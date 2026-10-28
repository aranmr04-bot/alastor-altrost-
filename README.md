
import makeWASocket, { useMultiFileAuthState, DisconnectReason, downloadMediaMessage } from '@whiskeysockets/baileys'
import pino from 'pino'
import axios from 'axios'
import yts from 'yt-search'
import figlet from 'figlet'

console.log(figlet.textSync('ALASTOR BOT V3', { font: 'Small' }))

// ===== قائمة الكلمات الممنوعة - تقدر تضيف عليها =====
const BAD_WORDS = [
    'كس', 'طيز', 'قحبة', 'نيك', 'متناك', 'شرموطة', 
    'fuck', 'shit', 'bitch', 'كس امك', 'يا حمار', 'يا كلب'
]

// ===== دالة فحص السب =====
function containsBadWords(text) {
    if (!text) return false
    let lower = text.toLowerCase()
    return BAD_WORDS.some(word => lower.includes(word))
}

async function startBot() {
    const { state, saveCreds } = await useMultiFileAuthState('auth')
    const sock = makeWASocket({
        auth: state,
        logger: pino({ level: 'silent' }),
        printQRInTerminal: true,
        browser: ["Alastor Ultra V3", "Chrome", "3.0"]
    })

    sock.ev.on('creds.update', saveCreds)

    sock.ev.on('connection.update', async (update) => {
        const { connection, lastDisconnect } = update
        if (connection === 'close') {
            if (lastDisconnect?.error?.output?.statusCode !== DisconnectReason.loggedOut) startBot()
        } else if (connection === 'open') {
            console.log('✅ Alastor V3 Online - بكل المميزات!')
        }
    })

    // ===== 1- نظام الترحيب التلقائي =====
    sock.ev.on('group-participants.update', async (update) => {
        try {
            const { id, participants, action } = update
            for (let p of participants) {
                if (action === 'add') {
                    let welcomeText = `
*أهلا وسهلا نورت القروب!* 🎉

👋 مرحبا @${p.split('@')[0]}

*🤖 أنا بوت ألاستور V3*
✦ استخدم .menu لمعرفة أوامري
✦ الجداول، التحميل، الملصقات
✦ ممنوع السب - الحذف تلقائي ⚠️

*نورتنا!* ✨
                    `
                    await sock.sendMessage(id, { text: welcomeText, mentions: [p] })
                }
                if (action === 'remove') {
                    await sock.sendMessage(id, { text: `👋 غادر @${p.split('@')[0]} القروب`, mentions: [p] })
                }
            }
        } catch (e) { console.log(e) }
    })

    // ===== 2- نظام الرسائل والأوامر =====
    sock.ev.on('messages.upsert', async ({ messages }) => {
        const m = messages[0]
        if (!m.message || m.key.fromMe) return
        const from = m.key.remoteJid
        const isGroup = from.endsWith('@g.us')
        const body = m.message.conversation || m.message.extendedTextMessage?.text || m.message.imageMessage?.caption || ""
        const args = body.trim().split(/ +/)
        const command = args[0].toLowerCase()
        const q = args.slice(1).join(" ")
        const reply = (text) => sock.sendMessage(from, { text })

        // ===== 3- نظام الحذف التلقائي للسب =====
        if (isGroup && containsBadWords(body)) {
            try {
                // حذف الرسالة
                await sock.sendMessage(from, { delete: m.key })
                // تحذير
                await sock.sendMessage(from, { 
                    text: `⚠️ @${m.key.participant.split('@')[0]} ممنوع السب والكلام السيء! تم حذف رسالتك.`,
                    mentions: [m.key.participant]
                })
                console.log(`تم حذف رسالة سب من ${m.key.participant}`)
                return // لا تكمل باقي الأوامر
            } catch (err) {
                console.log('البوت ليس مشرف - لا يستطيع الحذف')
                await reply('⚠️ كلام سيء! الرجاء احترام القروب - فعل خاصية المشرف للبوت ليحذف تلقائياً')
            }
        }

        // ===== الأوامر =====
        if (command == '.menu' || command == '.help' || command == 'القائمة') {
            await reply(`
*🤖 ALASTOR ULTRA V3 - القائمة*

*📊 الجداول:*
.جدول الاسم/العمر/البلد - يسوي جدول
مثال: .جدول احمد/25/مصر - علي/30/اليمن

*📥 التحميل:*
.تيك رابط - تيك توك بدون علامة
.يوتيوب اسم الفيديو - بحث
.انستا رابط - ريلز انستا

*📱 التطبيقات:*
.تطبيق اسم التطبيق - رابط بلاي ستور

*🛠️ أدوات:*
.ملصق - حول صورة لملصق (ارسل صورة مع الأمر)
.صورة - حول الملصق لصورة
.ترجمة نص

*🛡️ الحماية:*
الحذف التلقائي للسب فعال ✅
ترحيب تلقائي فعال ✅
            `)
        }

        if (command == '.جدول') {
            if (!q) return reply('استخدم: .جدول احمد/25/مصر - محمد/30/السعودية')
            let rows = q.split('-')
            let table = '┌─────────┬─────────┬─────────┐\n│ الاسم │ العمر │ البلد │\n├─────────┼─────────┼─────────┤\n'
            rows.forEach(r => {
                let cols = r.split('/')
                table += `│ ${cols[0]||''} │ ${cols[1]||''} │ ${cols[2]||''} │\n`
            })
            table += '└─────────┴─────────┴─────────┘'
            await reply(table)
        }

        if (command == '.تيك') {
            if (!q) return reply('حط رابط التيك توك')
            await reply('⏳ جاري التحميل...')
            try {
                let { data } = await axios.get(`https://api.tiklydown.eu.org/api/download?url=${q}`)
                await sock.sendMessage(from, { video: { url: data.video.noWatermark }, caption: '✅ بدون علامة - Alastor Bot' })
            } catch { reply('❌ فشل، تأكد من الرابط') }
        }

        if (command == '.يوتيوب' || command == '.yt') {
            if (!q) return reply('اكتب اسم الفيديو')
            let search = await yts(q)
            let video = search.videos[0]
            await sock.sendMessage(from, { image: { url: video.thumbnail }, caption: `*${video.title}*\n${video.url}` })
        }

        if (command == '.تطبيق' || command == '.apk') {
            if (!q) return reply('اكتب اسم التطبيق')
            let url = `https://play.google.com/store/search?q=${encodeURIComponent(q)}&c=apps`
            await reply(`*📱 ${q}*\nرابط التحميل الرسمي:\n${url}`)
        }

        if (command == '.ملصق') {
            let media = m.message.imageMessage ? await downloadMediaMessage(m, 'buffer', {}) : null
            if (!media) return reply('ارسل صورة مع كلمة .ملصق')
            await sock.sendMessage(from, { sticker: media })
        }

        if (command == '.ترجمة') {
            if (!q) return reply('اكتب: .ترجمة hello')
            try {
                let { data } = await axios.get(`https://api.mymemory.translated.net/get?q=${encodeURIComponent(q)}&langpair=en|ar`)
                await reply(`الترجمة: ${data.responseData.translatedText}`)
            } catch { reply('فشل الترجمة') }
        }

        if (command == '.ping') {
            await reply('Pong! 🏓 البوت شغال 100%')
        }
    })
}

startBot()node_modules
auth
*.log
