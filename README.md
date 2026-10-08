# Game_Card — Rune Dominion Arena (โครงการเกมการ์ด)

repo นี้เป็น "โฟลเดอร์แม่" ของโครงการ เก็บเอกสารออกแบบ/แผนพัฒนา/กฎการทำงาน
ส่วน **โค้ดเกมทั้งหมดอยู่ใน submodule** [`rune-dominion-arena`](rune-dominion-arena)
(คนละ repo: <https://github.com/woravik-ship-it/rune-dominion-arena>)

## สถานะปัจจุบัน (2026-10-07)

| หัวข้อ | สถานะ |
|---|---|
| แผนพัฒนา Phase 0–12 | ✅ ครบทุกข้อ (209/209 · ดู [`DEVELOPMENT_PLAN.md`](DEVELOPMENT_PLAN.md)) |
| งานต่อเนื่อง Phase 13–45 | ✅ ทำแล้วและ **commit ครบ** (Phase 43–45 สรุปท้ายแผน: ไอเทมแยกชิ้น/ตีบวก · แผนที่เก็บของ · เครื่องประดับอวตาร · 45.4 ความยากดันไล่ทุกชั้น + Map ยึดโซนที่อยู่ · 45.5 น้ำหนักบอสที่วัดใหม่ + e2e ลบบัญชีทดสอบเอง · 45.6 กลไกบอสคู่พยุงกัน + คู่มือผู้เล่น 117 หน้า) |
| ตรวจแผน vs โค้ด (`npm run audit`) | ✅ **68/73 มีหลักฐานจริง + 5 ข้อเลือกใช้ทางอื่นโดยเจตนา** (73/73 ครบ) |
| เทสต์ | ✅ Jest **867 passed / 60 suites** · `tsc --noEmit` 0 error |
| ฐานข้อมูลจริง | ✅ **7 บัญชี** (woravik · TCM · Abcd · KJ_SAM · player1 · secadmin · t5678) · ไม่มีบัญชีทดสอบค้าง |
| ตรวจด้วยเบราว์เซอร์จริง (สคริปต์ของโปรเจกต์) | ✅ `verify:map-zone` 10/10 · `verify:dungeon-curve` 8/8 · `verify:collection` 14/14 · `test:e2e` 12/12 |
| Production build | ✅ ผ่าน + รันจริง (`rune-dominion-arena.service`) |
| E2E critical flow (HTTP) | ✅ 30/30 ผ่าน · E2E เบราว์เซอร์ (Playwright) ดู `tests/e2e/` |
| รันจริงบนเครื่องนี้ | ✅ `systemd --user`: `rune-dominion-postgres` · `rune-dominion-arena` (พอร์ต 3000) · `rune-dominion-images.timer` · `rune-dominion-backup.timer` (สำรอง DB รายวัน 04:30) |
| Public URL | ✅ **<https://rune.e2sv.link>** (Cloudflare named tunnel `e2sv` — คงที่ ไม่เปลี่ยนทุกครั้งที่รีสตาร์ท) |
| สำรองฐานข้อมูล | ✅ `~/backups/rune-dominion/` · ตรวจกู้คืนได้จริงด้วย `npm run backup:verify` |
| โค้ดขึ้น GitHub | ✅ `rune-dominion-arena` (โค้ดเกม) + repo แม่ `Game_Card` (เอกสาร) — ทั้งคู่ push แล้ว |

## โคลนโปรเจกต์

```bash
git clone --recurse-submodules git@github.com:woravik-ship-it/rune-dominion-arena.git
# หรือโคลนเฉพาะโค้ดเกม (ไม่ต้องใช้ repo แม่ก็ได้)
```

> ถ้าโคลน repo นี้ (โฟลเดอร์แม่) ต้องมี repo ปลายทางบน GitHub ก่อน แล้วใช้ `--recurse-submodules`

## เอกสารในโฟลเดอร์นี้

| ไฟล์ | เนื้อหา |
|---|---|
| [`DEVELOPMENT_PLAN.md`](DEVELOPMENT_PLAN.md) | แผนพัฒนา Phase 0–12 + หลักฐาน/ผลทดสอบจริง + บั๊กที่แก้ |
| [`Markdown.md`](Markdown.md) | GDD (Game Design Document) ฉบับเต็ม |
| [`AGENTS.md`](AGENTS.md) | กฎของ session Cline ที่รันในโฟลเดอร์นี้ (Telegram) |
| [`rune-dominion-arena/README.md`](rune-dominion-arena/README.md) | วิธีรัน/ทดสอบ/ดีพลอยตัวเกม |
| [`rune-dominion-arena/docs/`](rune-dominion-arena/docs) | API · คู่มือผู้เล่น · Deployment · Security · Marketing |

## คำสั่งที่ใช้บ่อย (ใน submodule)

```bash
cd rune-dominion-arena
npm test                    # unit + integration tests
npm run e2e:flow            # E2E critical flow (ยิง HTTP จริง)
npm run build && npm start  # production build + รัน
npm run backup              # สำรองฐานข้อมูล
```
