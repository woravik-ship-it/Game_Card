# AGENTS.md — กฎควบคุม Cline Telegram (session รันในโฟลเดอร์นี้)

โฟลเดอร์นี้คือ `--cwd` ของ `cline-telegram.service`
(ดู `ExecStart` ใน `~/.config/systemd/user/cline-telegram.service`)
ดังนั้นทุก session ที่มาจาก Telegram จะมี cwd = `/home/woravik/E2_Lab/Game_Card`
และอ่านกฎจากไฟล์นี้

> กฎฉบับเต็มของตัวบอทอยู่ใน `~/E2_Lab/cline-bot/AGENTS.md`
> ไฟล์นี้คือสำเนาเฉพาะ **กฎที่ต้องมีผลกับ session Telegram** (ผู้ใช้สั่ง 2026-09-20)

---

## 🤖 กฎการตอบทุกข้อความ: แจ้ง Model ทั้ง "ตอนเริ่ม" และ "ตอนจบ"

**ทุกคำตอบ ไม่มีข้อยกเว้น**

1. **ก่อนเริ่มทำงาน** รัน:

   ```bash
   python3 /home/woravik/E2_Lab/cline-ctl/current-model.py
   ```

   แล้วขึ้นต้นคำตอบด้วยผลลัพธ์นั้นเป็น **บรรทัดแรก** (ก่อนบอกแผน ก่อนเรียกเครื่องมือ)

   > ⚠️ **แก้ 2026-09-25**: footer เป็นแบบสั้น `By <model>` (ผู้ใช้ขอ)
   > — สองฝั่ง (extension/CLI) ใช้รุ่นเดียวกัน = `By <model>` บรรทัดเดียว
   > — ถ้าเพิ่งเปลี่ยนโมเดลแล้ว session เดิมยังรันด้วยรุ่นเก่า =
   > `By <รุ่นใหม่> · CLI: <รุ่นเก่า>` (การเปลี่ยนมีผลกับ session ใหม่เท่านั้น)
   > ⇒ **คัดลอกบรรทัดผลลัพธ์ตามจริง ห้ามตัดทอน/แต่งเพิ่ม**

2. **เมื่อทำงานเสร็จ** แนบบรรทัดเดิมเป็น **บรรทัดสุดท้าย** ของคำตอบอีกครั้ง

ตัวอย่าง

```
By cline-free/mimo-v2.6-flash

... เนื้อหางาน ...
By cline-free/mimo-v2.6-flash
```

- อ่านค่าจริงจาก `~/.cline/data/settings/providers.json` — **ห้ามเดา/ห้ามแต่งชื่อ model เอง**
- ถ้าสคริปต์ error ให้เขียน `🤖 สถานะ model: อ่านไม่ได้` แล้วรายงาน error ตามจริง

---

## 🔀 กฎการสลับ model

**การสลับรุ่นโดยคน/agent → ผ่าน dashboard API เท่านั้น** (มี .bak + ตรวจค่าให้):

```bash
bash /home/woravik/E2_Lab/cline-ctl/scripts/set-model.sh --provider opencode-go --model deepseek-v4.1-flash --wait 45
```

> ⚠️ **แก้กฎ 2026-09-24** (เดิมเขียนว่า "ห้ามแก้ `providers.json` ตรงๆ เด็ดขาด"):
> ห้าม **แก้ด้วยมือ/hand-edit** ไฟล์นี้ แต่ **watchdog ซ่อมอัตโนมัติได้** — เขียนแบบ atomic
> (temp + `os.replace` + สำรอง `providers.json.repair-bak.<ts>` + อ่านกลับตรวจว่าตรง)
>
> **เหตุจริงที่ทำให้ต้องแก้กฎ:** ตัว CLI refresh OAuth token ของ provider `cline` ทุกชั่วโมง
> แล้วเขียน `lastUsedProvider = cline` ทับรุ่นที่ผู้ใช้เลือก (โค้ดในไบนารี:
> `saveProviderSettings` → `lastUsedProvider: setLastUsed!==!1 ? X : $.lastUsedProvider`)
> → ทุกชั่วโมงรุ่นจะเด้งกลับเป็น `cline/google/gemma-4-31b-it:free` (รุ่นฟรีที่โควต้าหมดแล้ว
> = chat ไม่ตอบ) · ซ่อมผ่าน panel API อย่างเดียวไม่พอ เพราะ panel ล่ม/rate limit 429/
> restart connector ทับงานที่กำลังรัน
> ⇒ ตัวซ่อมคือ `watch/check.py` (timer 2 นาที + `cline-ctl-drift.path` เฝ้าไฟล์ → ซ่อมใน ~2 วิ)
> และ **การซ่อมไม่ต้อง restart connector** (connector อ่าน providers.json ตอนเริ่ม session ใหม่)

- `--wait N` = หน่วง N วินาทีก่อนยิงคำขอ (จำเป็นเมื่อสั่งจากในแชท เพราะ POST สำเร็จ
  แล้ว dashboard จะ restart connector — ต้องให้คำตอบถึงผู้ใช้ก่อน)
- `--show` = ดู provider/model ปัจจุบัน
- ค่า model ของ BYOK provider **ไม่ต้องใส่ prefix**: `deepseek-v4.1-flash`
  (ไม่ใช่ `opencode-go/deepseek-v4.1-flash`)
- การเปลี่ยนมีผลกับ **session ถัดไป** เท่านั้น — ถ้าสลับกลางบทสนทนา ต้องบอกผู้ใช้ตรงๆ
- รหัส dashboard อ่านจาก `~/.config/cline-dashboard.env` — **ห้าม hardcode รหัสในสคริปต์/แชท**

---

## 📋 ตรวจสถานะ model / แจ้งผล

- `bash ~/E2_Lab/cline-ctl/scripts/model-status-notify.sh --check` → ส่งรายงานเข้า Telegram เอง
- `python3 ~/E2_Lab/cline-ctl/watch/model_health.py --status` → สรุปว่าโมเดลไหนใช้ได้

## 🔒 ความปลอดภัย

- **ห้ามเปลี่ยนไปใช้ provider ที่เสียเงินเองโดยไม่ได้รับอนุญาตจากผู้ใช้**
  (ถ้าผู้ใช้พิมพ์สั่งเองชัดเจน = ได้รับอนุญาตแล้ว ให้ทำทันที)
- Token/API key **ห้ามส่งกลับในแชท**
- สคริปต์คืน error → รายงานตามจริง **ห้ามเดาว่าน่าจะสำเร็จ**

---

## 🔧 ความเสถียรของบอท (เพิ่ม 2026-09-22 หลังเหตุ "บอทเงียบ")

**สาเหตุที่เคยเกิดจริง (พบใน log):**
1. cline CLI อัปเดตตัวเองกลางคัน (`npm update -g cline` → 3.0.62→3.0.63) ทำให้ binary ถูกสลับระหว่างรัน
   → hub ตายด้วย `ETXTBSY` / `Unable to locate executable .../cline` → connector หยุด → บอทเงียบ
2. `cline-telegram.service` เป็น `Type=oneshot` + `RemainAfterExit=yes` → systemd มอง unit ว่า active
   แม้ process ลูก (`connect telegram … -i`) ตายแล้ว จึงไม่มีอะไร restart ให้ ต้องสั่งมือ

**แก้ถาวรแล้ว:**
1. **ปิด auto-update ของ CLI** — `~/.cline/data/settings/global-settings.json` → `"autoUpdateEnabled": false`
   อัปเดตเองเมื่อต้องการ: `npm i -g cline@latest && systemctl --user restart cline-hub.service`
2. **Watchdog ทุก 2 นาที** — `cline-ctl-check.timer` → `~/E2_Lab/cline-ctl/watch/check.py`
   - hub ไม่ตอบ → restart `cline-hub` · hub ดีแต่ connector หาย → restart `cline-telegram`
   - เพดาน 6 ครั้ง/ชม. + ส่ง Telegram แจ้งเองเมื่อซ่อม
   - เช็คมือ: `bash ~/E2_Lab/cline-ctl/watch/check.py --status` · log: `~/E2_Lab/cline-ctl/watch/check.log`
3. **ช่องทางสำรองส่งข้อความ** — `python3 ~/E2_Lab/cline-ctl/tg/tg-send.py --text "..."` (หรือ `--file`)
   = Bot API ตรง, plain text (ไม่ตั้ง parse_mode), ตัดท่อนละ 3,800 ตัวอักษร, retry 3 ครั้ง

**กฎการตอบในแชท (กัน `Telegram reply failed: Bad Request`):**
- ตอบสั้น ≤ ~2,500 ตัวอักษร · เลี่ยงตาราง markdown ใหญ่ / โค้ดบล็อกยาว / `*` `_` กระจัดกระจาย
- ถ้าต้องรายงานยาว: เขียนไฟล์ในเครื่อง → ตอบสรุปสั้นๆ + บอกพาธ (หรือส่งไฟล์ผ่าน `tg-send.py --file`)
- ผู้ใช้บอก "ไม่เห็นข้อความ / บอทเงียบ" → ดู `watchdog.log` แล้วทดสอบส่งด้วย `tg-send.py`

---

## 📘 งานคู่มือ/หนังสือ (ผู้ใช้สั่ง 2026-09-26)

**ห้ามทำ/อัปเดต/ส่งคู่มือ (หนังสือ `docs/manual/`, PDF) เองโดยไม่ได้รับคำสั่ง** — เปลือง token
(สร้างหนังสือ 1 รอบ = ถ่ายภาพทั้งชุด + พิมพ์ PDF + ส่งไฟล์ ≈ หลายแสน token)

- ✅ ทำเมื่อผู้ใช้สั่งชัดเจนเท่านั้น เช่น "ทำคู่มือ", "อัปเดตคู่มือ", "ส่งคู่มือใหม่"
- ❌ ห้ามทำต่อ **โดยอัตโนมัติ** หลังแก้ฟีเจอร์/UI (เช่น เพิ่มเสียง แก้ปุ่ม) — ให้รอคำสั่งก่อน
- ถ้าเห็นว่าหนังสือจะไม่ตรงกับของจริง ให้ **บอกสั้น ๆ ว่าต้องอัปเดตตรงไหน** แล้วรอผู้ใช้ตัดสินใจ
- เอกสารที่อัปเดตได้เลยโดยไม่ต้องรอ (ไม่นับเป็นงานคู่มือ): `DEVELOPMENT_PLAN.md` · `README.md`
  · คอมเมนต์ในโค้ด · เทสต์

