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

   > ⚠️ **แก้ 2026-09-24**: เดิมสคริปต์อ่านแค่ `lastUsedProvider` ใน providers.json
   > = "รุ่นที่ถูกสั่งสลับล่าสุด" ไม่ใช่รุ่นที่ connector รันจริง → รายงานผิด
   > (ขึ้น gemma ทั้งที่บอทใช้ deepseek) **ตอนนี้อ่านจาก session ที่กำลังรันจริงก่อน**
   > ถ้าไม่ตรงกับค่าที่ตั้งไว้จะต่อท้าย `· (ตั้งไว้ถัดไป: ...)` ให้เห็นทั้งสองค่า
   > ⇒ **คัดลอกบรรทัดผลลัพธ์ทั้งบรรทัดตามจริง ห้ามตัดทอน**

2. **เมื่อทำงานเสร็จ** แนบบรรทัดเดิมเป็น **บรรทัดสุดท้าย** ของคำตอบอีกครั้ง

ตัวอย่าง

```
🤖 กำลังรัน: deepseek-v4.1-flash · OpenCode Go (subscription $10/เดือน) (opencode-go)

... เนื้อหางาน ...
🤖 กำลังรัน: deepseek-v4.1-flash · OpenCode Go (subscription $10/เดือน) (opencode-go)
```

- อ่านค่าจริงจาก `~/.cline/data/settings/providers.json` — **ห้ามเดา/ห้ามแต่งชื่อ model เอง**
- ถ้าสคริปต์ error ให้เขียน `🤖 สถานะ model: อ่านไม่ได้` แล้วรายงาน error ตามจริง

---

## 🔀 กฎการสลับ model

**ห้ามแก้ `~/.cline/data/settings/providers.json` ตรงๆ** (ทำให้ connector พัง)
ให้ผ่าน dashboard API เท่านั้น:

```bash
bash /home/woravik/E2_Lab/cline-ctl/scripts/set-model.sh --provider opencode-go --model deepseek-v4.1-flash --wait 45
```

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
