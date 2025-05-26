const {
    default: makeWASocket,
    useMultiFileAuthState,
    DisconnectReason,
    fetchLatestBaileysVersion,
    makeInMemoryStore, // Opsional jika Anda ingin menggunakan store
    jidNormalizedUser,
    proto
} = require('@whiskeysockets/baileys');
const pino = require('pino');
const qrcode = require('qrcode-terminal'); // Untuk login bot utama
const fs = require('fs');
const path = require('path');
const settings = require('./settings'); // Pastikan settings.js ada dan benar

// Path ke file database
const dbPath = path.join(__dirname, 'database');
const userDbPath = path.join(dbPath, 'database.json');
const premiumDbPath = path.join(dbPath, 'premium.json');
const ownerDbPath = path.join(dbPath, 'owner.json');

// Inisialisasi database
let userDB = {};
let premiumDB = [];
let ownerDB = [];

function loadDatabases() {
    try {
        if (fs.existsSync(userDbPath)) userDB = JSON.parse(fs.readFileSync(userDbPath, 'utf-8'));
        else fs.writeFileSync(userDbPath, JSON.stringify({ users: {} }, null, 2), userDB = { users: {} });

        if (fs.existsSync(premiumDbPath)) premiumDB = JSON.parse(fs.readFileSync(premiumDbPath, 'utf-8'));
        else fs.writeFileSync(premiumDbPath, JSON.stringify([], null, 2), premiumDB = []);

        if (fs.existsSync(ownerDbPath)) ownerDB = JSON.parse(fs.readFileSync(ownerDbPath, 'utf-8'));
        else fs.writeFileSync(ownerDbPath, JSON.stringify([], null, 2), ownerDB = []);
        console.log("Database loaded successfully.");
    } catch (error) {
        console.error("Failed to load databases:", error);
        userDB = { users: {} }; premiumDB = []; ownerDB = [];
    }
}

function saveDatabase(type) {
    try {
        if (type === 'user' || type === 'all') fs.writeFileSync(userDbPath, JSON.stringify(userDB, null, 2));
        if (type === 'premium' || type === 'all') fs.writeFileSync(premiumDbPath, JSON.stringify(premiumDB, null, 2));
        if (type === 'owner' || type === 'all') fs.writeFileSync(ownerDbPath, JSON.stringify(ownerDB, null, 2));
    } catch (error) {
        console.error("Failed to save database:", error);
    }
}

// Pindahkan deklarasi ini ke scope global modul
const commands = new Map();
let sockInstance = null;
let qrHasBeenDisplayedThisAttempt = false;

// --- FUNGSI getPanelServerSpecs (Biarkan seperti sebelumnya atau implementasi live Anda) ---
async function getPanelServerSpecs() {
    // Ganti dengan implementasi live Anda jika sudah ada, atau biarkan placeholder
    const { exec } = require('child_process');
    function executeCommand(command) {
        return new Promise((resolve) => {
            exec(command, (error, stdout, stderr) => {
                resolve(stdout.trim() || "");
            });
        });
    }
    console.log("[INFO] Mengambil spesifikasi server...");
    const specs = { /* ... (isi dengan data dummy atau hasil exec) ... */ };
    // Contoh mengambil hostname
    specs.hostname = await executeCommand("hostname");
    specs.cpu = await executeCommand("lscpu | grep 'Model name:' | sed 's/Model name:[ \t]*//'");
    // ... tambahkan perintah lain untuk spek ...
    return specs;
}
// -----------------------------------------------------------------------------

// --- MODIFIKASI FUNGSI loadPlugins ---
function loadPlugins(currentSock, clearPrevious = false) { // Terima 'currentSock'
    if (clearPrevious) {
        commands.clear();
        console.log("[PLUGIN LOADER] Cache perintah lama dibersihkan.");
    }

    const pluginsDir = path.join(__dirname, 'plugins');
    if (!fs.existsSync(pluginsDir)) {
        console.warn(`[PLUGIN LOADER] Folder plugins tidak ditemukan di ${pluginsDir}. Tidak ada plugin yang dimuat.`);
        return;
    }

    fs.readdirSync(pluginsDir).forEach(file => {
        if (file.endsWith('.js')) {
            try {
                const pluginPath = path.join(pluginsDir, file);
                delete require.cache[require.resolve(pluginPath)]; // Hapus cache modul
                const plugin = require(pluginPath);

                if (plugin.command && typeof plugin.execute === 'function') {
                    const cmdName = Array.isArray(plugin.command) ? plugin.command[0] : plugin.command;
                    
                    if (clearPrevious || !commands.has(cmdName.toLowerCase())) {
                        commands.set(cmdName.toLowerCase(), plugin);
                        if (Array.isArray(plugin.command)) {
                            plugin.command.forEach(alias => {
                                if (clearPrevious || !commands.has(alias.toLowerCase())) {
                                    commands.set(alias.toLowerCase(), plugin);
                                }
                            });
                        }
                        console.log(`[PLUGIN LOADER] Loaded plugin: ${cmdName} dari ${file}`);
                    } else if (!clearPrevious && commands.has(cmdName.toLowerCase())) {
                        console.log(`[PLUGIN LOADER] Plugin ${cmdName} dari ${file} sudah ada, dilewati (mode tidak membersihkan).`);
                    }
                } else {
                    console.warn(`[PLUGIN LOADER] Plugin ${file} tidak memiliki struktur command/execute yang valid.`);
                }
            } catch (error) {
                console.error(`[PLUGIN LOADER] Gagal memuat plugin ${file}:`, error);
            }
        }
    });
    console.log(`[PLUGIN LOADER] Total ${commands.size} perintah berhasil dimuat/diperbarui.`);
}
// --- AKHIR MODIFIKASI loadPlugins ---


async function connectToWhatsApp() {
    qrHasBeenDisplayedThisAttempt = false;
    const { state, saveCreds } = await useMultiFileAuthState(settings.sessionPath);
    const { version, isLatest } = await fetchLatestBaileysVersion();
    console.log(`Menggunakan Baileys versi: ${version.join('.')}, Terbaru: ${isLatest}`);

    const sock = makeWASocket({
        version,
        logger: pino({ level: 'silent' }),
        printQRInTerminal: false,
        auth: state,
        browser: [settings.botName, "Chrome", "1.0.0"],
        getMessage: async key => proto.Message.fromObject({})
    });
    sockInstance = sock; // Simpan instance sock global

    // Pemanggilan awal loadPlugins dipindahkan ke event 'open'

    sock.ev.on('connection.update', async (update) => {
        const { connection, lastDisconnect, qr } = update;

        if (qr && !qrHasBeenDisplayedThisAttempt) {
            console.log("------------------------------------------------");
            try {
                const specs = await getPanelServerSpecs();
                if (specs) {
                    console.log("✨ Spesifikasi Server ✨");
                    console.log(`   Hostname: ${specs.hostname || "N/A"}`);
                    console.log(`   CPU: ${specs.cpu || "N/A"}`);
                    // ... (tampilkan spek lain) ...
                    console.log("------------------------------------------------");
                }
            } catch (specError) {
                console.warn("[PERINGATAN] Gagal mendapatkan spesifikasi server:", specError.message);
                console.log("------------------------------------------------");
            }

            console.log("Scan QR Code ini dengan WhatsApp Anda:");
            qrcode.generate(qr, { small: true });
            console.log("------------------------------------------------");
            qrHasBeenDisplayedThisAttempt = true;
        }

        if (connection === 'close') {
            const statusCode = (lastDisconnect.error)?.output?.statusCode;
            const shouldReconnect = statusCode !== DisconnectReason.loggedOut;
            console.log('Koneksi ditutup karena:', lastDisconnect.error, ', mencoba menghubungkan ulang:', shouldReconnect);

            if (statusCode === DisconnectReason.restartRequired) {
                console.log("Restart diperlukan, menghubungkan ulang...");
                connectToWhatsApp();
            } else if (shouldReconnect) {
                connectToWhatsApp();
            } else {
                console.log("Tidak bisa menghubungkan ulang, Anda telah logout. Hapus folder session dan scan ulang.");
            }
        } else if (connection === 'open') {
            console.log(`${settings.botName} terhubung!`);
            console.log(`Nomor Owner: ${settings.ownerNumbers.join(', ')}`);
            qrHasBeenDisplayedThisAttempt = false;
            loadPlugins(sockInstance, true); // <--- PANGGIL loadPlugins DI SINI
                                            // Gunakan sockInstance yang global
                                            // true untuk membersihkan command map sebelumnya.
        }
    });

    sock.ev.on('creds.update', saveCreds);

    sock.ev.on('messages.upsert', async (m) => {
        const msg = m.messages[0];
        if (!msg.message || (msg.key && msg.key.remoteJid === 'status@broadcast')) return;
        await handleMessage(sockInstance, msg); // Gunakan sockInstance global
    });
}

async function handleMessage(currentSock, msg) { // Terima 'currentSock'
    const messageType = Object.keys(msg.message)[0];
    let body = "";

    if (messageType === 'conversation') body = msg.message.conversation;
    else if (messageType === 'extendedTextMessage') body = msg.message.extendedTextMessage.text;
    else if (msg.message.imageMessage?.caption) body = msg.message.imageMessage.caption;
    else if (msg.message.videoMessage?.caption) body = msg.message.videoMessage.caption;

    const senderJid = jidNormalizedUser(msg.key.remoteJid);
    const pushName = msg.pushName || "Pengguna";
    const isGroup = senderJid.endsWith('@g.us');
    const actualSender = isGroup ? jidNormalizedUser(msg.key.participant) : senderJid;

    if (!body || !body.startsWith(settings.prefix)) return;

    const args = body.slice(settings.prefix.length).trim().split(/ +/);
    const commandName = args.shift().toLowerCase();
    const command = commands.get(commandName);

    if (command) {
        const normalizedOwnerDB = (Array.isArray(ownerDB) ? ownerDB : []).map(owner => jidNormalizedUser(owner));
        const allOwners = [...settings.ownerNumbers.map(owner => jidNormalizedUser(owner)), ...normalizedOwnerDB];
        
        if (command.ownerOnly && !allOwners.includes(actualSender)) {
            return currentSock.sendMessage(senderJid, { text: "Maaf, perintah ini hanya untuk Owner." }, { quoted: msg });
        }
        const normalizedPremiumDB = (Array.isArray(premiumDB) ? premiumDB : []).map(prem => jidNormalizedUser(prem));
        if (command.premiumOnly && !normalizedPremiumDB.includes(actualSender)) {
            return currentSock.sendMessage(senderJid, { text: "Maaf, perintah ini hanya untuk pengguna Premium." }, { quoted: msg });
        }
        try {
            console.log(`[COMMAND] ${commandName} oleh ${pushName} (${actualSender}) di ${senderJid}`);
            // --- MODIFIKASI OBJEK UTILS ---
            await command.execute(currentSock, msg, args, {
                settings,
                userDB,
                premiumDB,
                ownerDB,
                saveDatabase,
                // fs dan path diimpor lokal oleh plugin jika butuh, atau bisa di-pass jika mau:
                // fs: require('fs'),
                // path: require('path'),
                jidNormalizedUser, // Baileys sudah menyediakan ini, tapi bisa di-pass juga
                isGroup,
                // Fungsi untuk me-reload plugin:
                reloadPlugins: (clearCacheAndCommands = true) => loadPlugins(currentSock, clearCacheAndCommands),
            });
            // --- AKHIR MODIFIKASI OBJEK UTILS ---
        } catch (error) {
            console.error(`Error menjalankan command ${commandName}:`, error);
            currentSock.sendMessage(senderJid, { text: 'Terjadi kesalahan saat menjalankan perintah.' }, { quoted: msg });
        }
    }
}

// Panggil loadDatabases di awal sekali
loadDatabases();
// Jalankan bot
connectToWhatsApp().catch(err => console.error("Gagal memulai koneksi WhatsApp:", err));

// --- Handler Sinyal dan Error (Biarkan seperti sebelumnya) ---
process.on('SIGINT', async () => {
    console.log("Menutup bot...");
    if (sockInstance) await sockInstance.logout();
    console.log("Bot telah ditutup.");
    process.exit(0);
});
process.on('unhandledRejection', (reason, promise) => console.error('Unhandled Rejection:', promise, 'alasan:', reason));
process.on('uncaughtException', (error) => console.error('Uncaught Exception:', error));

let file = require.resolve(__filename)
fs.watchFile(file, () => {
fs.unwatchFile(file)
console.log(chalk.redBright(`Update ${__filename}`))
delete require.cache[file]
require(file)
})
