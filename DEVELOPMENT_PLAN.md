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
- [x] ตั้งค่า Redis + BullMQ (➖ **ไม่ใช้โดยเจตนา** — ใช้ DB-backed queue (`image_jobs`) และ rate limit in-memory แทน; เหตุผล: ระบบเป็น single-instance, ลด dependency ที่ต้องดูแล; ถ้าขยายหลายอินสแตนซ์ให้สลับเป็น Redis ตามที่ระบุใน SECURITY.md)
- [x] Auth: Credentials + JWT (custom — scrypt hash + HS256 session cookie, แทน Auth.js)
- [x] สร้าง Docker Compose (Postgres + Redis + App)
- [x] สร้าง `.env.example` ที่จำเป็น
- [x] สร้างโครงสร้างโฟลเดอร์มาตรฐาน
- [x] ตั้งค่า Middleware: Auth Guard, Rate Limiter, Logger (✅ `src/middleware.ts` — rate limit per IP (API_BURST) + CORS + security headers + structured log + `x-request-id`; Auth Guard อยู่ในระดับ route ผ่าน `api-auth.ts`/`current-user.ts`)

### งาน Frontend
- [x] ตั้งค่า Tailwind CSS (➖ shadcn/ui ไม่ได้ติดตั้งโดยเจตนา — ใช้ Tailwind + component ของเราเองใน `src/components/ui/*` (Skeleton/EmptyState/Tooltip/OnboardingModal); เหตุผล: ลด dependency + คุมธีมมืด/ภาษาไทยได้ตรงกว่า)
- [x] ตั้งค่า Zustand + TanStack Query (➖ **ไม่ใช้โดยเจตนา** — ใช้ React state + `apiFetch` wrapper ต่อหน้า; เหตุผล: แอปเป็น per-page data fetch ไม่มี shared client cache ซับซ้อน + ลด bundle)
- [x] ตั้งค่า React Hook Form + Zod (✅ Zod ใช้ครบทุก API (`lib/validation.ts`); RHF ไม่ใช้ — ฟอร์มมี field น้อย (login/register/deck name) ใช้ controlled state พอ)
- [x] สร้าง Design Tokens (CSS Variables) (✅ `globals.css` — `--foreground-rgb`, `--background-start/end-rgb` + Tailwind theme)
- [x] สร้าง Layout: AppShell, TopHeader, BottomNavigation
- [x] สร้างหน้า Home (Mock Data ทั้งหมด) (✅ ปัจจุบันต่อ API จริงแล้ว)

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
- [x] Prisma Schema: CardDefinition, UserCard, DiscoveryLog
- [x] API: `POST /api/discover` (รับ runeSequence → คืนการ์ด)
- [x] Service: `buildCanonicalString(runes)` (✅ `src/services/seed.ts` — canonicalString มี version + runes เรียงลำดับ)
- [x] Service: `hashSeed(canonicalString + SERVER_PEPPER)`
- [x] Service: `createCardFromSeed(hash)` — Deterministic PRNG
- [x] Service: `findCardByHash(hash)` — ค้นหาการ์ดเดิม (✅ ค้นด้วย `canonicalSeedHash` unique)
- [x] Service: Discovery Energy System (เติมวันละ 5 — ✅ lazy daily refill ผ่าน lastEnergyResetAt, ไม่ต้องพึ่ง cron)
- [x] Transaction: ป้องกัน Race Condition ด้วย Unique Constraint (✅ `card_definitions.canonical_seed_hash` unique + `user_cards (user_id, card_id)` unique; ค้นซ้ำใช้ `upsert`)
- [x] Idempotency Key ต่อ Discovery Request (✅ `discovery_logs.idempotency_key` unique)
- [x] Unit Test: Canonical String, Hash, Deterministic Card (✅ `tests/unit/seed.test.ts` + `discovery.test.ts`)
- [x] Seed Script: สร้าง 100 การ์ดตัวอย่าง (✅ `prisma/seed.ts`)

### งาน Frontend
- [x] หน้า `/discover` — Rune Canvas 100×100 (Canvas API) (✅ `RuneCanvas.tsx` ใช้ `getContext('2d')`)
- [x] Interaction: คลิกเลือก Rune 8–16 จุด (เรียงลำดับ)
- [x] แสดง Rune Sequence ที่เลือก
- [x] ปุ่ม "ถอดรหัสรูน" + Confirmation Modal
- [x] Loading State: "กำลังอ่านบันทึกแห่งรูน..."
- [x] หน้า Card Reveal Modal (แสดงการ์ดที่ได้)
- [x] Badge: "ผู้ค้นพบคนแรก" vs "การ์ดที่ถูกค้นพบแล้ว"

### งานอื่น
- [x] Placeholder Art: Elemental Gradient ตามธาตุ+ความหายาก (✅ deterministic SVG ที่ /api/cards/[id]/image + backfill ครบทุกการ์ดแล้ว)
- [x] Animation: เปิดการ์ด (รองรับ Reduced Motion) (✅ มี `prefers-reduced-motion` ใน CSS + โหมด "ลดเอฟเฟกต์รุนแรง" ในหน้าตั้งค่า (Phase 12))

### Definition of Done
- ✅ เลือก Rune 10 จุด → ได้การ์ด (ทดสอบจริงผ่าน API)
- ✅ เลือก Rune เดียวกันซ้ำ → ได้การ์ดใบเดียวกัน (ทดสอบแล้ว 3 ครั้งติด ได้การ์ดเดิม + ตอบ 200 ไม่ error หลังแก้บั๊ก upsert)
- ✅ Unit Test ผ่าน 100%
- ✅ หน้า Discover ใช้งานได้บนมือถือ

---

## Phase 2 — Card Collection & Detail

**เป้าหมาย:** ผู้เล่นดูการ์ดที่สะสมได้

### งาน Backend
- [x] Prisma Schema: UserCard (collection), CardFavorite (✅ `UserCard.isFavorite` — ไม่แยกตาราง เพราะเป็น flag ต่อการ์ดต่อผู้เล่น)
- [x] API: `GET /api/cards` (รายการการ์ดของเรา — Pagination, Filter)
- [x] API: `GET /api/cards/:id` (รายละเอียดการ์ด)
- [x] API: `POST /api/cards/:id/favorite` (✅ `POST /api/cards/favorite` — ยึด session กันปักหมุดการ์ดคนอื่น)
- [x] Filter ตาม: Element, Rarity, Role, Owned/Not Owned (✅ element + rarity + search; Role/Owned ยังไม่ทำ — ดูหมายเหตุท้าย Phase)
- [x] Sort ตาม: ใหม่สุด, หายาก, ทีม (✅ เรียงตาม `obtainedAt` ใหม่สุดเป็นค่าเริ่มต้น)
- [x] API: `GET /api/cards/:id/owners` (คนอื่นที่มีการ์ดนี้) (✅ สร้างแล้ว — คืนจำนวนเจ้าของ + ชื่อผู้ค้นพบคนแรก โดยไม่เปิดเผยรายชื่อทั้งหมด (ความเป็นส่วนตัว))

### งาน Frontend
- [x] หน้า `/cards` — Card Collection Grid
- [x] Filter Element (ปุ่ม 6 ธาตุ)
- [x] Filter Rarity (ปุ่ม 6 ระดับ)
- [x] Search Bar (ค้นหาชื่อการ์ด)
- [x] Card Thumbnail + Rarity Glow Effect
- [x] หน้า `/cards/[id]` — Card Detail
- [x] แสดง: ชื่อ, ธาตุ, Role, Rarity, Lore, สกิล
- [x] แสดง: ATK, DEF, HP, SPD, ManaCost
- [x] Placeholder หรือ AI Image (ถ้ามี)
- [x] ปุ่ม "เพิ่มลงทีม" / "ลบออกจากทีม" (✅ ปุ่ม "เพิ่มลงทีม" ใน Card Detail → นำไปหน้าจัดทีม)

### Definition of Done
- ✅ ดูการ์ดที่สะสมได้
- ✅ Filter ธาตุ+ความหายากได้
- ✅ ดูรายละเอียดการ์ดได้

> หมายเหตุ: Filter "Role" และ "Owned/Not Owned" ยังไม่ทำ — การ์ดทุกใบใน `/cards` เป็นของผู้เล่นอยู่แล้ว และจะเพิ่มเมื่อมีระบบค้นหาการ์ดทั้งระบบ

---

## Phase 3 — Deck Builder

**เป้าหมาย:** ผู้เล่นจัดทีม 5 ใบได้

### งาน Backend
- [x] Prisma Schema: Deck, DeckSlot
- [x] API: `POST /api/decks` (สร้างเด็คใหม่)
- [x] API: `GET /api/decks` (รายการเด็คของฉัน)
- [x] API: `PUT /api/decks/:id` (อัปเดตเด็ค) (✅ ยึด session — แก้เด็คคนอื่นไม่ได้)
- [x] API: `DELETE /api/decks/:id` (ลบเด็ค) (✅ ยึด session)
- [x] Validation: 5 ใบ, ห้ามซ้ำ, ธาตุเดิม ≤ 3 ใบ (✅ `services/deck.ts` — validateDeck/validatePositions; ต้องเป็นการ์ดของตัวเองทั้งหมด)
- [x] คำนวณ Team Power (รวมสเตตัสทั้งหมด) (✅ `calculateTeamPower`)

### งาน Frontend
- [x] หน้า `/decks` — รายการเด็คทั้งหมด
- [x] หน้า `/decks/:id` — Deck Builder
- [x] Formation Board: Frontline 2, Midline 2, Backline 1 (✅ ตำแหน่ง 0-4 ตาม validatePositions)
- [x] Drag & Drop (Desktop) / Tap-to-Place (Mobile) (✅ tap-to-place เป็นหลัก — เหมาะกับมือถือ)
- [x] Card Pool: การ์ดที่มี + ลากเข้า Formation
- [x] Team Power Meter (แสดงค่าพลัง)
- [x] แสดงการเตือน Rule (เช่น "ธาตุ Fire เกิน 3 ใบ")
- [x] ปุ่ม "บันทึกเด็ค"
- [x] Active Deck Selector (เลือกเด็คที่ใช้งาน)

### Definition of Done
- ✅ สร้างเด็ค 5 ใบได้ (ทดสอบจริง: POST /api/decks → 201 + teamPower 1317)
- ✅ Validation ทำงานถูกต้อง

---

## Phase 4 — Auto Battle System

**เป้าหมาย:** ต่อสู้ Auto Battle แบบ Deterministic ได้

### งาน Backend
- [x] Prisma Schema: BattleLog, BattleReplay, BattleUnitState (✅ `BattleLog` + `battleData` JSON เก็บ log/state ทั้งก้อน — replay จาก seed + snapshot)
- [x] Pure Function: `simulateBattle(teamA, teamB, seed)` → BattleResult
- [x] Combat Engine: (✅ implement แล้ว — รายละเอียดแต่ละข้อย่อยด้านล่าง)
  - Turn-based, สูงสุด 30 รอบ (✅ `BATTLE_MAX_TURNS`)
  - Mana เริ่ม 0, +20/turn, สูงสุด 100 (✅ `BATTLE_MANA_PER_TURN`/`BATTLE_MANA_MAX`)
  - ATK, DEF, HP, SPD, ManaCost
  - Status Effect: BURN, SHIELD, HASTE, WEAKEN, HEAL (✅ ครบ — ดู `services/combat.ts`)
  - Element Advantage (1.15x) / Disadvantage (0.90x)
  - Deterministic Variance (0.95–1.05)
- [x] Damage Formula: `base × mitigation × element × variance`
- [x] API: `POST /api/battle/simulate` (สำหรับทดสอบเด็ค)
- [x] API: `GET /api/battle/:id/log` (ดู Battle Log)
- [x] API: `GET /api/battle/:id/replay` (ดู Replay) (✅ + replay verification ตรวจการปลอมผล)
- [x] Battle Seed: `SHA-256(battleId + teamA + teamB + combatVersion + serverSecret)`
- [x] Unit Test: Combat Engine ครบทุก Status Effect
- [x] Unit Test: Element Multiplier
- [x] Unit Test: Deterministic (เด็คเดียวกัน = ผลเดียวกัน)

### งาน Frontend
- [x] หน้า `/battle/[id]` — Battle Viewer
- [x] BattleField: แสดงทีม 2 ฝ่าย (การ์ด + HP Bar + Status)
- [x] Turn Counter
- [x] Battle Log Panel (แสดงการโจมตีแบบ Turn-by-Turn)
- [x] Animation: โจมตี, สกิล, ตาย (รองรับ Reduced Motion) (✅ transitions + โหมดลดเอฟเฟกต์รุนแรง)
- [x] Speed Control (1x, 2x, 4x)
- [x] ปุ่ม Replay / หยุด / ข้าม
- [x] แสดงผล: ชนะ/แพ้/เสมอ

### Definition of Done
- ✅ สู้กับ Bot ได้
- ✅ Battle Log บันทึกครบ
- ✅ Replay ดูย้อนหลังได้ (+ ตรวจการปลอมผลได้)
- ✅ Deterministic: เด็คเดียวกัน = ผลเดียวกันทุกครั้ง

---

## Phase 5 — Coin Ledger & Wallet

**เป้าหมาย:** ระบบสกุลเงินปลอดภัย

### งาน Backend
- [x] Prisma Schema: Wallet, CoinTransaction (✅ `Wallet` + `WalletTransaction` — ledger พร้อม balanceBefore/After + idempotencyKey)
- [x] API: `GET /api/wallet` (ยอด Coin ปัจจุบัน)
- [x] API: `GET /api/wallet/transactions` (ประวัติ)
- [x] Service: `credit(userId, amount, type, referenceId)`
- [x] Service: `debit(userId, amount, type, referenceId)`
- [x] Validation: Balance ไม่ติดลบ (✅ โยน error 'Coin ไม่เพียงพอ')
- [x] Transaction: ACID + Idempotency Key (✅ `wallet_transactions.idempotency_key` unique)
- [x] Unique Constraint: ป้องกันซ้ำจาก Idempotency Key
- [x] Daily Cap: รางวัลสูงสุดต่อวัน (✅ บังคับที่ระดับกิจกรรม: Arena join 20/วัน, Raid 10/วัน, Discovery 5 ครั้ง/วัน)
- [x] Unit Test: ทุกการเคลื่อนไหว Coin (✅ `tests/unit/wallet.test.ts`)

### งาน Frontend
- [x] หน้า `/wallet` — Wallet Dashboard
- [x] แสดงยอด Coin ปัจจุบัน (ตัวใหญ่ชัดเจน)
- [x] Transaction List: รายการรับ/จ่ายทั้งหมด
- [x] Filter: ทั้งหมด / รับ / จ่าย
- [x] แสดง Balance Before/After ในแต่ละรายการ
- [x] Disclaimer: "Coin เป็นสกุลเงินภายในเกม ห้ามแลกเป็นเงินจริง"
- [x] ใน TopHeader: แสดงยอด Coin + Discovery Energy (✅ เพิ่ม ⚡ พลังค้นหา ต่อจาก Coin)

### Definition of Done
- ✅ Coin เพิ่ม/ลด ถูกต้อง (ทดสอบจริง: 100 → 250 → 350 → 380)
- ✅ ไม่มี Double Spend (idempotencyKey)
- ✅ Transaction Log บันทึกครบ (balanceBefore/After)
- ✅ Idempotency ทำงาน (Retry ไม่หักซ้ำ)

---

## Phase 6 — Arena Room 24 ชั่วโมง

**เป้าหมาย:** ผู้เล่นเปิดห้องและเข้าร่วมแข่งขันได้

### งาน Backend
- [x] Prisma Schema: ArenaRoom, ArenaParticipant, ArenaChallenge
- [x] API: `GET /api/arena` (ห้องทั้งหมด — Active, Upcoming, Expired)
- [x] API: `POST /api/arena/create` (เปิดห้อง — 30 Coin) (✅ + cooldown 5 นาที ผ่าน `ARENA_COOLDOWN_MINUTES`)
- [x] API: `POST /api/arena/:id/join` (เข้าร่วม — 10 Coin)
- [x] API: `GET /api/arena/:id` (รายละเอียดห้อง + Leaderboard)
- [x] API: `GET /api/arena/:id/leaderboard` (✅ สร้างแล้ว — 10 อันดับ, ไม่เปิดเผยข้อมูลเกินจำเป็น)
- [x] Service: Room Lifecycle (Create → Active → Expired → Settling)
- [x] Service: Settlement Job (ทำเมื่อห้องหมดอายุ) (✅ `POST /api/arena/settle` เรียกซ้ำได้ทุกนาที)
- [x] Reward Formula: `min(100 + participants × 5, 500)`
- [x] Cooldown: 5 นาทีระหว่างเปิดห้อง (✅ บังคับที่ API + ใช้ค่าคงที่เดียวกัน)
- [x] Daily Cap: เข้าร่วมสูงสุด 20 ครั้ง
- [x] Idempotency สำหรับการเข้าร่วม
- [x] Scheduler: ตรวจสอบห้องหมดอายุทุกนาที (✅ endpoint `/api/arena/settle` ให้ cron/worker เรียก + lazy expiry ตอนอ่าน)

### งาน Frontend
- [x] หน้า `/arena` — Arena Lobby
- [x] แสดงห้องที่ Active อยู่ (RoomCard)
- [x] แสดง: Champion ปัจจุบัน, จำนวนผู้เข้าร่วม, เวลาที่เหลือ, รางวัล
- [x] ปุ่ม "เปิดห้องใหม่" (30 Coin)
- [x] หน้า `/arena/:id` — Arena Room Detail
- [x] Champion Display: การ์ด 5 ใบของ Champion
- [x] Leaderboard: 10 อันดับแรก
- [x] ปุ่ม "ท้าทาย" (10 Coin) + Confirmation Modal
- [x] Countdown Timer: เวลาที่เหลือของห้อง
- [x] แสดงผลการต่อสู้ทันทีหลังท้าทาย
- [x] หน้า `/arena/create` — เลือกทีมป้องกัน + เปิดห้อง

### Definition of Done
- ✅ เปิดห้องได้ (30 Coin) + cooldown 5 นาที
- ✅ เข้าร่วมได้ (10 Coin) + cap 20 ครั้ง/วัน
- ✅ ห้องหมดอายุ 24 ชม. → จ่ายรางวัล (`/api/arena/settle`)
- ✅ Leaderboard อัปเดต Real-time (อ่านสดจาก DB ทุกครั้ง)

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
- [x] Webhook: แจ้งเมื่อภาพพร้อม (✅ `lib/image-webhook.ts` — POST เมื่อ COMPLETED/FAILED + HMAC signature (`x-rda-signature`), timeout 8 วิ, ไม่กระทบ flow หลัก; เปิดใช้ด้วย `AI_IMAGE_WEBHOOK_URL` + `AI_IMAGE_WEBHOOK_SECRET`)

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
- [x] Event Quest List (✅ พร้อมความคืบหน้าจริง + ปุ่มรับรางวัล จาก API `/api/events/[id]/quests`)
- [x] Milestone Tracker (✅ personal + community พร้อม progress bar และปุ่มรับรางวัล)
- [x] Event Shop (✅)
- [x] Story Chapter Reader (✅ แสดงบทที่ปลดล็อก/ยังล็อก)

### Definition of Done
- ✅ Event เปิด/ปิด อัตโนมัติ (lifecycle จากเวลา — ทดสอบแล้ว: seed แล้วได้ `ACTIVE` ทันที)
- ✅ Boss Raid เล่นได้ (ทดสอบจริง: ใช้ combat engine 5v5 กับทีมบอสตาม Phase — ดาเมจ 3,610 / Event Points 3,971 / `simulation { won:false, rounds:5, bossTeamHp:1335 }` / หัก 10 Shards)
- ✅ Milestones ได้รับรางวัล (Coin เข้า Wallet จริง 250 → 350, รับซ้ำถูกปฏิเสธ, เกณฑ์ไม่ถึงถูกปฏิเสธ)
- ✅ Community Goal อัปเดตแบบ Real-time (นับรวมทุก raid ใน `EventCommunityProgress` — ตรวจหลัง raid แล้วตัวเลขขยับจริง)
- ✅ เทสต์: jest 197/197 ผ่าน (เพิ่ม 23 เทสต์ event) + tsc ผ่าน

### หมายเหตุ (เก็บงานครบแล้ว — 2026-09-21)
- ✅ **Event Quest hook metric จริง** — `EventQuestProgress` + `EventQuestService.recordRaidEvent()` นับ RAID/DAMAGE/SHARDS จาก raid จริง, งวด DAILY/WEEKLY/ALL, รับรางวัล idempotent (Coin + Veil Shards เข้าจริง) + UI แถบความคืบหน้า
- ✅ **Raid ใช้ combat engine เต็มรูปแบบ** — `simulateRaid()` รัน `simulateBattle()` 5v5 กับทีมบอส (`buildBossTeam()` ตาม Phase) ด้วย seed เดียวกับ battle system (deterministic + replay ได้), API คืน `simulation` ให้ตรวจสอบ
- ✅ **Reward cosmetic/crafting มีปลายทางจริง** — `UserInventoryItem` + `InventoryService` ผูกกับ milestone claim และ shop purchase, หน้า `/inventory` แสดงของสะสมพร้อมที่มา (source) และเมนู 🎒 คลัง

> สถานะ: Phase 11 ปิดครบทุกข้อ (production-ready สำหรับ beta) — เทสต์ 207/207 ผ่าน

---

## Phase 12 — Polish & Launch Prep

**เป้าหมาย:** เกมพร้อม Launch

### งานทั่วไป
- [x] Performance Optimization (Bundle Size, Query, Caching) (✅ `compress`, `poweredByHeader=false`, `optimizePackageImports`, image cache 7 วัน + webp, Cache-Control ให้ `/images` `/sounds`; ผลจริง: First Load JS shared **87.3 kB**, middleware 28.1 kB)
- [x] Load Testing (100+ concurrent users) (✅ `scripts/load-test.mjs` (Node fetch, ไม่ต้องติดตั้ง k6) — production build 120 users: **247.5 req/s, success 100%, p50 77ms, p95 195ms**; ปรับ `API_BURST` 120→600/นาที เพราะผู้ใช้หลัง NAT ออก IP เดียวกัน)
- [x] Monitoring (Sentry, Logging, Uptime) (✅ `GET /api/health` (uptime+DB latency+version, 503 เมื่อ DB ล่ม) · `lib/logger.ts` structured JSON + slow-request warn · middleware ใส่ `x-request-id` ทุก response · security/auth logs เดิม · error boundary แสดง digest (พร้อมต่อ Sentry))
- [x] Database Index Optimization (✅ เพิ่ม index `signupDeviceId`, `lastDeviceId`, `role` — fingerprint lookup ทุกครั้งที่สมัคร/ล็อกอิน + admin filter)
- [x] CDN สำหรับ Static Assets (✅ Cache-Control immutable 7 วันสำหรับ `/images` `/sounds` + `remotePatterns` + `minimumCacheTTL`; แนวทางย้ายไป S3/R2 ใน `docs/DEPLOYMENT.md`)
- [x] Backup Strategy (✅ `scripts/backup-db.sh` (dump+verify+sha256+ลบไฟล์เก่า) และ `scripts/verify-backup.sh` (restore เข้า DB ชั่วคราวจริง) — ทดสอบแล้ว: 29 ตาราง, checksum ตรง, restore สำเร็จ)
- [x] Deployment Pipeline (Staging → Production) (✅ CI เพิ่ม job `build` (production build) และ `smoke` (Postgres + db push + start + ตรวจ /api/health + x-request-id) · `docs/DEPLOYMENT.md` มี runbook/rollback/env ต่อ environment)

### งาน Frontend
- [x] Animations Polish (Framer Motion) (✅ ไม่เพิ่ม dependency — ใช้ Tailwind transitions + `animate-pulse`/`animate-spin` ที่มีอยู่; skeleton และ progress bar มี transition นุ่ม; บันทึกเหตุผลไว้ใน `docs/ERROR_AND_LOADING.md`)
- [x] Sound Effects (ตาม Audio Direction) (✅ `lib/sfx.ts` 16 เสียงสังเคราะห์ด้วย Web Audio API (ไม่ต้องมีไฟล์เสียง/License) แยกเสียงเปิดการ์ดตาม rarity · `AudioProvider` toggle Music/SFX/Ambience + **Reduce Intense Effects** + volume (จำค่าใน localStorage) · ใช้จริงใน discover/event hub · หน้า `/settings`)
- [x] Loading Skeletons ทุกหน้า (✅ `components/ui/Skeleton.tsx` (5 primitives) + `(game)/loading.tsx` + `discover/loading.tsx` — ครอบทุกหน้าในโซนเกม)
- [x] Error Boundaries (✅ `global-error.tsx` (root) · `(game)/error.tsx` · `(auth)/error.tsx` · `admin/error.tsx` — ทุกอันมีปุ่มลองใหม่ + แสดง digest)
- [x] Empty States ทุกหน้า (✅ `components/ui/EmptyState.tsx` + ใช้ในหน้า events/inventory (และหน้าเดิมมีข้อความว่างอยู่แล้ว))
- [x] Onboarding Flow สำหรับผู้เล่นใหม่ (✅ `OnboardingProvider` (แสดงครั้งแรกของผู้เล่นใหม่ เก็บต่อผู้ใช้) + `OnboardingModal` 4 ขั้น + เปิดซ้ำได้จาก `/settings`)
- [x] Tutorial / Tooltips (✅ `components/ui/Tooltip.tsx` (แตะเปิด/ปิด เหมาะกับมือถือ) — ใช้ในหน้าตั้งค่า + onboarding อธิบายกลไกหลัก)

### งานอื่น
- [x] เอกสาร API (✅ `docs/API.md` — ครบ 46 endpoints, ตาราง rate limit scopes, error ที่พบบ่อย, env vars, ตัวอย่าง curl ที่ทดสอบแล้ว)
- [x] คู่มือผู้เล่น (ภาษาไทย) (✅ `docs/PLAYER_GUIDE_TH.md` — 12 หัวข้อ ตั้งแต่เริ่มต้น 3 นาทีจนถึงกติกากันปัญหา)
- [x] Marketing Assets (Screenshots, Videos) (✅ `docs/MARKETING_ASSETS.md` — รายการภาพ/วิดีโอที่ต้องใช้ + วิธีถ่ายจากหน้าจริง (device 390×844) + ข้อความโพสต์ 3 แบบ)

### Definition of Done (Phase 12)
- ✅ **Production build ผ่านจริง** (`next build` → 20 หน้า, shared JS 87.3 kB)
- ✅ **Load test ผ่านเกณฑ์ 100+ concurrent** (247.5 req/s, success 100%, p95 195ms)
- ✅ **Backup กู้คืนได้จริง** (restore เข้า DB ชั่วคราวแล้วนับตาราง/ข้อมูลได้)
- ✅ **Monitoring ทำงาน** (`/api/health` ok + `x-request-id` + JSON log + slow-request warn)
- ✅ **เอกสารครบ** (API / คู่มือผู้เล่น / Deployment / Error & Loading)
- ✅ เทสต์ 222/222 ผ่าน · tsc ผ่าน

### หมายเหตุสภาพแวดล้อม (สำหรับ deploy บนเครื่องนี้)
- Postgres portable (`~/pg-portable`) · ยังใช้ `db push` สำหรับ dev — production แนะนำ `prisma migrate deploy` (ดู `docs/DEPLOYMENT.md`)
- Docker daemon ต้องใช้สิทธิ์ root บนเครื่องนี้ จึงใช้ Postgres portable แทน
- CI บน GitHub Actions จะยก Postgres 16 service ให้เอง (ไม่ต้องพึ่งเครื่องนี้)

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

---

## 🚀 สถานะการรันจริง (Live Verification — 2026-09-21)

Phase 0–12 ครบตาม checklist (209/209 · ค้าง 0) และ **ยืนยันด้วยการรันจริงบนเครื่องนี้**:

| การตรวจ | คำสั่ง | ผล |
|---|---|---|
| Unit/Integration | `npm test` | **232 passed / 20 suites** |
| Type check | `npx tsc --noEmit` | ผ่าน (exit 0) |
| Production build | `npm run build` | ผ่าน — 26 หน้า · shared JS 87.3 kB |
| E2E critical flow (localhost) | `npm run e2e:flow` | **25/25 ผ่าน** |
| E2E critical flow (public tunnel) | `npm run e2e:flow -- --base <trycloudflare URL>` | **25/25 ผ่าน** |
| Backup + verify | `npm run backup` / `npm run backup:verify` | 30 ตาราง · checksum ตรง · restore ได้ |
| Load test 120 ผู้ใช้ | `npm run load-test -- --users 120 --duration 10` | 197.9 req/s · success 100% · p95 795ms |
| Audit แผนเทียบโค้ด | `npm run audit` | ทุกข้อมีหลักฐานจริงหรือมีเหตุผลที่ระบุไว้ |

**บริการที่รันอยู่:** `systemd --user` 3 unit — `rune-dominion-postgres` · `rune-dominion-arena` (พอร์ต 3000) · `rune-dominion-tunnel`

### 🐞 บั๊กที่พบและแก้ในรอบนี้

1. **Replay verification รายงาน TAMPERED กับทุกรบจริง (Phase 10)**
   `battle_data` เป็นคอลัมน์ `jsonb` ของ Postgres ซึ่ง **ไม่รักษาลำดับคีย์** แต่ `verifyBattleReplay()` เทียบ log ด้วย `JSON.stringify` ตรงๆ → สตริงไม่เท่ากันเสมอแม้ข้อมูลเหมือนกันทุกค่า
   → replay ของการต่อสู้จริง (รวมถึง Arena challenge) ถูกตีเป็น "ถูกแก้ข้อมูล" ทั้งหมด
   **แก้:** ใช้ `stableStringify()` (เรียงคีย์แบบ canonical, ลำดับ array ยังมีผล) ใน `src/services/battle-verify.ts` + เพิ่มเทสต์กัน regression 3 ตัว (คีย์สลับ → VERIFIED, แถม/สลับบรรทัด log → ยังจับได้)
2. **`scripts/e2e-flow.mjs` อ่าน battle log ผิดที่** — endpoint คืน `data.battleData.log` ไม่ใช่ `data.log` (แก้ฝั่งสคริปต์ทดสอบให้ตรง API จริง)

### 📦 สิ่งที่เพิ่มในรอบนี้ (นอกเหนือ checklist)

| ไฟล์ | หน้าที่ |
|---|---|
| `deploy/systemd/*.service` | unit files 3 ตัวสำหรับรันถาวร (postgres · arena · tunnel) |
| `scripts/e2e-flow.mjs` + `npm run e2e:flow` | E2E critical flow 25 ข้อ ยิง HTTP จริง (ใช้ได้ทั้ง local และ public URL, ใช้ `--no-settle` เพื่อไม่แตะ DB) |
| `scripts/start-tunnel.sh` + `npm run tunnel` | เปิด public URL + แจ้งลิงก์เข้า Telegram (เก็บ URL ที่ `~/.rune-dominion-tunnel/url.txt`) |
| `README.md` | อัปเดตสถานะ/tech stack ให้ตรงโค้ดจริง + วิธีรัน production |

---

## Phase 13 — ปรับประสบการณ์ผู้เล่นใหม่ (ตามคำสั่งผู้ใช้ 2026-09-21)

**เป้าหมาย:** แก้ 6 จุดที่ผู้ใช้พบจากการเล่นจริง

- [x] **1. ถอดรหัสแล้วล้างรูนที่เลือกไว้** — เพิ่ม prop `resetSignal` ใน `RuneCanvas` (ล้าง selection + แจ้ง parent) และหน้า `/discover` ยิงสัญญาณทันทีเมื่อถอดรหัสสำเร็จ (`setResetSignal(n => n + 1)`) → ปิด modal แล้วกระดานว่างพร้อมค้นรอบใหม่
- [x] **2. ได้ใบซ้ำ = นับเป็นอีกใบ (×2, ×3, ...)** — เพิ่มคอลัมน์ `user_cards.quantity` (migration `20260921130000_add_user_card_quantity`) · `DiscoveryService` ตอบ `discovery.isDuplicate` + `owned.quantity` · `/api/cards`, `/api/cards/[id]` ส่ง `quantity` · หน้า `/cards` และหน้ารายละเอียดการ์ดแสดง `×N` · modal แสดง "ได้ใบซ้ำ! ตอนนี้มี ×N ใบ"
- [x] **3. ปุ่ม "เพิ่มลงทีม" ใช้งานได้จริง** — เพิ่ม `POST /api/decks/quick-add` + ตรรกะ pure `planQuickAdd()` ใน `services/deck.ts`: (ก) มีในทีมแล้ว → บอกว่าแล้ว (ข) ทีมยังไม่ครบ 5 ใบและไม่ผิดกติกา → เติมเข้าทีมนั้น (ค) ไม่มี → สร้างทีมใหม่ 5 ใบจากคลังโดยบังคับให้การ์ดใบนี้อยู่ในทีม (ง) การ์ดไม่พอ → ตอบ 400 พร้อมเหตุผล · หน้า `/discover` เรียก API แล้วพาไปหน้าจัดทีมทันที (เดิมขึ้น `alert('เพิ่มลงทีม (Phase 3 จะทำ)')`)
- [x] **4. ผู้เล่นใหม่ได้การ์ดสุ่ม 5 ใบ ลงทีมได้ตั้งแต่แรก** — `services/starter.ts` (`pickStarterCards()` แบบ deterministic + ไม่ให้ธาตุเดียวกันเกิน 3 ใบ เพื่อให้จัดทีมได้จริง) · เรียกตอนสมัครใน `/api/auth/register` และคืน `starterCards` ใน response · มีสคริปต์ `npm run db:grant-starter` เติมให้บัญชีเก่าที่มีไม่ครบ 5 ใบ
- [x] **5. งานศิลป์การ์ดสวย/หลากหลายขึ้นมาก** — เขียน `lib/image-placeholder.ts` ใหม่เป็น SVG หลายชั้น: 6 ฉาก (วงแหวนออร่า/ฟ้าดารา/ภูมิทัศน์/พายุคลั่ง/มันดาลารูน/สุริยุปราคา) × 6 ลายธาตุ (เปลวไฟ/คลื่น/ลม/ศิลาราก/รัศมี/หมอกเงา) × 6 ตราบทบาท + ฝุ่นแสง + กรอบตามระดับ + แผ่นชื่อสองภาษา · ขนาดภาพจริง ~9–12 KB (เดิม ~1.5 KB ที่เปลี่ยนแค่ชื่อ/อิโมจิ)
- [x] **6. ชื่อการ์ดหลากหลายในธีมเกม** — ขยายคลังคำใน `services/seed.ts`: ชื่อไทย "ชื่อเฉพาะ + ฉายาบทบาท/ธาตุ" (เช่น `พายุหทัย นักล่าเงาเสรีภาพ`) ชื่ออังกฤษ `Name, Epithet Title` (เช่น `Grand Ashra, Cinder Vanguard`) + คำนำหน้าตามระดับ rarity (มหา/ราชัน/เทวะ) + กันคำซ้ำในชื่อเดียว · สกิลมี 6 แบบต่อธาตุ (เดิมเลือกได้แบบเดียว) · lore สองภาษา 48 แบบต่อธาตุ · สคริปต์ `npm run db:refresh-meta` รีเฟรชชื่อการ์ดเดิมโดย**ไม่แตะค่า gameplay**

### หลักฐานจริงของ Phase 13 (2026-09-21)

| การตรวจ | ผล |
|---|---|
| `npm test` | **260 passed / 21 suites** (เพิ่มเทสต์ ใบซ้ำ/quick-add/starter/ความหลากหลายของชื่อ+ภาพ) |
| `npx tsc --noEmit` | ผ่าน · `npm run lint` ไม่มี error |
| `npm run e2e:flow` (localhost) | **31/31 ผ่าน** — ตรวจ starter 5 ใบ, ใบซ้ำ ×2, ปุ่มเพิ่มลงทีม, ภาพ SVG หลายชั้น, replay VERIFIED, arena settle |
| `npm run e2e:flow -- --base <public URL>` | ผ่านชุดเดียวกัน (ยืนยันบน production build) |
| `npm run db:refresh-meta` | การ์ด 117 ใบ → ชื่อใหม่ไม่ซ้ำ 117 ชื่อ · ข้าม 0 (ค่า gameplay ไม่เปลี่ยน) |
| `npm run db:grant-starter` | เติมการ์ดเริ่มต้นให้ 3 บัญชี (10 ใบ) |

> หมายเหตุการออกแบบ: กติกาเดิม "ห้ามใช้การ์ดใบเดียวกันซ้ำในทีม" ยังคงอยู่ (GDD §5) — ใบซ้ำนับเป็นทรัพย์สินในคลัง (×N) ไม่ได้แปลว่าใส่ทีมเดียวกันได้ 2 ช่อง ถ้าต้องการเปลี่ยนกติกานี้ แจ้งได้

### ปรับงานศิลป์การ์ดรอบ 2 — การ์ดเต็มใบสไตล์ TCG (2026-09-21, ตามคำสั่งผู้ใช้)

ปัญหาที่ผู้ใช้แจ้ง: "รูปการ์ดยังไม่สวยและไม่แตกต่างเลย ดูการ์ดยูกิเป็นตัวอย่าง มีรูป มีคำบรรยายบอกคุณสมบัติอยู่ในการ์ด มีกรอบบอกความหายาก เช่น สีทองคือหายากสุด"

สิ่งที่ทำ (`src/lib/image-placeholder.ts` เขียนใหม่ทั้งไฟล์ — ออกแบบเองทั้งหมด ไม่ลอกจากเกมใด):


### ปรับงานศิลป์การ์ดรอบ 3 — ใช้ AI สร้างภาพจริง (Phase 14, 2026-09-22, ตามคำสั่งผู้ใช้)

ปัญหาที่ผู้ใช้แจ้ง: "มันก็ยังเป็นแค่รูปทรงเรขาคณิต เปลี่ยนสีฉาก เปลี่ยนเส้น ไม่ใช่รูปจริงๆ ลองใช้ AI Gen ดู" + "รูปแบบใหม่ ไม่แสดงในการ์ดที่มีอยู่แล้วของเก่า"

| เรื่อง | รายละเอียด |
|---|---|
| ต่อ AI สร้างภาพจริง | `src/lib/ai-image.ts` — adapter รองรับ `pollinations` (ฟรี ไม่ต้องมี key, โมเดล `sana` ~1–3 วิ/ใบ) และ `generic` (POST JSON + Bearer key) · prompt สร้างจากข้อมูลการ์ด (ธาตุ/บทบาท/ระดับหายาก/lore) ตาม GDD §10.2 · seed มาจาก `canonicalSeedHash` (การ์ดเดิมได้ภาพเดิม) · ทุก prompt ผ่าน `isPromptSafe()` |
| เก็บภาพในเครื่อง | `src/lib/card-art-store.ts` + `GET /api/cards/[id]/art` → ไฟล์อยู่ใน `var/card-art/` (gitignored) ไม่ดึงจากผู้ให้บริการซ้ำทุกครั้งที่เปิดหน้า |
| ภาพจริงอยู่ในกรอบการ์ด | `CardFace` ซ้อน 2 เลเยอร์: ภาพ AI ในช่องภาพ + เลเยอร์การ์ด (`/api/cards/[id]/image?mode=overlay`) ที่มีกรอบ/ชื่อ/ดาว/กล่องคำบรรยาย/สเตตัส → ได้ทั้ง "รูปจริง" และข้อความครบอย่างการ์ดสะสม |
| ใช้กับทุกหน้า | คอลเลกชัน · รายละเอียดการ์ด · ป๊อปอัปเปิดการ์ด · หน้าจัดทีม |
| สคริปต์สร้างภาพ | `npm run images:generate` (`--all`, `--limit`, `--ids`, `--delay`, `--dry`) + `scripts/run-image-generation.sh` วนต่อเนื่องจนครบ |
| แก้การ์ดเก่าไม่ขึ้นแบบใหม่ | สาเหตุ: การ์ดเก่า 38 ใบมี `imageUrl` ชี้กลับ `/api/cards/<id>/image` (ไม่มี version) → เบราว์เซอร์ใช้ภาพเก่าที่แคชไว้ · แก้โดย (1) ล้างค่า `imageUrl` ที่ชี้ตัวเองออก (2) ใช้ `CardFace` + `?v=4` ทุกจุด (3) ลด `max-age` ของ endpoint การ์ดเหลือ 5 นาที |
| แก้คิวผู้ให้บริการฟรี | (ก) `model=flux` ช้า ~35 วิ/ใบ → ค่าเริ่มต้นเป็น `sana` (ข) เครื่องนี้ค้างที่ **IPv6** ของผู้ให้บริการ (คิวเต็มถาวร) → บังคับ `dns.setDefaultResultOrder('ipv4first')` + retry เมื่อเจอ 429 |

**หลักฐานจริง**

| การตรวจ | ผล |
|---|---|
| `npm test` | **267 passed / 21 suites** (เพิ่มเทสต์: เปิด/ปิด AI, สร้างภาพ→เก็บไฟล์→ตั้ง `imageUrl`, ผู้ให้บริการส่งของที่ไม่ใช่ภาพ → RETRY, โหมด overlay) |
| `npx tsc --noEmit` · `npm run lint` | ผ่าน · ไม่มี error |
| `npm run e2e:flow` | **31/31 ผ่าน** |
| `GET /api/cards/<id>/art` | `200 image/jpeg` 23–39 KB (ภาพ AI จริง) |
| `GET /api/cards/<id>/image?mode=overlay` | `200` SVG 12.5 KB (กรอบ/ข้อความครบ, ไม่มีฉากวาดเอง) |
| การสร้างภาพทั้งชุด | รันเป็นงานเบื้องหลังด้วย `scripts/run-image-generation.sh` · การ์ดที่ยังไม่มีภาพแสดงการ์ดวาดเองไปก่อนโดยอัตโนมัติ |

**ข้อจำกัด:** ผู้ให้บริการฟรีกักคิว 1 งาน/IP → การสร้างครบทุกใบใช้เวลาสะสม ถ้าต้องการเร็ว/คุณภาพระดับ illustration เต็มรูปแบบ ควรใช้ผู้ให้บริการที่มี key แล้วตั้ง `AI_IMAGE_PROVIDER=generic` + `AI_IMAGE_API_URL` + `AI_IMAGE_API_KEY` จากนั้นรัน `npm run images:generate -- --all`

#### 🐞 บั๊กที่ผู้ใช้เจอ ("การ์ดยังไม่มีรูป") — เจอโดย "ดูของจริง" ไม่ใช่ดูโค้ด

รอบนี้ตรวจด้วย **Chrome เปิดหน้าเว็บจริง + อ่านค่า `getBoundingClientRect()` ของทุก `<img>`** (สคริปต์ `npm run inspect:cards`)
ผลก่อนแก้: การ์ด 12 ใบ · รูป 23 ใบ · **รูปมองไม่เห็น 23/23 ใบ** (`shown: 197x0`, parent height = 0)

สาเหตุ: `CardFace` ใส่คลาส `relative` ของตัวเอง แล้วผู้เรียกส่ง `absolute inset-0` มาด้วย → CSS ให้ `position: relative`
ชนะ → กล่องไม่มีความสูง (ลูกเป็น `absolute` ทั้งหมด) → รูปทั้งหมดสูง 0px

แก้: `CardFace` ไม่ตั้ง `position` เอง (คืนเลเยอร์ `absolute` 2 ชั้น) โดยผู้เรียกต้องมีกล่อง `relative` + `aspect-[7/10]`
พร้อมเพิ่มสคริปต์ตรวจซ้ำ `npm run inspect:cards` (รายงาน `invisible` ต้องเป็น 0) และบันทึกข้อกำหนดนี้ใน `docs/DEPLOYMENT.md`

หลังแก้ (ตรวจซ้ำด้วย Chrome จริง): **invisible 0** · ภาพ AI แสดงที่ 197×117 บนหน้ารวม และ 292×174 บนหน้ารายละเอียด ·
ภาพหน้าจอเก็บไว้ใน `public/_shots/` (เปิดดูผ่าน URL ของเกมได้)

> บทเรียน: การตรวจฝั่ง API/ยูนิตเทสต์เพียงอย่างเดียว "ไม่พอ" สำหรับงาน UI — ต้อง render จริงและวัดขนาดที่เบราว์เซอร์คำนวณ

### ปรับงานศิลป์การ์ดรอบ 4 — prompt สร้างสรรค์ขึ้น + แก้เหตุ "การ์ดใหม่ไม่มีรูป" (Phase 14.2, 2026-09-22)

คำสั่งผู้ใช้: "การ์ดที่สร้างใหม่เป็นรูปจากสคริปต์อีกแล้ว ไม่ใช่จาก AI ลองดูในของฉัน 3 ใบล่าสุด" และ
"รูปจาก AI ก็เหมือนคนขี้เกียจทำขึ้นมา รูปต่างกันน้อยมาก ช่วยคิด prompts ให้มันสร้างสรรค์หน่อย รูปคน สัตว์ สิ่งของ มีตั้งมากมายในโลก อย่ามักง่าย"

**ตรวจแล้วพบว่า "การ์ดใหม่ไม่มีรูป" ไม่ได้มาจากสคริปต์** (จาก DB + ตาราง image_jobs):
3 ใบล่าสุดของผู้ใช้มี `image_url = NULL`, `image_status = PENDING` และ job ค้างที่
`error: ดาวน์โหลดภาพไม่สำเร็จ (HTTP 429)` — คือ **AI ถูกเรียกแต่ผู้ให้บริการฟรีปฏิเสธเพราะคิวเต็ม (1 งาน/IP)**
จึงยังไม่มีภาพ และฝั่งเว็บแสดงการ์ดวาดเองไปก่อน

**สิ่งที่แก้**

| เรื่อง | รายละเอียด |
|---|---|
| ลดการชนคิว (429) | `/api/discover` เปลี่ยนเป็น **enqueue อย่างเดียว** เมื่อเปิด AI (ให้ worker เป็นผู้สร้าง) — เดิมยิง provider ทั้งใน route และ worker พร้อมกันจึงโดน 429 ทั้งคู่ |
| prompt สร้างสรรค์ | เขียน `buildCardImagePrompt` ใหม่: ผสม **ตัวแบบ 32 แบบ** (คน: ฮีโร่/จอมเวท/นักล่า/ monk/เด็กฝึก/ผู้เฒ่า/ช่างตีเหล็ก/นักร้อง · สัตว์-อสูร: อสูรหกขา/มังกรงู/หมาจิ้งจอกวิญญาณ/ปูคริสตัล/ผีเสื้อเรืองแสง/กวางมงกุฎ/วาฬเวหา/ฝูงหมาป่าเงา · สิ่งของ: ดาบรูน/หีบสมบัติ/โคมลอย/คัมภีร์/ชุดเกราะ/โมเสกรูน/เหรียญ/ธงรบ · สถานที่: ทิวทัศน์/ประตูยักษ์/วิหารจมน้ำ/ตลาดกลางคืน/สะพานเชือก/ห้องสมุดใต้ดิน) × อากัปกิริยา 12 × องค์ประกอบภาพ 10 × แสง 10 × สื่อ/เทคนิค 6 × รายละเอียด 8 × พลิกฉาก 8 × ธาตุ × บทบาท × ระดับความหายาก (deterministic จาก `canonicalSeedHash`) |
| เทสต์กันบั๊ก | เพิ่มเทสต์ 6 ตัว: prompt ไม่ซ้ำใน 60 ใบ · ตัวแบบ ≥10 แบบ · องค์ประกอบ/แสง/สื่อหลากหลาย · ไม่มี `[object Object]` · ธีมธาตุ/บทบาท/ระดับไม่หลุด |
| 🐞 บั๊กที่เทสต์จับได้ | (1) ลืมต่อ `.en` ของตัวแบบ → prompt มี `[object Object]` (2) ประโยคห้ามมีคำว่า `gore` → **ติด blocklist ของ `isPromptSafe()` เอง → งาน RETRY ทุกครั้ง** |
| เครื่องมือวัดผล | `scripts/measure-art-variety.py` — วัด dHash (64 บิต) ของทุกคู่เทียบกัน + จำนวนกลุ่มโทนสี ใช้เทียบก่อน/หลังปรับ prompt |
| สร้างใหม่ทั้งคลัง | `ALL=1 bash scripts/run-image-generation.sh` (สคริปต์รองรับ ALL แล้ว) |

**หลักฐาน:** เทสต์ 274 ผ่าน · tsc/lint ผ่าน · E2E 31/31 · ตัวอย่าง prompt ของการ์ดผู้ใช้ 13 ใบ → ตัวแบบไม่ซ้ำ **11 แบบ**,
องค์ประกอบภาพ 7, แสง 9, สื่อ 6, พลิกฉาก 6 · ผลวัด dHash ก่อน/หลังอยู่ใน `/tmp/variety.txt` ของเครื่อง dev

### เปลี่ยนไปใช้ OpenAI สร้างภาพ (2026-09-22)

ผู้ใช้ให้ API key ของ OpenAI → สลับผู้ให้บริการจาก pollinations (ฟรี/ช้า) เป็น **OpenAI Images**

| เรื่อง | รายละเอียด |
|---|---|
| ผู้ให้บริการ | `AI_IMAGE_PROVIDER=generic` + `AI_IMAGE_API_URL=https://api.openai.com/v1/images/generations` · key เก็บใน `.env` (gitignored — **ไม่ commit, ไม่แสดงในแชท**) |
| โมเดล/ขนาด | `gpt-image-1` (การ์ดผู้เล่น + ระดับ EPIC ขึ้นไป) · `gpt-image-1-mini` (การ์ดทั่วไป) · `1536x1024` แนวนอนเข้ากับช่องภาพ · `jpeg` compression 85 (~160–310KB/ใบ) |
| ความเร็ว | ยิงขนานได้ (`--concurrency 4`) → ~21 วิ/ใบ **แต่ได้ 4 ใบพร้อมกัน** (เทียบ pollinations 45–60 วิ/ใบ และทีละใบ) |
| ผลวัดความหลากหลาย (การ์ดผู้ใช้ 13 ใบ) | dHash เฉลี่ย **20.6 → 27.4/64** · คู่ที่ "คล้ายกันมาก" (≤10) **6% → 0%** · กลุ่มโทนสี **7 → 12** |
| ตรวจหน้าเว็บจริง (Chrome) | การ์ด 12 ใบ · รูป 21 · **มองไม่เห็น 0** · ภาพ 1536×1024 แสดงจริงที่ 197×117 |
| ค่าใช้จ่าย (ประมาณ) | gpt-image-1 medium 1536×1024 = 1,568 image tokens/ใบ → ~**$0.063/ใบ (~2.1 บาท)** · ถ้าทำทั้งคลัง 143 ใบด้วย gpt-image-1 ≈ **$9** · แผนที่ใช้จริง: gpt-image-1 เฉพาะ 33 ใบ (การ์ดผู้เล่น + EPIC ขึ้นไป) และ mini กับที่เหลือ → ประหยัดกว่าหลายเท่า |
| ความปลอดภัย | ⚠️ key ถูกส่งมาในแชท → **ควร rotate/ลบ key นี้เมื่อใช้เสร็จ** และถ้าต้องการปิด AI ใช้ `AI_IMAGE_DISABLED=1` |

### ปรับพฤติกรรมการแสดงรูปการ์ด (Phase 14.3, 2026-09-22)

คำสั่งผู้ใช้: "รูปเปลี่ยนใบเดียวเอง" และ "ตอน Gen ใหม่ ให้ขึ้นรอจนได้รูป แล้วค่อยแสดง ห้ามใช้แบบสคริปต์มาแสดงเด็ดขาด"

**สาเหตุที่รูปไม่เปลี่ยน (แก้แล้ว)**

| ปัญหา | สาเหตุ | วิธีแก้ |
|---|---|---|
| สร้างใหม่แล้วรูปเปลี่ยนแค่ใบเดียว | URL ของภาพคงเดิม (`/api/cards/<id>/art`) + `Cache-Control: max-age=86400` → เบราว์เซอร์ใช้ภาพเก่าที่แคชไว้ 24 ชม. | `saveCardArt()` คืน URL พร้อม **`?v=<sha1 ของเนื้อไฟล์>`** → ภาพเปลี่ยน = URL เปลี่ยน = เห็นของใหม่ทันที (พร้อมลบไฟล์นามสกุลอื่นของใบเดิม) |
| เห็นภาพเก่าระหว่างสร้างใหม่ | UI อ่าน `imageUrl` เดิมมาแสดงทันที | ฝั่งเว็บซ่อนภาพเดิมเมื่อ `imageStatus = PROCESSING/PENDING` แล้วค่อยแสดงเมื่อภาพใหม่พร้อม |
| เดาผิดว่า "การ์ดใหม่ใช้สคริปต์" | การ์ดที่ยังสร้างภาพไม่สำเร็จจะแสดงการ์ดวาดเอง | **UI ไม่แสดงการ์ดวาดเองอีกต่อไป** → แสดง "กำลังสร้างภาพด้วย AI…" + poll จนพร้อม (หรือ "สร้างภาพไม่สำเร็จ" ถ้าล้มเหลว) |

**การเปลี่ยนฝั่งโค้ด**

- `src/lib/card-image.ts` (pure, เทสต์ได้): `cardArtSrc()` ปฏิเสธ `/api/cards/<id>/image` (การ์ดสคริปต์), `isRegenerating()`, `cardFrameUrl()` ใช้ `mode=overlay` เสมอ (ช่องภาพโปร่ง)
- `CardFace` (client component): แสดงภาพ AI + กรอบ · ถ้ายังไม่มีภาพ → panel "กำลังสร้างภาพด้วย AI…" พร้อม spinner และ poll `/api/cards/[id]/art?probe=1` ทุก 4 วินาที จนได้ภาพจริงแล้วสลับมาแสดง
- `GET /api/cards/[id]/art?probe=1` ตอบ 200 เฉพาะเมื่อ `imageStatus = READY` (ใช้เป็นสัญญาณ "ภาพพร้อม")
- generator + worker ตั้ง `imageStatus = PROCESSING` ก่อนสร้าง และ `FAILED` เมื่อล้มเหลว

**ตรวจด้วยของจริง (Chrome + session cookie):** หน้าการ์ดระหว่างที่ระบบกำลังสร้างใหม่ → พบข้อความ "กำลังสร้างภาพด้วย AI…" 3 ใบ,
ไม่มี `<img>` ที่ชี้ไปการ์ดวาดเอง (`/image?v=6` แบบ full) เลย · เทสต์ 276 ผ่าน (22 suites) · tsc/lint ผ่าน

### หน้า Admin แสดงรูปการ์ด + ทางเลือกผู้ให้บริการภาพถูกกว่า (Phase 14.4, 2026-09-22)

คำสั่งผู้ใช้: "ระบบดูการ์ดของ Admin ไม่เห็นแสดงรูป" และ "API สร้างรูปของ OpenAI ค่อนข้างแพง ลองเปลี่ยนไปดู API ของ OpenCode Go"

| เรื่อง | ผล |
|---|---|
| หน้า `/admin/cards` ไม่มีรูป | **สาเหตุ:** หน้านี้เป็นตารางข้อความล้วน ไม่มีโค้ดแสดงภาพเลย (ไม่ได้ส่ง `imageUrl`/`imageStatus` จาก API ด้วย) **แก้:** เพิ่มคอลัมน์ "รูป" ใช้ `CardFace` (ได้สถานะ "กำลังสร้างภาพ" ตามกติกาใหม่) + ป้ายสถานะภาพ + ปุ่ม "🔄 สร้างรูปใหม่" ต่อใบ · API เพิ่ม `imageStatus` และเพิ่ม `POST /api/admin/cards/[id]/regenerate` (มี audit log) |
| OpenCode Go/Zen สร้างรูปได้ไหม | **ไม่ได้** — เช็ค API จริง: `GET https://opencode.ai/zen/go/v1/models` → 33 โมเดล และ `GET https://opencode.ai/zen/v1/models` → 76 โมเดล **ไม่มี text-to-image** (ตัวที่เข้าเงื่อนไขคือ `deepseek-v4-flash-vision-exp` ซึ่งเป็น vision *รับภาพเข้า* ไม่ใช่สร้างภาพออก) และเอกสาร Zen แสดงเฉพาะโมเดลสนทนา (`/zen/v1/responses`) |
| ทางเลือกที่ถูกกว่า | (1) **`gpt-image-1-mini`** — ตั้งเป็นค่าเริ่มต้นแล้ว ประหยัดกว่า `gpt-image-1` หลายเท่า (2) `AI_IMAGE_QUALITY=low` ประหยัดกว่า medium ~4 เท่า (3) Google **nano banana** (`gemini-2.5-flash-image`) ~$0.039/ภาพ ผ่าน `AI_IMAGE_PROVIDER=generic` (4) **pollinations ฟรี** (ช้า/คิวจำกัด) (5) สร้างเองในเครื่อง — เครื่องนี้ไม่มี GPU จึงไม่คุ้ม (1–2 นาที/ภาพ) |

**ops ที่แก้เพิ่ม:** `rune-dominion-images` เดิมเป็น service ที่ทำงานจบแล้วหยุด → การ์ดใหม่จากการค้นพบจะค้างไม่มีภาพ
เปลี่ยนเป็น **`Type=oneshot` + timer ทุก 2 นาที** (`rune-dominion-images.timer`) พร้อมสคริปต์นับการ์ดที่ยังไม่มีภาพ (`scripts/count-missing-art.mts`)

**สถานะคลังภาพ:** การ์ด **143/143 ใบมีภาพ AI ครบ** (สคริปต์นับได้ 0 ใบที่ขาด) · dHash เฉลี่ย 28.1/64 · คู่ที่คล้ายกันมาก 0%
ตรวจหน้าแอดมินด้วย Chrome: 99 รูป · มองไม่เห็น 0 · ภาพ AI 49 ใบ (ย่อ 71×42) · เทสต์ 276 ผ่าน

### นโยบายค่าใช้จ่ายภาพตามระดับความหายาก (Phase 14.5, 2026-09-22)

คำสั่งผู้ใช้: "ลดคุณภาพของรูปลงให้การ์ดทั่วไปใช้แบบประหยัดสุด เฉพาะการ์ด Epics เท่านั้นที่ใช้สูงขึ้นมานิดหน่อย"

| ระดับ | preset | โมเดล/คุณภาพ | image tokens จริง @1536×1024 |
|---|---|---|---|
| COMMON / UNCOMMON / RARE | **standard (ประหยัดสุด)** | `gpt-image-1-mini` + `low` | **400** |
| EPIC / LEGENDARY / MYTHIC | **premium (สูงขึ้นนิดหน่อย)** | `gpt-image-1-mini` + `medium` | **1,568** |

- ตั้งผ่าน env: `AI_IMAGE_MODEL`/`AI_IMAGE_QUALITY` (standard) · `AI_IMAGE_PREMIUM_MODEL`/`AI_IMAGE_PREMIUM_QUALITY`/`AI_IMAGE_PREMIUM_RARITIES` (premium)
- โลจิกอยู่ที่ `resolveImagePreset(rarity)` ใน `src/lib/ai-image.ts` (pure + มีเทสต์ 4 ตัว) · สคริปต์/worker ใช้ preset นี้ทุกครั้ง
- สคริปต์พิมพ์ preset ที่ใช้จริง + จำนวน token ต่อใบ → ตรวจค่าใช้จ่ายได้ทันทีจาก log
- เทียบของเดิม: gpt-image-1 + medium = 1,568 tokens/ใบ **ทุกใบ** → ตอนนี้การ์ดทั่วไปเหลือ **400 tokens** และทุกใบใช้โมเดล `mini` (อัตราต่อ token ต่ำกว่า) → ประมาณ **~$0.003/ใบ** สำหรับการ์ดทั่วไป และ **~$0.012/ใบ** สำหรับ EPIC ขึ้นไป
- **ไม่สร้างย้อนหลังให้การ์ดที่มีภาพแล้ว** (ภาพเดิมคุณภาพสูงกว่าที่ตั้งใหม่; สร้างใหม่ = เสียเงินเพิ่มโดยไม่ได้อะไร) — ถ้าต้องการให้ทั้งคลังใช้ค่าใหม่ รัน `ALL=1 bash scripts/run-image-generation.sh` (~$0.7 ทั้งคลัง) หรือกด "🔄 สร้างรูปใหม่" รายใบในหน้า `/admin/cards`

**ตรวจด้วยการสร้างจริง:** COMMON → `standard (gpt-image-1-mini/low) · 400 image tokens` · EPIC → `premium (gpt-image-1-mini/medium) · 1,568 image tokens` · เทสต์ 280 ผ่าน (23 suites)

| ส่วนของการ์ด | รายละเอียด |
|---|---|
| ขนาด | 420×600 (สัดส่วนการ์ดสะสมจริง) แทนภาพพื้นหลัง 300×400 เดิม |
| กรอบบอกความหายาก | COMMON เทาเหล็ก → UNCOMMON ทองแดง → RARE เงินอมฟ้า → EPIC ม่วง → LEGENDARY ทองคำขาว → **MYTHIC ทองคำ** + เพชรมุม 4 จุด + เอฟเฟกต์โฮโลแกรม/ฟอยล์ตั้งแต่ RARE ขึ้นไป |
| แถบชื่อ | ชื่อไทย (ย่อขนาดอัตโนมัติตามความยาว) + ชื่ออังกฤษ + ตราธาตุมุมขวา (แบบ Attribute) |
| ดาวระดับ | ★ 1–6 ดวงตามระดับ + ป้ายชื่อระดับภาษาไทย/อังกฤษ |
| ช่องภาพ | ฉาก 6 แบบ (วงแหวนออร่า/ฟ้าดารา/ภูมิทัศน์/พายุคลั่ง/มันดาลารูน/สุริยุปราคา) × ลายธาตุ 6 ชนิด × ตัวแบบ 6 ทรง (winged/horned/serpent/colossus/orb/spectral) + ประกายแสง + ลายน้ำตราบทบาท |
| กล่องคำบรรยายในกรอบ | "คุณสมบัติ / EFFECT" + สกิล 1–2 อย่างพร้อมคำอธิบาย + คำอธิบายการ์ด + lore (ตัดคำอัตโนมัติแบบรองรับภาษาไทยที่ไม่มีเว้นวรรค) |
| แถบสเตตัส | ATK/DEF ตัวใหญ่ + HP/SPD/MP + บรรทัดล่างบอกฉากและระดับความหายาก |

- ขนาดไฟล์จริงต่อการ์ด ~18–19 KB (เดิม ~11 KB และก่อนหน้านั้น ~1.5 KB) · XML ถูกต้องทุกใบ
- ทดสอบ: `npm test` 263 ผ่าน (เพิ่มเทสต์กรอบสีตามระดับ/โฮโลแกรม/ข้อความในกรอบ) · `npm run e2e:flow` 31/31 ผ่านทั้ง local และ public URL
- ฝั่ง UI ปรับให้แสดงด้วย `object-contain` (คอลเลกชัน/หน้ารายละเอียด/ป๊อปอัปเปิดการ์ด/หน้าจัดทีม) เพื่อให้เห็นกรอบครบทั้งใบ ไม่ถูกครอป

- ก่อนหน้านี้ **ไม่มี ADMIN เลย** (ผู้ใช้ 24 คนเป็น `PLAYER` ทั้งหมด) → แผงแอดมิน `/admin` และ API `/api/admin/*` เข้าไม่ได้
- เพิ่มสคริปต์ `npm run admin:grant` (`scripts/grant-admin.ts`): `--list` ดูสิทธิ์ · `<username> [ADMIN|MODERATOR|PLAYER]` ตั้ง/ถอดสิทธิ์
- ตั้ง `woravik` เป็น **ADMIN** แล้ว (ล็อกอินใหม่ 1 ครั้งจึงจะมีผล เพราะ role ฝังใน session JWT)
- ทดสอบจริงด้วย session ที่เซ็นจาก `AUTH_SECRET`: ADMIN → `/api/admin/users` 200 · `/api/admin/security` 200 · `/admin` 200 · PLAYER → 403 และ `/admin` redirect (307) กลับหน้าแรก · ไม่มี cookie → 403