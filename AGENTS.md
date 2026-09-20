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
   python3 /home/woravik/E2_Lab/cline-bot/current-model.py
   ```

   แล้วขึ้นต้นคำตอบด้วยผลลัพธ์นั้นเป็น **บรรทัดแรก** (ก่อนบอกแผน ก่อนเรียกเครื่องมือ)

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
bash /home/woravik/E2_Lab/cline-bot/set-model.sh --provider opencode-go --model deepseek-v4.1-flash --wait 45
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

- `bash ~/E2_Lab/cline-dashboard/model-status-notify.sh --check` → ส่งรายงานเข้า Telegram เอง
- `python3 ~/E2_Lab/cline-dashboard/model_health.py --status` → สรุปว่าโมเดลไหนใช้ได้

## 🔒 ความปลอดภัย

- **ห้ามเปลี่ยนไปใช้ provider ที่เสียเงินเองโดยไม่ได้รับอนุญาตจากผู้ใช้**
  (ถ้าผู้ใช้พิมพ์สั่งเองชัดเจน = ได้รับอนุญาตแล้ว ให้ทำทันที)
- Token/API key **ห้ามส่งกลับในแชท**
- สคริปต์คืน error → รายงานตามจริง **ห้ามเดาว่าน่าจะสำเร็จ**
