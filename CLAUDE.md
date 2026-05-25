# CLAUDE.md — Mana Back Office Project Context

## Project Overview
ระบบ Back Office ของร้าน **Mana Mana** ร้านอาหารญี่ปุ่นแนว Homey Izakaya ประกอบด้วย 2 ส่วนหลัก:
1. **Stock Form** — ฟอร์มกรอกสต็อกวัตถุดิบประจำวัน (GitHub Pages)
2. **ระบบใบเสร็จ** — รับรูปใบเสร็จจาก LINE กลุ่ม แล้ว OCR ด้วย Claude AI

---

## 1. Stock Form (GitHub Pages)

**URL:** https://nuchiko.github.io/Stock/  
**GitHub Repo:** Nuchiko/Stock (branch: main, file: index.html)

### Items ในฟอร์ม
```
Salmon, Maguro, Madai, ShimaAji, Kanpachi, Aori Ika, Hokkigai, Ikura, Hotate
```

### Flow
1. User กรอกข้อมูลสต็อกใน form
2. html2canvas ถ่ายภาพหน้าจอ (onclone fix: overflow visible, width 860px)
3. ส่ง Base64 image + note ไปที่ GAS via fetch POST
4. GAS บันทึกลง Google Sheets + upload รูปไป Google Drive + ส่ง LINE

### GAS (Google Apps Script)
- **Project ID:** `1gDa2TnrBMmWdIrmh_4AFoeQjRMJ0a7cidGRop5FQ9eUsYogarkn1W07c`
- **Exec URL:** `https://script.google.com/macros/s/AKfycby0tNfxuwJVdq7j9WMXvdBbSXWsu4md7tyBHSkceykHg1xbenltdwTE265lO3jui7A6Xg/exec`
- **Current Version:** 11 (deployed May 24, 2026)

### GAS Key Variables
```javascript
var LINE_TOKEN = '<LINE_CHANNEL_ACCESS_TOKEN>';
var GROUP_ID = 'Cfe2e9e8d74453db7cd25c7dc3dc884b9';
```

### GAS Flow (doPost)
1. รับ `imageBase64` + `note` + stock data
2. บันทึกข้อมูลลง Google Sheets
3. Upload รูปไป Google Drive (folder วันที่)
4. Set sharing: ANYONE_WITH_LINK VIEW
5. `imageUrl = 'https://drive.google.com/thumbnail?id=' + fileId + '&sz=w1200'`
6. `sendLineImage(LINE_TOKEN, GROUP_ID, imageUrl)`
7. ถ้ามี note: `sendLineText(LINE_TOKEN, GROUP_ID, '📝 หมายเหตุ: ' + note)`

### html2canvas Fix (ใน index.html)
```javascript
html2canvas(document.getElementById('captureArea'), {
  scale: 1.5,
  useCORS: true,
  backgroundColor: '#ffffff',
  logging: false,
  onclone: function(doc) {
    var el = doc.getElementById('captureArea');
    el.style.overflow = 'visible';
    el.style.width = '860px';
    el.style.maxWidth = 'none';
    var tw = el.querySelector('.table-wrap');
    if (tw) { tw.style.overflow = 'visible'; }
  }
})
```

---

## 2. ระบบใบเสร็จ (n8n + LINE + Claude OCR)

**n8n URL:** https://nuchiko.app.n8n.cloud  
**Workflow:** "My workflow 2" (ID: `vhH93lTGcvz7pMZx`)

### Workflow Flow
```
LINE Webhook Trigger
  → Respond to Line (200 OK)   [parallel branch]
  → Is Image?
      true → Download Image from Line
           → Convert to Base64  [Code node]
           → Claude — Read Receipt (OCR)  [POST https://api.anthropic.com]
           → Parse Claude response
           → Add M... (ส่งกลับ LINE)
      false → (จบ)
```

### n8n Credentials
| ชื่อ | ประเภท | ค่า |
|------|--------|-----|
| Header Auth account | Header Auth | Name: `Authorization`, Value: `Bearer <LINE_CHANNEL_ACCESS_TOKEN>` |

### n8n Variables
| Key | Value |
|-----|-------|
| `LINE_CHANNEL_ACCESS_TOKEN` | `<LINE_CHANNEL_ACCESS_TOKEN>` |

### LINE Webhook URL
```
https://nuchiko.app.n8n.cloud/webhook/line-webhook
```

### Convert to Base64 (Code Node Logic)
```javascript
// 1. ดึง binary data จาก Download Image node
const binaryKey = Object.keys($input.item.binary)[0];
const binaryData = $input.item.binary[binaryKey];
const mimeType = binaryData.mimeType || 'image/jpeg';

let buffer;
try {
  buffer = await this.helpers.getBinaryDataBuffer(binaryData);
} catch(e1) {
  // fallback: re-fetch จาก LINE API ใช้ $vars.LINE_CHANNEL_ACCESS_TOKEN
  const wh = $("Line Webhook Trigger").item.json;
  const events = wh.events || (wh.body && wh.body.events) || [];
  const messageId = String(events[0].message.id);
  buffer = await this.helpers.request({
    method: 'GET',
    url: `https://api-data.line.me/v2/bot/message/${messageId}/content`,
    headers: { Authorization: `Bearer ${$vars.LINE_CHANNEL_ACCESS_TOKEN}` },
    encoding: null
  });
}
const base64 = Buffer.isBuffer(buffer)
  ? buffer.toString('base64')
  : Buffer.from(buffer).toString('base64');

// 2. ส่งไป Claude API (anthropic)
const anthropicBody = {
  model: 'claude-sonnet-4-6',
  max_tokens: 1024,
  messages: [{
    role: 'user',
    content: [
      { type: 'image', source: { type: 'base64', media_type: mimeType, data: base64 } },
      { type: 'text', text: 'You are an AI expert at reading Thai receipts...' }
    ]
  }]
};
```

---

## 3. LINE API Info

**Channel:** Mana Back Office (Messaging API)  
**LINE Developers Console:** https://developers.line.biz/console/channel/2010168854/messaging-api

**Channel Access Token (long-lived):**
```
<LINE_CHANNEL_ACCESS_TOKEN>
```
> ⚠️ โทเคนจริงห้ามเก็บในไฟล์นี้ (repo เป็น public) — เก็บไว้ใน LINE Developers Console / n8n Variables / GAS Script Properties เท่านั้น
> ⚠️ หมายเหตุเดิม: ตัวอักษรตัวที่ 2 จากท้ายก่อน `/w` เป็น **O ตัวใหญ่** ไม่ใช่ o ตัวเล็ก

**LINE Group ID:** `Cfe2e9e8d74453db7cd25c7dc3dc884b9`  
**Bot Basic ID:** `@381pgtpx`

---

## 4. ปัญหาที่แก้ไปแล้ว (Session History)

| ปัญหา | สาเหตุ | วิธีแก้ |
|-------|--------|---------|
| Screenshot ตัดตารางขวา | `overflow: hidden` บน container | เพิ่ม `onclone` callback ใน html2canvas |
| Input fields มีเลข 0 | `value="0"` ใน HTML | เปลี่ยนเป็น `value=""` |
| n8n Authorization failed | Token มี `o` ตัวเล็ก แทน `O` ตัวใหญ่ | อัปเดต Header Auth credential |
| n8n content is gone | Webhook Trigger ใช้ Pinned data เก่า | Unpin data ใน Line Webhook Trigger node |
| n8n 401 ใน Convert to Base64 | `LINE_CHANNEL_ACCESS_TOKEN` variable มีโทเค็นเก่า | อัปเดต Variable ใน n8n |

---

## 5. สถานะการทดสอบ

- [x] ส่งรูปใบเสร็จใน LINE กลุ่ม → bot ตอบกลับ OCR ถูกต้อง — **ผ่าน (25 พ.ค. 2026)** อ่านร้าน "HORIGOME GARDEN" ยอด 200 บาท ถูกต้อง
- [x] ตรวจสอบ "Parse Claude response" และ "Add M..." nodes — **ผ่าน** จัดรูปแบบข้อความตอบกลับ (emoji + ร้าน/ยอด/วันที่) ครบถ้วน

### ปรับปรุงแล้ว
- [x] **แยกข้อความตอบกลับ ใบเสร็จ vs รูปสินค้า** — **เสร็จ (25 พ.ค. 2026)** แก้ที่ node `Reply to Line Group` (Body JSON expression) เพิ่มเงื่อนไข `isReceipt = shop_name && shop_name !== '-' && total_amount && Number(total_amount) > 0`
  - ใบเสร็จ → "✅ บันทึกใบเสร็จสำเร็จ! 🧾 ร้าน:... 💰 ยอดรวม:... 📅 วันที่:..." (เหมือนเดิม)
  - รูปสินค้าทั่วไป → "✅ บันทึกรูปสินค้าเรียบร้อย 📅 วันที่:... 📁 บันทึกแล้วใน Google Drive"
  - การบันทึก Drive/Sheets **ไม่เปลี่ยน** (ยังบันทึกทุกรูป)
  - publish เป็น version ใหม่แล้ว (ย้อนกลับได้ผ่าน n8n Version History)

- [x] **เพิ่มชนิดรูป "เบียร์ล้างสาย"** — **เสร็จ (25 พ.ค. 2026)** แยกรูป 3 ชนิดผ่าน field `image_type` (`receipt`/`beer_cleaning`/`other`)
  - **prompt (node `Convert to Base64`)**: Claude คืน `image_type` (ลองให้อ่าน `beer_volume_ml` จากระดับในเหยือกด้วย แต่**อ่านไม่แม่น เลิกใช้**)
  - **รูปเบียร์**: บันทึกลง Drive + ตอบ "📸 รับรูปเบียร์ล้างสายแล้ว พิมพ์ปริมาณตามมา เช่น เบียร์ 250" (node `Reply to Line Group` สาขา beer_cleaning)
  - การบันทึก Sheet ฝั่งรูป: ทุกรูปลง tab `receipts` (Sheet Name คงที่ — รูปเบียร์ไม่เขียน beer_cleaning เพื่อกันแถว null)

- [x] **ปริมาณเบียร์มาจากข้อความ LINE** — **เสร็จ (25 พ.ค. 2026)** พนักงานพิมพ์ "เบียร์ <ตัวเลข>" (เช่น "เบียร์ 170") เป็นข้อความแยก
  - **สาขาใหม่**: `Line Webhook Trigger → Beer Text Filter (Code) → Append Beer Volume (Sheets) → Reply Beer Volume (HTTP)`
    - `Beer Text Filter`: Code (Run Once for All Items) กรองข้อความที่มี "เบียร์" + ตัวเลข ดึงเลข → `{beer_volume_ml, received_date, received_time, sender_id, group_id, reply_token}` ถ้าไม่เข้าเงื่อน return [] (หยุด)
    - `Append Beer Volume`: append ลง tab **`beer_cleaning`** (autoMap) หัวคอลัมน์: `received_date`, `received_time`, `beer_volume_ml`, `sender_id`
    - `Reply Beer Volume`: ตอบ "✅ บันทึกปริมาณเบียร์ล้างสาย X ml"
  - รูปกับตัวเลขบันทึกแยกกัน โยงด้วยวันที่ (ไม่ผูกแถวเดียวกัน)
  - **แก้บั๊กสำคัญ**: เดิม node `Is Image?` ต่อทั้ง true+false → Download ทำให้ข้อความ (ไม่ใช่รูป) วิ่งเข้า Download → "Bad request" → ทั้ง execution หยุด สาขาข้อความเลยไม่ตอบ → **ลบสาย false→Download ออก** เหลือเฉพาะ true (รูป) → Download
  - spreadsheet ID = `1to3ya3q47M2GKbyANcjsTzw8WvP9IQryKtxF-iiueMc`

- [x] **เปลี่ยน push API → reply API** — **เสร็จ (25 พ.ค. 2026)** บอตหยุดตอบเพราะ **โควต้า push ของ LINE เต็ม 300/300/เดือน** (LINE นับ push แบบต่อผู้รับ × จำนวนคนในกลุ่ม จึงหมดเร็ว) → error "too many requests"
  - แก้ที่ node `Reply to Line Group` + `Reply Beer Volume`: เปลี่ยน URL `/message/push` → `/message/reply` และ body `{ to: group_id }` → `{ replyToken: reply_token }`
  - **reply API ฟรี ไม่จำกัด ไม่นับโควต้า 300** (เหมาะกับบอทตอบข้อความ) — replyToken มาจาก webhook event (Parse node เก็บ `reply_token`, Beer Text Filter เก็บ `reply_token`)
  - ข้อจำกัด: replyToken ใช้ได้ครั้งเดียว + ต้องตอบภายใน ~1 นาที (OCR ~15-25 วิ ทัน)
  - เช็กโควต้า: `curl https://api.line.me/v2/bot/message/quota` และ `.../quota/consumption` (Bearer LINE token)

- [x] **แก้เวลาเป็นเวลาไทย (UTC+7)** — **เสร็จ (25 พ.ค. 2026)** เดิม `received_date`/`received_time` เป็น UTC (n8n server รัน UTC) ช้ากว่าจริง 7 ชม. → แก้ใน node `Parse Claude JSON + Add Metadata` และ `Beer Text Filter`: เปลี่ยน `new Date()` → `new Date(Date.now() + 7*60*60*1000)` และ time ใช้ `now.toISOString().split('T')[1].slice(0,8)` (เลี่ยงพึ่ง timezone ของ server)

---

## 6. Files & Links สำคัญ

| รายการ | URL/Path |
|--------|----------|
| Stock Form | https://nuchiko.github.io/Stock/ |
| GAS Editor | https://script.google.com/home/projects/1gDa2TnrBMmWdIrmh_4AFoeQjRMJ0a7cidGRop5FQ9eUsYogarkn1W07c |
| n8n Workflow | https://nuchiko.app.n8n.cloud/workflow/vhH93lTGcvz7pMZx |
| LINE Developers | https://developers.line.biz/console/channel/2010168854/messaging-api |
