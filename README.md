# Game_Card — Rune Dominion Arena (โครงการเกมการ์ด)

repo นี้เป็น "โฟลเดอร์แม่" ของโครงการ เก็บเอกสารออกแบบ/แผนพัฒนา/กฎการทำงาน
ส่วน **โค้ดเกมทั้งหมดอยู่ใน submodule** [`rune-dominion-arena`](rune-dominion-arena)
(คนละ repo: <https://github.com/woravik-ship-it/rune-dominion-arena>)

## สถานะปัจจุบัน (2026-09-21)

| หัวข้อ | สถานะ |
|---|---|
| แผนพัฒนา Phase 0–12 | ✅ ครบทุกข้อ (209/209 · ดู [`DEVELOPMENT_PLAN.md`](DEVELOPMENT_PLAN.md)) |
| เทสต์ | ✅ Jest 232 passed / 20 suites · `tsc --noEmit` ผ่าน |
| Production build | ✅ ผ่าน (26 หน้า · shared JS 87.3 kB) |
| E2E critical flow | ✅ 25/25 ผ่าน (localhost **และ** ผ่าน tunnel สาธารณะ) |
| รันจริงบนเครื่องนี้ | ✅ `systemd --user` 3 unit: `rune-dominion-postgres` · `rune-dominion-arena` (พอร์ต 3000) · `rune-dominion-tunnel` |
| Public URL | `~/.rune-dominion-tunnel/url.txt` (Cloudflare quick tunnel — เปลี่ยนทุกครั้งที่รีสตาร์ท) |
| โค้ดขึ้น GitHub | ✅ `rune-dominion-arena` push แล้ว · ⚠️ repo แม่นี้ยังไม่มี remote |

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
