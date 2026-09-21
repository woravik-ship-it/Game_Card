# Rune Dominion Arena — Development Plan

> เอกสารแผนการพัฒนาแบบ Phase ตาม GDD
> เป้าหมาย: ส่งมอบ MVP ที่เล่นได้จริงทุก Phase
> ทุก Phase ต้องมี PR + Test + Demo ผ่านก่อนขึ้น Phase ถัดไป

---

## 🎯 กฎระหว่างพัฒนา (ทุก Phase)

- ✅ **TypeScript strict mode** ทุกไฟล์
- ✅ **ใช้ Integer เท่านั้น** สำหรับ Currency ห้าม Float
- ✅ **Server-side เท่านั้น** สำหรับการคำนวณสำคัญ (Combat, Discovery, Reward)
- ✅ **Deterministic** ห้าม `Math.random()` ในระบบสำคัญ
- ✅ **Idempotent** ทุก Endpoint ที่มีผลกระทบ Currency
- ✅ **Mobile-first UI** ทุกหน้าจอ
- ✅ **i18n-ready** ภาษาไทยทุกข้อความ
- ✅ **Unit Test** สำหรับ Core Logic (Seed, Combat, Reward)
- ✅ **E2E Test** สำหรับ Critical Flow (Discovery → Battle → Arena)

---

## Phase 0 — Project Setup & Architecture

**เป้าหมาย:** โครงสร้างโปรเจกต์ครบ รันได้เลยแบบ Mock Data

### งาน Backend
- [x] ตั้งค่า Next.js App Router + TypeScript strict
- [x] ติดตั้งและตั้งค่า Prisma + PostgreSQL
- [ ] ตั้งค่า Redis + BullMQ
- [x] Auth: Credentials + JWT (custom — scrypt hash + HS256 session cookie, แทน Auth.js)
- [x] สร้าง Docker Compose (Postgres + Redis + App)
- [x] สร้าง `.env.example` ที่จำเป็น
- [x] สร้างโครงสร้างโฟลเดอร์มาตรฐาน
- [ ] ตั้งค่า Middleware: Auth Guard, Rate Limiter, Logger (✅ Logger + Security headers — เหลือ Auth Guard/Rate Limiter รอ auth flow นำไปใช้)

### งาน Frontend
- [x] ตั้งค่า Tailwind CSS (❌ shadcn/ui ยังไม่ได้ติดตั้ง)
- [ ] ตั้งค่า Zustand + TanStack Query
- [ ] ตั้งค่า React Hook Form + Zod
- [x] สร้าง Design Tokens (CSS Variables)
- [x] สร้าง Layout: AppShell, TopHeader, BottomNavigation
- [x] สร้างหน้า Home (Mock Data ทั้งหมด)

### งานอื่น
- [x] เขียน README วิธีรันโปรเจกต์
- [x] สร้าง Mock Data เบื้องต้น (100 การ์ดผ่าน seed — ผู้ใช้สร้างผ่าน /register)
- [x] ตั้งค่า GitHub Actions (Type check + Lint + Test)

### Definition of Done
- `docker compose up` รันได้สมบูรณ์
- เปิดหน้า Home เห็น Mock Data
- Auth ล็อกอินได้ (Mock)

---

## Phase 1 — Rune Discovery & Seed System

**เป้าหมาย:** ผู้เล่นเลือก Rune → ได้การ์ดแบบ Deterministic

### งาน Backend
- [ ] Prisma Schema: CardDefinition, UserCard, DiscoveryLog
- [ ] API: `POST /api/discover` (รับ runeSequence → คืนการ์ด)
- [ ] Service: `buildCanonicalString(runes)`
- [ ] Service: `hashSeed(canonicalString + SERVER_PEPPER)`
- [ ] Service: `createCardFromSeed(hash)` — Deterministic PRNG
- [ ] Service: `findCardByHash(hash)` — ค้นหาการ์ดเดิม
- [x] Service: Discovery Energy System (เติมวันละ 5 — ✅ lazy daily refill ผ่าน lastEnergyResetAt, ไม่ต้องพึ่ง cron)
- [ ] Transaction: ป้องกัน Race Condition ด้วย Unique Constraint
- [ ] Idempotency Key ต่อ Discovery Request
- [ ] Unit Test: Canonical String, Hash, Deterministic Card
- [ ] Seed Script: สร้าง 100 การ์ดตัวอย่าง

### งาน Frontend
- [ ] หน้า `/discover` — Rune Canvas 100×100 (Canvas API)
- [ ] Interaction: คลิกเลือก Rune 8–16 จุด (เรียงลำดับ)
- [ ] แสดง Rune Sequence ที่เลือก
- [ ] ปุ่ม "ถอดรหัสรูน" + Confirmation Modal
- [ ] Loading State: "กำลังอ่านบันทึกแห่งรูน..."
- [ ] หน้า Card Reveal Modal (แสดงการ์ดที่ได้)
- [ ] Badge: "ผู้ค้นพบคนแรก" vs "การ์ดที่ถูกค้นพบแล้ว"

### งานอื่น
- [x] Placeholder Art: Elemental Gradient ตามธาตุ+ความหายาก (✅ deterministic SVG ที่ /api/cards/[id]/image + backfill ครบทุกการ์ดแล้ว)
- [ ] Animation: เปิดการ์ด (รองรับ Reduced Motion)

### Definition of Done
- เลือก Rune 10 จุด → ได้การ์ด
- เลือก Rune เดียวกันซ้ำ → ได้การ์ดใบเดียวกัน
- Unit Test ผ่าน 100%
- หน้า Discover ใช้งานได้บนมือถือ

---

## Phase 2 — Card Collection & Detail

**เป้าหมาย:** ผู้เล่นดูการ์ดที่สะสมได้

### งาน Backend
- [ ] Prisma Schema: UserCard (collection), CardFavorite
- [ ] API: `GET /api/cards` (รายการการ์ดของเรา — Pagination, Filter)
- [ ] API: `GET /api/cards/:id` (รายละเอียดการ์ด)
- [ ] API: `POST /api/cards/:id/favorite`
- [ ] Filter ตาม: Element, Rarity, Role, Owned/Not Owned
- [ ] Sort ตาม: ใหม่สุด, หายาก, ทีม
- [ ] API: `GET /api/cards/:id/owners` (คนอื่นที่มีการ์ดนี้)

### งาน Frontend
- [ ] หน้า `/cards` — Card Collection Grid
- [ ] Filter Element (ปุ่ม 6 ธาตุ)
- [ ] Filter Rarity (ปุ่ม 6 ระดับ)
- [ ] Search Bar (ค้นหาชื่อการ์ด)
- [ ] Card Thumbnail + Rarity Glow Effect
- [ ] หน้า `/cards/[id]` — Card Detail
- [ ] แสดง: ชื่อ, ธาตุ, Role, Rarity, Lore, สกิล
- [ ] แสดง: ATK, DEF, HP, SPD, ManaCost
- [ ] Placeholder หรือ AI Image (ถ้ามี)
- [ ] ปุ่ม "เพิ่มลงทีม" / "ลบออกจากทีม"

### Definition of Done
- ดูการ์ดที่สะสมได้
- Filter ธาตุ+ความหายากได้
- ดูรายละเอียดการ์ดได้

---

## Phase 3 — Deck Builder

**เป้าหมาย:** ผู้เล่นจัดทีม 5 ใบได้

### งาน Backend
- [ ] Prisma Schema: Deck, DeckSlot
- [ ] API: `POST /api/decks` (สร้างเด็คใหม่)
- [ ] API: `GET /api/decks` (รายการเด็คของฉัน)
- [ ] API: `PUT /api/decks/:id` (อัปเดตเด็ค)
- [ ] API: `DELETE /api/decks/:id` (ลบเด็ค)
- [ ] Validation: 5 ใบ, ห้ามซ้ำ, ธาตุเดิม ≤ 3 ใบ
- [ ] คำนวณ Team Power (รวมสเตตัสทั้งหมด)

### งาน Frontend
- [ ] หน้า `/decks` — รายการเด็คทั้งหมด
- [ ] หน้า `/decks/:id` — Deck Builder
- [ ] Formation Board: Frontline 2, Midline 2, Backline 1
- [ ] Drag & Drop (Desktop) / Tap-to-Place (Mobile)
- [ ] Card Pool: การ์ดที่มี + ลากเข้า Formation
- [ ] Team Power Meter (แสดงค่าพลัง)
- [ ] แสดงการฝื่อน Rule (เช่น "ธาตุ Fire เกิน 3 ใบ")
- [ ] ปุ่ม "บันทึกเด็ค"
- [ ] Active Deck Selector (เลือกเด็คที่ใช้งาน)

### Definition of Done
- สร้างเด็ค 5 ใบได้
- Validation ทำงานถูกต้อง

---

## Phase 4 — Auto Battle System

**เป้าหมาย:** ต่อสู้ Auto Battle แบบ Deterministic ได้

### งาน Backend
- [ ] Prisma Schema: BattleLog, BattleReplay, BattleUnitState
- [ ] Pure Function: `simulateBattle(teamA, teamB, seed)` → BattleResult
- [ ] Combat Engine:
  - Turn-based, สูงสุด 30 รอบ
  - Mana เริ่ม 0, +20/turn, สูงสุด 100
  - ATK, DEF, HP, SPD, ManaCost
  - Status Effect: BURN, SHIELD, HASTE, WEAKEN, HEAL
  - Element Advantage (1.15x) / Disadvantage (0.90x)
  - Deterministic Variance (0.95–1.05)
- [ ] Damage Formula: `base × mitigation × element × variance`
- [ ] API: `POST /api/battle/simulate` (สำหรับทดสอบเด็ค)
- [ ] API: `GET /api/battle/:id/log` (ดู Battle Log)
- [ ] API: `GET /api/battle/:id/replay` (ดู Replay)
- [ ] Battle Seed: `SHA-256(battleId + teamA + teamB + combatVersion + serverSecret)`
- [ ] Unit Test: Combat Engine ครบทุก Status Effect
- [ ] Unit Test: Element Multiplier
- [ ] Unit Test: Deterministic (เด็คเดียวกัน = ผลเดียวกัน)

### งาน Frontend
- [ ] หน้า `/battle/[id]` — Battle Viewer
- [ ] BattleField: แสดงทีม 2 ฝ่าย (การ์ด + HP Bar + Status)
- [ ] Turn Counter
- [ ] Battle Log Panel (แสดงการโจมตีแบบ Turn-by-Turn)
- [ ] Animation: โจมตี, สกิล, ตาย (รองรับ Reduced Motion)
- [ ] Speed Control (1x, 2x, 4x)
- [ ] ปุ่ม Replay / หยุด / ข้าม
- [ ] แสดงผล: ชนะ/แพ้/เสมอ

### Definition of Done
- สู้กับ Bot ได้
- Battle Log บันทึกครบ
- Replay ดูย้อนหลังได้
- Deterministic: เด็คเดียวกัน = ผลเดียวกันทุกครั้ง

---

## Phase 5 — Coin Ledger & Wallet

**เป้าหมาย:** ระบบสกุลเงินปลอดภัย

### งาน Backend
- [ ] Prisma Schema: Wallet, CoinTransaction
- [ ] API: `GET /api/wallet` (ยอด Coin ปัจจุบัน)
- [ ] API: `GET /api/wallet/transactions` (ประวัติ)
- [ ] Service: `credit(userId, amount, type, referenceId)`
- [ ] Service: `debit(userId, amount, type, referenceId)`
- [ ] Validation: Balance ไม่ติดลบ
- [ ] Transaction: ACID + Idempotency Key
- [ ] Unique Constraint: ป้องกันซ้ำจาก Idempotency Key
- [ ] Daily Cap: รางวัลสูงสุดต่อวัน
- [ ] Unit Test: ทุกการเคลื่อนไหว Coin

### งาน Frontend
- [ ] หน้า `/wallet` — Wallet Dashboard
- [ ] แสดงยอด Coin ปัจจุบัน (ตัวใหญ่ชัดเจน)
- [ ] Transaction List: รายการรับ/จ่ายทั้งหมด
- [ ] Filter: ทั้งหมด / รับ / จ่าย
- [ ] แสดง Balance Before/After ในแต่ละรายการ
- [ ] Disclaimer: "Coin เป็นสกุลเงินภายในเกม ห้ามแลกเป็นเงินจริง"
- [ ] ใน TopHeader: แสดงยอด Coin + Discovery Energy

### Definition of Done
- Coin เพิ่ม/ลด ถูกต้อง
- ไม่มี Double Spend
- Transaction Log บันทึกครบ
- Idempotency ทำงาน (Retry ไม่หักซ้ำ)

---

## Phase 6 — Arena Room 24 ชั่วโมง

**เป้าหมาย:** ผู้เล่นเปิดห้องและเข้าร่วมแข่งขันได้

### งาน Backend
- [ ] Prisma Schema: ArenaRoom, ArenaParticipant, ArenaChallenge
- [ ] API: `GET /api/arena` (ห้องทั้งหมด — Active, Upcoming, Expired)
- [ ] API: `POST /api/arena/create` (เปิดห้อง — 30 Coin)
- [ ] API: `POST /api/arena/:id/join` (เข้าร่วม — 10 Coin)
- [ ] API: `GET /api/arena/:id` (รายละเอียดห้อง + Leaderboard)
- [ ] API: `GET /api/arena/:id/leaderboard`
- [ ] Service: Room Lifecycle (Create → Active → Expired → Settling)
- [ ] Service: Settlement Job (ทำเมื่อห้องหมดอายุ)
- [ ] Reward Formula: `min(100 + participants × 5, 500)`
- [ ] Cooldown: 5 นาทีระหว่างเปิดห้อง
- [ ] Daily Cap: เข้าร่วมสูงสุด 20 ครั้ง
- [ ] Idempotency สำหรับการเข้าร่วม
- [ ] Scheduler: ตรวจสอบห้องหมดอายุทุกนาที

### งาน Frontend
- [ ] หน้า `/arena` — Arena Lobby
- [ ] แสดงห้องที่ Active อยู่ (RoomCard)
- [ ] แสดง: Champion ปัจจุบัน, จำนวนผู้เข้าร่วม, เวลาที่เหลือ, รางวัล
- [ ] ปุ่ม "เปิดห้องใหม่" (30 Coin)
- [ ] หน้า `/arena/:id` — Arena Room Detail
- [ ] Champion Display: การ์ด 5 ใบของ Champion
- [ ] Leaderboard: 10 อันดับแรก
- [ ] ปุ่ม "ท้าทาย" (10 Coin) + Confirmation Modal
- [ ] Countdown Timer: "ห้องน้าจะสิ้นสุดใน"
- [ ] แสดงผลการต่อสู้ทันทีหลังท้าทาย
- [ ] หน้า `/arena/create` — เลือกทีมป้องกัน + เปิดห้อง

### Definition of Done
- เปิดห้องได้ (30 Coin)
- เข้าร่วมได้ (10 Coin)
- ห้องหมดอายุ 24 ชม. → จ่ายรางวัล
- Leaderboard อัปเดตแบบ Real-time

---

## Phase 7 — Quest & Mission System

**เป้าหมาย:** ภารกิจรายวัน/สัปดาห์ให้ Coin

### งาน Backend
- [x] Prisma Schema: Quest, QuestProgress (+ periodKey ต่องวด, code, metric)
- [x] API: `GET /api/quests` (รายการเควสทั้งหมด)
- [x] API: `POST /api/quests/:id/claim` (รับรางวัล)
- [x] Quest Type: Daily Discovery, Daily Battle, Weekly Win, etc.
- [x] Auto-reset Daily Quest เที่ยงคืน (Server Time) — ด้วย periodKey ต่องวด
- [x] Auto-reset Weekly Quest วันจันทร์ — ISO week key
- [x] Idempotency สำหรับการ Claim (updateMany lock + wallet idempotencyKey)

### งาน Frontend
- [x] หน้า `/quests` — Quest Dashboard
- [x] Tab: รายวัน / รายสัปดาห์ / ถาวร
- [x] แสดง: ความคืบหน้า + ปุ่ม Claim
- [x] Progress Bar แต่ละเควส

### Definition of Done
- รับเควสทำได้
- Claim รางวัลได้
- รีเซ็ตรายวันทำงาน

---

## Phase 8 — AI Image Generation

**เป้าหมาย:** การ์ดใหม่มีภาพ AI

### งาน Backend
- [x] Prisma Schema: ImageJob (มีใน schema อยู่แล้ว — ใช้ตารางเดิม)
- [x] API: `POST /api/admin/images/requeue`
- [x] Queue: DB-backed queue จากตาราง ImageJob (แทน BullMQ เพราะยังไม่ติดตั้ง Redis)
- [x] Worker: `/api/admin/images/process` ดึงงาน → AI API หรือ Placeholder → บันทึก URL
- [x] Retry: Exponential Backoff (30s→60s→120s, cap 10 นาที) 3 ครั้ง แล้ว FAILED
- [x] Placeholder แทนภาพขณะรอ — deterministic SVG ตามธาตุ/ระดับความหายาก (route `/api/cards/[id]/image`)
- [x] Content Moderation (blocklist ตรวจ prompt ก่อนเรียก AI)
- [ ] Webhook: แจ้งเมื่อภาพพร้อม

### งาน Frontend
- [x] Card Thumbnail แสดง Placeholder (ผ่าน image route + onError fallback)
- [x] Card Detail แสดงภาพเมื่อ READY
- [x] Badge "กำลังสร้างภาพ..." ถ้า PENDING

### Definition of Done
- การ์ดใหม่สร้างภาพอัตโนมัติ
- Placeholder แสดงระหว่างรอ
- Retry ล้มเหลว → สถานะ FAILED, Admin Requeue ได้

---

## Phase 9 — Admin Tools

**เป้าหมาย:** Admin จัดการระบบได้

### งาน Backend
- [x] Prisma Schema: AdminUser, AdminActionLog (✅ ใช้ User.role (ADMIN/MODERATOR) แทน AdminUser แยก + ตาราง admin_action_logs)
- [x] Role-based Access: Admin แยกจาก User ทั่วไป (getAdminSession + /admin layout guard)
- [x] API: Admin CRUD สำหรับ Cards, Quests, Events (✅ Cards + Quests — Events รอทำพร้อม Phase 11)
- [x] API: `GET /api/admin/users` (ดูผู้เล่น — search + pagination + สถิติรายคน)
- [x] API: `GET /api/admin/analytics` (สถิติภาพรวม)
- [x] API: `POST /api/admin/images/requeue` (สร้างภาพใหม่ — มีตั้งแต่ Phase 8 + GET /api/admin/images)
- [x] Audit Log: บันทึกทุก Action ของ Admin (PATCH card/quest บันทึก before/after)

### งาน Frontend
- [x] หน้า `/admin` — Admin Dashboard (สถิติ 8 การ์ด + สถานะคิวภาพ)
- [x] หน้า `/admin/cards` — จัดการการ์ด (ค้นหา + แก้ชื่อไทย/lore)
- [x] หน้า `/admin/quests` — จัดการเควส (เปิด/ปิด + แก้เป้าหมาย/รางวัล)
- [x] หน้า `/admin/users` — ดูผู้เล่น
- [x] หน้า `/admin/images` — ดูสถานะภาพ + Requeue/Process

### Definition of Done
- ✅ Admin จัดการทุก Entity ได้ (Cards, Quests, Images — Events รอ Phase 11)
- ✅ Audit Log บันทึกครบ (ทดสอบ UPDATE_QUEST ผ่าน)
- ✅ ป้องกัน User ทั่วไปเข้า Admin (API 403 + layout redirect)

---

## Phase 10 — Security & Anti-Cheat

**เป้าหมาย:** ระบบปลอดภัยตาม Checklist

### งาน Backend
- [x] Rate Limiting: ต่อ User/IP/Device (✅ sliding-window in-memory (`src/lib/rate-limit.ts`) + middleware จำกัด burst 120 req/นาที/IP — identity เรียงลำดับ User → Device → IP; ปรับผ่าน env `RATE_LIMIT_<SCOPE>_LIMIT`; scale หลายอินสแตนซ์ค่อยสลับ store เป็น Redis)
- [x] Bot Pattern Detection (Action เร็วผิดปกติ) (✅ `src/lib/anti-cheat.ts` — FAST_ACTIONS (≥6 ครั้ง ห่าง <250ms) + UNIFORM_CADENCE (jitter ≤20ms × 12 ครั้ง) → 429 + SecurityEvent BOT_PATTERN — ใช้กับ /api/discover)
- [x] Alt-account Fingerprinting (✅ header `x-device-id` + IP → เก็บ signup/last ในตาราง users; สมัคร/ล็อกอินซ้ำ device เดียวกัน → SecurityEvent ALT_ACCOUNT_SUSPECT — ไม่บล็อกอัตโนมัติ กัน false positive บน LAN NAT)
- [x] Input Validation ทุก API (Zod) (✅ `src/lib/validation.ts` — schemas ครอบ discover / auth / battle / arena create-join-challenge / quest claim / decks / favorite + `parseJsonBody` ตอบ 400 ระบุ path)
- [x] SQL Injection Prevention (Prisma จัดการให้) (✅ ตรวจแล้ว — ไม่มี `$queryRaw`/`$executeRaw` ใน src เลย)
- [x] XSS Protection (Next.js + React จัดการให้) (✅ ตรวจแล้ว — ไม่มี `dangerouslySetInnerHTML` + CSP รองรับ)
- [x] CORS Configuration (✅ `src/lib/cors.ts` — same-origin เป็นค่าเริ่มต้น, เพิ่มผ่าน `CORS_ALLOWED_ORIGINS`, preflight 204/403, guard ใน middleware)
- [x] Security Headers (Helmet) (✅ ใช้ middleware แทน Helmet เพราะ Next.js ไม่ใช่ Express — เพิ่ม CSP + HSTS (prod) + X-Frame-Options/nosniff/Referrer-Policy/Permissions-Policy)
- [x] Battle Replay Verification (ตรวจสอบความถูกต้อง) (✅ `src/services/battle-verify.ts` — สร้าง battle เก็บ `teams` snapshot + seed → replay re-simulate เทียบ winner/rounds/HP/log ทั้งก้อน; TAMPERED → SecurityEvent severity HIGH; battle เก่า → UNVERIFIABLE)

### งานอื่น
- [x] Security Audit Log (✅ โมเดล `SecurityEvent` (ตาราง security_events) + `src/lib/security-log.ts` (fail-safe) + `GET /api/admin/security` (guard ADMIN/MODERATOR, pagination + filter by type))
- [x] Penetration Test เบื้องต้น (✅ เทสอัตโนมัติ 5 ชุดใหม่: rate-limit / anti-cheat / battle-verify / cors / validation — jest ผ่าน 164/164 + typecheck ผ่าน; checklist manual อยู่ SECURITY.md §4)
- [x] Document Security Policy (✅ `SECURITY.md` — สถาปัตยกรรม 8 ชั้น, กลไกแต่ละอัน, accepted risks, วิธีรายงานช่องโหว่)

### Definition of Done
- ✅ ทุก Checklist ใน GDD ผ่าน (เหลือ Image Moderation เป็น N/A — ยังไม่มี user upload)
- ✅ ไม่มีช่องโหว่ที่ร้ายแรง (jest 168 ผ่าน, tsc ผ่าน)
- ✅ **ตาราง security_events + คอลัมน์ fingerprint push ลง DB แล้ว** (`prisma db push` สำเร็จ — 20 ตาราง)
- ✅ **ทดสอบ E2E ผ่านจริง:** ล็อกอินผิด 10 ครั้ง → 429 + AUTH_FAILURE_SPIKE ลง DB, ยิง discover รัวๆ → 429 + BOT_PATTERN (FAST_ACTIONS), CORS origin แปลกปลอม → 403, CSP/security headers ครบ, fingerprint (signup/last device_id) บันทึกถูกต้อง

### หมายเหตุสภาพแวดล้อม (Postgres)
- Docker daemon ต้องใช้สิทธิ์ root ที่เครื่องนี้ (user ไม่ได้อยู่กลุ่ม docker) — ใช้ **PostgreSQL 18.6 portable** แทน:
  - ติดตั้งอยู่ที่ `~/pg-portable/postgresql-18.6.0-x86_64-unknown-linux-gnu/`, data dir `~/pg-portable/data`
  - สตาร์ท: `export LD_LIBRARY_PATH=~/pg-portable/lib && ~/pg-portable/postgresql-18.6.0-x86_64-unknown-linux-gnu/bin/pg_ctl -D ~/pg-portable/data -l ~/pg-portable/pg.log -o "-p 5432" start`
  - ฐานข้อมูล `rune_dominion` / user `postgres` / password `postgres` — ตรงกับ `.env` เดิม (port 5432 เหมือน docker-compose)
  - หมายเหตุ: ใช้ auth=trust ตอน initdb (แต่ตั้งรหัสผ่านให้ postgres แล้ว) — เหมาะกับ dev เครื่องตัวเองเท่านั้น

---

## Phase 11 — Seasonal Event (Beta)

**เป้าหมาย:** Event "Call of the Moonless Gate" 14 วัน

### งาน Backend
- [x] Prisma Schema: Event, EventQuest, EventParticipation, EventReward (✅ `EventBoss`, `EventRaidAttempt`, `EventMilestone` (personal/community), `EventShopItem`, `EventShopPurchase`, `EventStoryChapter`, `EventCommunityProgress` + `EventParticipation` เพิ่ม `eventPoints`/`damageDealt` — push ลง DB แล้ว)
- [x] Event Lifecycle: UPCOMING → ACTIVE → GRACE_PERIOD → ENDED (✅ `EventService.syncStatuses()` คำนวณจากเวลา = lazy ไม่ต้องพึ่ง cron; grace 24 ชม.; POST /api/admin/events/sync สำหรับ scheduler)
- [x] Event Currency: Veil Shards (✅ เก็บใน `EventParticipation.currencyEarned/Spent` แบบ integer — แยกจาก Coin ของ Wallet)
- [x] Boss Raid System (4 Phase, Mechanics 8 แบบ) (✅ 4 Phase ตามช่วง HP 100–76/75–51/50–26/25–0% + mechanics ครบ 10 แบบ: Veil Shield, Rune Fracture, Moonless Mark, Eclipse Pulse, Rift Hunger, Ember Break, Gale Shift, Rooted Guard, Tide Cleanse, Dawn Resonance)
- [x] Personal Milestones (✅ 7 ระดับตาม GDD §13.6 — 2,000→75,000 points, claim idempotent)
- [x] Community Milestones (✅ 5 ระดับตาม GDD §13.7 — 1M→50M damage)
- [x] Event Shop (✅ 4 ไอเทม ซื้อด้วย Veil Shards + จำกัดต่อผู้ใช้ + idempotency key)
- [x] Story Chapter Unlock (✅ 4 บท ปลดล็อกตาม community damage)
- [x] Scheduler: เปิด/ปิด Event อัตโนมัติ (✅ `EventService.syncStatuses()` + admin endpoint)

### งาน Frontend
- [x] หน้า `/events/[eventId]` — Event Hub (✅ Banner + Countdown + Boss + Milestone + Quest + Shop + Story)
- [x] Event Banner + Countdown (✅ แสดงเวลาที่เหลือ + สถานะ)
- [x] Boss Raid UI (Boss HP, Phase, ทีมโจมตี) (✅ หลอด HP + ชื่อ Phase + ค่าเข้า/cap รายวัน/คะแนน/ดาเมจของตัวเอง)
- [x] Event Quest List (✅)
- [x] Milestone Tracker (✅ personal + community พร้อม progress bar และปุ่มรับรางวัล)
- [x] Event Shop (✅)
- [x] Story Chapter Reader (✅ แสดงบทที่ปลดล็อก/ยังล็อก)

### Definition of Done
- ✅ Event เปิด/ปิด อัตโนมัติ (lifecycle จากเวลา — ทดสอบแล้ว: seed แล้วได้ `ACTIVE` ทันที)
- ✅ Boss Raid เล่นได้ (ทดสอบจริง: ดาเมจ 3,395 / Event Points 3,734 (+10% element bonus) / mechanics 5 รายการ / หัก 10 Shards → ได้ 7 / boss 1,000,000 → 993,210)
- ✅ Milestones ได้รับรางวัล (Coin เข้า Wallet จริง 250 → 350, รับซ้ำถูกปฏิเสธ, เกณฑ์ไม่ถึงถูกปฏิเสธ)
- ✅ Community Goal อัปเดตแบบ Real-time (นับรวมทุก raid ใน `EventCommunityProgress` — ตรวจหลัง raid แล้วตัวเลขขยับจริง)
- ✅ เทสต์: jest 197/197 ผ่าน (เพิ่ม 23 เทสต์ event) + tsc ผ่าน

### หมายเหตุ (งานที่เหลือของ Phase 11)
- Event Quest progress ยังไม่ hook เข้า metric จริง (มีคำนิยามใน DB แล้ว — ต่อ hook เมื่อทำ Phase 12)
- Raid ปัจจุบันใช้สูตรดาเมจจากพลังทีม (`teamAtk × 12`) — ยังไม่ผูก combat engine 5v1 เต็มรูปแบบ
- Community milestone reward (VEIL_SHARDS/CRAFTING_DUST/COSMETIC) บันทึกเป็น `milestonesReached` แล้ว — ยังไม่มีระบบ cosmetic/crafting ปลายทาง

---

## Phase 12 — Polish & Launch Prep

**เป้าหมาย:** เกมพร้อม Launch

### งานทั่วไป
- [ ] Performance Optimization (Bundle Size, Query, Caching)
- [ ] Load Testing (100+ concurrent users)
- [ ] Monitoring (Sentry, Logging, Uptime)
- [ ] Database Index Optimization
- [ ] CDN สำหรับ Static Assets
- [ ] Backup Strategy
- [ ] Deployment Pipeline (Staging → Production)

### งาน Frontend
- [ ] Animations Polish (Framer Motion)
- [ ] Sound Effects (ตาม Audio Direction)
- [ ] Loading Skeletons ทุกหน้า
- [ ] Error Boundaries
- [ ] Empty States ทุกหน้า
- [ ] Onboarding Flow สำหรับผู้เล่นใหม่
- [ ] Tutorial / Tooltips

### งานอื่น
- [ ] เอกสาร API
- [ ] คู่มือผู้เล่น (ภาษาไทย)
- [ ] Marketing Assets (Screenshots, Videos)

---

## 📊 สรุป Timeline (โดยประมาณ)

| Phase | ระยะเวลาประมาณ | สถานะหลังเสร็จ |
|-------|----------------|-----------------|
| Phase 0 | 1 สัปดาห์ | โปรเจกต์รันได้ |
| Phase 1 | 2 สัปดาห์ | Rune Discovery ใช้ได้ |
| Phase 2 | 1 สัปดาห์ | ดู Collection ได้ |
| Phase 3 | 1 สัปดาห์ | จัดทีมได้ |
| Phase 4 | 2 สัปดาห์ | Auto Battle ได้ |
| Phase 5 | 1 สัปดาห์ | Coin ปลอดภัย |
| Phase 6 | 2 สัปดาห์ | Arena เล่นได้ |
| Phase 7 | 1 สัปดาห์ | Quest รับรางวัลได้ |
| Phase 8 | 1 สัปดาห์ | การ์ดมีภาพ AI |
| Phase 9 | 1 สัปดาห์ | Admin ใช้ได้ |
| Phase 10 | 1 สัปดาห์ | ปลอดภัย |
| Phase 11 | 2 สัปดาห์ | Event เล่นได้ |
| Phase 12 | 2 สัปดาห์ | พร้อม Launch |

**รวมประมาณ 17–20 สัปดาห์ (4–5 เดือน)** สำหรับทีม 1–2 คน

---

## 🏗️ โครงสร้างโฟลเดอร์ (สรุป)

```
rune-dominion-arena/
├── prisma/
│   ├── schema.prisma
│   └── seed.ts
├── src/
│   ├── app/
│   │   ├── page.tsx              # Home
│   │   ├── (auth)/
│   │   │   ├── login/page.tsx
│   │   │   └── register/page.tsx
│   │   ├── (game)/
│   │   │   ├── discover/page.tsx
│   │   │   ├── cards/page.tsx
│   │   │   ├── cards/[id]/page.tsx
│   │   │   ├── decks/page.tsx
│   │   │   ├── decks/[id]/page.tsx
│   │   │   ├── battle/[id]/page.tsx
│   │   │   ├── arena/page.tsx
│   │   │   ├── arena/create/page.tsx
│   │   │   ├── arena/[id]/page.tsx
│   │   │   ├── wallet/page.tsx
│   │   │   ├── quests/page.tsx
│   │   │   ├── profile/page.tsx
│   │   │   └── events/[id]/page.tsx
│   │   ├── api/
│   │   │   ├── auth/[...nextauth]/route.ts
│   │   │   ├── discover/route.ts
│   │   │   ├── cards/route.ts
│   │   │   ├── cards/[id]/route.ts
│   │   │   ├── decks/route.ts
│   │   │   ├── battle/route.ts
│   │   │   ├── arena/route.ts
│   │   │   ├── wallet/route.ts
│   │   │   ├── quests/route.ts
│   │   │   └── admin/
│   │   └── admin/
│   │       └── page.tsx
│   ├── components/
│   │   ├── ui/                   # shadcn/ui
│   │   ├── layout/
│   │   ├── cards/
│   │   ├── rune/
│   │   ├── battle/
│   │   └── arena/
│   ├── lib/
│   │   ├── prisma.ts
│   │   ├── auth.ts
│   │   ├── redis.ts
│   │   └── queue.ts
│   ├── services/
│   │   ├── seed.ts               # Seed Service
│   │   ├── card-discovery.ts
│   │   ├── combat-engine.ts
│   │   ├── arena.ts
│   │   └── wallet.ts
│   ├── hooks/
│   ├── stores/
│   ├── types/
│   └── utils/
├── tests/
│   ├── unit/
│   │   ├── seed.test.ts
│   │   ├── combat.test.ts
│   │   └── reward.test.ts
│   └── e2e/
│       ├── discover.spec.ts
│       └── battle.spec.ts
├── public/
│   ├── images/
│   └── sounds/
├── docker-compose.yml
├── .env.example
├── tailwind.config.ts
├── tsconfig.json
├── package.json
└── README.md
```

---

## 📌 หมายเหตุสำคัญ

1. **ทุก Phase ต้อง Demo ผ่านก่อนไป Phase ถัดไป**
2. **เขียน Test คู่กับ Feature** ไม่ต้องเขียนทีหลัง
3. **Commit เล็กๆ บ่อยๆ** — Feature หนึ่ง Commit
4. **Review PR ด้วยตัวเองก่อน Merge**
5. **อย่ากลัว Refactor** — โค้ดที่ดีคือโค้ดที่เขียนใหม่ได้ง่าย

---

> ✍️ เอกสารนี้เป็น Living Document — ปรับได้ตามความจำเป็นระหว่างพัฒนา
- Mobile ใช้ Tap-to-Place ได้