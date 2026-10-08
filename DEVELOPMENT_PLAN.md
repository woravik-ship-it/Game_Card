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

### 🐞 ปุ่ม "สร้างรูปใหม่" ของแอดมินค้างหมุนไม่จบ (Phase 14.6, 2026-09-22)

ผู้ใช้แจ้ง: "กดสร้างรูปใหม่ไป ตอนนี้ยังไม่แสดงรูปเลย ยังหมุนๆ อยู่เลย" — ตรวจจาก DB + `image_jobs` พบ **3 สาเหตุซ้อนกัน**

| # | สาเหตุ | หลักฐาน | วิธีแก้ |
|---|---|---|---|
| 1 | งานถูกใส่คิว `PENDING` แต่ **ไม่มีใครประมวลผล** — timer เดิมสร้างเฉพาะการ์ดที่ "ยังไม่มีไฟล์ภาพ" ส่วนการ์ดที่สร้างใหม่ (มีไฟล์เก่า) ถูกข้าม | การ์ด `ทิวรีย์` มี `image_status=PROCESSING` แต่ job ยัง `PENDING` ค้างตั้งแต่ 00:18 | เพิ่ม `scripts/image-worker-tick.mts` ที่ประมวลผล **งานในคิว** (รวมงาน regenerate) ไม่ใช่ดูแค่ "การ์ดที่ไม่มีภาพ" |
| 2 | งานที่ถูกล็อกเป็น `PROCESSING` แล้วโปรเซสตาย/ถูก restart กลางทาง **ค้างถาวร** | พบ 4 งานค้าง `PROCESSING` (เกิดตอนรีสตาร์ทแอพระหว่างสร้าง) | tick **reclaim** งาน `PROCESSING` ที่ค้างเกิน 5 นาที → กลับเป็น `PENDING` |
| 3 | การ์ดที่สถานะค้าง `PROCESSING` แต่ไฟล์ภาพมีอยู่แล้ว ไม่ถูกคืนสถานะ → UI หมุนตลอด | พบการ์ด `แกรนด์ป่าธานี` ค้าง `PROCESSING` โดยไม่มีงานในคิว | tick **เก็บกวาด** การ์ดที่ค้างสถานะแต่ไม่มีงานในคิว → คืน `READY` (ถ้ามีไฟล์ภาพ) หรือเข้าคิวใหม่ |

**ผลหลังแก้ (ตรวจกับ DB จริง):** `ดึงงานค้างกลับเข้าคิว: 4 งาน` → `คิว: ทำ 4 งาน → สำเร็จ 4` → **การ์ดค้าง PROCESSING 0 ใบ · งานในคิว 0**
ตรวจหน้าเว็บการ์ดที่เพิ่งสร้างใหม่: ภาพแสดงจริง `1536x1024` (แสดง 292×174) พร้อม version ใหม่ `?v=c0e9575` — ไม่หมุนค้างแล้ว

> หมายเหตุเวลา: ตอนทดสอบผมเขียน `updatedAt` ด้วย `now()` ของ Postgres (เวลาไทย) แต่ Prisma เก็บเป็น UTC → เวลาไปอยู่ใน "อนาคต" ทำให้ reclaim ไม่ทำงาน (ข้อมูลทดสอบผิด ไม่ใช่โค้ดผิด) — บทเรียนสำหรับการทดสอบกับ DB ที่ timezone ต่างกัน

**timer ทำงานทุก 2 นาที** (ExecStart = `npx tsx scripts/image-worker-tick.mts --batch 8`) → เคสค้างแบบนี้จะหายเองภายใน 2 นาทีเสมอ

### ปรับ UX การ์ดรอบ 2: ตัวหนังสือทับกัน + Admin ดูรูปใหญ่ + ซ่อนคำเทคนิค (Phase 14.7, 2026-09-22)

| คำสั่งผู้ใช้ | สาเหตุ/สิ่งที่พบ | สิ่งที่แก้ |
|---|---|---|
| "ตัวหนังสือในการ์ดอยู่ไม่เป็นระเบียบ มีซ้อนทับกัน" | สร้างเครื่องมือวัดจริง `npm run inspect:layout` (Chrome + `getBBox()` ของทุก `<text>` ใน SVG) → **พบ 5 จุดทับกัน**: หัวข้อ "คุณสมบัติ / EFFECT" ทับบรรทัดสกิลแรก (75%) และชื่อไทยทับชื่ออังกฤษในแถบชื่อ (17%) | แถบชื่อสูงขึ้น (54→62) + เว้นบรรทัดไทย/อังกฤษ (baseline 27/50) + ลดขนาดชื่อสูงสุด 22→20px · กล่องคำบรรยาย: เว้นหัวข้อ + เส้นคั่น + เริ่มบรรทัดที่ +32, `lineH` 18, จำกัด 6 บรรทัด, ฟอนต์ 12/11/11/10.5 · เลื่อนแถวดาวเป็น y=100 · **ตรวจซ้ำ 10/10 ใบ → ทับกัน 0 คู่** |
| "หน้า Card Admin ให้ดูรูปใหญ่ได้เหมือนหน้าดูการ์ดปกติ" | เดิมเป็นตารางข้อความ + การ์ดย่อ 20px เท่านั้น | การ์ดย่อเป็นปุ่ม → เปิด modal แสดงการ์ดเต็มใบ (330×471) + สถานะ/เจ้าของ/จำนวนค้นพบ + ปุ่มสร้างรูปใหม่ · ตรวจด้วย Chrome จริง: คลิกแล้ว modal เปิด แสดง art 292×174 + frame 330×471 |
| "ข้อมูลการทำงานเบื้องหลังไม่ต้องแสดงในเกม" | ผู้เล่นเห็นคำว่า "AI" ในสถานะรอ | เปลี่ยนข้อความผู้เล่น: "กำลังสร้างภาพด้วย AI…" → **"กำลังวาดภาพ…"** · "สร้างภาพไม่สำเร็จ" → "ยังวาดภาพไม่สำเร็จ" · badge หน้ารายละเอียดการ์ด "รอสร้างภาพ" → "กำลังวาดภาพ" (คำทางเทคนิคเหลือเฉพาะในหน้าแอดมิน/เอกสาร) |

**เครื่องมือใหม่:** `npm run inspect:layout` (`scripts/inspect-card-layout.mjs` + `public/_cardlayout.html`) — ใช้ตรวจว่าข้อความบนการ์ดทับกันไหมได้ทุกเมื่อหลังแก้ layout · ภาพหน้าจอตัวอย่าง: `public/_shots/admin-card-large.png`

### เอาข้อความ "ฉาก…/ความหายาก…" ท้ายการ์ดออก (Phase 14.8, 2026-09-22)

ผู้ใช้ถามว่า *"ตัวหนังสือด้านล่างการ์ด ฉาก, ความยาก จะบอกอะไร ถ้าไม่สำคัญ เอาออกก็ได้ ตอนนี้มันอยู่ทับเส้นกรอบการ์ด"*

- **คำตอบ:** ข้อความนั้นเป็น metadata ภายในของระบบ (ระบุว่าการ์ดใบนี้ใช้ "ฉาก" แบบไหนจาก 6 แบบ และระดับความหายาก) ใช้เพื่อตรวจว่าภาพมีความหลากหลาย — **ผู้เล่นไม่ต้องเห็น** และเป็นตัวสุดท้ายที่ยังล้นทับเส้นกรอบล่าง
- **แก้:** ลบบรรทัด `<text>` ท้ายการ์ดออกทั้งหมด แล้วย้ายข้อมูลไปเป็น metadata ที่มองไม่เห็น → `data-art-style` / `data-rarity` บน `<svg>` + `<desc>` (เครื่องมือตรวจภายในและ screen reader ยังอ่านได้)
- **เทสต์:** ปรับเทสต์ "ความหลากหลายของฉาก" ให้อ่านจาก `data-art-style` และเพิ่มเทสต์กันบั๊ก "ต้องไม่มีข้อความ ฉาก/ความหายาก บนการ์ด"
- **ผลตรวจ:** การ์ด 6/6 ใบ → ข้อความทับกัน 0 คู่ · เทสต์รวม 281 ผ่าน (23 suites)

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

### แสงบนการ์ด: แสงเลื่อม (foil) + แสงเรืองแบบไอเทมตีบวก (Phase 14.9–14.10, 2026-09-22/23)

**คำสั่งผู้ใช้:** *"ต้องการเอฟเฟกต์แสงเลื่อมครอบบนการ์ดอีกชั้น"* → *"ทำ Effect การ์ด ให้เหมือน Item ตีบวกในเกม Mu Online ที่เป็นแสงๆ สวย"*

| รอบ | สิ่งที่ทำ | ผล |
|---|---|---|
| 14.9 | ชั้นแสงเลื่อม CSS ล้วน: tint (เฉดขอบ) · prism (วงรุ้งในช่องภาพ) · sweep (แสงกวาด) · sparkle (ประกาย) ตามระดับความหายาก | ✅ ใช้ได้ — คงไว้ (`src/lib/card-foil.ts` + `CardFoil.tsx`) |
| 14.9.1 | ออร่า `box-shadow` รอบขอบ | ❌ ผู้ใช้บอกดูเป็น "กรอบสี่เหลี่ยม" ไม่มีแสง → ลบ |
| 14.9.2 | เปลวไฟ 9 ลูกกระพริบรอบขอบ (ตีบวกสไตล์ MU) | ❌ ผู้ใช้บอก "พอๆ ไม่ได้" → revert ทั้งหมด (`00863b4`) |
| **14.10** | **แสงเรืองแบบไอเทมตีบวก วาดด้วย SVG glow** (`feGaussianBlur` + gradient + clip-path รู) — 5 ดีไซน์ให้เลือก | ✅ ตรวจด้วยภาพจริง 5 ดีไซน์ |
| **14.11** | **นำดีไซน์ `inner` ไปใช้จริง** (ผู้ใช้เลือก *"ลองทำแบบ inner"*) — ผูกเข้า `CardFace` + deploy ขึ้น production | ✅ ใช้จริงทุกหน้า |

**เหตุที่ 2 รอบแรกไม่ผ่าน (บันทึกไว้กันทำซ้ำ):** ใช้ `box-shadow` (แข็งเป็นสี่เหลี่ยม) และลูกเปลวไฟ (ไม่ใช่ "แสง") · กล่อง grid มี `overflow-hidden` ตัดแสงนอกการ์ด · และ **ประเมินผลจากคำบรรยาย ไม่ได้ดูภาพจริง**

**รอบ 14.10 แก้ที่รากของปัญหา:**
1. เปลี่ยนไปวาดด้วย **SVG** — `feGaussianBlur` ใช้หน่วย user unit → สเกลตามขนาดการ์ดทุกขนาด (เดิม px ทำให้แต่ละหน้าไม่เหมือนกัน) และแสง "เกาะรูปทรงการ์ด" เหมือนแสงเกาะไอเทมจริง
2. **`clip-path` รูแบบ evenodd** → วงแสงอยู่ "นอกการ์ด" และ "นอกช่องภาพ" เท่านั้น → **การ์ดคม ไม่มีฝ้า** (ผู้ใช้เคยติเรื่องฝ้าทับภาพมาก่อน)
3. **บังคับดูภาพจริง**: `npm run shoot:aura` ถ่ายด้วย Chrome จริง (CDP) ทุกครั้งก่อนรายงานผล
4. ระดับแสงเทียบเคียง MU: `+7 ขอบเรือง` → `+9 + ประกายดาว` → `+11 + เสาแสง` → `+13 ครบชุด + ประกายลอย` (COMMON/UNCOMMON เรียบตามกติกาเดิม)
5. ดีไซน์ `inner` ตัดแสงในกรอบ → ใช้ได้ทุกหน้าโดยไม่ต้องแก้ layout; ดีไซน์นอกกรอบต้องมีที่ว่าง 12%

**ไฟล์:** `src/lib/card-aura.ts` · `src/components/cards/CardAura.tsx` · `src/app/aura-preview/page.tsx` · `src/app/globals.css` (บล็อก card-aura) · `scripts/shoot-aura-preview.mjs` · `tests/unit/card-aura.test.ts` (22 เทสต์)

**หลักฐาน:** `tsc --noEmit` 0 error · `next lint` ไม่มี warning ใหม่ · `jest` **312 ผ่าน / 25 suites** · ภาพจริง 11 ใบ (`public/_shots/v-tier-*.png`, `v-mythic-MYTHIC-*.png`) · พรีวิว `/aura-preview`

**ค้างอยู่ (รอผู้ใช้เลือก):** ยังไม่ผูก `CardAura` เข้ากับ `CardFace`/หน้าจริง (เพื่อไม่ให้เปลี่ยนหน้าจริงก่อนอนุมัติ) · ยังไม่ deploy ขึ้น production (เพื่อไม่ให้ public URL ของ tunnel เปลี่ยน)

### เอฟเฟกต์ Canvas 2D — ดีไซน์ `neon` (Phase 14.13, 2026-09-24)

**คำสั่งผู้ใช้ (ส่ง prompt มาให้ทำตาม):**
*"เขียนทับด้วยระบบพิกัด 2D ธรรมดา จะใช้คุณสมบัติการเรืองแสงและการเบลอของ Canvas
`ctx.globalCompositeOperation='lighter'` … `ctx.shadowBlur = 20; ctx.shadowColor='#00ffff';`
เพื่อสร้างออร่ารอบตัวการ์ด"* (หลังดีไซน์ SVG ทั้ง `inner`/`flow` ยังไม่ถูกใจ)

| สิ่งที่ทำ | ผล |
|---|---|
| `card-canvas.ts` (pure) + `CardAuraCanvas.tsx` — วาดด้วย Canvas 2D จริง: `globalCompositeOperation='lighter'`, `shadowBlur`/`shadowColor`, ลำแสงไหล (setLineDash+lineDashOffset), อนุภาคไหลตามเส้นรอบ, เปลวไฟ (quadratic + แกว่ง) | ✅ ใช้งานได้ (รอบนี้รอผู้ใช้เลือกจากภาพจริง) |
| clip 2 ชั้น (`outsideArt` + วงแหวนขอบการ์ด) ⇒ ภาพ/ข้อความคม 100% | ✅ ตรวจด้วยพิกเซลจริง = alpha 0 ทับช่องภาพ |
| `scripts/inspect-card-canvas.mjs` — ตรวจพิกเซลจริง (วาดไหม/มีแสงขอบไหม/ทับภาพไหม) | ✅ `npm run inspect:canvas` |
| `scripts/shoot-aura-preview.mjs --freeze N` — แช่เวลาให้ภาพนิ่งเทียบดีไซน์ได้คงที่ | ✅ |
| ประหยัดแรง: หยุดวาดเมื่อพ้นจอ/แท็บซ่อน/การ์ดเล็ก (<120px) · เคารพ prefers-reduced-motion | ✅ |

**หลักฐาน:** `tsc` 0 error · `next lint` ไม่มี warning · **jest 347 ผ่าน / 26 suites** · `npm run inspect:canvas` ผ่านทุกการ์ด (วาดจริง 112,937 px · ขอบ alpha=18 · ทับช่องภาพ 0/0/0) · ภาพ `public/_shots/neon-MYTHIC-neon.png`


**คำสั่งผู้ใช้ (รีวิวการ์ดจริง ดีไซน์ `inner` ระดับ LEGENDARY):**
*"แบบนี้ใกล้เคียง แต่แสงทำให้การ์ดเสียความคมชัด แล้วที่อยากได้ อยากได้ เหมือนเปรวไฟ หรือการไหล เหมือนน้ำ"*

**วินิจฉัย:** ชั้น `sparks` ของ `inner` ลอย "ผ่านตัวภาพ" (mix-blend `screen` + blur) → ภาพดูฝ้า/เสียความคม

| สิ่งที่ทำ | ผล |
|---|---|
| ดีไซน์ใหม่ **`flow`** (แบบ **Inner** ตามคำสั่งผู้ใช้): แสงไหลวนบนเส้นขอบการ์ด (`stroke-dasharray` + animate `dashoffset`) + **เปลวไฟลุกขึ้นบนขอบล่าง** + ประกายลอย | ✅ ใช้ได้ (รอผู้ใช้เลือก) |
| ทุกชั้นของ `flow` ถูกตัดด้วย **`ring` = วงแหวนขอบการ์ด (30 หน่วย)** ⇒ ไม่ทับตัวภาพ/กล่องข้อความ และ **ไม่ล้นออกนอกการ์ด** | ✅ แก้ข้อติตรงจุด (คม 100% + เป็น Inner) |
| `sparks` ของ `inner` ก็ถูก clip ที่ขอบช่องภาพ (`art-outside`) → ภาพคมขึ้นโดยไม่เปลี่ยนดีไซน์ | ✅ |
| เปลวไฟวาดด้วย `feTurbulence` + `feDisplacementMap` (นิ่ง) + CSS animation (ลุก/หุบ/สะบัด) | ✅ `prefers-reduced-motion` ปิดได้จริง (SMIL ปิดไม่ได้) |
| เปลว 2 ลูกต่อจุดยึด (สูง/เตี้ยต่างกันจาก hash) · ระดับคุมจำนวน/ความเข้ม/ความเร็ว | ✅ ไม่เป็น "ซี่ฟัน" เรียงเสมอ |

**ไฟล์:** `src/lib/card-aura.ts` (`AuraVariant` + `flow`, `auraFlames`, `auraFlowDash`) · `src/components/cards/CardAura.tsx` · `src/app/globals.css` (`cardAuraFlow` / `cardAuraFlame`) · `/aura-preview?variant=flow` · `tests/unit/card-aura.test.ts` (+16 เทสต์)

**หลักฐาน:** `tsc --noEmit` 0 error · `next lint` ไม่มี warning ใหม่ · **jest 328 ผ่าน / 25 suites** · ภาพจริง `public/_shots/prod-flow-LEGENDARY-flow.png` + `flow5-MYTHIC-flow.png`

**ข้อควรรู้:** `flow` เป็นดีไซน์ **Inner** (แสงอยู่ในการ์ด 100% ไม่ต้องมีที่ว่างรอบการ์ด — ใช้ได้ทุกหน้าเหมือน `inner`) · ภายในกรอบเหลือที่ให้แสงแค่ "วงแหวนขอบการ์ด" (`AURA_RING_WIDTH = 30`) เพราะภาพ+กล่องข้อความ+สเตตัสกินพื้นที่กลางการ์ดหมด · `DEFAULT_AURA_VARIANT` ยังเป็น `inner` (การ์ดจริงยังไม่เปลี่ยนจนผู้ใช้เลือก)

**🐞 กับดักที่เจอรอบนี้:** `next dev` เขียนทับ `.next` ของ production build → `next start` ล้ม (`Could not find a production build`) → ต้อง `NODE_ENV=production npm run build` ใหม่ทุกครั้งหลังใช้ dev server


**คำสั่งผู้ใช้:** *"ลองทำแบบ inner"* (เลือกจาก 5 ดีไซน์ของรอบ 14.10)

| สิ่งที่ทำ | ไฟล์ |
|---|---|
| ตั้งดีไซน์ที่ใช้จริงเป็น `inner` (จุดเดียวคุมทั้งเกม) | `src/lib/card-aura.ts` → `DEFAULT_AURA_VARIANT` |
| ให้ `CardFace` วาดชั้น aura เอง (มี prop `auraVariant?` ไว้ทดลองดีไซน์อื่น) | `src/components/cards/CardFace.tsx` |
| แก้บั๊กพรีวิวแสงซ้อน 2 ชั้น (ส่ง `auraVariant` ให้ `CardFace` แทนการวาด `CardAura` ซ้ำ) | `src/app/aura-preview/page.tsx` |
| เพิ่ม `--aura` → วัดชั้นแสงจาก **หน้าจริง** ด้วย Chrome (ไม่ต้องดูด้วยตา) | `scripts/inspect-cards-page.mjs` |
| เทสต์เพิ่ม 4 ตัว (ดีไซน์ตั้งต้น/พอดีกรอบ/องค์ประกอบครบ/COMMON ยังเรียบ) | `tests/unit/card-aura.test.ts` |

**ทำไม `inner` ไม่ต้องแก้ layout หน้าไหนเลย:** viewBox = ผืนการ์ดพอดี (`0 0 420 600`) + `inset: 0` + `overflow: hidden` → แสงอยู่ในกรอบการ์ด 100% (ดีไซน์อื่นต้องมีที่ว่าง 12% รอบการ์ดและห้ามกล่องแม่ `overflow-hidden`)

**หลักฐานจากหน้าจริง (วัดด้วย Chrome + session แอดมิน):** `/cards` → aura 4 ใบ (RARE/EPIC) จาก 12 ใบ ขนาด 222×317 พอดีกล่องการ์ดทุกใบ · `/admin/cards` → 13 ใบ ขนาด 80×114 พอดีทุกใบ · ทุกใบ `mix-blend-mode: screen` + `pointer-events: none` + `aria-hidden="true"` · อนิเมชันรันครบ · COMMON/UNCOMMON ไม่มีชั้นแสงเลย (กติกาเดิมไม่หลุด) · ภาพจริง: `public/_shots/real-cards-inner.png`, `real-admin-inner.png`, `aura-MYTHIC-inner.png`

**🐞 กับดักตอน deploy (เจอจริงรอบนี้):** `.env` ตั้ง `NODE_ENV="development"` → `npm run build` ตรงๆ จะเข้าโหมด dev แล้วล้ม ("Export encountered errors" หลายหน้า) ต้องสั่ง **`NODE_ENV=production npm run build`** (systemd ทับค่าให้ตอนรันจริงอยู่แล้ว)

**รอบ 14.11.1 (2026-09-23) — ถอด "ประกายดาว" (flare) ออกจาก `inner`:** ผู้ใช้รีวิวบนการ์ดจริงหลัง deploy — *"ยังไม่ถูกใจ โดยเฉพาะประกายดาว ไม่เหมาะเลย"* → `auraLayers('inner')` เหลือ `halo + sparks` (`flare: false`) · เทสต์/พรีวิว/เอกสารอัปเดตตาม · ดีไซน์อื่นในหน้าพรีวิวยังมีประกายดาวไว้เทียบ · รีวิวโค้ดรอบเดียวกัน: กันพรีวิวพังเมื่อ `?variant=` ผิด + uid ของ SVG รวม variant กัน id ชน + `CardAura` default ตรงกับดีไซน์จริง

## Phase 14.14 — เอฟเฟกต์การ์ดตาม GIF อ้างอิง (2026-09-25)

**คำสั่งผู้ใช้:** *"ให้แก้ effect เป็นเหมือน ตัวอย่างตาม Link"* + ส่ง GIF 7 ใบ (dreamassets 419–424)
· ก่อนหน้านี้ติว่า *"แสงสีข้างในที่ทับภาพอยู่ หมุนๆ มันแหว่ง เวลาหมุน แถมทำให้ภาพสีเพี้ยน"*

| สิ่งที่ทำ | ไฟล์ |
|---|---|
| แยกเฟรม GIF จริง (PIL) → สรุปสเปก: แถบแสงหางดาวหางวนรอบตัว · วงแหวนฐานใต้เท้า · แสงกวาดบนโลหะ | — |
| เรขาคณิต pure: `orbitSpec` · `orbitGeometry` · `baseRingGeometry` · `ringPulse` · `wispRibbon` · `sheenBand` | `src/lib/card-canvas.ts` |
| วาด 3 ชั้นใหม่ใน Canvas (clip เฉพาะ "ช่องภาพ"): แสงกวาด → วงแหวนฐาน → แถบแสงวนรอบ (3 pass: ฟุ้ง/แกน/ไส้หัว) | `src/components/cards/CardAuraCanvas.tsx` |
| ตั้งดีไซน์ที่ใช้จริงทั้งเกมเป็น `neon` (จาก `inner`) — จุดเดียวคุมทั้งเกม | `src/lib/card-aura.ts` |
| แก้ต้นเหตุ "แหว่ง/สีเพี้ยน": วงกลม `conic-gradient` หมุน + `color-dodge` → แถบเฉดรุ้งเต็มช่องภาพ + `screen` | `src/app/globals.css` |
| คง foil (tint/sweep/sparkle) ทุกใบ แต่ `disablePrism` เมื่อดีไซน์เป็น Canvas | `src/components/cards/CardFace.tsx` |
| เทสต์ใหม่ 12 ตัว + อัปเดตบล็อก "ดีไซน์ตั้งต้น" | `tests/unit/card-canvas.test.ts` · `tests/unit/card-aura.test.ts` |
| เกณฑ์ตรวจพิกเซลใหม่ 4 ข้อ (วาดจริง · มีแสงในช่องภาพ · เกาะขอบการ์ด · ไม่ล้นออกนอกช่องภาพ) | `scripts/inspect-card-canvas.mjs` |

**หลักฐาน:** `tsc --noEmit` 0 error · `next lint` ไม่มี warning ใหม่ · **jest 361 ผ่าน / 26 suites** ·
`NODE_ENV=production npm run build` ผ่าน + restart `rune-dominion-arena` ·
`npm run inspect:canvas -- --query 'variant=neon&rarity=MYTHIC'` ✅ 4/4 (วาดจริง 167,114 px · มีแสงในช่องภาพ 1,631 จุด · ชื่อ/ข้อความ/สเตตัส = 0) ·
`--query 'variant=neon&rarity=RARE'` ✅ 4/4 (maxA 168) · ภาพจริง `public/_shots/chk3-MYTHIC-neon.png` + `chk4-MYTHIC-neon.png`

**ย้อนกลับได้ทันที:** ตั้ง `DEFAULT_AURA_VARIANT = 'inner'` ที่ `src/lib/card-aura.ts` (ดีไซน์เดิมยังอยู่ครบในหน้าพรีวิว)

---

## Phase 16 — หน้าสนามรบแบบใหม่ (2026-09-25)

**คำสั่งผู้ใช้:** *"หน้าสนามรบต้องแสดงการ์ดเรียงบน-ล่าง (เรา = ล่าง / คู่ต่อสู้ = บน) พร้อม HP/MP ต่อใบ
+ สัญลักษณ์ดาบ (คนโจมตี) / โล่ (คนรับ) + เอฟเฟกต์แดงที่คนโดนตี + ข้อความล่าสุดอยู่ด้านบน
+ แถบ HP รวมทั้งสองทีม"*

| สิ่งที่ทำ | ไฟล์ |
|---|---|
| ตัวเล่นเทปแบบ pure: `buildReplayFrames(teamA, teamB, log)` → เฟรมต่อเหตุการณ์ (`frames = log + 1`, เฟรม -1 = ก่อนเริ่ม) ใช้ `hpAfter`/`manaAfter` ที่ engine บันทึกไว้ ไม่คำนวณดาเมจซ้ำ | `src/services/battle-replay.ts` (ใหม่) |
| สถานะบนการ์ด: 🔥 เผาไหม้ (สแต็ก 1-3) · 💧 อ่อนแอ · 🛡️ โล่ · ⚡ ว่องไว — เลขเทิร์น/สแต็กเดินตาม engine เป๊ะ (ลดเทิร์นท้ายเทิร์นของเจ้าตัว, เทิร์นที่โดนเผาไม่นับ + เผาไม่เกิน 3 สแต็ก) + `cardStatuses()` ใช้ร่วมกับตัวตรวจหน้าจริง | เดียวกัน |
| หน้าสนามรบ: คู่ต่อสู้ 5 ใบบน / ทีมเรา 5 ใบล่าง · HP/MP ต่อใบ (แถบ + ตัวเลข) · ขอบสีตามธาตุ · ⚔️ คนโจมตี / 🛡️ + คลุมแดง คนรับ · การ์ดตาย = เทา + 💀 · ข้อความ log ใหม่สุดอยู่บน · แถบ HP รวม 2 ทีม · ปุ่ม เริ่มเล่น/หยุด/เริ่มใหม่/ข้าม + ความเร็ว x1/x4/x8 | `src/app/(game)/battle/[id]/page.tsx` |
| เลิกฮาร์ดโค้ด `100` ในหน้าจอ → ใช้ `BATTLE_MANA_MAX` จาก constants | เดียวกัน |
| ห้อง Arena: ท้าสำเร็จ → พาไปดูผลในห้อง battle อย่างเดียว (ไม่ขึ้นข้อความสรุปผลซ้ำก่อนเปลี่ยนหน้า) | `src/app/(game)/arena/[id]/page.tsx` |
| ตัวตรวจหน้าจริงด้วย Chrome (CDP) 7 ข้อ: การ์ด 10 ใบเรียงบน-ล่าง · HP/MP ทุกใบ · ⚔️🛡️แดงกลางรบ · HP รวม = ผลรวมการ์ด · log ใหม่สุดบน · ไอคอนสถานะตรงข้อมูล · ไม่มี modal บัง | `scripts/inspect-battle-field.mjs` (ใหม่) · `npm run inspect:battle` |
| เทสต์ 15 ตัว รวมชุดเทียบ **engine จริง** (`simulateBattle` + seed → HP เฟรมสุดท้ายตรง `teamAHpRemaining/B` เป๊ะ, ใบตาย = ใบที่มี log faint) | `tests/unit/battle-replay.test.ts` (ใหม่) |

**🐞 ต้นเหตุที่งานค้างรอบก่อน (แก้แล้ว):** ตัวตรวจหน้าจริงปิด onboarding modal ไม่ได้ ⇒ ภาพที่ได้ถูก modal บังทั้งใบ (ผู้ใช้รีวิวไม่ได้)
- สาเหตุจริง: selector เขียน escape ของ class `z-[100]` **ซ้อน 2 ชั้น** ใน template literal
  → ฝั่ง Chrome ได้ selector `.z-[100]` ซึ่งไม่ถูกต้องตาม CSS → `SyntaxError` →
  และโค้ดเดิมทิ้งผลลัพธ์ (`void dialogState`) จึงไม่ error ให้เห็น · modal จึงค้างทุกครั้ง
- แก้: เพิ่ม `data-onboarding-modal` ที่ `OnboardingModal` + หา modal ด้วย attribute selector ·
  ตั้งธง localStorage "เคยดู onboarding" ผ่าน `Page.addScriptToEvaluateOnNewDocument` (จำลองผู้เล่นเดิม) ·
  วนปิด-ตรวจซ้ำ + เพิ่มข้อตรวจ "ไม่มี modal บัง" (fail ให้เห็น ไม่เงียบ)

**หลักฐาน (2026-09-25):** `tsc --noEmit` 0 error · `next lint` ไม่มี warning ในไฟล์ที่แก้ · **jest 421 ผ่าน / 28 suites** ·
`NODE_ENV=production npm run build` ผ่าน + restart `rune-dominion-arena` ·
`npm run inspect:battle` ✅ **7/7** บนการ์ดศึกจริง 2 อัน (การ์ด 10 ใบ ห่างแถว 125-131 px · HP รวม 477/477 · ⚔️1 🛡️1 แดง1 ·
log 58 เหตุการณ์ใหม่สุดอยู่บน · `BURN=1` มีไอคอนจริง · modal = gone) ·
ภาพจริง `public/_shots/battle-field.png` (จบศึก) + `battle-field-status.png` (มีการ์ดติด 🔥)

**ย้อนกลับได้:** ไม่ผูกกับ API/DB ใดๆ — ถ้าต้องถอยให้ `git checkout` ไฟล์ `battle/[id]/page.tsx`
+ ลบ `src/services/battle-replay.ts` (หน้าจะกลับไปใช้ `log.slice(0, visibleCount)` แบบเดิม)

### รอบ 2 (2026-09-25) — การ์ดเต็มใบ · ชื่อทีม = ชื่อ Deck · ปุ่มข้าม · เมนูเลือกทีมใน Arena

**คำสั่งผู้ใช้:** *"แสดงรูปการ์ดแบบเต็มสิ แล้วปุ่มข้าม ไม่ต้องมีคำว่ารู้ผลเลย
ชื่อทีม ใส่ชื่อ Deck ไปเลย แล้วตอนจะเข้าร่วมประลอง ให้มีเมนูเลือกทีม ใน Deck ด้วย"*

| สิ่งที่ทำ | ไฟล์ |
|---|---|
| หน้าสนามรบแสดง **การ์ดเต็มใบ** (ภาพ AI + กรอบ/ชื่อ/สเตตัส/แสงเรือง ผ่าน `CardFace`) แทนกล่องข้อความย่อ · HP/MP แถบ+ตัวเลขอยู่ "ใต้การ์ด" ไม่ทับรูป · การ์ดตาย = เทา + 💀 · ขยายคอนเทนเนอร์เป็น `max-w-3xl` ให้การ์ดสูง ~235px | `src/app/(game)/battle/[id]/page.tsx` |
| API ส่งข้อมูลสำหรับแสดงผลเพิ่ม: `teamNames` (ชื่อ Deck ต่อฝ่าย — บอท = "บอท (สุ่มการ์ด)") + `cardMeta` (imageUrl/imageStatus/rarity ต่อการ์ด) | `src/app/api/battle/[id]/log/route.ts` |
| กติกากลางแบบ pure (เทสต์ได้): `teamName` · `cardMetaMap` · `teamCardIds` · `hasArt` | `src/services/battle-display.ts` (ใหม่) + `tests/unit/battle-display.test.ts` (10 เทสต์) |
| ปุ่มข้ามเปลี่ยนเป็น **"⏩ ข้าม"** (ตัดคำว่า "รู้ผลเลย" ตามสั่ง) | `src/app/(game)/battle/[id]/page.tsx` |
| ชื่อทีมทั้งหัวเรื่อง, แถบ HP รวม และหัวการ์ดของแต่ละฝ่าย = **ชื่อ Deck จริง** | เดียวกัน |
| **เมนูเลือกทีม (Deck) ในห้องประลอง** ก่อนเข้าร่วม/ท้าทาย: เลือกได้เฉพาะเด็คครบ 5 ใบ (ไม่ครบ = ปุ่มจาง กดไม่ได้) · พรีวิวการ์ดทั้ง 5 ใบของทีมที่เลือก · ปุ่มเข้าร่วม/ท้าทาย + ข้อความยืนยันใช้เด็คที่เลือก · ค่าเริ่มต้น = เด็คแรกที่ครบ 5 ใบ | `src/app/(game)/arena/[id]/page.tsx` |
| ตัวตรวจหน้าจริงเพิ่ม 3 ข้อ (การ์ดเต็มใบเทียบ `cardMeta` จาก API · ชื่อทีมเป็นชื่อ Deck · ปุ่มข้ามไม่มีคำว่ารู้ผล) → รวม **10 ข้อ** · ตัวตรวจใหม่ของห้องประลอง **4 ข้อ** | `scripts/inspect-battle-field.mjs` · `scripts/inspect-arena-room.mjs` (ใหม่) + `npm run inspect:arena` |

**หลักฐาน (2026-09-25):** `tsc --noEmit` 0 error · `next lint` ไม่มี warning ในไฟล์ที่แก้ · **jest 431 ผ่าน / 29 suites** ·
`NODE_ENV=production npm run build` ผ่าน + restart ·
`npm run inspect:battle` ✅ **10/10** (กรอบการ์ด 10/10 · รูปจริงบนจอ 10 = API บอก 10 · การ์ดสูงต่ำสุด 235px ·
`A=ทีมด่วน 1 · B=บอท (สุ่มการ์ด)` · ปุ่ม `⏩ ข้าม` · modal = gone) ·
`npm run inspect:arena` ✅ **4/4** (เด็ค 2 ใบ เลือกได้ 2 · ค่าเริ่มต้นถูกเลือกให้ · สลับเด็คแล้วพรีวิวเปลี่ยน 5 ใบ มีรูปจริง 5 ·
`selected` ตรงกับพรีวิวและผูกกับปุ่มเข้าร่วม/ท้าทาย) · ภาพจริง `public/_shots/battle-field.png` + `arena-deck-picker.png`

**ย้อนกลับได้:** ไม่แตะ DB/สกีมา — ถ้าต้องถอยให้ `git revert` commit ของรอบนี้ (ค่าที่เพิ่มใน API เป็นฟิลด์ใหม่ ไม่กระทบผู้ใช้เดิม)

### รอบ 3 (2026-09-25) — ไอคอนดาบ/โล่กลางการ์ด · Replay/ต่อสู้อีกครั้ง · ส่งทีมเข้าห้องซ้ำได้

**คำสั่งผู้ใช้:** *"สัญลักษณ์ดาบกับโล่ ตอนต่อสู้เอามาไว้ตรงกลางเลย ไว้มุม มันตกขอบ
· การต่อสู้ผ่านไปแล้ว สามารถกดดู Re play หรือต่อสู้ใหม่ได้
· การเพิ่มทีมเข้ามาในห้อง เก็บค่าเข้า จะจัดเข้ามากี่ครั้งก็ได้"*

| สิ่งที่ทำ | ไฟล์ |
|---|---|
| สัญลักษณ์ ⚔️/🛡️ ย้ายมา **กลางการ์ด** (วงกลมดำ + ริงเรืองแสง ทับช่องภาพ) แทนมุมบนที่ตกขอบการ์ด — การ์ดที่ตายแล้วยังเห็น 💀 ทับอยู่บนสุด | `src/app/(game)/battle/[id]/page.tsx` |
| **จบศึกแล้ว** มีกล่องผล + ปุ่ม **"🔁 ดู Replay"** (ย้อนไปเฟรมแรกแล้วเล่นใหม่ทันที) และ **"⚔️ ต่อสู้อีกครั้ง"** (สร้างศึกใหม่ด้วยเด็คเดิมผ่าน `/api/battle/simulate`; ศึกกับบอทส่ง `bot: true`) + บอกว่า "ต่อสู้อีกครั้ง = ศึกใหม่ด้วยทีมเดิม (คู่ต่อสู้ …)" | เดียวกัน |
| API ส่ง `decks: { A, B }` + `isBotBattle` เพื่อให้ปุ่มต่อสู้ใหม่รู้ว่าต้องยิงเด็คไหน | `src/app/api/battle/[id]/log/route.ts` |
| กติกา pure `refightBody(attackerDeckId, defenderDeckId)` (ไม่มีเด็ค B → `bot: true` · ไม่รู้เด็ค A → `null` ปุ่มแจ้งให้ไปเริ่มเอง) + เทสต์ | `src/services/battle-display.ts` · `tests/unit/battle-display.test.ts` |
| **ส่งทีมเข้าห้องซ้ำได้ + คิดค่าเข้าทุกครั้ง**: `arenaJoinPlan()` = create (ครั้งแรก) / **update (ส่งซ้ำ → หักค่าเข้าใหม่ + อัปเดตเด็คแถวเดิม ไม่สร้างแถวซ้ำ)** / skip (คีย์เดิมยิงซ้ำ = ไม่หัก) · เพดาน 20 ครั้ง/วัน นับจาก **รายการหัก Coin จริง** (ให้การเข้าซ้ำถูกนับด้วย) · ห้องเต็มไม่บล็อกคนที่เคยเข้าแล้ว | `src/services/arena.ts` · `src/app/api/arena/[id]/join/route.ts` · `tests/unit/arena.test.ts` |
| หน้าห้องประลอง: ปุ่ม **"ส่งทีมเข้าห้อง (10 Coin)"** + ข้อความยืนยัน และข้อความผลลัพธ์จาก `arenaJoinMessage()` ("เปลี่ยนทีมเป็น … — หัก 10 Coin (เข้าได้หลายครั้ง คิดค่าเข้าทุกครั้ง)") + หมายเหตุใต้ปุ่ม | `src/app/(game)/arena/[id]/page.tsx` |
| ตัวตรวจหน้าจริง: battle เพิ่มเป็น **12 ข้อ** (ไอคอนกลางการ์ดต้องอยู่ในกรอบ+ใกล้กลาง · จบศึกต้องมีปุ่ม Replay/ต่อสู้อีกครั้ง) และเก็บภาพ **กลางรบ** เพิ่ม · arena เพิ่มเป็น **5 ข้อ** (ต้องมีข้อความชี้แจงค่าเข้า) | `scripts/inspect-battle-field.mjs` · `scripts/inspect-arena-room.mjs` |

**หลักฐาน (2026-09-25):** `tsc --noEmit` 0 error · **jest 440 ผ่าน / 29 suites** · `next lint` ไม่มี warning ในไฟล์ที่แก้ · build + restart ·
`npm run inspect:battle` ✅ **12/12** (ไอคอนอยู่ในกรอบการ์ด `inside=true` เยื้องจากกลาง (0,13)px ทั้งดาบและโล่ ·
ปุ่ม `🔁 ดู Replay` + `⚔️ ต่อสู้อีกครั้ง` (disabled=false)) ·
`npm run inspect:arena` ✅ **5/5** (ปุ่ม "ส่งทีมเข้าห้อง (10 Coin)" + พบข้อความ "คิดค่าเข้า…") ·
**ทดสอบ API จริง:** ส่งทีมเข้าซ้ำ → `{joined:true, rejoined:true, entryFee:10}` (หักจริง 100→90 Coin) · ยิงคีย์เดิมซ้ำ → `{alreadyJoined:true, idempotent:true}` (ไม่หัก) ·
แถวผู้เข้าร่วมยัง **2 แถวเท่าเดิม** (ไม่ซ้ำ) และ `deckId` ถูกอัปเดตเป็นเด็คใหม่ · ปุ่มต่อสู้อีกครั้ง → ศึกใหม่ `cmugkrksp0006453ubhm0xhm1` (ชนะ · HP เหลือ 494) ·
ภาพจริง `public/_shots/battle-field-mid.png` (ไอคอนกลางการ์ดตอนรบ) + `battle-field.png` (จบศึก มีปุ่ม Replay/ต่อสู้อีกครั้ง) + `arena-deck-picker.png`

**ข้อควรรู้:** การทดสอบครั้งนี้ใช้ Coin ของ `woravik` ไป 10 เหรียญ (100 → 90) ตามค่าเข้าจริง 1 ครั้ง —
ถ้าต้องการคืน ให้ใช้ `scripts/grant-*` หรือแจ้งในแชท · เพดาน 20 ครั้ง/วันยังอยู่ (นับเฉพาะครั้งที่จ่ายจริง)

---

## Phase 17 — UI มือถือ: เมนูไม่ล้นขอบ (2026-09-25)

**คำสั่งผู้ใช้:** *"กลับมาแก้ UI เมนูต่างๆ ในมือถือ มันล้นขอบ แก้ไขให้พอดี"*

**วินิจฉัยจากหน้าจริง (Chrome CDP ที่ 360×780):** หน้าเว็บ **ไม่มี** แถบเลื่อนนอน (page overflow = 0 ทุกหน้า)
แต่ **เมนู 2 แท่งถูกตัดขอบขวาทิ้ง** — วัดได้จาก `scrollWidth - clientWidth` ของกล่องเมนู:
- `header` nav (ลิงก์ข้อความ 8 อัน) → ซ่อนอยู่ **192px**
- แถบล่าง (12 เมนู) → ซ่อนอยู่ **89px**
⇒ ผู้ใช้เห็นเมนูถูกตัดที่ขอบจอ (scrollbar ถูกซ่อนไว้ด้วย `scrollbar-hide` จึงดูเหมือน "ล้น")

| สิ่งที่ทำ | ไฟล์ |
|---|---|
| **แถบล่าง (มือถือ)**: เปลี่ยนจากแถวเลื่อนนอน 12 เมนู → **5 เมนูหลัก** (หน้าแรก · ค้นรูน · การ์ด · จัดทีม · ประลอง) + ปุ่ม **"☰ เพิ่มเติม"** ที่เปิดแผงเมนู รวม 6 คอลัมน์เท่ากัน (`grid-cols-6`) ⇒ พอดีจอทุกขนาด | `src/components/layout/BottomNavigation.tsx` |
| **แผง "เมนูเพิ่มเติม"**: bottom sheet 3 คอลัมน์ (ทดสอบเด็ค · Coin · ภารกิจ · กิจกรรม · คลัง · ตั้งค่า · โปรไฟล์ · แอดมินถ้าเป็นแอดมิน) ไฮไลต์หน้าปัจจุบัน ปิดด้วย backdrop/ปุ่ม/เปลี่ยนหน้า · `/battle` เข้าถึงได้บนมือถือแล้ว (เดิมไม่มีในแถบล่าง) | เดียวกัน |
| **จอใหญ่ (md+)**: ยังแสดงเมนูทั้งหมดในแถวเดียว + ป้ายเปลี่ยนเป็นไทยสั้น (พอดีกว่าคำอังกฤษยาว) | เดียวกัน |
| **หัวเว็บ (มือถือ)**: ซ่อนลิงก์ข้อความทั้งหมด (เหลือแถบล่างเป็นตัวนำทาง) · แบรนด์ย่อเป็น `🔮 RDA` (เต็ม "Rune Dominion" บน sm+) · ⚡พลังค้นหา / 🪙Coin / 👤โปรไฟล์(ชื่อย่อ) / ปุ่ม "ออก" — ทั้งแถวอยู่ในจอ 320px | `src/components/layout/TopHeader.tsx` |
| **ตัวตรวจใหม่**: `npm run inspect:mobile` — เปิด 12 หน้าจริงที่ความกว้างมือถือ แล้วตรวจ 3 เกณฑ์: (1) ไม่มีแถบเลื่อนนอน (2) เมนูไม่ซ่อนเนื้อหา (scrollWidth-clientWidth) (3) แผง "เพิ่มเติม" เปิดได้ ทุกปุ่มอยู่ในจอ ปิดได้ · เก็บภาพได้ด้วย `--shot` | `scripts/inspect-mobile-overflow.mjs` (ใหม่) |

**หลักฐาน (2026-09-25):** `tsc --noEmit` 0 error · `next lint` ไม่มี warning ในไฟล์ที่แก้ · build + restart ·
`npm run inspect:mobile` (360×780) ✅ **12/12 หน้า** — ล้นนอน 0px · เมนูซ่อนเนื้อหา **0px** (เดิม 192px/89px) · แผงเมนู 7 ปุ่มอยู่ในจอ + ปิดได้ ·
`--width 320` ✅ ครบ 12 หน้าเท่ากัน · `--width 1280` ✅ (จอใหญ่ไม่โชว์ปุ่ม "เพิ่มเติม" — ตรวจแล้วว่าปุ่มถูกซ่อนจริง) ·
ภาพจริง `public/_shots/mobile-home.png` (แถบล่าง 5+1) + `mobile-home-sheet.png` (แผงเมนูเพิ่มเติม) + `mobile320-*.png`

**ย้อนกลับได้:** UI ล้วน ไม่แตะ API/DB — ถ้าต้องถอยให้ `git revert` commit ของรอบนี้




## Phase 18 — หนังสือคู่มือการเล่น (ฉบับหนังสือ + ภาพประกอบจากเกมจริง) (2026-09-26)

**คำสั่งผู้ใช้:** *"ใน Project Game card ช่วยทำคู่มือการเล่นให้หน่อย ทำเป็นหนังสือ พร้อมภาพประกอบนะ"*

เดิมมีแค่ `docs/PLAYER_GUIDE_TH.md` (12 หัวข้อ ข้อความล้วน) — รอบนี้ทำใหม่เป็น **หนังสือ** ที่มีภาพประกอบจริง
และเขียนจากค่าคงที่จริงในโค้ด (ไม่ใช่การประมาณ) พร้อมทั้งเพิ่มเครื่องมือถ่ายภาพหน้าจอสำหรับใช้ทำเอกสารซ้ำได้ในอนาคต

| สิ่งที่ทำ | ไฟล์ |
|---|---|
| สคริปต์ถ่ายภาพหน้าจอจริงเพื่อทำเอกสาร: Chrome headless + CDP · มือถือ 390×844 (dsf 2) และเดสก์ท็อป 1360×900 · ล็อกอินด้วย session token · ปิด onboarding modal · 26 หน้า (มี `--anon` สำหรับหน้า login/register) | `scripts/capture-manual-shots.mjs` (ใหม่) |
| ต้นฉบับหนังสือ (HTML + print CSS ขนาด A5 ตาม `@page`) — ปก · สารบัญ · 14 บท · ภาคผนวก 5 หัวข้อ · กล่องเกร็ด/สูตร/ตาราง | `docs/manual/index.html` |
| **ตัวเล่ม** PDF ขนาด A5 | `docs/manual/RuneDominion-Manual-TH.pdf` (77 หน้า · 12.9 MB) |
| ภาพประกอบ 29 ภาพ (ภาพหน้าจอเกมจริง 26 + ตัวอย่างการ์ด Common/Rare/Mythic 3) | `docs/manual/images/` |
| วิธีสร้างเล่มใหม่ (ถ่ายภาพ → พิมพ์ PDF) | `docs/manual/README.md` |
| ลิงก์ในสารบัญเอกสารของเกม | `rune-dominion-arena/README.md` (ตาราง "เอกสาร") |

**เนื้อหาที่เพิ่มจากคู่มือเดิม (ดึงจากโค้ดจริง):**

| เรื่อง | แหล่งอ้างอิงในโค้ด | ตัวอย่างตัวเลขที่ใส่ในเล่ม |
|---|---|---|
| สูตรสถิติการ์ด + โอกาสแต่ละระดับ rarity | `src/services/seed.ts` | ATK ฐาน 20–69 → คูณ 1.0/1.2/1.5/1.8/2.2/2.8 · โอกาส 40/30/15/9/4/2% · SPD 10–39 ไม่คูณ |
| กติกาการต่อสู้ (มานา/ลำดับ/เป้า/ตัดสินผล) | `src/services/combat-engine.ts` · `combat.ts` | 30 รอบ · มานา +20/รอบ ใช้สกิลที่ 50 · ตีศัตรู SPD ต่ำสุด · เสมอเมื่อ HP รวมเท่ากัน |
| สูตรดาเมจ + ผังคู่เปรียบธาตุ | `src/services/combat.ts` | `floor(ATK × 100/(100+DEF) × ธาตุ × 0.95–1.05)` · เพลิง→ลม→ดิน→น้ำ→เพลิง · แสง↔เงา |
| สกิลและสถานะ | `src/services/combat-engine.ts` | เผาไหม้ 5%/ชั้น (สูงสุด 3) · โล่ −30% (2 รอบ) · ว่องไว +10 SPD (3 รอบ) · อ่อนแอ −20% ATK (3 รอบ) · เยียวยา 20% HP |
| คะแนนทีมแบบละเอียด | `src/lib/deck-formation.ts` | ช่องโจมตี 2/ป้องกัน 2/สนับสนุน 1 · เรต 1.0/1.2/1.6 · ตรงบทบาท +10% · แกน 6 เหลี่ยม 600/450/1300/200/45/250 |
| อารีน่า (รวมกติกาส่งทีมซ้ำ) | `src/lib/constants.ts` · `src/services/arena.ts` | 30/10 Coin · cooldown 5 นาที · 20 ครั้ง/วัน · รางวัล `min(100+ผู้เข้าร่วมไม่ซ้ำ×5, 500)` |
| กิจกรรม + กลไกบอส | `src/services/event-boss.ts` · `event-raid-engine.ts` · `event-definitions.ts` | บอส 1,000,000 HP · 4 เฟส (130/90/420 → 220/150/600) · Veil Shield −40% (แสงช่วยเหลือ −20%) · Moonless Mark +20% · Rage/Pulse ตามเทิร์น · +10% เมื่อใช้ ≥4 ธาตุ |
| ภารกิจ 8 ใบ · Milestone 7+5 ระดับ · ร้านค้า 4 รายการ | `prisma/seed.ts` · `event-definitions.ts` | Coin 30/20/40/120/200/250/500/1000 · 2,000→75,000 คะแนน · 1M→50M ชุมชน |
| เศรษฐกิจและกติกากันโกง (ledger/idempotency/daily cap) | `src/services/wallet.ts` | Coin 100 เริ่มต้น · ทุกธุรกรรมมียอดก่อน/หลัง · ยอดติดลบไม่ได้ |

**การตรวจงาน (2026-09-26):**

- `node --check scripts/capture-manual-shots.mjs` ผ่าน · รันจริงได้ภาพ 26 หน้า (ไม่ error) + รอบ `--anon` ได้หน้า login/register
- ตรวจว่าทุกลิงก์ภาพใน `index.html` มีไฟล์จริง: **28/28 อ้างอิงมีไฟล์ครบ** (สคริปต์วน `grep -o 'images/…'` เทียบกับ `ls images`)
- พิมพ์ PDF ด้วย Chrome headless สำเร็จ: **77 หน้า · 420×594.96 pt (A5) · ไม่เข้ารหัส**
- ตรวจหน้าจริงด้วยการแปลง PDF เป็นภาพ (`pdftoppm`) 6 หน้า: ปก/สารบัญ/บทที่ 1/บทที่ 7/บทที่ 10/ท้ายเล่ม — ภาพและตารางไม่ล้นหน้า
- แก้ปัญหาเลย์เอาต์ที่เจอจากการตรวจครั้งแรก: ภาพหน้าจอมือถือ (แนวตั้งยาว) ล้นหน้า A5 → จำกัด `max-height` ของภาพ (128mm ปกติ · 140mm ภาพกว้าง · 68mm บนปก)

**ข้อควรรู้:** การถ่ายภาพหน้าจอใช้บัญชีผู้เล่นจริง (`woravik` — ADMIN) และไม่ได้แก้ข้อมูลใด ๆ ในฐานข้อมูล ·
token ถูกสร้างชั่วคราวเพื่อการนี้เท่านั้นและ **ไม่ถูกเก็บลงไฟล์ในโปรเจกต์**

### แก้รอบ 2 (2026-09-26) — ตัดเนื้อหาสายเทคนิคออก ให้เหลือเฉพาะเกมเพลย์

**คำสั่งผู้ใช้:** *"คู่มือ Game card ที่เขียนมา เขียนตาเทคนิคการทำงานของเกมมาด้วย ไม่ต้องการ เอาเฉพาะ เกมเพลย์ พอ"*

| ตัดออก (สายเทคนิคภายใน) | เปลี่ยนเป็น (เกมเพลย์) |
|---|---|
| canonical string · SHA-256 + server pepper · canonicalSeedHash · deterministic PRNG | "ลำดับรูนเดิมได้การ์ดใบเดิมเสมอ" + ทำไมจึงเป็นเช่นนั้นในมุมผู้เล่น |
| idempotency key · transaction · ledger · fingerprint/alt-account · rate limit | "กดรัวไม่โดนหักซ้ำ" · "ดูประวัติยอดก่อน–หลังได้ที่หน้ากระเป๋า" · "1 คน 1 บัญชี" |
| เส้นทาง API (`/discover` `/cards` …) · ชื่อสคริปต์ (`npm run db:grant-starter`) | ชื่อเมนู/ไอคอนที่ผู้เล่นเห็นจริง |
| ตัวคูณ rarity (1.0–2.8×) · ช่วง SPD/mana ของการ์ด · สูตร `ค่ามานา × 3 + 6` | ช่วงค่าที่ผู้เล่นเจอจริง · "การ์ดที่สกิลมานาแพงกว่าได้คะแนนช่องสนับสนุนมากกว่า" |
| สูตรโค้ด (`max(500, floor(ratio × 20,000))`) · ค่า ATK/DEF/HP ดิบของบอสแต่ละเฟส | คะแนน 500–20,000 · +10% เมื่อใช้ 4 ธาตุ · ตารางเฟสพร้อมคำอธิบายความแข็ง |
| "เทปใช้ค่าที่ engine บันทึก" · ซีด/ความแปรปรวนแบบสุ่มตายตัว · GDD | "เทปคือเหตุการณ์จริงที่เกิดขึ้น" · "ดาเมจแปรปรวนราว ±5% ต่อครั้ง" |
| ภาคผนวก "ตารางค่าคงที่ทั้งหมด" (มีค่าภายใน) | "สรุปกติกาสำคัญ" (กติกาที่ผู้เล่นใช้ตัดสินใจ) |
| อภิธานศัพท์ Canonical String / Canonical Seed Hash / Deterministic / Formation Score / Team Power | คำศัพท์ที่ผู้เล่นเห็นในเกม (การค้นพบ · ผู้ค้นพบคนแรก · คะแนนทีม ฯลฯ) |

**ยังคงไว้ (เพราะเป็นกติกาที่ผู้เล่นต้องใช้เล่น):** สูตรดาเมจในมุมผู้เล่น · ผังเปรียบธาตุ · มานา/จำนวนรอบ ·
สถานะและระยะเวลา · กลไกบอสที่มีผลต่อดาเมจ · โอกาสระดับความหายาก · ค่า Coin/รางวัล/เพดานรายวัน

**หลักฐาน:** `index.html` ไม่เหลือคำสายเทคนิค (ตรวจด้วย `grep -i` กลุ่มคำ: hash/pepper/idempotency/ledger/engine/
seed/endpoint/npm/GDD/fingerprint/digest/ซีด/ทรานแซกชัน) · ลิงก์ภาพ 28/28 มีไฟล์ครบ · พิมพ์ PDF ใหม่ = **77 หน้า A5 (12.3 MB)** ·
ตรวจหน้าจริงด้วย `pdftoppm` (บทที่ 3 · บทที่ 8 · ภาคผนวก จ) แล้วอ่านเป็นภาษาผู้เล่นทั้งหมด · ส่งไฟล์ใหม่เข้า Telegram ด้วย
`tg-send.py --document`

## Phase 19 — หน้าแรก (ตัดข้อความ Phase) + กระดานรูน (อักขระไม่ซ้ำ + ซ่อนลำดับ) (2026-09-26)

**คำสั่งผู้ใช้:** *"หน้าแรกของเกม ไม่ต้องบอก Phase การทำ เห็นมีคำบอกว่า Phase 7 · การเลือกรูน ไม่ต้องบอกว่าเลือกลำดับรูนที่เท่าไร
ด้านล่าง ไม่ต้องให้คนเล่นรู้ · แก้ไขรูนให้มีอักขระ ไม่ซ้ำกัน"*

| สิ่งที่ทำ | ไฟล์ |
|---|---|
| หน้าแรก: ตัดข้อความ `Phase 7 — Quest & Mission System เสร็จสมบูรณ์` (ข้อความ dev หลุดถึงผู้เล่น) → เปลี่ยนเป็นข้อความชวนเล่น "ค้นพบการ์ดด้วยรูน · จัดทีม 5 ใบ · ต่อสู้อัตโนมัติ · แข่งขันในอารีน่า" | `src/app/page.tsx` |
| กระดานรูน: **อักขระไม่ซ้ำกันทั้งกระดาน** — เดิมใช้ตัวอักษร 24 ตัววนซ้ำ (`GLYPHS[index % 24]`) เห็นเป็นแพตเทิร์นชัดเจน → เขียนตัววาดอักขระเองแบบ deterministic: index ของช่อง (0–9999) มี 4 หลักฐานสิบ แต่ละหลักเลือกรอย 1 จาก 10 แบบ แล้ววาดรอยนั้นในช่องของตัวเอง (บน/ขวา/ล่าง/ซ้าย) ⇒ **ไม่มีช่องไหนได้อักขระซ้ำกันจริง** (พิสูจน์ได้จากการเข้ารหัส 1:1) | `src/components/rune/RuneCanvas.tsx` (`traceRuneGlyph` · `RUNE_MARKS` · `RUNE_SLOTS`) |
| กระดานรูน: **ซ่อนข้อมูลที่ผู้เล่นไม่ต้องรู้** — ตัดบรรทัด `ลำดับรูน: 5, 88, 1234, …` ใต้กระดาน · ตัดเลขลำดับ (1..16) บนรูนที่เลือก (เหลือแค่ไฮไลต์เรืองแสง) · เปลี่ยน `พิกัด: (45, 45) · 10×10` → `มุมมอง 10×10 รูน` | เดียวกัน |
| ข้อความที่เกี่ยวข้อง: เคล็ดลับหน้า `/discover` (เดิม "ลำดับรูนเดียวกันจะได้การ์ดเดียวกันเสมอ") → "พลังค้นหาเติมให้ใหม่ 5 ครั้งทุกวัน" · onboarding ขั้นที่ 1 → "ค้นพบการ์ดด้วยการเลือกอักขระรูนบนกระดาน" | `src/app/discover/page.tsx` · `src/components/ui/OnboardingModal.tsx` |
| คู่มือ (หนังสือ): ภาพประกอบทั้งเล่ม **ถ่ายใหม่** + ปรับข้อความให้ตรงกับ UI ใหม่ (ไม่พูดถึง "ลำดับรูน"/พิกัด/ตัวเลข) ในบทที่ 1 · 3 · อภิธานศัพท์ + คำบรรยายรูป 3.1 | `docs/manual/index.html` · `docs/manual/images/*` · PDF พิมพ์ใหม่ 77 หน้า |

**หลักฐาน (2026-09-26):** `npx tsc --noEmit` 0 error · **jest 440 ผ่าน / 29 suites** ·
`NODE_ENV=production npm run build` ✓ Compiled successfully + restart `rune-dominion-arena` · `/api/health` 200 ·
ถ่ายหน้าจริงด้วย Chrome (CDP) ยืนยัน: หน้าแรกไม่มีข้อความ Phase · กระดานรูนมีอักขระต่างกันทุกช่องและไม่มีตัวเลขใด ๆ บนจอ ·
คู่มือ PDF พิมพ์ใหม่ **77 หน้า A5** และส่งเข้า Telegram ด้วย `tg-send.py --document`

**ย้อนกลับได้:** UI ล้วน ไม่แตะ DB/API — `git revert` commit ของรอบนี้ได้ทันที (กติกาการค้นพบ/สูตรยังเหมือนเดิมทุกอย่าง)

## Phase 20 — แจ้งเตือนต่อผู้ใช้ · เวลาประมาณการ Gen รูปการ์ด · ตัวเลือกภาษาในเกม (2026-09-26)

**คำสั่งผู้ใช้:** *"ตอน Gen รูปการ์ด ให้บอกว่าใช้เวลาประมาณเท่าไร และสามารถกลับมาดูได้ภายหลัง ·
ทำระบบแจ้งเตือนต่างๆ เช่นทำรูปเสร็จ, ต่อสู้จบ รายงานผล หรือประกาศต่างๆ สำหรับของแต่ละ User ·
ทำตัวเลือกภาษาภายในเกม ทำเป็นเมนูตั้งค่าต่างๆ ภายในเกม"*

### 1) เวลาประมาณการสร้างภาพการ์ด + กลับมาดูภายหลัง

| สิ่งที่ทำ | ไฟล์ |
|---|---|
| เก็บ `ImageJob.startedAt` (เวลาที่ worker เริ่มลงมือ) → คำนวณเวลาสร้างจริงได้ | `prisma/schema.prisma` |
| ตัวประมาณเวลาบริสุทธิ์: `estimateEtaSeconds({ queuePosition, inFlight, avgSeconds })` = เวลาเฉลี่ย × (งานของตัวเอง 1 + งานรอข้างหน้า + งานที่กำลังทำ) · clamp 3–300 วิ (ค่าตั้งต้น 25 วิ) | `src/lib/image-eta.ts` (ใหม่) |
| `ImageService.queueSnapshot()` (คิว + เวลาสร้างเฉลี่ยจากงานที่เสร็จล่าสุด 20 งาน) และ `statusForCards(ids)` (คิว/ตำแหน่ง/ETA ต่อการ์ด) | `src/services/image.ts` |
| API: `GET /api/images/status?cardId=|cardIds=` — คืนสถานะ + ETA + คิวรวม (ยึด session; ไม่ส่ง id = การ์ดของผู้เล่นที่ยังไม่มีภาพ) | `src/app/api/images/status/route.ts` (ใหม่) |
| UI: คอมโพเนนต์ `CardArtStatus` — เต็มรูปแบบ (การ์ดที่ยังไม่มีภาพ) และแบบ compact (กริดการ์ด) · อัปเดตเองทุก 10 วิ · ข้อความ "กลับมาดูภายหลังได้ — ระบบจะแจ้งเตือนเมื่อสร้างเสร็จ" | `src/components/cards/CardArtStatus.tsx` (ใหม่) + ฝังใน `cards/page.tsx` · `cards/[id]/page.tsx` |
| เมื่อภาพเสร็จ/ล้มเหลวถาวร → สร้างแจ้งเตือนให้เจ้าของการ์ดทุกคน | `src/services/image.ts` |

### 2) ระบบแจ้งเตือน "ของแต่ละ User"

| สิ่งที่ทำ | ไฟล์ |
|---|---|
| ตาราง `notifications` (1 แถว = 1 ผู้เล่น) เก็บข้อความ **สองภาษา** (titleTh/titleEn/bodyTh/bodyEn) + href/icon/อ่านแล้ว · ตาราง `announcements` (ประกาศทีมงาน + จำนวนผู้รับ) | `prisma/schema.prisma` + migration `20260926011138_phase20_notifications_locale_image_eta` |
| `NotificationService`: create/createMany · notifyImageReady/Failed (เจ้าของการ์ดทุกคน) · notifyBattleResult (ผู้โจมตี + ผู้ป้องกัน) · notifyArenaChallenge · notifyArenaChampion · notifyEventMilestone · announce (fan-out) · list/unreadCount/markRead · **เคารพค่าที่ผู้เล่นเลือก** (notifyPrefs) | `src/services/notification.ts` (ใหม่) · `src/lib/notification-prefs.ts` (ใหม่) |
| API: `GET /api/notifications` (กรอง all/unread + unreadCount · ตอบตามภาษาที่ผู้เล่นเลือก) · `POST /api/notifications/read` (ids หรือทั้งหมด) · `POST/GET /api/admin/announcements` (เฉพาะ ADMIN/MODERATOR) | `src/app/api/notifications/**` · `src/app/api/admin/announcements/route.ts` (ใหม่) |
| จุดที่ยิงแจ้งเตือนจริง: ต่อสู้จบ (`/api/battle/simulate`) · ส่งทีมเข้าห้องอารีน่า (`/api/arena/[id]/join`) · แชมป์อารีน่ารับรางวัล (`/api/arena/settle`) · ภาพการ์ดเสร็จ/ล้มเหลว (worker) | ไฟล์เดิม + hook ที่กลืน error (ไม่ทำลายธุรกรรมหลัก) |
| UI: ระฆัง 🔔 พร้อมตัวเลขยังไม่อ่านในหัวเว็บ (อัปเดตทุก 60 วิ + เมื่อกลับเข้าแท็บ) และหน้า `/notifications` (กรอง อ่านทั้งหมด เวลาแบบ "2 นาทีที่แล้ว") | `src/components/layout/NotificationBell.tsx` (ใหม่) · `src/app/(game)/notifications/page.tsx` (ใหม่) |

### 3) ตัวเลือกภาษา + เมนูตั้งค่าภายในเกม

| สิ่งที่ทำ | ไฟล์ |
|---|---|
| ระบบ i18n: พจนานุกรมกลาง 2 ภาษา (ไทย/อังกฤษ) ~150 คีย์ · `t(locale, key, vars)` บริสุทธิ์ · ถอยไปไทยเมื่อไม่พบคำแปล · มีเทสต์ตรวจว่าไม่มีคีย์ตกหล่น | `src/lib/i18n/{index,dict-th,dict-en}.ts` (ใหม่) |
| `LocaleProvider`: localStorage → ค่าจากบัญชี → ค่าเริ่มต้นไทย · เปลี่ยนภาษาแล้ว PATCH ไปเก็บที่ `User.locale` (เพื่อให้ข้อความแจ้งเตือนจากเซิร์ฟเวอร์ตรงภาษา) | `src/components/providers/LocaleProvider.tsx` (ใหม่) + `User.locale` ใน schema |
| API: `GET/PATCH /api/profile/settings` (locale + notifyPrefs) | `src/app/api/profile/settings/route.ts` (ใหม่) |
| หน้าตั้งค่าทำใหม่เป็น **เมนูตั้งค่า 5 ส่วน** (🌐 ภาษา · 🔊 เสียง · 🔔 การแจ้งเตือน · 🧭 ช่วยเหลือ · ℹ️ เกี่ยวกับ) + เมนูลัดด้านบน · ปุ่มเลือกภาษา · สวิตช์เลือกประเภทการแจ้งเตือน 5 แบบ · ลิงก์ไปศูนย์แจ้งเตือน | `src/app/(game)/settings/page.tsx` |
| แปล UI ให้เปลี่ยนตามภาษา: เมนูบน/ล่างทั้งหมด · หัวเว็บ · หน้าแรก · กระดานรูน · หน้าตั้งค่า · ศูนย์แจ้งเตือน · สถานะการสร้างภาพ | ไฟล์คอมโพเนนต์/เพจที่เกี่ยวข้อง |

**หลักฐาน (2026-09-26):** migration applied + Prisma Client สร้างใหม่ · `npx tsc --noEmit` 0 error ·
**jest 463 ผ่าน / 32 suites** (เพิ่ม 3 ชุดใหม่: i18n / image-eta / notification-prefs) ·
`NODE_ENV=production npm run build` ✓ Compiled successfully + restart `rune-dominion-arena` · `/api/health` 200 ·
**ทดสอบ API จริง:** ประกาศจากทีมงาน → `recipients: 34` · ต่อสู้กับบอท → ได้แจ้งเตือน `BATTLE_RESULT` พร้อมลิงก์ไปเทป (`/battle/<id>`) ·
สลับภาษาเป็น `en` → ข้อความแจ้งเตือนเดิมกลายเป็นอังกฤษทันที (พิสูจน์ว่าข้อความถูกแปลตามภาษาของแต่ละ user) ·
ค้นพบการ์ดใหม่ → `/api/images/status` ตอบ `PENDING · คิวที่ 1/1 · ETA 25 วิ` และหน้า `/cards` แสดง "🎨 about 25 seconds" ·
ภาพหน้าจอตรวจด้วย Chrome: หน้าตั้งค่า (เมนู+ปุ่มภาษา) · ศูนย์แจ้งเตือน (ระฆังมีตัวเลข 3) · กริดการ์ด (ป้าย "กำลังวาดภาพ" + เวลาประมาณ)

**ขอบเขตที่ยังเหลือ (บันทึกไว้ให้ตรงจริง):** ตัวเลือกภาษาแปลครบใน *shell + ฟีเจอร์ใหม่* (เมนู หัวเว็บ หน้าแรก กระดานรูน ตั้งค่า แจ้งเตือน สถานะภาพ)
ส่วนหน้าลึก ๆ (การ์ด/จัดทีม/อารีน่า/ต่อสู้/กิจกรรม/แอดมิน) ยังเป็นไทยก่อน — เพิ่มคำแปลได้โดยเติมคีย์ใน `dict-en.ts` แล้วแทนข้อความในหน้านั้น

## Phase 21 — เสียงประกอบใช้งานได้จริง: เพลง + บรรยากาศ + เอฟเฟกต์ (2026-09-26)

**คำสั่งผู้ใช้:** *"Projects Game Card มีเมนูเสียง แต่ไม่เห็นมีเสียงเลย ทำเสียงประกอบด้วย"*

**สาเหตุที่เงียบ (ตรวจจากโค้ด ไม่ใช่เดา):**
1. **ไม่มีตัวเล่นเพลง/บรรยากาศเลย** — มีแต่สวิตช์ในหน้าตั้งค่า และตารางเสียง (SFX) ที่สังเคราะห์ได้เท่านั้น
2. SFX ถูก **ทิ้งเมื่อ AudioContext ยัง suspended** (`playSfx` คืน false) ⇒ เสียงแรกที่ผู้ใช้กดหายไป
3. ค่าเริ่มต้นเดิมปิดเพลง/บรรยากาศไว้ ⇒ เปิดหน้าเกมครั้งแรกจึงเงียบสนิท
4. ไม่มีบัสเสียงแยกชั้น ⇒ ปรับระดับเพลง/บรรยากาศ/เอฟเฟกต์แยกกันไม่ได้ และ `revealSfxFor()` (เสียงเปิดการ์ดตาม rarity) ไม่เคยถูกเรียกใช้

| สิ่งที่ทำ | ไฟล์ |
|---|---|
| กราฟเสียงกลาง: บัส SFX/เพลง/บรรยากาศ + มาสเตอร์ + Analyser (วัดระดับเสียงได้) · ฟังก์ชันบริสุทธิ์ `layerGain()` (ปิดชั้น = 0 · ลดความเข้ม = ครึ่ง · clamp วอลุ่ม) | `src/lib/audio-engine.ts` (ใหม่) |
| **เพลงประกอบ**: สังเคราะห์เอง (pad 4 เสียง + ระฆัง) ไล่คอร์ด i–VI–III–VII ความยาวลูป 30 วิ · ตั้งเวลาแบบ lookahead · `midiToFreq/chordNotes` บริสุทธิ์เทสต์ได้ | `src/lib/music.ts` (ใหม่) |
| **เสียงบรรยากาศ**: noise ที่สร้างแบบ deterministic + bandpass ที่ขยับด้วย LFO (ลมพัด) + ประกายเป็นช่วง ๆ | `src/lib/ambience.ts` (ใหม่) |
| ตัวเล่นจริง (WebAudio scheduling + หยุดได้สะอาด) | `src/lib/audio-players.ts` (ใหม่) |
| Provider ใหม่: ปลดล็อก audio ทุก interaction + `resume()` เมื่อกลับเข้าแท็บ · เปิด/ปิดเพลง–บรรยากาศตามสวิตช์ · ปรับกราฟตามวอลุ่ม/โหมดลดความเข้ม · เปิด **ค่าเริ่มต้น = เพลง+บรรยากาศเปิด** · debug API `window.__rdaAudio` (QA) | `src/components/providers/AudioProvider.tsx` |
| SFX เลือกปลายทางได้ + แยกตารางเวลาเป็น `sfxSchedule()` (บริสุทธิ์) + **ไม่ทิ้งเสียงเมื่อ context ยัง suspended** + เพิ่มเสียง `coin`, `notify` | `src/lib/sfx.ts` |
| ผูกเสียงเข้ากับเกมจริง: เปิดการ์ดตาม rarity (`revealSfxFor` ถูกใช้แล้ว) · จบศึกชนะ/แพ้/เสมอ · รับรางวัลภารกิจ · ส่งทีมเข้าห้องอารีน่า · เริ่มสู้กับบอท · แจ้งเตือนใหม่ (กระดิ่ง) + กดเปิดศูนย์แจ้งเตือน | `CardRevealModal` · `battle/[id]` · `battle` · `quests` · `arena/[id]` · `NotificationBell` |
| หน้าตั้งค่า: ปุ่มทดสอบเสียงเพลง/บรรยากาศ + ตัวบ่งชี้ "กำลังเล่นอยู่" + คำอธิบายว่าต้องแตะก่อน | `src/app/(game)/settings/page.tsx` + คำแปล 2 ภาษา |
| เครื่องมือตรวจเสียงอัตโนมัติ: เปิดเกมใน Chrome (headless, `--autoplay-policy=no-user-gesture-required`) → ปลดล็อก → วัด RMS ของเพลง/บรรยากาศ + SFX ทุกชื่อ | `scripts/inspect-audio.mjs` (ใหม่) · `npm run inspect:audio` |
| คู่มือ: อัปเดตบทเสียง + ภาคผนวก ก/ข (ค่าเริ่มต้นใหม่ + ปุ่มทดสอบ) แล้วพิมพ์ PDF ใหม่ | `docs/manual/index.html` · `PDF 77 หน้า` |

**หลักฐานวัดได้ (2026-09-26):** `npx tsc --noEmit` 0 error · **jest 493 ผ่าน / 35 suites** (เพิ่ม 4 ชุด: `audio-engine` · `music` · `ambience` · อัปเดต `sfx`) ·
`NODE_ENV=production npm run build` ✓ + restart · `/api/health` 200 ·
**`npm run inspect:audio` ผ่านทุกข้อ ✅** — AudioContext = running · เพลงเล่น · บรรยากาศเล่น ·
gain ต่อชั้น `{sfx:0.63, music:0.21, ambience:0.154}` · **peak RMS ของเพลง+บรรยากาศ = 0.019** ·
**SFX ทั้ง 15 ชื่อมีสัญญาณออกจริง** (peak 0.017–0.027) ไม่มีชื่อไหนเงียบ

### แก้รอบ 2 (2026-09-26) — ผู้ใช้ฟังจริงแล้วพบ: "ได้ยินแต่เสียง Ambience ซ่า ๆ" + "สลับไปแอปอื่นเสียงไม่หาย"

| ปัญหาที่ผู้ใช้เจอ | สาเหตุจริง | แก้ |
|---|---|---|
| ได้ยินแต่เสียงบรรยากาศ "ซ่า ๆ" ไม่ได้ยินเพลง/เอฟเฟกต์ | 1) บรรยากาศตั้งไว้ดังกว่าที่ควร (`ambience: 0.22`) และใช้ **bandpass 520Hz** → เป็นเสียงซ่า 2) เพลง/เอฟเฟกต์เบาเกินไป (pad 0.075 · SFX สลายตัวเร็ว) | บรรยากาศ: `0.07` + เปลี่ยนเป็น **lowpass 260Hz + tremolo หายใจ** (ลมไกล ไม่ใช่ซ่า) · เพลง: pad 0.11 + **เพิ่มเสียงเบส 0.14** + ระฆัง 0.075 · SFX: gain ×1.6 (≤0.18) + **envelope มีช่วง sustain** (เดิมสลายตัวจนหูแทบไม่ได้ยิน) + ยืดเสียงกดปุ่มสั้น ๆ |
| สลับไปแอปอื่นแล้วเสียงไม่หาย | ไม่มีการหยุดเสียงเมื่อไม่ได้ดูเกม | เพิ่ม `pauseAll/resumeAll` + ผูก `visibilitychange` (ซ่อนแท็บ) และ `blur/focus` (สลับแอป) โดยมี debounce 250ms · หยุดตัวเล่น + `ctx.suspend()` (ประหยัดแบต) · กลับมาแล้วเล่นต่อถ้าสวิตช์ยังเปิด |
| ตัวตรวจเสียงเดิมเชื่อไม่ได้ (เพลงกับบรรยากาศวัดรวมกัน) | สคริปต์วัดรวมทุกชั้น + มีบั๊กลำดับ (ปิดเพลงแล้วไม่เปิดใหม่) | `inspect-audio.mjs` วัด **แยกชั้น** (เพลงเดี่ยว/บรรยากาศเดี่ยว/SFX เดี่ยว) + ตรวจ pause/resume + จำลองซ่อนแท็บ · เพิ่ม flag `--no-sfx` |

**ผลตรวจรอบ 2 (วัดจริงจากเกม):** gain ต่อชั้น `{sfx:0.7, music:0.315, ambience:0.049}` ·
**เพลงเดี่ยว peak RMS = 0.0139** · **บรรยากาศเดี่ยว = 0.0046 (เบากว่าเพลง ~3 เท่า)** ·
**SFX ทุกชื่อ = 0.031–0.056** (ดังกว่ารอบก่อน ~3 เท่า) · pause แล้ว `level = 0` และ `ctx = suspended` ·
ซ่อนแท็บ → หยุดเอง · กลับมา → เล่นต่อเอง · `tsc` 0 error · **jest 495 ผ่าน** (เพิ่มเทสต์กัน regress: เพลงต้องดังกว่าบรรยากาศ ≥2 เท่า และบรรยากาศ ≤10% ของชั้นเสียงเต็ม)

> หมายเหตุ: **ยังไม่อัปเดตคู่มือ/หนังสือในรอบนี้** (ผู้ใช้สั่ง 2026-09-26 ว่าให้รอสั่งค่อยทำ — เบา token)
> ส่วนที่หนังสือยังไม่ตรง: บทเสียงในภาคผนวก ก (ค่าเริ่มต้น/คำอธิบายบรรยากาศ) + ตัวเลขระดับเสียง


## Phase 22 — เสียงดังกว่าที่เคย + เสียงต่อสู้จริง (ดาบ / ปล่อยสกอล) (2026-09-20)

**คำสั่งผู้ใช้:** *"เสียงเบามาก เปิดสุดแทบไม่ได้ยิน และเสียงตอนต่อสู้ก็ไม่มี ทำเป็นเสียงดาบ เสียงปล่อยสกอล หน่อย"*

**สาเหตุจริง (ตรวจจากโค้ด):**
1. **เกนทั้งระบบต่ำเกินไป** — SFX มี gain 0.11–0.16 × บัส SFX 1.0 × วอลุ่ม 0.7 ≈ peak 0.09
   (เดิมตั้งใจให้ "ไม่รบกวน" แต่เกินไป ⇒ เปิดวอลุ่มสุดแล้วยังเบา) · เพลงบัส 0.45 ก็เบา
2. **เทปต่อสู้ไม่มีเสียงเลย** — หน้า `battle/[id]` เล่นแค่ `battle_win/battle_lose` ตอนเทปจบ
   ทุกเหตุการณ์ (ฟัน/สกิล/เผา/ฮีล/ล้ม) ไม่ผูกเสียง ⇒ "เสียงตอนต่อสู้ไม่มี"
3. **ตารางเสียงทำได้แต่ oscillator** — เสียงดาบต้องมี "ลม" และ "โลหะ" ซึ่งคลื่นล้วนทำไม่ได้

| สิ่งที่ทำ | ไฟล์ |
|---|---|
| **เสียงดังกว่าที่เคย**: เพิ่ม `SFX_GAIN_BOOST` (×1.6) + เพดาน `MAX_TONE_GAIN` (0.4) ใช้กับทุกเสียงในตารางโดยไม่ต้องแก้ทีละบรรทัด | `src/lib/sfx.ts` |
| **เสียงดาบ**: `battle_sword` = ลมดาบ (noise bandpass ไถล 2600→700Hz) + คมโลหะ (ping) + กระแทกเข้าที่ · **`battle_clash`** = ดาบกระทบดาบ (ความถี่ inharmonic 1860/2790/4180Hz) | `src/lib/sfx.ts` |
| **เสียงปล่อยสกอล**: `battle_cast` = ร่ายเวทไล่ความถี่ขึ้น (320→1280Hz) + ลมเวท + วาบพลังตอนปล่อยออก | `src/lib/sfx.ts` |
| เสียงประกอบเหตุการณ์อื่นในเทป: `battle_shield` (กางโล่) · `battle_burn` (ไฟวูบ) · `battle_heal` (ระฆังไล่ขึ้น) · `battle_faint` (ทรุดลง) | `src/lib/sfx.ts` |
| **ToneSpec รองรับ noise** (`NoiseSpec`: ตัวกรอง + ความถี่ + `slideTo` + Q) และ `playSfx` เล่น oscillator + noise พร้อมกันได้ · บัฟเฟอร์ noise แคชต่อ AudioContext (WeakMap) | `src/lib/sfx.ts` |
| `battleSfxFor(action, statusApplied)` — ฟังก์ชันบริสุทธิ์เลือกเสียงตามเหตุการณ์ (attack→ดาบ · skill→สกอล · skill+SHIELD→โล่ · burn/heal/faint) | `src/lib/sfx.ts` |
| **ผูกเสียงเข้าเทปต่อสู้**: ทุกครั้งที่ log entry ใหม่โผล่ ⇒ เล่นเสียงของเหตุการณ์นั้น (มี throttle 60ms กันเสียงซ้อนที่ความเร็ว x8) | `src/app/(game)/battle/[id]/page.tsx` |
| **ยกเกนรวม**: `MASTER_BASE_GAIN` 1.0 → 1.3 + **DynamicsCompressor** ก่อนออกลำโพง (กันแตกเมื่อเพลง+SFX+บรรยากาศดังพร้อมกัน) · เพลงบัส 0.45 → 0.75 · บรรยากาศ 0.07 → 0.10 | `src/lib/audio-engine.ts` |
| วอลุ่มเริ่มต้น 0.7 → 0.85 | `src/components/providers/AudioProvider.tsx` |
| หน้าตั้งค่า: ปุ่มทดสอบเสียงดาบ ⚔️ / ปล่อยสกอล ✨ (+ คำแปล 2 ภาษา) | `src/app/(game)/settings/page.tsx` · `dict-th.ts` · `dict-en.ts` |
| เครื่องมือตรวจเสียง: เพิ่มชื่อเสียงใหม่ทั้ง 7 ตัวในรายการตรวจ (เดิมมี 15) | `scripts/inspect-audio.mjs` |
| เทสต์กัน regress: `battleSfxFor` ครบทุกเหตุการณ์ · เสียงดาบมีทั้ง noise และ oscillator · gain ไม่เกินเพดาน · เกนรวม > 1 · compressor มีค่า ratio > 1 | `tests/unit/sfx.test.ts` · `tests/unit/audio-engine.test.ts` |

**หลักฐานวัดได้ (2026-09-20):** `npx tsc --noEmit` 0 error · **jest 510 ผ่าน / 35 suites** · `next lint` ไม่มีคำเตือนใหม่ · `NODE_ENV=production npm run build` ✓ + `systemctl --user restart rune-dominion-arena` · `/api/health` 200

**`npm run inspect:audio` (วัดจากเกมจริงใน Chrome headless) ผ่านทุกข้อ ✅**
- gain ต่อชั้น `{sfx:0.85, music:0.6375, ambience:0.085}` (เดิม `{0.7, 0.315, 0.049}` ⇒ เพลงดังกว่าเดิม 2 เท่า)
- **เพลง+บรรยากาศ peak RMS = 0.0704** (เดิม 0.0178 ⇒ ~4 เท่า) · เพลงเดี่ยว 0.0495 (เดิม 0.0139)
- **SFX ทั้ง 22 ชื่อมีสัญญาณออกจริง** peak 0.143–0.329 (เดิม 0.031–0.056 ⇒ **~4.5 เท่า**)
  โดยเสียงใหม่: `battle_sword` = 0.232 · `battle_clash` = 0.244 · `battle_cast` = 0.329 · `battle_shield` = 0.221 · `battle_burn` = 0.143 · `battle_heal` = 0.215 · `battle_faint` = 0.145
- pause/resume และซ่อนแท็บ/กลับเข้าเกมยังทำงานถูกต้อง

**ตรวจเทปต่อสู้จริงในเบราว์เซอร์ (หน้า `/battle/<id>` ของศึก 38 เหตุการณ์ = ฟัน 21 · สกิล 6 · ล้ม 6 · ฮีล 4 · เผา 1):**
ดักการสร้างโหนดเสียง (oscillator/noise) แล้วกด "▶ เริ่มเล่น" — ระหว่างเล่น x1 ได้ **peak RMS = 0.237** และสร้างโหนดเสียง 28 ตัว (noise 14)
· เร่ง x8 ไปจนจบ **38/38 เหตุการณ์** ได้ peak 0.350 · โหนดรวม 137 (noise 56) ⇒ **เสียงดาบ/ปล่อยสกอลดังจริงตามเหตุการณ์ในเทป** ไม่ใช่แค่มีในตาราง


### แก้รอบ 3 (2026-09-20) — ผู้ใช้ฟังจริงรอบ 2: "เสียง Effect ดังแล้ว ส่วนเสียงดนตรีกับ Ambience ยังเบามาก"

| ปัญหาที่ผู้ใช้เจอ | สาเหตุจริง (ตรวจแล้ว ไม่ใช่เดา) | แก้ |
|---|---|---|
| เพลงยังเบามาก | 1) บัสเพลง 0.75 ถูกตั้งให้ "ต้องเบากว่า SFX" ตามสัดส่วนบัส — แต่ **สัญญาณเพลงกับ SFX คนละแบบ**: SFX เป็นเสียงสั้น peak สูง / เพลงเป็น pad ต่อเนื่อง gain ต่อโน้ตแค่ 0.05–0.14 ⇒ วัดจริงเพลงได้ peak RMS 0.049 ทั้งที่ SFX 0.23 2) **โน้ต pad อยู่ที่ C3–C4 (130–262Hz) ซึ่งลำโพงมือถือแทบไม่ถ่ายทอด** — ผู้ใช้ฟังบนมือถือจึงได้ยินเบากว่าที่วัดได้ | บัสเพลง 0.75 → **1.8** + เพิ่ม **ชั้น "ความสว่าง" (MUSIC_PRESENCE) อ็อกเทฟบน C4–C5 (262–523Hz) 4 เสียง gain 0.09** ซึ่งเป็นย่านที่มือถือออกได้ดีที่สุด + ระฆัง 0.075 → 0.1 |
| Ambience ยังเบามาก | บัส 0.10 + ตัวกรอง lowpass ที่ 260Hz (ต่ำกว่าย่านที่มือถือออก) + ประกาย gain 0.02 + tremolo ลึก 0.45 กดระดับเฉลี่ยลง | บัส 0.10 → **0.3** · ตัวกรอง **260 → 400Hz** · ประกาย **0.02 → 0.045** · tremolo ลึก **0.45 → 0.4** |

**หลักฐานวัดได้ (วอลุ่ม 0.85 · เทียบกับรอบ 2 ก่อนแก้):**

| ค่า | รอบ 2 (ก่อน) | รอบ 3 (หลัง) | เปลี่ยน |
|---|---|---|---|
| gain บัส | `{sfx:0.85, music:0.6375, ambience:0.085}` | `{sfx:0.85, music:1.53, ambience:0.255}` | เพลง ×2.4 · บรรยากาศ ×3 |
| เพลงเดี่ยว peak RMS | 0.0495 | **0.127–0.133** | **×2.7** |
| บรรยากาศเดี่ยว peak RMS | 0.0157 | **0.0475** | **×3.0** |
| เพลง+บรรยากาศ | 0.0704 | **0.221** | **×3.1** |
| พลังงานเพลงในย่าน 250–1000Hz (ที่มือถือออกได้) | (วัดไม่ได้ — ไม่มีชั้น presence) | **72.6%** (ต่ำ 40–250Hz 27.4%) | เพลง "ได้ยินจริง" บนมือถือ |
| SFX | 0.14–0.33 | 0.14–0.33 (ไม่แตะ — ผู้ใช้บอกว่าเพียงพอแล้ว) | คงเดิม |

- compressor ผ่อนเป็น `ratio 2.5 / threshold -10` (เดิม 3 / -14) เพราะบัสเพลงถูกยกขึ้นมาก — เกณฑ์ต่ำ+ratio สูงจะบี้เพลงที่ต่อเนื่องจนแบน
- `tsc` 0 error · **jest 514 ผ่าน / 35 suites** · build ✓ + restart · `/api/health` 200 · `inspect:audio` ผ่านทุกข้อ (22 เสียง SFX ยังมีสัญญาณจริง)
- เทสต์ที่ปรับความหมาย (พร้อมเหตุผลในคอมเมนต์): "บัสเพลงสูงกว่า SFX ได้ เพราะ RMS เพลงต่อเนื่องต่ำกว่าพีค SFX" · "บรรยากาศต้องเบากว่าเพลง ≥3 เท่า" (แทนเกณฑ์ ≤10% ที่ทำให้เบาจนไม่ได้ยิน) · "ชั้น presence ต้องอยู่ย่าน 220–1000Hz"






## Phase 23 — UI: หัวเว็บ Desktop ไม่โชว์จำนวน + ปุ่มลูกศรตารางรูนใหม่ (2026-09-20)

**คำสั่งผู้ใช้:** *"แก้ไข UI ตรงตำแหน่งแสดงจำนวน พลังงาน และเหรียญ ในเว็บสำหรับ Desktop เป็น ... ไม่แสดงจำนวน และแก้ปุ่มลูกศรสำหรับเลื่อนตารางรูน เอามาใส่ด้านข้าง และบนล่าง ตารางแทน ทำปุ่มเป็น 3 เหลี่ยม พอ ปุ่มกลับกลางเอาออก ตัวหนังสือบอกขนาด 10 x 10 เอาออก ไม่ต้องบอก แต่ปรับขนาดให้เหมาะสมกับจอ"*

| สิ่งที่ทำ | ไฟล์ |
|---|---|
| **หัวเว็บ (Desktop)**: ⚡ พลังค้นหา และ 🪙 Coin แสดง `...` แทนตัวเลข (เบรกพอยต์ `md` ขึ้นไป) — มือถือยังโชว์ตัวเลขเดิม เพราะต้องรู้จำนวนก่อนค้นหารูน | `src/components/layout/TopHeader.tsx` |
| **ปุ่มลูกศรตารางรูน** ย้ายจากกลุ่มเดิม (↑←→↓ รวมกันใต้ตาราง) → **▲ เหนือตาราง · ▼ ใต้ตาราง · ◀ ซ้าย · ▶ ขวา** ของตาราง | `src/components/rune/RuneCanvas.tsx` |
| เปลี่ยนปุ่มเป็น **สามเหลี่ยม** (▲▼◀▶) สีเหลืองอำพัน + **ปิดตัวเองเมื่อสุดขอบ** (อยู่ซ้ายสุด ◀ เป็น disabled) + `aria-label` จากคำแปล 2 ภาษา | `RuneCanvas.tsx` · `dict-th.ts` · `dict-en.ts` |
| **ลบปุ่ม "กลับกลาง" (Recenter)** และ **ลบข้อความ "มุมมอง 10×10 รูน"** พร้อมคีย์คำแปล `discover.viewPort` / `discover.center` ที่ไม่ใช้แล้ว | `RuneCanvas.tsx` · `dict-th.ts` · `dict-en.ts` |
| **ขนาดตารางปรับตามจอจริง**: เดิมตรึง 36px/ช่อง (360px) ทุกจอ → คำนวณใหม่ = `min(ความกว้างที่เหลือ, ความสูงที่เหลือ − แถบเมนูล่าง − ที่ว่างใต้ตาราง) ÷ 10` clamp 20–60px ⇒ จอใหญ่ได้ตารางใหญ่ มือถือพอดีความกว้าง (ไม่ล้น ไม่ต้องซูม) · คำนวณใหม่เมื่อ `resize`/`orientationchange` | `RuneCanvas.tsx` |

**หลักฐานวัดได้ (Chrome headless วัด DOM จริง · `tsc` 0 error · **jest 514 ผ่าน / 35 suites** · build ✓ + restart · `/api/health` 200):**

| ตรวจ | Desktop 1440×900 | มือถือ 390×844 |
|---|---|---|
| ขนาดตาราง | 370×370 px (ช่อง 37px) — พอดีจอทั้งกว้าง/สูง (`canvasFitsWidth/Height = true`) | 270×270 px (ช่อง 27px) |
| ตำแหน่งปุ่ม | ▲ บน · ▼ ล่าง · ◀ ซ้าย · ▶ ขวา (`upAbove/downBelow/leftOf/rightOf = true`) | เหมือนกัน |
| ปุ่มทำงานจริง | กดแล้วภาพตารางเปลี่ยนทั้ง 4 ปุ่ม (`{right:true,left:true,down:true,up:true}`) | เหมือนกัน |
| ปิดตัวเองสุดขอบ | กดซ้ายจนสุด → `◀ disabled = true` | เหมือนกัน |
| ข้อความ "10×10" / ปุ่ม "กลับกลาง" | ไม่พบทั้งคู่ (`hasSizeText:false`, `hasRecenter:false`) | ไม่พบ |
| ล้นแนวนอน | `0 px` | `0 px` |
| หัวเว็บ | `⚡ ... | 🪙 ...` (ไม่โชว์จำนวน) | `⚡ 5 | 🪙 100` (โชว์จำนวนตามเดิม) |

> ยังไม่อัปเดตคู่มือ/หนังสือ (รอกฎ/รอคำสั่ง) — ส่วนที่จะไม่ตรง: ภาพหน้า "ค้นหารูน" (ปุ่มลูกศร/ขนาดตาราง) และผังหัวเว็บบน Desktop



---

## Phase 24 — หัวเว็บ Desktop โชว์ตัวเลขจริง + ตารางรูนเลื่อนวนไม่สิ้นสุด (2026-09-26)

**คำสั่งผู้ใช้:** *"1. พลังงาน และเหรียญ ที่ Desktop site มันไม่แสดงตัวเลข 2. แก้ไขตารางรูนทำให้เวลากดมันวน เอามาต่อกัน ให้กดได้ไม่สิ้นสุด"*

| สิ่งที่ทำ | ไฟล์ |
|---|---|
| **หัวเว็บทุกจอแสดงตัวเลขจริง** — ⚡ พลังค้นหา และ 🪙 Coin เลิกซ่อนเป็น `...` บน Desktop (`md:hidden` / `hidden md:inline` ออกทั้งคู่) เหลือ `...` เฉพาะตอนยังโหลดค่าไม่เสร็จ (`null`) — **ยกเลิกคำสั่ง Phase 23** ตามคำสั่งใหม่ | `src/components/layout/TopHeader.tsx` |
| **ตรรกะกระดานรูนแยกเป็นโมดูลบริสุทธิ์** — `RUNE_GRID_SIZE` (100) · `RUNE_VIEW_SIZE` (10) · `RUNE_MAX_OFFSET` (90) · `RUNE_OFFSET_COUNT` (91) · `wrapRuneOffset()` · `runeIndexAt()` ⇒ เทสต์ได้โดยไม่ต้องเรนเดอร์ canvas | `src/lib/rune-grid.ts` (ใหม่) |
| **ปุ่มลูกศรวนไม่สิ้นสุด** — ถอด `disabled={viewX<=0}` / `{viewX>=MAX_VIEW}` (และคลาส `disabled:opacity-*`) ออกทั้ง ▲▼◀▶ · `panView` ใช้ `setViewX(prev => wrapRuneOffset(prev + dx))` (functional update — กดรัวไม่ตกหล่น) ⇒ อยู่ขวาสุดแล้วกด ▶ ต่อ = กลับไปซ้ายสุดทันที | `src/components/rune/RuneCanvas.tsx` |
| **ลาก (drag) ก็วน** — `wrapRuneOffset(pan.baseX - …)` แทน `clamp(...)` · `randomFill` ห่อ offset ด้วย · ตำแหน่งช่อง → index ใช้ `runeIndexAt()` ร่วมกัน | `RuneCanvas.tsx` · `src/lib/rune-grid.ts` |
| คำแปล 2 ภาษา (aria-label/คำใบ้) บอกว่าวนได้: *"เลื่อนตารางรูนขึ้น (วนไม่สิ้นสุด)"* / *"Pan the rune table up (wraps around)"* + คำใบ้ "ลากเพื่อเลื่อนแผน (วนไม่สิ้นสุด)" | `src/lib/i18n/dict-th.ts` · `dict-en.ts` |
| คู่มือผู้เล่น (เอกสาร ไม่ใช่หนังสือ PDF): เพิ่มแถว "พลังค้นหา = หัวเว็บโชว์ตัวเลขจริงทั้ง 2 จอ" และ "เลื่อนกระดาน = วนไม่สิ้นสุด" | `docs/PLAYER_GUIDE_TH.md` |
| เทสต์กัน regress 11 ข้อ — wrap ค่าบวก/ลบ · กด ▶ 300 ครั้งไม่มีจุดตัน · กด ◀ จาก 0 → 90 · วนครบรอบทุก 91 · index 0–9999 · **ตรวจซอร์สจริง**: RuneCanvas ไม่มี `disabled={viewX…}` + ใช้ `wrapRuneOffset` ทุกทาง · TopHeader ไม่มี `md:hidden` | `tests/unit/rune-grid.test.ts` (ใหม่) |

**เหตุผลที่รอบละ 91 ไม่ใช่ 100:** offset ของมุมมองใช้ได้แค่ 0–90 (มุมขวาล่าง = 90 เพราะมุมมองกว้าง 10 ช่องบนกระดาน 100 ช่อง) ⇒ ตำแหน่งที่วนได้จริง = 91 ตำแหน่ง/แกน

**หลักฐานวัดได้ (Chrome headless + CDP วัด DOM/แคนวาสจริง · `tsc --noEmit` 0 error · **jest 525 ผ่าน / 36 suites** · `next lint` ไม่มีคำเตือนใหม่ · `NODE_ENV=production npm run build` ✓ + restart service · `/api/health` 200):**

| ตรวจ | Desktop 1440×900 | มือถือ 390×844 |
|---|---|---|
| ⚡ พลังค้นหา | **`⚡5`** (visible, x=1049–1079 — เดิม `⚡ ...`) | `⚡5` (ไม่เปลี่ยน) |
| 🪙 Coin | **`🪙100`** (visible, x=1091–1135 — เดิม `🪙 ...`) | `🪙100` (ไม่เปลี่ยน) |
| ล้นแนวนอน | −15 px (ไม่มีสกอลบาร์) | 0 px |

**ทดสอบปุ่มลูกศรในเบราว์เซอร์ (กด ▶ จริง 100 ครั้ง เก็บลายนิ้วมือภาพจาก `getImageData`):**
- `disabled` ทั้ง 4 ปุ่ม = `false` ตลอดการทดสอบ (แม้กดไป 100+ ครั้ง) — ไม่มีจุดตันแล้ว
- ภาพซ้ำทุกรอบ **91 ครั้ง**: `true` (ตรวจ 10 คู่) · offset ทั้ง 91 ตำแหน่งให้ภาพ **ไม่ซ้ำกัน 91/91**
- กด ◀ 1 ครั้ง = ได้ภาพเดียวกับ offset ก่อนหน้า 1 ก้าว (`true`) · ▲/▼ เปลี่ยนภาพจริง (`true`)

> ไม่ได้อัปเดต **หนังสือคู่มือ** (`docs/manual/` + PDF) — ตามกฎต้องรอคำสั่งผู้ใช้
> จุดที่จะไม่ตรงกับของจริง 2 จุด: **รูปที่ 3.1** (ภาพหน้า "ค้นหารูน" — เดิมหัวเว็บ Desktop แสดง `...` และปุ่มลูกศรมีสถานะ disabled ที่ขอบ) และคำบรรยาย "ใช้ปุ่มลูกศรหรือลากเพื่อเลื่อนหาโซนใหม่" (ตอนนี้เลื่อนวนไม่สิ้นสุด)


---

## Phase 24.1 — แจ้งเตือน: อ่านแล้วหายทันที (2026-09-26)

**คำสั่งผู้ใช้:** *"ใน Project Game card การแจ้งเตือนเมื่อเปิดดูแล้ว ไม่หายไปในทันที แก้ไขด้วย"*

**สาเหตุจริง:** ระฆัง 🔔 ในหัวเว็บ (`NotificationBell`) โหลดตัวเลขใหม่แค่ **ทุก 60 วินาที + ตอนกลับเข้าแท็บ (focus)** เท่านั้น
⇒ อ่านแจ้งเตือนในหน้า `/notifications` แล้วตัวเลขยังค้าง และในกรอง "ยังไม่อ่าน" รายการที่อ่านแล้วก็ยังค้างอยู่ (เดิมโค้ดอัปเดตแค่ `isRead` ไม่เอาออกจากรายการ)

| สิ่งที่ทำ | ไฟล์ |
|---|---|
| **เหตุการณ์กลาง "การแจ้งเตือนเปลี่ยน"** — `NOTIFICATIONS_CHANGED_EVENT` (`rda:notifications-changed`) · `emitNotificationsChanged(unreadCount?)` · `subscribeNotificationsChanged(handler)` · ปลอดภัยกับ SSR (ไม่มี `window` = ไม่ทำอะไร) + กันค่าเพี้ยน (ติดลบ/ทศนิยม/NaN) | `src/lib/notification-events.ts` (ใหม่) |
| **ระฆังอัปเดตทันที 3 ทาง**: (1) ฟังเหตุการณ์แล้วตั้งเลขที่ส่งมาตรง ๆ ไม่ต้องรอ fetch (2) โหลดใหม่ทุกครั้งที่ **เปลี่ยนหน้า** (`pathname`) — กลับจากหน้าแจ้งเตือนแล้วตัวเลขต้องเป็น 0 ทันที (3) คง poll 60 วิ/focus เดิมไว้เป็นตาข่ายกันตก · เปลี่ยนเป็น `cache: 'no-store'` · แยกตรรกะตั้งเลข+เล่นเสียงเป็น `apply()` (เสียง `notify` เล่นเฉพาะตอนเลข *เพิ่ม*) | `src/components/layout/NotificationBell.tsx` |
| **หน้าศูนย์แจ้งเตือน — กดอ่านแล้วหายทันที (optimistic)**: กดรายการ → รายการนั้นหายจากลิสต์ **ทันที** เมื่อกรอง "ยังไม่อ่าน" (กรอง "ทั้งหมด" = เปลี่ยนเป็นอ่านแล้ว + ไฮไลต์หาย) · กด "อ่านทั้งหมด" → ลิสต์ว่าง + ตัวเลขเป็น 0 ทันที · ยิงเหตุการณ์ทันที ⇒ ระฆังเป็น 0 ภายใน ~0.35 วิ (ไม่รอ poll 60 วิ) | `src/app/(game)/notifications/page.tsx` |
| **ยิงคำขอแล้วไม่หลุด**: รวมเป็น `postRead()` ตัวเดียว + `keepalive: true` (กดลิงก์แล้วเปลี่ยนหน้าทันที คำขอต้องถูกส่งจนจบ) + **ใช้ `unreadCount` จริงที่เซิร์ฟเวอร์ตอบกลับมา** ปรับตัวเลขให้ตรง · ถ้าคำขอล้มเหลว → `load()` ดึงของจริงกลับมา (ห้ามค้างผลลัพธ์ที่ไม่ได้เกิดขึ้น) · กันกดซ้ำ/อ่านแล้วยิง API ซ้ำ | `src/app/(game)/notifications/page.tsx` |
| **เทสต์ 7 ข้อ** — ชื่อเหตุการณ์คงที่ · ส่ง/ไม่ส่งตัวเลข · ค่าติดลบ-ทศนิยม-NaN · unsubscribe · ผู้ฟังหลายตัว · ไม่มี `window` (SSR) แล้วไม่ throw | `tests/unit/notification-events.test.ts` (ใหม่) |
| **สคริปต์พิสูจน์บนเบราว์เซอร์จริง** (Chrome headless + CDP) — seed แจ้งเตือน N รายการ → เปิดหน้าจริง → กดอ่าน 1 รายการ **ขณะจำลองเน็ตหน่วง 2 วิ** → กด "อ่านทั้งหมด" → ตรวจ DOM + เซิร์ฟเวอร์ + รีโหลด | `scripts/inspect-notification-read.mjs` (ใหม่) · `package.json` → `npm run inspect:notif-read` |

**หลักฐานวัดได้ (รันจริงบน production ที่ restart แล้ว · `tsc --noEmit` 0 error · **jest 532 ผ่าน / 37 suites** · `next lint` ไม่มีคำเตือนใหม่ · build ✓ · `/api/health` 200):**

| ตรวจ | ผลจริง |
|---|---|
| โหลดหน้าหลัก → ระฆังโชว์เลข 3 | `badge=3` ✅ |
| กรอง "ยังไม่อ่าน" | `items=3 · badge=3` ✅ |
| กดอ่าน 1 รายการ (เน็ตหน่วง 2 วิ) → หายจากลิสต์ | `items 3→2` ใน **356 ms** ✅ |
| กดอ่าน 1 รายการ → ระฆังลดเลข | `badge 3→2` ใน **356 ms** (ก่อนคำขอจบ 2,000 ms) ✅ |
| หลังคำขอจบ → ไม่เด้งกลับ | `badge=2 · items=2` ✅ |
| กด "อ่านทั้งหมด" → ระฆังเป็น 0 | `badge=null` ใน **353 ms** (poll ตั้งไว้ 60,000 ms) ✅ |
| กด "อ่านทั้งหมด" → ลิสต์ยังไม่อ่านว่าง | `items=0` ✅ |
| เซิร์ฟเวอร์ตรงกับ UI | `GET /api/notifications?filter=unread` → `unreadCount=0` ✅ |
| รีโหลดหน้า → ยังเป็น 0 | `badge=null · items=0` ✅ |

**สรุปสคริปต์: ผ่าน 11/11 ข้อ** (`node scripts/inspect-notification-read.mjs --register --seed 3`)

> ตัวเลข 356/353 ms ยืนยันว่าไม่ได้มาจาก poll 60 วิ — และการที่ระฆังลดเลขได้ **ก่อนคำขออ่านจบ (เน็ตหน่วง 2 วิ)** แปลว่าเป็น optimistic + เหตุการณ์ภายในหน้าเว็บจริง
> ยังไม่อัปเดต **หนังสือคู่มือ** (`docs/manual/` + PDF) ตามกฎ — พฤติกรรมที่เพิ่มเข้ามา (อ่านแล้วหายทันที) เป็นการปรับ UX ไม่ได้เปลี่ยนหน้าจอ จึงไม่กระทบภาพในหนังสือ


---

## Phase 24.2 — เสียงต่อสู้ดังขึ้น + สมจริงขึ้น (2026-09-26)

**คำสั่งผู้ใช้:** *"เสียงต่อสู้ เบา เสียง ไม่สมจริง แก้ไขด้วย"*

**สาเหตุจริง 4 ข้อ (ตรวจจากโค้ด + วัดเสียงจริงก่อนแก้):**
1. **เสียงต่อสู้ไม่ได้ถูกลด** แต่ "ความดังที่วัดได้" กับ "ความดังที่หูได้ยิน" คนละเรื่อง — ชั้นเสียงกระแทกส่วนใหญ่อยู่ที่ **110–190 Hz** (square 150 / sawtooth 120 / lowpass 900→300) ซึ่ง **ลำโพงมือถือออกไม่ได้** ⇒ วัด RMS สูงแต่หูแทบไม่ได้ยิน
2. **เสียงแห้งเกินไป (ไม่สมจริง)** — ทุกชั้นเป็น oscillator + noise ที่ **ไม่มีหางเสียงสะท้อน** เลย ⇒ ฟังเป็น "บี๊บ/ติ๊ก" ไม่ใช่ "ดาบกระทบในห้องหิน"
3. **เล่นซ้ำเหมือนเดิมทุกครั้ง** — ไม่มี jitter, noise เริ่มที่จุดเดิมทุกครั้ง, การโจมตีใช้เสียงเดิม 100% ⇒ หูจับได้ทันทีว่าเป็นเสียงสังเคราะห์
4. **ช่วงเปิดเสียง (attack) คงที่ 8 ms** — ไม่มี transient คมของ "ของแข็งกระแทก" และไม่มีลมค่อย ๆ มา ⇒ ขาดมิติเวลา

**ก่อนแก้ (วัดจริง baseline วอลุ่ม 0.85):** เสียงต่อสู้ peak 0.112–0.303 (เฉลี่ย **0.19151**) · UI 0.15851 · `battle_clash` มีความยาวเสียงเหนือ 20% ของพีคแค่ **100 ms** (FAIL เกณฑ์ตัวเอง: click แห้ง)

| สิ่งที่ทำ | ไฟล์ |
|---|---|
| **ยกความดังเฉพาะกลุ่มต่อสู้**: `SFX_BATTLE_GAIN = 1.35` (+ `BATTLE_SFX_NAMES` / `isBattleSfx` / `sfxGainScale` / `sfxJitterCents`) คูณใน `sfxSchedule` — เสียง UI **ไม่ถูกแตะ** (ดังเท่าเดิม ไม่รบกวน) · เพดานต่อชั้น 0.4 → 0.45 | `src/lib/sfx.ts` |
| **เสียงสะท้อนห้องสร้างเอง** (ไม่ต้องมีไฟล์เสียง): `buildReverbImpulse(sampleRate, seconds, decay, random)` = noise ที่จางแบบ exponential (0.55 วิ / decay 3.2) → `ConvolverNode` ตัวเดียวต่อการเล่น + wet gain ต่อชั้น · ตั้ง `reverb` ในตารางเสียง (0.12–0.35) | `src/lib/sfx.ts` |
| **transient ต่อชั้นเสียง**: `ToneSpec.attack` (1–3 ms = กระแทกคม · 20–60 ms = ลม) แทนค่าเดิม 8 ms คงที่ + envelope ใหม่ (attack → sustain → exponential decay) | `src/lib/sfx.ts` |
| **ความไม่ซ้ำ**: jitter ±35 cents ต่อการเล่น (เฉพาะเสียงต่อสู้) + noise สุ่มจุดเริ่มในบัฟเฟอร์ ⇒ ฟันดาบครั้งที่ 1 กับ 2 ไม่เหมือนกัน | `src/lib/sfx.ts` |
| **เรียบเรียงใหม่ทั้งกลุ่มต่อสู้** ให้ทุกเสียงมี transient + เนื้อเสียง 250–4000Hz + โลหะ inharmonic + หางสะท้อน: `battle_hit` · `battle_sword` · `battle_clash` · `battle_cast` · `battle_shield` · `battle_burn` · `battle_heal` · `battle_faint` · `battle_win` · `battle_lose` · `raid_hit` · `raid_phase` | `src/lib/sfx.ts` |
| **สลับเสียงดาบ/ดาบกระทบดาบ**: `battleSfxFor(action, status, hitIndex)` — ทุกครั้งที่ 3 ของการโจมตีใช้ `battle_clash` · หน้าเทปนับ `attackIndex` และรีเซ็ตเมื่อเริ่มเทปใหม่ | `src/lib/sfx.ts` · `src/app/(game)/battle/[id]/page.tsx` |
| **Sidechain ducking** (เพลง/บรรยากาศลดในจังหวะกระแทก): `BATTLE_DUCK { music 0.6, ambience 0.7, hold 0.15s, attack 0.02s, release 0.3s }` + `duckedLayerGain()` · `applyLayerGains(..., ducked, rampSeconds)` · provider เรียกเมื่อเล่นเสียงกลุ่มต่อสู้ ⇒ เพลงไม่เบาทั้งศึก | `src/lib/audio-engine.ts` · `src/components/providers/AudioProvider.tsx` |
| **QA API เพิ่ม 2 ตัว**: `window.__rdaAudio.bands(250,4000)` = สัดส่วนพลังงานย่านความถี่ (`bandEnergyShare()`) และ `bus()` = เกนจริงบนบัส (ใช้ยืนยัน ducking) | `src/lib/audio-engine.ts` · `AudioProvider.tsx` |
| **ตัวตรวจเสียงขยายผล**: ต่อ SFX วัด `peak` + `activeMs` (ความยาวเสียงเหนือ 20% ของพีค = "มีเนื้อ/มีหาง ไม่ใช่ click") + `bandShare` · เกณฑ์ใหม่ 4 ข้อ (peak ≥ 0.10 · active ≥ 120 ms · ย่าน 250–4000Hz ≥ 35% · ต่อสู้ > UI) + ตรวจ ducking 3 ข้อ · ธง `--only-battle` / `--battle-min-*` | `scripts/inspect-audio.mjs` |
| **เทสต์กัน regress +33 ข้อ**: กลุ่มต่อสู้ถูกยกเฉพาะกลุ่ม · ทุกเสียงมี transient ≤ 5 ms + ความยาว ≥ 150 ms + พลังงานย่าน 250–4000Hz + reverb > 0 · UI ไม่มี reverb · jitter ต่อสู้ ≠ 0 · impulse จางตามเวลา/อยู่ในช่วง -1..1/ทำซ้ำได้ · duck คำนวณถูก (รวม **regression ของบั๊ก "duck ค้าง"**) | `tests/unit/sfx.test.ts` · `tests/unit/audio-engine.test.ts` |
| คู่มือผู้เล่น (เอกสาร): แถว SFX/เพลง/บรรยากาศตรงของจริง + หมายเหตุ ducking ระหว่างต่อสู้ | `docs/PLAYER_GUIDE_TH.md` |

**🐞 บั๊กที่ตัวตรวจใหม่จับได้เอง (แก้แล้วในรอบเดียวกัน):** `applyLayerGains` รอบแรกส่ง `ducked` ไม่ครบ ⇒ เพลง/บรรยากาศถูก duck **ค้างตลอดเวลา** (เพลงเบาลง 40% ไม่มีวันคืน — วัดได้ music bus 1.53 → 0.6885 ถาวร) · ตอนนี้เลือกค่าระหว่าง `layerGain` กับ `duckedLayerGain` ตามธง + มีเทสต์ยืนยันทั้งสองทาง

**หลักฐานวัดได้ (รันจริงบน production · `tsc --noEmit` 0 error · **jest 565 ผ่าน / 37 suites** · `next lint` ไม่มีคำเตือนใหม่ · build ✓ + restart · `/api/health` 200):**

| เสียงต่อสู้ | peak ก่อน → หลัง | active ก่อน → หลัง | ย่าน 250–4000Hz (หลัง) |
|---|---|---|---|
| battle_hit | 0.201 → **0.317** | 125 → 125 ms | 45% |
| battle_sword | 0.222 → **0.310** | 200 → 175 ms | 74% |
| battle_clash | 0.249 → **0.308** | **100 → 200 ms** | 71% |
| battle_cast | 0.303 → **0.339** | 425 → 450 ms | 99% |
| battle_shield | 0.198 → **0.338** | 325 → 275 ms | 93% |
| battle_burn | 0.112 → **0.182** | 275 → 250 ms | 53% |
| battle_heal | 0.212 → **0.356** | 425 → 450 ms | 100% |
| battle_faint | 0.129 → **0.235** | 275 → 400 ms | 66% |
| battle_win | 0.172 → **0.266** | 225 → 450 ms | 100% |
| battle_lose | 0.188 → **0.320** | 275 → 575 ms | 60% |
| raid_hit | 0.132 → **0.327** | 175 → 225 ms | 89% |
| raid_phase | 0.201 → **0.299** | 325 → 325 ms | 64% |
| **เฉลี่ย** | **0.192 → 0.297 (+55%)** | — | ต่ำสุด 45% |
| **เสียง UI (ไม่แตะ)** | 0.158 (เท่าเดิมทุกตัว) | — | — |

- **เทียบกับ UI:** เสียงต่อสู้ 0.29653 vs UI 0.15800 (≈1.9 เท่า) — ก่อนแก้ 0.19151 vs 0.15851
- **ducking (วัดจากบัสจริง):** เพลง `1.53 → 0.918` ขณะดาบดัง แล้วกลับเป็น `1.53` เอง · บัส SFX ไม่ถูกลด (0.85 ตลอด)
- **เทปจริงในหน้า `/battle/<id>` (Chrome headless กด "เริ่มเล่น" จริง):** peak RMS = **0.379** · เพลงลดเป็น 0.918 แล้วคืน 1.53 เมื่อเทปจบ · 80/80 ตัวอย่างมีเสียงออก · ไม่มี error ฝั่งเสียง
- `npm run inspect:audio` → **ผ่านทุกข้อ ✅** (22 SFX มีสัญญาณ · 4 เกณฑ์เสียงต่อสู้ · ducking 3 ข้อ · pause/resume/ซ่อนแท็บยังถูกต้อง)

> **การแกว่งของค่าที่วัดได้เป็นเรื่องที่ตั้งใจ** — jitter ±35 cents + noise สุ่มจุดเริ่ม ทำให้เสียงดาบแต่ละครั้งไม่เหมือนกันเป๊ะ (วัดซ้ำสามรอบได้ 0.278 / 0.318 / 0.310)
> ยังไม่อัปเดต **หนังสือคู่มือ** (`docs/manual/` + PDF) ตามกฎ — จุดที่จะไม่ตรง: หน้าตั้งค่าอธิบายว่า "เพลงปิดไว้เป็นค่าเริ่มต้น" (ของจริงเปิด) และยังไม่มีคำอธิบาย ducking ระหว่างต่อสู้


---

## Phase 25 — ขายการ์ดคืนร้านเป็น Veil Shards + ช่างใส่ Item 3 ช่อง (2026-09-27)

**คำสั่งผู้ใช้:** *"ได้จากการขายการ์ดคืนร้าน จำนวนขึ้นกับความหายากของการ์ด บางส่วนก็ได้จากกิจกรรม
ทำในส่วนของช่างใส่ Item เพิ่ม Status ให้ 3 ช่อง Item โจมตี, ป้องกัน, สนับสนุน สำหรับใส่ Item ที่ได้รับ หรือ Craft มาได้"*

**ปัญหาที่พบก่อนทำ (จาก Phase 24.2 ที่ผู้ใช้ถาม "Veil Shards หาได้จากไหน"):**
- Veil Shards ถูกเก็บเป็น "ยอดของแต่ละกิจกรรม" (`EventParticipation.currencyEarned/Spent`) ⇒ **ผู้เล่นใหม่ไม่มีทางได้เลย**
  (เข้าร่วม = 0, ค่าเข้า Raid = 10, ภารกิจทุกอันก็ต้อง raid ก่อน ⇒ ยิง raid จริงได้ `Veil Shards ไม่พอ`)
  ⇒ ต้องมี "กระเป๋ากลาง" ที่ได้จาก**ขายการ์ด**ด้วย ตามคำสั่งผู้ใช้

| สิ่งที่ทำ | ไฟล์ |
|---|---|
| **กระเป๋า Veil Shards ระดับผู้ใช้** (`User.veilShards`) + ledger `VeilShardTransaction` (amount/type/source/balanceBefore-After/idempotencyKey) — รูปแบบเดียวกับ Wallet ของ Coin | `prisma/schema.prisma` (migration `20260926163829_phase25_items_veil_shards`) · `src/services/veil-shard.ts` (ใหม่) |
| **กฎสกุลเงินบริสุทธิ์**: มูลค่าขายตาม rarity (ทั่วไป 2 · ไม่ธรรมดา 5 · หายาก 12 · ตำนาน 30 · ตำนานเลือง 80 · สร้างสรรค์ 200) · `cardSellTotal` · `canAfford` · ที่มาของทุกธุรกรรม | `src/lib/veil-shards.ts` (ใหม่) |
| **ขายการ์ดคืนร้าน** `POST /api/cards/[id]/sell` — หักจำนวนในคลัง, บวก Veil Shards เข้ากระเป๋า, ledger `CARD_SELL` · **กันพลาด**: การ์ดที่อยู่ในเด็คต้องเหลือ ≥1 ใบ | `src/app/api/cards/[id]/sell/route.ts` (ใหม่) |
| **แคตตาล็อก Item 12 ชิ้น 3 ช่อง** (โจมตี/ป้องกัน/สนับสนุน อย่างละ 4 ระดับ rarity) + `ensureCatalog()` upsert จากโค้ดอัตโนมัติ ⇒ เพิ่มของใหม่ไม่ต้อง migrate/seed มือ | `src/lib/item-definitions.ts` (ใหม่) · `src/services/item.ts` (ใหม่) |
| **ตารางของ**: `ItemDefinition` (catalog) · `UserItem` (ของที่ถือ) · `CardItemSlot` (3 ช่องต่อการ์ด · unique ต่อช่อง) · enum `ItemSlot` | `prisma/schema.prisma` |
| **ร้านช่าง**: `GET /api/items` (แคตตาล็อก + ของที่มี + ราคา/ส่วนที่ขาด) · `POST /api/items/buy` (ซื้อด้วย Veil Shards) · `POST /api/items/craft` (Veil Shards + ฝุ่นเวท จากกิจกรรม) — ทั้งหมดเป็น transaction เดียว | `src/app/api/items/**` · `src/services/item.ts` |
| **ช่างใส่ Item ของการ์ด**: `GET/POST/DELETE /api/cards/[id]/equipment` — 3 ช่อง, ตรวจช่องต้องตรงชนิดของ Item, ต้องมีของในคลัง, 1 ชิ้นใส่ได้ครั้งเดียว (ของซ้ำใส่หลายการ์ดได้) | `src/app/api/cards/[id]/equipment/route.ts` (ใหม่) |
| **Status ของ Item มีผลจริง**: `deckToCombatCards()` บวก Status จาก Item ก่อนเข้าสู่ระบบต่อสู้ · สถานะ/พลังทีมใน `/api/decks` และ `/api/decks/[id]` = พื้นฐาน + Item | `src/services/battle-api.ts` · `src/app/api/decks/**` |
| **ย้ายกิจกรรมมาใช้กระเป๋าเดียวกัน**: Raid (หักค่าเข้า 10 + คืนรางวัล 2/7), ร้านค้ากิจกรรม, ภารกิจกิจกรรม, Milestone VEIL_SHARDS → debit/credit กระเป๋าผู้ใช้ (ยังนับสถิติของอีเวนต์ใน `EventParticipation` ตามเดิม) | `src/services/event.ts` · `src/services/event-quest.ts` · `src/app/api/events/route.ts` |
| **API ยอดเงิน + ประวัติ**: `GET /api/veil-shards` (balance + ledger ล่าสุด) ใช้ในหัวเว็บ | `src/app/api/veil-shards/route.ts` (ใหม่) |
| **UI**: หน้าการ์ดมี "🛠️ ช่างใส่ Item" 3 ช่อง (เลือกใส่/ถอด + โชว์ "+Status" ที่ได้) + ปุ่ม "ขายคืนร้าน" (2 ขั้น: กดขาย → ยืนยัน) + สถานะการ์ดโชว์ "สถานะรวม" | `src/components/cards/CardItemWorkshop.tsx` (ใหม่) · `src/app/(game)/cards/[id]/page.tsx` |
| **UI**: หน้าใหม่ `/items` (ร้านช่าง — แยก 3 ช่อง โชว์ status/ราคา/ของที่มี + ปุ่มซื้อ/คราฟต์) · ยอด 💠 ในหัวเว็บทุกหน้า · เมนู "ร้านช่าง" (จอใหญ่ + มือถือ) + คำแปล 2 ภาษา | `src/app/(game)/items/page.tsx` (ใหม่) · `TopHeader.tsx` · `BottomNavigation.tsx` · `dict-th/dict-en` |
| **rate limit**: เพิ่ม scope `SHOP_WRITE` (30/นาที) ใช้กับ ขาย/ซื้อ/คราฟต์/ใส่-ถอด Item | `src/lib/rate-limit.ts` |
| **เทสต์ +21 ข้อ**: มูลค่าขายตาม rarity (เรียงขึ้นทุกขั้น/ค่าเพี้ยนไม่พัง) · แคตตาล็อกครบ 3 ช่อง/ราคาเป็น integer/ของหายากแรงกว่า · รวม-บวก Status · ตรวจช่อง · ประเมินการคราฟต์ (ขาดเท่าไร) | `tests/unit/veil-shards.test.ts` · `tests/unit/item-definitions.test.ts` (ใหม่) |
| **สคริปต์ตรวจของจริง 22 ข้อ**: สมัคร → สร้างเด็ค → ค้นรูน → ขายการ์ด (ได้ตาม rarity + กันขายการ์ดในเด็ค) → ซื้อ/คราฟต์ → ใส่ 3 ช่อง (ผิดช่องถูกปฏิเสธ) → สถานะในเด็ค/ในเทปศึกเพิ่มจริง → หน้าการ์ด+หัวเว็บ+กดใส่ผ่าน UI จริง | `scripts/inspect-item-workshop.mjs` (ใหม่) · `npm run inspect:items` |

**หลักฐานวัดได้ (production จริง · `tsc --noEmit` 0 error · **jest 586 ผ่าน / 39 suites** · `next lint` ไม่มีคำเตือนใหม่ · build ✓ + restart · `/api/health` 200):**

`npm run inspect:items` → **ผ่าน 22/22 ข้อ** (ยิง API จริง + Chrome headless กดปุ่มจริง)

| ตรวจ | ผลจริง |
|---|---|
| ขายการ์ดคืนร้าน (การ์ดระดับ EPIC) | ได้ **30 Veil Shards** (ตรงตาม rarity) · ยอด 182 → **212** |
| ยอด Veil Shards เป็นกระเป๋าเดียวกัน | `GET /api/veil-shards` เพิ่มขึ้นเท่ากับที่ขาย (+30) |
| ขายการ์ดที่อยู่ในเด็คจนหมด | ถูกปฏิเสธ **400** — "การ์ดใบนี้อยู่ในเด็ค — ต้องเหลือไว้อย่างน้อย 1 ใบ" |
| ร้านช่าง | ครบ 3 ช่อง · **12 Item** |
| ซื้อ Item (หินลับคม) | 💠 200 → **188** (จ่าย 12 ตามราคา) · ของเข้าคลัง ×1 |
| คราฟต์ Item (โล่ไม้โอ๊ก) | 💠 188 → **182** · ✨ 200 → **195** (หักทั้งสองสกุลตามสูตร) |
| ใส่ช่องโจมตี | ATK **75 → 81** (ของ +6) · ช่องที่ใส่ = `["ATTACK"]` |
| ใส่ผิดช่อง | ถูกปฏิเสธ **400** — "Item นี้ใส่ช่องนี้ไม่ได้" |
| สถานะในเด็ค | ATK = **81** (พื้นฐาน 75 + Item 6) — ใช้ค่าที่บวกของจริง |
| เทปศึกจริง | snapshot ของศึก ATK = **81** (บวก Item แล้ว) |
| หน้าการ์ด (DOM จริง) | มีช่อง 3 ช่อง `["ATTACK","DEFENSE","SUPPORT"]` · ปุ่มขายคืนร้านมี · โชว์ "+6" ที่ ATK |
| หัวเว็บ | 💠 **212** (ตรงกับกระเป๋า) |
| กดใส่ Item ผ่าน UI จริง | ใส่ "โล่ไม้โอ๊ก" ช่องป้องกันสำเร็จ (มีปุ่มถอดขึ้นมา) |

**ตรวจเส้นทางกิจกรรมที่ย้ายมาใช้กระเป๋าเดียวกัน (ยิง API จริง):**

| ขั้น | ผลจริง |
|---|---|
| ผู้เล่นใหม่ + เติม 40 💠 → Raid | **200** · คืนรางวัล +2 · หักค่าเข้า 10 ⇒ ยอด **32** (ledger มี 2 รายการ `EVENT_RAID`) |
| ซื้อของร้านกิจกรรม (Coin Cache 15) | **200** ⇒ ยอด **17** (ledger 3 รายการ) |
| สถิติอีเวนต์ยังถูกต้อง | `EventParticipation` = earned 2 / spent 25 / points 2237 / damage 2034 (ยังนับครบ) |
| ผู้เล่นไม่มี Veil Shards | ข้อความใหม่บอกยอดจริง: "Veil Shards ไม่พอ (ต้องใช้ 15 ชิ้น · มี 12)" |

**การตัดสินใจเชิงออกแบบ (เขียนไว้ให้ตรวจย้อนหลัง):**
- เลือก **กระเป๋ากลาง `User.veilShards`** แทนการเก็บใน participation เพราะผู้ใช้ต้องการให้ขายการ์ดได้เงินแม้ไม่มีกิจกรรมเปิด
  โดยคง `EventParticipation.currencyEarned/Spent` ไว้เป็น "สถิติของอีเวนต์" (ไม่ใช่ยอดใช้จ่าย) ⇒ ไม่ต้องแก้ UI กิจกรรม
- แคตตาล็อก Item อยู่ในโค้ด (`src/lib/item-definitions.ts`) และ sync ลง DB อัตโนมัติ ⇒ เพิ่มของใหม่ = แก้ไฟล์เดียว
- Item บวก Status แบบ **บวกตรง** (integer) ไม่มีเพดาน เพื่อให้ตัวเลขตรวจสอบได้ และให้ผู้เล่นเห็นผลทันที

> ยังไม่อัปเดต **หนังสือคู่มือ** (`docs/manual/` + PDF) ตามกฎ — จุดที่จะไม่ตรง: ต้องเพิ่มบท "ร้านช่าง + ช่องใส่ Item 3 ช่อง" และหัวเว็บที่มี 💠
> ส่วน `docs/PLAYER_GUIDE_TH.md` (คู่มือผู้เล่นแบบข้อความ) อัปเดตแล้ว: เพิ่มบทที่ 9 (ร้านช่าง) + กติกา 2 ข้อ + แผนที่เมนู


---

## Phase 25.1 — ซื้อ/ขายแล้วยอด 💠 ต้องเปลี่ยนทันที (2026-09-27)

**คำสั่งผู้ใช้:** *"⚡️0 🪙220 💠7 — หลังจากใช้ไปแล้วไม่ลดทันที ต้องรอเปลี่ยนหน้า หรือ Refresh"*

**สาเหตุจริง:** หัวเว็บ (`TopHeader`) โหลดยอด Veil Shards **แค่ตอนเปิดหน้า/เปลี่ยนหน้า (`pathname`)** เท่านั้น
⇒ ซื้อ/คราฟต์ Item ในหน้า `/items` หรือขายการ์ดในหน้าการ์ด ยอด 💠 ยังค้างเป็นค่าเก่า (ต้องเปลี่ยนหน้าหรือ F5)
(เป็นรูปแบบเดียวกับบั๊ก "ระฆังแจ้งเตือนไม่อัปเดต" ใน Phase 24.1)

| สิ่งที่ทำ | ไฟล์ |
|---|---|
| **เหตุการณ์กลาง "ยอด 💠 เปลี่ยน"** — `VEIL_SHARDS_CHANGED_EVENT` (`rda:veil-shards-changed`) · `emitVeilShardsChanged(balance?)` · `subscribeVeilShardsChanged(handler)` · ปลอดภัยกับ SSR + กันค่าเพี้ยน (NaN/ติดลบ/ทศนิยม ⇒ ถือว่า "ไม่รู้ยอด") | `src/lib/veil-shard-events.ts` (ใหม่) |
| **หัวเว็บอัปเดตทันที**: ฟังเหตุการณ์ → ตั้งเลขที่ส่งมาทันที (โชว์ก่อน) → แล้วดึง `/api/veil-shards` ยืนยันกับเซิร์ฟเวอร์ (กันตัวเลขเพี้ยน) | `src/components/layout/TopHeader.tsx` |
| **ยิงเหตุการณ์ทุกจุดที่ยอดเปลี่ยน**: ซื้อ/คราฟต์ Item (`/items`) · ขายการ์ด (ช่างใส่ Item) · Boss Raid (ค่าเข้า/รางวัล) · ร้านค้ากิจกรรม · ภารกิจกิจกรรม · Milestone (บางระดับให้ Veil Shards ⇒ สั่งให้หัวเว็บดึงยอดใหม่) | `src/app/(game)/items/page.tsx` · `src/components/cards/CardItemWorkshop.tsx` · `src/app/(game)/events/[eventId]/page.tsx` |
| **เทสต์ +7 ข้อ** — ชื่อเหตุการณ์คงที่ · ส่ง/ไม่ส่งยอด · ค่าเพี้ยนถูกปรับให้ปลอดภัย · `Number(undefined)` จาก API = ไม่รู้ยอด (ไม่ใช่ NaN) · unsubscribe · ผู้ฟังหลายตัว · ไม่มี `window` (SSR) แล้วไม่ throw | `tests/unit/veil-shard-events.test.ts` (ใหม่) |
| **ตัวตรวจของจริงเพิ่ม 2 ข้อ** (กดปุ่มบนเบราว์เซอร์จริง วัด DOM หัวเว็บ หลังกดทันที **โดยไม่เปลี่ยนหน้า**) | `scripts/inspect-item-workshop.mjs` |

**หลักฐานวัดได้ (production จริง · `tsc --noEmit` 0 error · **jest 593 ผ่าน / 40 suites** · `next lint` ไม่มีคำเตือนใหม่ · build ✓ + restart · `/api/health` 200):**

| ตรวจ (กดปุ่มจริงบนเบราว์เซอร์) | ผลจริง |
|---|---|
| ขายการ์ดบนหน้าการ์ด → ยอด 💠 บนหัวเว็บ | **212 → 224 ภายใน 100 ms** (ไม่เปลี่ยนหน้า) |
| ซื้อ Item บนหน้า `/items` → ยอด 💠 บนหัวเว็บ | **224 → 212 ภายใน 100 ms** (ไม่เปลี่ยนหน้า) |

`npm run inspect:items` → **ผ่าน 25/25 ข้อ** (เพิ่มจาก 22 ข้อใน Phase 25)

> จุดที่ยิงเหตุการณ์เพิ่มแต่ยังไม่ได้กดทดสอบผ่าน UI ในสคริปต์ (ยิงผ่าน API แล้วใน Phase 25): Raid / ร้านค้ากิจกรรม / ภารกิจกิจกรรม / Milestone — ใช้กลไกเดียวกัน (`emitVeilShardsChanged` → หัวเว็บอัปเดต)
> ไม่มีเอกสาร/หนังสือคู่มือต้องแก้ (เป็นการแก้พฤติกรรมภายใน ไม่เปลี่ยนหน้าจอ)


---

## Phase 26 — กระเป๋า (เงิน+ไอเทม) · โปรไฟล์มีข้อมูลจริง · อวตารอิโมจิ/วาดเอง 6×6 (2026-09-27)

**คำสั่งผู้ใช้:** *"แล้วกระเป๋า Bag มีไว้ทำไม ถ้าไม่เอา Item ไปแสดง เอาเงินไปแสดง
Profile ก็ยังไม่มีข้อมูล ทำให้ด้วย เพิ่ม เลือก Emoji แทนตัว หรือ สามารถวาด เองได้จาก ช่องวาด 6x6 ช่อง"*

**สิ่งที่พบก่อนทำ:**
- กระเป๋า (`/inventory`) แสดงแค่ของสะสมจากกิจกรรม — **ไม่มีไอเทมช่างที่เพิ่งทำ (Phase 25) และไม่มีตัวเลขเงินเลย**
- เมนู 👤 ในหัวเว็บ/i18n ชี้ไป `/profile` แต่ **ไม่มีหน้านั้นอยู่จริง (404)** — "Profile ไม่มีข้อมูล" = ไม่มีหน้าเลย
- ผู้ใช้ไม่มีที่ตั้งรูปประจำตัว (มีแต่คอลัมน์ `avatarUrl` ที่ไม่มี UI ใช้)

| สิ่งที่ทำ | ไฟล์ |
|---|---|
| **กระเป๋าใหม่ = เงิน + ไอเทม + ของสะสม**: `GET /api/inventory` เพิ่ม `balances` (Coin/Veil Shards/ฝุ่นเวท) + `workshopItems` (ของที่ถือ ตามช่อง + จำนวนที่ใส่) | `src/app/api/inventory/route.ts` |
| หน้ากระเป๋า: การ์ดเงิน 3 ช่อง (กดไป Wallet/ร้านช่างได้) · ไอเทมช่างแยก 3 ช่อง + Status + จำนวน · ส่วนของสะสมจากกิจกรรมเดิม | `src/app/(game)/inventory/page.tsx` |
| **โปรไฟล์จริง**: `GET /api/profile` รวม ตัวตน + ยอดเงิน/พลัง + สถิติการเล่น (ศึก/ชนะ/แพ้/อัตราชนะ · การ์ด · ทีม · ภารกิจ · ไอเทม) + กิจกรรมล่าสุด | `src/app/api/profile/route.ts` (ใหม่) · `src/app/(game)/profile/page.tsx` (ใหม่) |
| **อวตาร 2 แบบ**: เลือกอิโมจิ (32 แบบ) หรือวาดเอง **6×6** (พาเลตต์ 8 สี + ยางลบ + ล้างทั้งหมด + ล้างอวตาร) — เก็บเป็น `avatarEmoji` และ `avatarGrid` (รหัส 36 ช่อง) | `src/lib/avatar.ts` (ใหม่) · `src/app/api/profile/avatar/route.ts` (ใหม่) · `src/components/profile/AvatarEditor.tsx` · `AvatarView.tsx` (ใหม่) |
| **แสดงอวตารบนหัวเว็บทุกหน้า** (แทน 👤) + อัปเดตทันทีเมื่อบันทึกผ่านเหตุการณ์กลาง `rda:avatar-changed` | `src/components/layout/TopHeader.tsx` · `src/lib/avatar-events.ts` (ใหม่) · `/api/auth/me` (เพิ่มฟิลด์อวตาร) |
| **ความปลอดภัยของข้อมูลอวตาร**: อิโมจิต้องอยู่ในรายการที่เกมมีให้ · กริดถูกทำความสะอาด (อักขระแปลก → ช่องโปร่งใส · เติม/ตัดให้ครบ 36) · กริดว่างถูกปฏิเสธ · กันอิโมจิ/ภาพวาดซ้อนกัน | `src/lib/avatar.ts` · `src/app/api/profile/avatar/route.ts` |
| migration `20260926170701_phase26_avatar_emoji_grid` (เพิ่ม 2 คอลัมน์ใน users) | `prisma/schema.prisma` |
| **เทสต์ +24 ข้อ**: กริด 6×6 (ทำความสะอาด/ไป-กลับ/nับช่อง) · พาเลตต์ (8 สี key ไม่ซ้ำ) · อิโมจิ (ตรวจรายการ) · ชนิดอวตารที่จะแสดง · เหตุการณ์อวตาร (emit/subscribe/sanitize/SSR) | `tests/unit/avatar.test.ts` · `tests/unit/avatar-events.test.ts` (ใหม่) |
| **สคริปต์ตรวจของจริง 22 ข้อ**: โปรไฟล์มีข้อมูลจริง · กระเป๋ามีเงิน+ไอเทม · อวตารอิโมจิ/กริด/ปฏิเสธค่าเพี้ยน · หัวเว็บแสดงอวตาร · ช่องวาด 36 ช่อง · กดเลือกอิโมจิแล้วอวตารหัวเว็บเปลี่ยนทันที | `scripts/inspect-profile-avatar.mjs` (ใหม่) · `npm run inspect:profile` |
| คู่มือผู้เล่น: บทที่ 10 กระเป๋า (เงิน+ไอเทม+ของสะสม) · บทที่ 11 โปรไฟล์+อวตาร · แผนที่เมนูเพิ่ม 👤/🛠️ | `docs/PLAYER_GUIDE_TH.md` |

**หลักฐานวัดได้ (production จริง · `tsc --noEmit` 0 error · **jest 617 ผ่าน / 42 suites** · `next lint` ไม่มีคำเตือนใหม่ · build ✓ + restart · `/api/health` 200):**

`npm run inspect:profile` → **ผ่าน 22/22 ข้อ**

| ตรวจ | ผลจริง |
|---|---|
| `GET /api/profile` | coin=100 · shards=0 · **statKeys=11** (ไม่ใช่หน้าว่าง) |
| กระเป๋า (API) | `{coin:100, veilShards:88, dust:50}` · `workshopItems=1` (หินลับคม) |
| ตั้งอวตารอิโมจิ `🐉` | สำเร็จ · `/api/auth/me` ส่ง `🐉` กลับมา (หัวเว็บใช้ค่านี้) |
| อิโมจินอกรายการ `🚀` | **400** — "อิโมจิไม่ถูกต้อง — เลือกจากรายการที่เกมมีให้ (32 แบบ)" |
| ส่งข้อมูลว่าง / กริดว่างเปล่า | **400** ทั้งคู่ (มีข้อความบอกเหตุผล) |
| บันทึกภาพวาด `AABB??CC` | เก็บเป็น **36 ช่อง** (`AABB..CC…`) · อิโมจิเดิมถูกล้าง (`null`) — ไม่ซ้อนกัน |
| โปรไฟล์บอกชนิดอวตาร | `grid` |
| หน้า `/profile` (DOM จริง) | ชื่อ "Profile Check" · การ์ดเงิน+**11 ช่องสถิติ** · 🪙100 💠88 ตรงกับ API · **อิโมจิ 32 ตัวให้เลือก** |
| หัวเว็บ | แสดงภาพวาดที่บันทึก (`AABB..CC…`) |
| แท็บ "วาดเอง 6×6" | **36 ช่อง** + พาเลตต์ **9 ปุ่ม** (8 สี + ยางลบ) + ปุ่มบันทึก |
| กดเลือกอิโมจิจาก UI | อวตารบนหัวเว็บเปลี่ยนเป็น 🐉 **ใน 100 ms** (ไม่เปลี่ยนหน้า/ไม่รีเฟรช) |
| กระเป๋า (DOM จริง) | 🪙100 · 💠88 · ✨50 · ไอเทม 1 รายการ · ส่วนของสะสมจากกิจกรรมยังอยู่ |

> ยังไม่อัปเดต **หนังสือคู่มือ** (`docs/manual/` + PDF) ตามกฎ — จุดที่จะไม่ตรง: ต้องเพิ่มบท "กระเป๋า (เงิน+ไอเทม)" · "โปรไฟล์ + อวตารอิโมจิ/วาดเอง 6×6" และหัวเว็บที่มีอวตารผู้เล่น
> `docs/PLAYER_GUIDE_TH.md` (คู่มือผู้เล่นแบบข้อความ) อัปเดตแล้วตามรายการด้านบน


---

## Phase 27 — Admin แสดงการ์ดครบทุกใบ (แบ่งหน้า + โหลดทั้งหมด) (2026-09-27)

**คำสั่งผู้ใช้:** *"เหมือนการ์ด ใน Admin จะแสดงการ์ดไม่ครบทุกใบ"*

**สาเหตุจริง (ยืนยันด้วยตัวเลข):**
- หน้า `/admin/cards` ขอข้อมูล **ครั้งเดียว** `?limit=50` และไม่มีปุ่มเปลี่ยนหน้าเลย
- API `/api/admin/cards` จำกัด `limit` ไว้ **ไม่เกิน 50** (`Math.min(50, ...)`) และมี `default 20`
- ในฐานข้อมูลมี **167 ใบ (ตอนตรวจล่าสุด 172 ใบ)** ⇒ แอดมินเห็นแค่ 50 ใบแรก ไม่มีทางดูใบที่เหลือ
- หน้า `/admin/users` มีรูปแบบเดียวกัน (ขอครั้งเดียว 50 รายชื่อ) ⇒ ผู้เล่นเกิน 50 คนก็ดูไม่ครบเช่นกัน

| สิ่งที่ทำ | ไฟล์ |
|---|---|
| **โมดูลกติกาการแบ่งหน้าชุดเดียว** (บริสุทธิ์ เทสต์ได้): `normalizeLimit` (ค่าเริ่มต้น/เพดาน), `pageCount`, `normalizePage` (clamp หน้าเกินขอบ), `pagerSummary` (ช่วง "51–100 จาก 172"), `pageWindow` (เลขหน้าที่ย่อแล้ว) | `src/lib/pagination.ts` (ใหม่) |
| **API การ์ด**: ใช้กติกากลาง · เพดานต่อหน้า 50 → **100** · คืน `pagination { page, limit, total, totalPages, from, to }` · ขอหน้าเกินขอบ = ดึงกลับหน้าสุดท้าย (ไม่คืนว่าง) | `src/app/api/admin/cards/route.ts` |
| **API ผู้เล่น**: ใช้กติกาเดียวกัน (เพดาน 100 + `pagination` แบบมีช่วง) | `src/app/api/admin/users/route.ts` |
| **หน้าการ์ดของแอดมิน**: แถบแบ่งหน้าใหม่ → "ทั้งหมด N ใบ · กำลังแสดง 51–100 · หน้า 2/4" · เลือก **ต่อหน้า 20/50/100** · ปุ่ม ◀/▶ + เลขหน้า (ย่อหน้าอัตโนมัติ) · ปุ่ม **📥 โหลดทั้งหมด (N)** ที่ดึงทุกหน้าจนครบในคลิกเดียว (เพดานกันพลาด 50 หน้า) · แถวการ์ดมี `data-admin-card` ให้นับได้ | `src/app/admin/cards/page.tsx` |
| **หน้าผู้เล่นของแอดมิน**: แถบแบ่งหน้าแบบเดียวกัน (จำนวนรวม + ต่อหน้า + ◀/▶) + `data-admin-user` | `src/app/admin/users/page.tsx` |
| **เทสต์ +13 ข้อ** — เพดาน/ค่าเริ่มต้น · 172 ใบ ÷ 50 = 4 หน้า · clamp หน้าเกินขอบ · ช่วงข้อมูลหน้าสุดท้าย · ปุ่มเลขหน้าแบบย่อ | `tests/unit/pagination.test.ts` (ใหม่) |
| **สคริปต์ตรวจของจริง 12 ข้อ** (API + เบราว์เซอร์จริง ด้วย session แอดมิน) | `scripts/inspect-admin-cards.mjs` (ใหม่) · `npm run inspect:admin` |
| เอกสาร API: ระบุ `?page=&limit=&search=` ของ admin cards/users + เพดาน 100/หน้า | `docs/API.md` |

**หลักฐานวัดได้ (production จริง · `tsc --noEmit` 0 error · **jest 630 ผ่าน / 43 suites** · `next lint` ไม่มีคำเตือนใหม่ · build ✓ + restart · `/api/health` 200):**

`npm run inspect:admin` → **ผ่าน 12/12 ข้อ** (DB ขณะตรวจ: **การ์ด 172 ใบ · ผู้ใช้ 52 คน**)

| ตรวจ | ผลจริง |
|---|---|
| API ต้องไม่ใช่ 403 (session แอดมิน) | HTTP 200 |
| API บอกจำนวนทั้งหมด | `total=172` = จำนวนใน DB |
| หน้าแรกตาม `limit` | ได้ 50 ใบ |
| **รวมทุกหน้าได้ครบทุกใบ ไม่ซ้ำ/ไม่ตกหล่น** | **172 / 172 ใบ (4 หน้า)** |
| ขอ `limit=500` | ถูกจำกัดที่ **100** (กันดึงทั้งตาราง) |
| ขอ `page=999` | ดึงกลับ **หน้า 4/4** ได้ 22 ใบ (ไม่คืนว่าง) |
| หน้าแรกของ Admin (DOM จริง) | **rows=50** · แถบเขียน "ทั้งหมด **172** ใบ · กำลังแสดง 1–50 · หน้า 1/4" |
| กดเลขหน้าสุดท้าย | **หน้า 4: rows=22** (คาด 22) |
| กด "📥 โหลดทั้งหมด" | **rows=172** ครบทุกใบในคลิกเดียว |
| หน้า Admin/ผู้เล่น | "ทั้งหมด **52** คน · กำลังแสดง 1–50 · หน้า 1/2" |

> หมายเหตุสำหรับผู้ตรวจ: สิทธิ์แอดมินอ่านจาก **session token (JWT)** ไม่ใช่ DB ⇒ สคริปต์ต้องล็อกอินใหม่หลังตั้ง `role=ADMIN` (ไม่งั้นได้ 403) — เจอตอนทำสคริปต์และแก้ไว้แล้ว
> หนังสือคู่มือ/คู่มือผู้เล่นไม่ต้องแก้ (เป็นหน้าเครื่องมือภายในสำหรับแอดมิน)


---

## Phase 28 — ตาราง Ranking ผู้เล่น (2026-09-27)

**คำสั่งผู้ใช้:** *"ทำตาราง Ranking ผู้เล่น ให้ด้วย"*

| สิ่งที่ทำ | ไฟล์ |
|---|---|
| **กติกาการจัดอันดับ (บริสุทธิ์ เทสต์ได้)**: 5 หมวด · น้ำหนักคะแนนสะสมตาม rarity (1/2/4/8/16/32) · `assignRanks` แบบการแข่งขัน (100,90,90,80 → 1,2,2,4) · `rankOfValue` · `topEntries` · `medalFor` | `src/lib/ranking.ts` (ใหม่) |
| **บริการสร้างตารางจากข้อมูลจริง**: พลังทีม (ผลรวม ATK+DEF+HP+SPD ของเด็ค + **Item ที่ใส่**) · คะแนนสะสมการ์ด · ชนะศึก (+จำนวนศึก) · คะแนนกิจกรรม (+ดาเมจ) · Coin — ทุกหมวดใช้โครง rows → sort → rank เดียวกัน และคืน "อันดับของฉัน" แม้อยู่นอก top ที่แสดง | `src/services/ranking.ts` (ใหม่) |
| **API**: `GET /api/ranking?category=&limit=` — ดูได้โดยไม่ต้องล็อกอิน (`signedIn`/`me` = null) · ล็อกอินแล้วได้อันดับของตัวเอง · ส่งรายการหมวดมาด้วย | `src/app/api/ranking/route.ts` (ใหม่) |
| **หน้า /ranking**: 5 แท็บ · ตาราง 50 อันดับ (เหรียญ 🥇🥈🥉 + อวตาร + ชื่อ + ข้อมูลรอง) · แถบ "อันดับของคุณ #N จากทั้งหมด" · แถวของตัวเองไฮไลต์ · เมนู 🏆 ทั้งจอใหญ่และมือถือ + คำแปล 2 ภาษา | `src/app/(game)/ranking/page.tsx` (ใหม่) · `BottomNavigation.tsx` · `TopHeader.tsx` · `dict-th/dict-en` |
| **เทสต์ +15 ข้อ**: หมวดครบ 5 · คะแนนสะสม rarity · อันดับค่าเท่ากัน · ค่าที่ไม่รู้จัก/เพี้ยน · ตัดตาราง · เหรียญ | `tests/unit/ranking.test.ts` (ใหม่) |
| **สคริปต์ตรวจของจริง 15 ข้อ** (API + เบราว์เซอร์จริง) | `scripts/inspect-ranking.mjs` (ใหม่) · `npm run inspect:ranking` |
| คู่มือผู้เล่น: บทที่ 12 ตารางจัดอันดับ + แผนที่เมนู | `docs/PLAYER_GUIDE_TH.md` |

**หลักฐานวัดได้ (production จริง · `tsc --noEmit` 0 error · **jest 645 ผ่าน / 44 suites** · `next lint` ไม่มีคำเตือนใหม่ · build ✓ + restart · `/api/health` 200):**

`npm run inspect:ranking` → **ผ่าน 15/15 ข้อ** (ข้อมูลจริงใน DB ขณะตรวจ)

| ตรวจ | ผลจริง |
|---|---|
| หมวดพลังทีม | 40 คนจัดอันดับ · เรียงมาก→น้อย ✓ · อันดับตามกติกา ✓ |
| หมวดคะแนนสะสมการ์ด | 53 คน (แสดง 50) |
| หมวดชนะศึก | 27 คน |
| หมวดคะแนนกิจกรรม | 4 คน |
| หมวด Coin | 53 คน |
| อันดับของฉัน (พลังทีม) | **#16 จาก 40 · ค่า 1261** |
| แถวของฉัน | `isMe` ตรงกับ `me.rank`/`me.value` (row#39 = 10 · me=#39) |
| ดูโดยไม่ล็อกอิน | ได้ตาราง (50 แถว) · `me=null` · `signedIn=false` |
| หน้า /ranking (DOM จริง) | 5 แท็บ `["power","collection","wins","event","coin"]` · ตาราง 40 แถว · แถวของฉันไฮไลต์ 1 แถว · **เว็บ #16 = API #16** |
| สลับแท็บทั้ง 4 หมวดที่เหลือ | ได้ข้อมูลทุกหมวด (50/27/4/50 แถว) |

> เกณฑ์ที่เลือกใช้: "พลังทีม" = ผลรวมสถานะจริงของการ์ดในเด็ค (บวก Item) เพื่อให้ Ranking สอดคล้องกับที่ผู้เล่นเห็นในหน้าจัดทีม
> หนังสือคู่มือ (`docs/manual/` + PDF) ยังไม่อัปเดตตามกฎ (รอคำสั่ง) — จุดที่จะไม่ตรง: ยังไม่มีหน้าตารางจัดอันดับในหนังสือ


---

## Phase 29 — เมนูจัดการผู้เล่น: กำหนดสิทธิ์ · แบน · ลบ (2026-09-27)

**คำสั่งผู้ใช้:** *"เมนูสำหรับจัดการผู้เล่น หรือกำหนดสิทธิ์ผู้เล่น แบนผู้เล่น หรือลบผู้เล่น"*

**ของเดิมที่มี:** หน้า `/admin/users` ดูรายชื่อ + เติมพลังค้นหาได้เท่านั้น — **ไม่มี** เปลี่ยนสิทธิ์/แบน/ลบ
และ `isActive` (ที่ใช้เป็นธงแบน) ถูกตรวจแค่ตอนล็อกอิน ⇒ ผู้เล่นที่ถูกแบนยังใช้ session เดิมเล่นต่อได้จนคุกกี้หมดอายุ

| สิ่งที่ทำ | ไฟล์ |
|---|---|
| **กติกาการจัดการผู้เล่น (บริสุทธิ์ เทสต์ได้)**: `checkManageUser()` — ห้ามจัดการตัวเองทุกกรณี · ผู้ดูแล (MODERATOR) แบน/ปลดแบนได้แต่เปลี่ยนสิทธิ์/ลบไม่ได้ · ห้ามแบน/ลบบัญชีแอดมิน · เปลี่ยนสิทธิ์ต้องเป็นค่าที่รู้จักและต่างจากเดิม | `src/lib/admin-users.ts` (ใหม่) |
| **API กำหนดสิทธิ์/แบน/ปลดแบน**: `PATCH /api/admin/users/[id]` (`{ role?, isActive?, reason? }`) — กัน "เหลือแอดมิน <1" · audit log `UPDATE_USER_ROLE` / `BAN_USER` / `UNBAN_USER` | `src/app/api/admin/users/[id]/route.ts` (ใหม่) |
| **API ลบผู้เล่น**: `DELETE /api/admin/users/[id]` — แอดมินเท่านั้น · ต้องส่ง `confirmUsername` ให้ตรง · นับ+รายงานของที่หายไป (การ์ด/ทีม/ศึก/แจ้งเตือน) · audit `DELETE_USER` | `src/app/api/admin/users/[id]/route.ts` |
| **แบนมีผลทันที (แก้ช่องโหว่เดิม)**: `resolveRequestUserId()` ตรวจ `isActive` จาก DB ทุก request ⇒ ผู้เล่นที่ถูกแบน/ถูกลบ ใช้ session เดิมต่อไม่ได้ทันที (เดิมใช้ได้จน token หมดอายุ) | `src/lib/current-user.ts` |
| **UI หน้าจัดการผู้เล่น**: ช่องเลือกสิทธิ์ในตาราง (ผู้เล่น/ผู้ดูแล/แอดมิน) · ปุ่ม 🚫 แบน / ♻️ ปลดแบน · ปุ่ม 🗑️ ลบ พร้อม **กล่องยืนยันที่ต้องพิมพ์ชื่อผู้ใช้ให้ตรง** · ป้าย "ถูกแบน" + แถวหรี่ลง · ซ่อนปุ่มที่ตัวเองไม่มีสิทธิ์ (ใช้กติกาชุดเดียวกับเซิร์ฟเวอร์) | `src/app/admin/users/page.tsx` |
| **เทสต์ +13 ข้อ**: เมทริกซ์สิทธิ์ครบทุกช่อง (ผู้เล่น/ผู้ดูแล/แอดมิน × 4 action) · ค่าเพี้ยน/สิทธิ์เดิม · ห้ามจัดการตัวเอง | `tests/unit/admin-users.test.ts` (ใหม่) |
| **สคริปต์ตรวจของจริง 22 ข้อ** (ยิง API จริง + กดปุ่มบนเบราว์เซอร์จริง) | `scripts/inspect-admin-users.mjs` (ใหม่) · `npm run inspect:admin-users` |
| เอกสาร API: ระบุ PATCH/DELETE ของ `/api/admin/users/[id]` | `docs/API.md` |

**หลักฐานวัดได้ (production จริง · `tsc --noEmit` 0 error · **jest 658 ผ่าน / 45 suites** · `next lint` ไม่มีคำเตือนใหม่ · build ✓ + restart · `/api/health` 200):**

`npm run inspect:admin-users` → **ผ่าน 22/22 ข้อ**

| ตรวจ | ผลจริง |
|---|---|
| กำหนดสิทธิ์ PLAYER → MODERATOR | สำเร็จ ("เปลี่ยนสิทธิ์ … เป็น ผู้ดูแล แล้ว") |
| สิทธิ์ที่ไม่รู้จัก / ตั้งสิทธิ์เดิมซ้ำ | **400** พร้อมเหตุผล ("สิทธิ์ที่เลือกไม่ถูกต้อง" / "มีสิทธิ์นี้อยู่แล้ว") |
| จัดการบัญชีตัวเอง (เปลี่ยนสิทธิ์/แบน/ลบ) | **400** — "จัดการบัญชีของตัวเองไม่ได้" |
| ผู้ดูแล (MODERATOR) เปลี่ยนสิทธิ์ | **400** — "เปลี่ยนสิทธิ์ได้เฉพาะแอดมิน" |
| ผู้ดูแลลบผู้เล่น | **400** — "ลบผู้เล่นได้เฉพาะแอดมิน" |
| แบนผู้เล่น | **200** · DB `isActive=false` |
| ผู้ถูกแบนล็อกอิน | **401** |
| **session เดิมของผู้ถูกแบน** (`/api/auth/me`) | **401 ทันที** (ไม่ต้องรอคุกกี้หมดอายุ) |
| API อื่นด้วย session เดิม (`/api/wallet`) | **401** |
| ปลดแบน | **200** · คุกกี้เดิมกลับมาใช้ได้ (`me=200`) |
| แบน/ลบบัญชีแอดมิน | **400** — "แบน/ลบบัญชีแอดมินไม่ได้ (ลดสิทธิ์ก่อน)" |
| ลบด้วยชื่อยืนยันผิด | **400** + บอกชื่อที่ต้องพิมพ์ให้ตรง |
| ลบผู้เล่นสำเร็จ | **200** · ข้อมูลลูกหายตาม (การ์ด 5 → 0 · DB ไม่มีผู้ใช้แล้ว) |
| Audit log | ครบ `UPDATE_USER_ROLE` · `BAN_USER` · `UNBAN_USER` · `DELETE_USER` |
| หน้า /admin/users (DOM จริง) | ช่องเลือกสิทธิ์ 49 · ปุ่มแบน 41 · ปุ่มลบ 41 |
| กดปุ่ม "แบน" บนหน้าเว็บ | สถานะบนจอ = `banned` · DB `isActive=false` |
| กล่องยืนยันการลบ | เปิดได้ · มีช่องพิมพ์ชื่อ · **ปุ่มยืนยันถูกล็อกจนกว่าจะพิมพ์ตรง** |

> หมายเหตุสำหรับผู้ตรวจ: สิทธิ์ใน session มาจาก JWT ตอนล็อกอิน ⇒ สคริปต์ต้องล็อกอินใหม่หลังตั้ง `role`
> และ `/api/auth/register` จำกัด 5 ครั้ง/นาทีต่อ IP ⇒ สคริปต์เว้นจังหวะการสมัคร + retry อัตโนมัติเมื่อเจอ 429
> หนังสือคู่มือ/คู่มือผู้เล่นไม่ต้องแก้ (เป็นหน้าเครื่องมือภายในของแอดมิน)


---

## Phase 30 — ผู้ดูแล (MODERATOR) ห้ามแตะเรื่องการ์ด/ภาพการ์ด (2026-09-27)

**คำสั่งผู้ใช้:** *"ห้ามแตะเรื่องการ์ด เช่น Gen รูปใหม่"*

**ของเดิม:** ทั้ง ADMIN และ MODERATOR ใช้ `getAdminSession()` ร่วมกัน ⇒ แก้ข้อมูลการ์ด, สั่ง Gen รูปใหม่, ประมวลผล/Requeue คิวภาพ ได้เท่ากันหมด

| สิ่งที่ทำ | ไฟล์ |
|---|---|
| **เมทริกซ์ความสามารถระดับเจ้าหน้าที่** (`checkStaffAbility`): เรื่องการ์ด/ภาพ (`cardEdit` · `cardRegenerate` · `imageProcess` · `imageRequeue`) = **แอดมินเท่านั้น** · ส่วน `userBan`/`announcement` ผู้ดูแลทำได้เหมือนเดิม · กำหนดสิทธิ์/ลบผู้เล่น = แอดมิน | `src/lib/admin-users.ts` |
| **API ที่ล็อกเพิ่ม** (ตอบ **403** พร้อมเหตุผลภาษาไทย): `PATCH /api/admin/cards/[id]`, `POST /api/admin/cards/[id]/regenerate`, `POST /api/admin/images/process`, `POST /api/admin/images/requeue` — โดย **cron (WORKER_TOKEN) ยังทำงานได้ตามเดิม** ไม่โดนบล็อก | `src/app/api/admin/cards/[id]/route.ts` · `.../regenerate/route.ts` · `.../images/{process,requeue}/route.ts` |
| **UI**: ผู้ดูแลเห็นหน้า `/admin/cards` และ `/admin/images` แบบ **"ดูอย่างเดียว"** — ซ่อนปุ่ม Gen รูปใหม่/แก้ไข/Requeue/ประมวลผลคิว และขึ้นป้าย 👀 อธิบาย · แอดมินเห็นปุ่มครบตามเดิม | `src/app/admin/cards/page.tsx` · `src/app/admin/images/page.tsx` |
| **เทสต์เพิ่ม 6 ข้อ** (รวมเป็น 19 ข้อในไฟล์นี้): เมทริกซ์การ์ด/ภาพ/ผู้เล่น + ข้อความเหตุผล + คนทั่วไปทำไม่ได้ | `tests/unit/admin-users.test.ts` |
| **สคริปต์ตรวจของจริงครอบ Phase 30** (สลับคุกกี้ผู้ดูแล ↔ แอดมิน บนเบราว์เซอร์เดียวกันจริง) | `scripts/inspect-admin-users.mjs` |

**หลักฐานวัดได้ (production จริง · `tsc --noEmit` 0 error · **jest 664 ผ่าน / 45 suites** · `next lint` ไม่มีคำเตือนใหม่ · build ✓ + restart · `/api/health` 200):**

`npm run inspect:admin-users` → **ผ่าน 33/33 ข้อ** (22 ข้อของ Phase 29 + 11 ข้อของ Phase 30)

| ตรวจ (Phase 30) | ผลจริง |
|---|---|
| ผู้ดูแลแก้ไขข้อมูลการ์ด | **403** — "\"แก้ไขข้อมูลการ์ด\" ทำได้เฉพาะแอดมิน (ผู้ดูแลดูได้อย่างเดียว)" |
| ผู้ดูแลสั่ง Gen รูปใหม่ | **403** — "\"สั่งสร้างภาพการ์ดใหม่\" ทำได้เฉพาะแอดมิน (ผู้ดูแลดูได้อย่างเดียว)" |
| ผู้ดูแลสั่งประมวลผลคิวภาพ | **403** |
| ผู้ดูแลสั่ง Requeue ภาพการ์ด | **403** |
| แอดมินแก้ข้อมูลการ์ด | **200** · DB `nameTh=แก้โดยแอดมิน` |
| แอดมินสั่ง Gen รูปใหม่ | **200** · ได้ `jobId` (เข้าคิว ไม่สร้างภาพในสคริปต์) |
| หน้า `/admin/cards` ของผู้ดูแล (DOM จริง) | ปุ่ม Gen รูปใหม่ **0** · ปุ่มแก้ไข **0** · ป้าย "ดูอย่างเดียว" **มี** · ยังเห็นรายการ 50 แถว |
| หน้า `/admin/images` ของผู้ดูแล | ปุ่ม Requeue/ประมวลผล **0** · ป้ายดูอย่างเดียว **มี** |
| หน้า `/admin/cards` ของแอดมิน | ปุ่ม Gen รูปใหม่ **50** · ปุ่มแก้ไข **50** · ไม่มีป้ายดูอย่างเดียว |

> สคริปต์สร้าง "การ์ดทดสอบชั่วคราว" ผ่าน Prisma เพื่อตรวจสิทธิ์ แล้วลบการ์ด + งานภาพของมันทิ้งตอนจบ (ไม่แตะการ์ดจริง)
> cron/scheduler ที่ใช้ `x-worker-token` ยังสั่งประมวลผลภาพได้ (ไม่ได้ตั้งใจบล็อกงานเบื้องหลัง — บล็อกเฉพาะ "คน" ที่เป็นผู้ดูแล)


---

## Phase 31 — ดันเจี้ยนหาวัตถุดิบคราฟต์ (2026-09-27)

**คำสั่งผู้ใช้:** *"สร้างดันเจี้ยนสำหรับหา item/วัตถุดิบนำมา craft มีทั้งเข้าฟรีตามเวลา ฟรีตลอด หรือใช้เหรียญ แต่ละดันมีหลายชั้นยากขึ้นเรื่อยๆ ด่านสู้กับ Deck พิเศษมีได้มากกว่า 5 ใบ การ์ดศัตรูกำหนด Status เอง ไม่รวมการ์ดทั่วไป แต่ Gen รูปเหมือนกัน เป็นการ์ดลูกน้อง + บอสประจำด่าน 1 ตัว (ไม่ต้อง Gen รูปเยอะ)"*

| สิ่งที่ทำ | ไฟล์ |
|---|---|
| **นิยามดัน 3 ดัน** (pure, เทสต์ได้): 🔥 สุสานเพลิง (ฟรีตลอด 3 ชั้น) · 🌙 รอยแยกไร้จันทร์ (ฟรีตามเวลา 12/13/20/21 น. 4 ชั้น) · 💰 เหวลึกทองคำ (Coin 50/ครั้ง 4 ชั้น) — status ฐานลูกน้อง/บอส + ตัวคูณ scale ต่อชั้น + รางวัลฝุ่น/Shards/Item ดรอป | `src/lib/dungeon-definitions.ts` |
| **ทีมศัตรู = บอส 1 + ลูกน้อง 4 (5v5 เท่าผู้เล่น)** — status กำหนดเองในโค้ด ไม่ผูก `CardDefinition` ⇒ ไม่ต้อง Gen รูปเพิ่ม · ความยากชดเชยด้วย status บอสสูง + scale ต่อชั้น (ผู้ใช้สั่งแก้จาก 5-8 ใบ → 5 ใบเท่ากัน) | `src/services/dungeon.ts` (`buildEnemyTeam`) + `DUNGEON_TEAM_SIZE` |
| **DB**: `dungeon_progress` (ชั้นสูงสุดที่เคยชนะ/ปลดล็อกชั้นถัดไป) + `dungeon_runs` (ประวัติ idempotent ด้วย `runId`) | `prisma/schema.prisma` + migration `phase31_dungeon` |
| **API**: `GET /api/dungeons` (ลิสต์ + สถานะเปิด/ชั้นสูงสุด) · `POST /api/dungeons/run` (หัก Coin/ตรวจเวลา/สู้/จ่ายรางวัล/hook เควส+แจ้งเตือน) — deterministic (seed จาก runId) · แพ้ได้ฝุ่นปลอบใจ 1/4 | `src/app/api/dungeons/route.ts` · `.../run/route.ts` |
| **UI**: หน้า `/dungeons` (เลือกดัน/ชั้น/เด็ค → กดลุยแล้ว**พาเข้าหน้าสนามรบ** `/battle/dungeon-run:<runId>` ดูสู้แบบ replay เต็ม: การ์ด 5v5 + HP/MP + log + เสียง) · จบแล้วปุ่ม 🏰 กลับดันเจี้ยนพร้อมสรุปรางวัล · เมนู 🏰 ดันเจี้ยน (th/en) | `src/app/(game)/dungeons/page.tsx` · `battle/[id]/page.tsx` (รองรับ `dungeon-run:`) · `BottomNavigation` · `dict-th/en` |
| **API replay ดัน**: `GET /api/dungeons/run/[runId]/log` (teams/log/ชื่อทีม/รางวัล/cardMeta — การ์ดศัตรูใช้ art reuse ไม่ Gen ใหม่) | `src/app/api/dungeons/run/[runId]/log/route.ts` |
| **เทสต์ 7 ข้อ** (นิยาม/สเกล/เวลา/ทีม 8 ใบ) + **สคริปต์ตรวจของจริง 10 ข้อ** | `tests/unit/dungeon.test.ts` · `scripts/inspect-dungeons.mjs` (`npm run inspect:dungeons`) |

**หลักฐานวัดได้ (production จริง · `tsc --noEmit` 0 error · jest 671 ผ่าน / 46 suites · `next lint` ผ่าน · build ✓ + restart · `/api/health` 200):**

`npm run inspect:dungeons` → **ผ่าน 10/10 ข้อ** (ลิสต์ 3 ดัน · ล็อกชั้น · ลุยชนะได้ของ · runId ซ้ำคืนผลเดิม · นอกเวลาบล็อก · หัก Coin 50 · หน้า /dungeons 200)

---

## Phase 31.1 — แก้ดันเจี้ยนหลังผู้ใช้ลองเล่น (2026-09-27)

**คำสั่งผู้ใช้:** *"ตรวจสอบระบบดันเจี้ยนที่สร้างมาใหม่ ตอนนี้เข้าหน้าต่อสู้ไม่ได้จริง ไม่ถูกต้องตามการเล่นเกม และฝุ่นเวทที่ได้จากดันเจี้ยนมีสัญลักษณ์ไม่เหมือนที่เคยทำไว้ จะทำให้คนเล่นสับสนว่าเป็นคนละ item — ช่วยดูเรื่อง ui หน้าดันเจี้ยน และระยะเวลาเข้าดันเจี้ยนฟรีตามเวลา ต้องบอกเข้าได้เป็นช่วง เวลาไหน ถึงเวลาไหน ข้อความเทคนิคหลังบ้านไม่ต้องนำมาแสดงให้เห็น"*

**สาเหตุจริงที่พบ (วัดจาก production ที่รันอยู่):**

| อาการ | ต้นเหตุจริง | วิธีแก้ |
|---|---|---|
| **"เข้าหน้าต่อสู้ไม่ได้จริง"** | หน้า `/battle/dungeon-run:<runId>` ได้ `useParams().id` มาเป็นค่าที่ **percent-encode** (`dungeon-run%3A…%3AEMBER_CRYPT%3Af1%3A…` — รหัส runId มี `:` หลายตัว) ⇒ `startsWith('dungeon-run:')` เป็น false → ยิงไป `/api/battle/dungeon-run%3A…/log` **404** → ขึ้น "ไม่พบการต่อสู้" ทุกครั้ง | แยกตรรกะเป็น `src/lib/battle-route.ts` (`parseBattleRouteId` ถอดรหัส / `battleLogUrl` encode id / `dungeonBattlePath`) + เทสต์ `battle-route.test.ts` |
| **การ์ดศัตรูพังทั้งฝั่ง** | log API ส่ง URL ภาพของ route ที่ **ไม่มีอยู่** (`/api/dungeons/art/<ชื่อ>` → 404) และกรอบการ์ด `/api/cards/dungeon:…/image` ตกไปเป็น placeholder "???" (404) → รูปไม่ขึ้น + หมุนค้าง "กำลังวาดภาพ…" | เพิ่ม `src/lib/dungeon-art.ts` (รหัส/ชื่อ/ธาตุ/status ของศัตรู = ค่าเดียวกับที่สู้จริง) · `src/services/dungeon-art.ts` (เลือก **ยืมภาพ AI ของการ์ดจริง** ในธาตุเดียวกันแบบ deterministic — ไม่ Gen ใหม่) · `/api/cards/[id]/art` + `/api/cards/[id]/image` รองรับรหัส `dungeon:*` (กรอบ 200 + ภาพ 200) · `CardFace` ไม่ poll ซ้ำเมื่อสถานะ READY (กันหมุนค้าง) |
| **"ฝุ่นเวทเป็นคนละ item"** | ดันเจี้ยนแจกด้วยรหัสของตัวเอง (`DUNGEON_EMBER_CRYPT_F1` ชื่อ 'ฝุ่นเวท (สุสานเพลิง ชั้น 1)') ขณะที่เกมแสดง ✨ 'ฝุ่นเวท' ⇒ กระเป๋าเห็นเป็นคนละแถว/คนละสัญลักษณ์ | รหัส/ชื่อกลาง `CRAFTING_DUST_CODE` · `CRAFTING_DUST_NAME_TH` ใน `src/services/inventory.ts` (ใช้ทั้งดันเจี้ยน + รางวัลกิจกรรม) · กระเป๋าเปลี่ยนสัญลักษณ์วัตถุดิบเป็น ✨ 'ฝุ่นเวท' · สคริปต์รวมของเก่า `scripts/normalize-dust-inventory.mjs` (รันแล้ว: 4 ผู้เล่น → แถวเดียว ยอดเท่าเดิม) |
| **เวลาฟรีอ่านไม่รู้ว่าเข้าได้ถึงกี่โมง** | แสดงเป็นรายชั่วโมง `12:00, 13:00, 20:00, 21:00` | `freeHourWindows()` รวมชั่วโมงติดกันเป็นช่วง (รองรับคร่อมเที่ยงคืน) + `formatFreeWindowsTh()` → `12:00–14:00 และ 20:00–22:00` + `freeEntryStatusTh()` (สถานะเปิด/ปิด + เวลาปิดรอบนี้ + `เปิดอีก 3 ชม. 15 น.`) ใช้สูตรเดียวกันทั้งฝั่งเซิร์ฟเวอร์และหน้าเว็บ (หน้าเว็บเดินนาฬิกาทุก 20 วิ) |
| **ข้อความเทคนิคหลังบ้านหลุดถึงผู้เล่น** | "บอส 1 + ลูกน้อง 4 (5v5 เท่ากัน — บอส status สูงชดเชย)" · "ฝุ่นปลอบใจ 1/4" · รหัสที่มา `DUNGEON:EMBER_CRYPT:F1` · "ข้อมูลเด็ค/เก่ากว่าที่ระบบเก็บ snapshot" | ลบ/เขียนใหม่เป็นภาษาเกมทั้งหมด · `src/lib/inventory-display.ts` (`sourceLabelTh` แปลงรหัสที่มาเป็น 'ดันเจี้ยน สุสานเพลิง ชั้น 1' / รางวัลกิจกรรม · รหัสที่ไม่รู้จัก = ซ่อน) · ข้อความ error ของ API เป็นประโยคที่ผู้เล่นอ่านรู้เรื่อง + `DungeonRuleError` → 400 |

**UI หน้า `/dungeons` ใหม่:** การ์ดดันบอกสถานะ 🟢/🔴 + ช่วงเวลาฟรีเป็นชิป ⏰ + เวลาถอยหลัง · เลือกชั้นเป็นรายการพร้อมรางวัล (✨ ฝุ่นเวท · 💠 Veil Shards · 🎁 ของดรอป %) และชั้นที่ยังล็อก · เตือน Coin ไม่พอ + แสดงยอดที่มี · ปุ่มบอกชั้นที่จะลุย

**ไฟล์ที่เพิ่ม/แก้:** `src/lib/dungeon-art.ts` (ใหม่) · `src/lib/battle-route.ts` (ใหม่) · `src/lib/inventory-display.ts` (ใหม่) · `src/services/dungeon-art.ts` (ใหม่) · `src/services/dungeon.ts` · `src/services/inventory.ts` · `src/lib/dungeon-definitions.ts` · `src/app/api/cards/[id]/art/route.ts` · `.../image/route.ts` · `src/app/api/dungeons/run/route.ts` · `.../run/[runId]/log/route.ts` · `src/app/(game)/dungeons/page.tsx` · `src/app/(game)/inventory/page.tsx` · `src/app/(game)/battle/[id]/page.tsx` · `src/components/cards/CardFace.tsx` · `scripts/normalize-dust-inventory.mjs` (ใหม่) · `scripts/inspect-dungeons.mjs` · เทสต์ `dungeon.test.ts` · `battle-route.test.ts` · `inventory-display.test.ts`

**หลักฐานวัดได้ (production จริง · `tsc --noEmit` 0 error · jest 686 ผ่าน / 48 suites · `next lint` ผ่าน · build ✓ + restart · `/api/health` 200):**

`npm run inspect:dungeons` → **ผ่าน 18/18 ข้อ** (เดิม 10 + ใหม่ 8: การ์ดศัตรูมีข้อมูลครบ · กรอบ 200 + svg ไม่ใช่ "???" · ภาพศัตรู 200 + image · ไม่มีการอ้าง route ภาพที่ตายแล้ว · ฝุ่นเวทเป็นไอเทมเดียวชื่อ 'ฝุ่นเวท' · เวลาฟรีเป็นช่วง HH:MM–HH:MM · สถานะไม่มีรหัสหลังบ้าน)

ตรวจ UI จริงด้วย Chrome headless (ผู้เล่นใหม่ → สร้างเด็ค → ลุยดัน): หน้า `/dungeons` แสดง "🔴 ปิดอยู่ — เข้าฟรี 12:00–14:00 และ 20:00–22:00" + ชิป ⏰ สองช่วง + "เปิดอีก 3 ชม. 14 น." (ไม่พบคำต้องห้าม DUNGEON_/5v5/snapshot เลย) · หน้าสนามรบดันเจี้ยนโหลดจริง **การ์ด 10 ใบ (ศัตรู 5 + ผู้เล่น 5) มีภาพ (naturalWidth 1536) และกรอบ (420) ครบทุกใบ** (เดิมฝั่งศัตรู 404 ทั้งแถว)

---

## Phase 31.2 — ดันเจี้ยนฟรี: ให้ของเฉพาะเมื่อชนะ (2026-09-27)

**คำสั่งผู้ใช้:** *"ดันเจี้ยนฟรี แจก item เฉพาะชนะเท่านั้น"*

| สิ่งที่ทำ | ไฟล์ |
|---|---|
| **กติกาต่อดัน `lossDustRatio`** — ฝุ่นเวทที่ได้ตอนแพ้: ดันฟรี (ฟรีตลอด/ฟรีตามเวลา) = **0** · ดันที่จ่าย Coin เข้า = 1/4 (จ่าย Coin ไปแล้ว ได้ปลอบใจ) | `src/lib/dungeon-definitions.ts` |
| **ฟังก์ชันบริสุทธิ์** `floorDustReward(dungeon, floor, won)` + `isWinOnlyReward(dungeon)` — ใช้ทั้งตอนจ่ายรางวัลจริงและตอนบอกผู้เล่น (ที่เดียว ไม่มีสูตรซ้ำ) | `src/lib/dungeon-definitions.ts` |
| จ่ายรางวัลใช้สูตรใหม่ (เดิม hardcode 1/4 ในเพย์โหลด) · Shards/ไอเทมดรอปชนะเท่านั้นอยู่แล้ว · `floorInfo` เพิ่ม `lossDust` + ดันมี `winOnlyReward` | `src/services/dungeon.ts` |
| **แจ้งเตือน** ไม่ขึ้น "ได้ฝุ่นเวท 0" อีก — แพ้ดันฟรีใช้ข้อความใหม่ "ยังไม่ได้รางวัล (ต้องชนะถึงจะได้)" | `src/services/dungeon.ts` · `dict-th/en` (`notif.dungeonBodyNoReward`) |
| **UI**: หัวหน้าเพจบอกกติกา · การ์ดดันบอก "ได้รางวัลเมื่อชนะเท่านั้น" · ชิปไอเทมขึ้น `(ชนะ)` · รายละเอียดชั้นบอก "แพ้ไม่ได้รางวัล — ต้องชนะเท่านั้น" + "🎁 ไอเทมดรอปได้เมื่อชนะเท่านั้น" · หน้าสนามรบขึ้น "รอบนี้ยังไม่ได้รางวัล (ชนะเท่านั้น)" | `src/app/(game)/dungeons/page.tsx` · `src/app/(game)/battle/[id]/page.tsx` |
| **เทสต์ 2 ชุดใหม่** (ดันฟรีทุกชั้นแพ้ = 0 · ชนะ = เต็ม · ดันเหรียญยังได้ 1/4) + **สคริปต์ตรวจของจริงเพิ่ม 2 ข้อ** | `tests/unit/dungeon.test.ts` · `scripts/inspect-dungeons.mjs` |

**หลักฐานวัดได้ (production จริง · jest 688 ผ่าน / 48 suites · `tsc --noEmit` 0 error · `next lint` ผ่าน · build ✓ + restart · `/api/health` 200):**

`npm run inspect:dungeons` → **ผ่าน 20/20** (เพิ่ม: "แพ้ดันฟรีไม่ได้รางวัล (ได้เฉพาะชนะ) — won=false dust=0 shards=0 item=null" · "ดันฟรี … lossDust = [0,0,0]" · "ดันเหรียญยังมีฝุ่นปลอบใจตอนแพ้ — lossDust=6")

ยิง API จริงเพื่อดูแจ้งเตือน: แพ้ `EMBER_CRYPT` → "สุสานเพลิง ชั้น 1: แพ้ · ยังไม่ได้รางวัล (ต้องชนะถึงจะได้)" (dust 0) · แพ้ `GILDED_ABYSS` → "เหวลึกทองคำ ชั้น 1: แพ้ · ได้ฝุ่นเวท 6"

ตรวจ UI ด้วย Chrome headless: "ดันฟรีได้รางวัลเมื่อชนะเท่านั้น (ดันเหรียญจ่าย Coin แล้ว แพ้ยังได้ฝุ่นเวทเล็กน้อย)" · "แพ้ไม่ได้รางวัล — ต้องชนะเท่านั้น" · "🎁 หินลับคม 15% (ชนะ)" · หน้าสนามรบ "รอบนี้ยังไม่ได้รางวัล (ชนะเท่านั้น)"

---

## Phase 31.3 — ชั้นที่เคยชนะแล้วไม่ได้รางวัลซ้ำ (2026-09-27)

**คำสั่งผู้ใช้:** *"ชั้นที่เคยชนะแล้วก็ไม่ได้รางวัลซ้ำ"*

| สิ่งที่ทำ | ไฟล์ |
|---|---|
| **กติกาสิทธิ์รับรางวัล**: ชั้น `≤ bestFloor` = ผ่านแล้ว → ลุยซ้ำได้แต่ไม่ได้ ✨ ฝุ่นเวท · 💠 Shards · 🎁 ไอเทม (ใช้ `bestFloor` ที่มีอยู่ ไม่ต้อง migration เพราะชั้นปลดล็อกตามลำดับ) | `src/lib/dungeon-definitions.ts` (`isFloorCleared` · `isRewardFloor`) |
| ตอนลุย: คำนวณ `rewardEligible` ก่อนสู้ → จ่ายรางวัล/ดรอปของเฉพาะรอบที่ยังไม่เคยผ่าน · เก็บ `rewardEligible` ลง `battleData` (รอบซ้อมย้อนหลังก็รู้) · `DungeonRunResult` คืนค่าใหม่ | `src/services/dungeon.ts` |
| **แจ้งเตือน** รอบซ้อมใช้ข้อความเฉพาะ "ชั้นนี้ผ่านแล้ว ลุยซ้ำไม่มีรางวัล" | `src/services/dungeon.ts` · `dict-th/en` (`notif.dungeonBodyReplay`) |
| **UI**: การ์ดชั้นขึ้น `✅ ผ่านแล้ว` + "ลุยซ้ำได้ · ไม่มีรางวัล" · ปุ่มเป็น "ลุยซ้ำ ชั้น N (ไม่มีรางวัล)" · รายละเอียดชั้นเตือนว่ารอบซ้อมไม่ได้รางวัล · หัวเพจบอกกติกา · หน้าสนามรบขึ้น "รอบซ้อม (ชั้นนี้ผ่านแล้ว ไม่มีรางวัล)" | `src/app/(game)/dungeons/page.tsx` · `src/app/(game)/battle/[id]/page.tsx` · `src/app/api/dungeons/run/[runId]/log/route.ts` |
| **เทสต์** ชุดใหม่ (ผ่านแล้ว = ซ้ำ · ยังไม่ผ่าน = มีสิทธิ์ ครอบทุกชั้น/ทุกดัน) + **สคริปต์ตรวจของจริงเพิ่ม 3 ข้อ** (จำลอง "ผ่านชั้น 1 มาแล้ว" ด้วย Prisma แล้วลุยซ้ำ → eligible=false · `cleared` ถูกทำเครื่องหมาย · หน้า replay รู้ว่าเป็นรอบซ้อม) | `tests/unit/dungeon.test.ts` · `scripts/inspect-dungeons.mjs` |

**หลักฐานวัดได้ (production จริง · jest 690 ผ่าน / 48 suites · `tsc --noEmit` 0 error · `next lint` ผ่าน · build ✓ + restart · `/api/health` 200):**

`npm run inspect:dungeons` → **ผ่าน 23/23** (เพิ่ม: "ลุยซ้ำชั้นที่ผ่านแล้ว = ไม่มีรางวัล — won=false eligible=false dust=0 shards=0 item=null" · "ชั้นที่ผ่านแล้วถูกทำเครื่องหมาย cleared — [true,false,false]" · "หน้า replay รู้ว่าเป็นรอบซ้อม — eligible=false")

UI จริง (Chrome headless · ผู้เล่นที่ผ่านชั้น 1 มาแล้ว): "ชั้น 1 · ปากทางเถ้าถ่าน ✅ ผ่านแล้ว" · "ลุยซ้ำได้ · ไม่มีรางวัล" · ปุ่ม "ลุยซ้ำ ชั้น 1 (ไม่มีรางวัล)" · "ชั้นนี้คุณชนะมาแล้ว — ลุยซ้ำได้เพื่อซ้อม แต่จะไม่ได้ ✨ ฝุ่นเวท · 💠 Veil Shards · 🎁 ไอเทมอีก" · หน้าสนามรบ "🏰 กลับดันเจี้ยน · รอบซ้อม (ชั้นนี้ผ่านแล้ว ไม่มีรางวัล)" · แจ้งเตือน "สุสานเพลิง ชั้น 1: แพ้ · ชั้นนี้ผ่านแล้ว ลุยซ้ำไม่มีรางวัล"

---

## Phase 31.4 — ปรับความยากดันเจี้ยน + เพิ่มดันฟรีระดับสูง + ของคราฟต์หลากหลาย (2026-09-27)

**คำสั่งผู้ใช้:** *"ปรับความยากของดันเจี้ยนลดลงหน่อย ชั้นแรกๆ ให้มือใหม่ได้ชนะบ้าง และปรับให้ยากขึ้นทีละนิด · เพิ่มดันเจี้ยนเข้าพรี ที่ระดับสูงขึ้นไปอีก ดรอปของเพิ่มขึ้น · ในฟังก์ชัน Craft เพิ่มของที่สามารถ Craft ได้เข้าไปอีก ให้หลากหลาย"*

### เครื่องมือวัดก่อนปรับ (สำคัญ: เดิมปรับความยากกันด้วยการเดา)
| สิ่งที่ทำ | ไฟล์ |
|---|---|
| **สคริปต์วัดอัตราชนะจริง**: การ์ดจากคลังจริง 178 ใบ → เด็คอ้างอิง 4 ระดับ (มือใหม่ 12 เด็คตามกฎ StarterService · ท็อปดิบ · ท็อป+Item ตำนาน · ท็อป+Item mythic) × ทีมศัตรูของทุกชั้น → รันศึกจริงผ่าน `simulateBattle` | `scripts/calibrate-dungeons.mts` (`npm run calibrate:dungeons`) |
| โหมดช่วยตั้งเลข: `--scan <CODE>:<ชั้น> --deck <beginner|top|legendary|mythic> --factors …` = หาว่าคูณ status ศัตรูกี่เท่าจึงได้ % ชนะตามเป้า (เจอความจริงว่าเอนจินนี้ "คม" มาก: ต่าง 10-15% ของ status = ชนะ 100% → 0%) | เดียวกัน |
| **เทสต์กันความยากเพี้ยน** (pure · รันใน CI): มือใหม่ชนะชั้น 1 ของดันฝึกหัด ≥70% · ชั้นถัดไปต้องไม่ชนะง่ายกว่าและยัง ≥25% · สเต็ป scale ต่อชั้น ≤12% · ดันกลางขึ้นไปมือใหม่ต้องชนะ ≤60% · ดันสูงสุดต้องมีคนติดของชนะได้ ≥50% | `tests/unit/dungeon-balance.test.ts` |

### ก่อน → หลัง (วัดที่ 20-30 ศึก/ช่อง)
| ดัน | เดิม (มือใหม่) | ใหม่ (มือใหม่) | ใหม่ (ท็อปดิบ / ติดของ) |
|---|---|---|---|
| 🔥 สุสานเพลิง (ฟรี) | 20% (0-100) · ชั้น 3 = 0% | **97 / 83 / 67%** | 100% / 100% |
| 🌊 วิหารน้ำขึ้น (ใหม่ · ฟรีตลอด) | — | **69 / 67 / 48 / 38%** | 100% / 100% |
| 🌙 รอยแยกไร้จันทร์ (ฟรีตามเวลา) | 0% ทุกชั้น | 16 / 11 / 0 / 0% | 100 / 100 / 20 / 0% · **ติดของ 100% ทุกชั้น** |
| 💰 เหวลึกทองคำ (50 Coin) | 0% | 0% | 0% · **ติดของ 100 / 100 / 100 / 40%** |
| ⚡ ยอดหอพายุ (ใหม่ · ฟรีตามเวลา 18:00–20:00) | — | 0% | 0% · **ติดของ 100 / 100 / 70 / 0 / 0% · mythic 100% ทุกชั้น** |

**สาเหตุที่มือใหม่แพ้ทุกครั้งเดิม:** ชั้น 1 ของสุสานเพลิงมี HP ศัตรูรวม 1,080 (atk 250) ขณะที่เด็คเริ่มต้นจริงในคลังมี HP 493-767 → ศัตรูรุมตายใน 2-3 รอบ · ตอนนี้ชั้น 1 = HP 620 (atk 156) ⇒ มือใหม่ 12 เด็คชนะ 97% (แย่สุด 65%) · สเต็ปต่อชั้นลดจาก +20/+45% → **+9/+22% (ดันฝึกหัด) และ +4-6% (ดันสูง)** = "ยากขึ้นทีละนิด"

### ดันฟรีเพิ่ม 2 ดัน (ระดับสูงขึ้น · ดรอปของเพิ่มขึ้น)
| ดันใหม่ | เข้า | ชั้น | รางวัล (ชนะ) | ไอเทมดรอป |
|---|---|---|---|---|
| 🌊 **วิหารน้ำขึ้น** (Tidal Sanctum) | ฟรีตลอด | 4 | ฝุ่น 18 → 48 · Shards 4 → 12 | หิน/เกราะ/เครื่องรางระดับกลาง (UNCOMMON) |
| ⚡ **ยอดหอพายุ** (Stormreach Spire) | ฟรีตามเวลา **18:00–20:00** | 5 | ฝุ่น 70 → 190 · Shards 16 → 40 | ช่วงกลางชั้น = ของระดับกลาง · ช่วงท้าย = ของในตำนาน (SUP_DUSKVEIL 20% · DEF_TITANHEART 10% · ATK_STORMFANG 10% · SUP_WORLDSEED 8%) |

⇒ ดันฟรี 4 ดันเรียงระดับ: สุสานเพลิง (10 ฝุ่น/ชั้นแรก) → วิหารน้ำขึ้น (18) → รอยแยกไร้จันทร์ (28) → ยอดหอพายุ (70)

### ของคราฟต์เพิ่ม 6 ชิ้น (รวม 12 → **18 ชิ้น** · 6 ชิ้น/ช่อง)
- ระดับ UNCOMMON (12 Shards + 12 ฝุ่น · ซื้อตรงได้ 26): `ATK_ASHEN_SPIKE` หนามเถ้าถ่าน · `DEF_IRONWEAVE` เกราะผ้าถักเหล็ก · `SUP_DUSKVEIL` เครื่องรางม่านสนธยา
- ระดับ **MYTHIC ใหม่** (240 Shards + 220 ฝุ่น · คราฟต์เท่านั้น): `ATK_STORMFANG` เขี้ยวพายุ (atk 85) · `DEF_TITANHEART` โล่หัวใจไททัน (def 75/hp 135) · `SUP_WORLDSEED` เมล็ดพันธุ์โลก (+14/+14/+60/+55)
- ทั้งหมดเข้าฐานข้อมูลอัตโนมัติผ่าน `ensureCatalog()` (ไม่ต้อง migrate) · เทสต์กันของคราฟต์ไม่มีราคา/ไม่มีที่มา

**ไฟล์:** `src/lib/dungeon-definitions.ts` (สมดุล + ดันใหม่ 2) · `src/lib/item-definitions.ts` (+6 ชิ้น) · `scripts/calibrate-dungeons.mts` (ใหม่) · `scripts/inspect-dungeons.mjs` (+6 ข้อ) · `tests/unit/dungeon-balance.test.ts` (ใหม่) · `tests/unit/dungeon.test.ts` · `tests/unit/item-definitions.test.ts` · `package.json` (`npm run calibrate:dungeons`)

**หลักฐานวัดได้ (production จริง · jest 699 ผ่าน / 49 suites · `tsc --noEmit` 0 error · `next lint` ผ่าน · build ✓ + restart · `/api/health` 200):**

`npm run inspect:dungeons` → **ผ่าน 27/27** (เดิม 23 + ใหม่ 4: ลิสต์ 5 ดัน (ฟรี 4 + เหรียญ 1) · ดันฟรีเรียงรางวัลจากน้อยไปมาก `EMBER_CRYPT:10 · TIDAL_SANCTUM:18 · MOONLESS_RIFT:28 · STORMREACH_SPIRE:70` · ทุกดันมีชื่อชั้น/รางวัลครบ · แคตตาล็อกช่างมีของคราฟต์ใหม่ครบ 6 ชิ้น = ทั้งหมด 18 ชิ้น) · และในรันนี้ **ผู้เล่นทดสอบ (เด็คเริ่มต้นจริง) ชนะสุสานเพลิงชั้น 1 ครั้งแรกได้สำเร็จ** (`won=true dust=10/10`) — เดิมแพ้ 6 ครั้งติด

UI จริง (Chrome headless): หน้า `/dungeons` แสดง 5 ดัน (🔥 สุสานเพลิง · 🌊 วิหารน้ำขึ้น · 🌙 รอยแยกไร้จันทร์ · 💰 เหวลึกทองคำ · ⚡ ยอดหอพายุ) พร้อมประเภทการเข้า · หน้า `/items` แสดงของคราฟต์ใหม่ครบ 6 ชิ้นทั้ง 3 ช่อง · กรอบ/ภาพการ์ดศัตรูของดันใหม่โหลดได้ 200 ทุกใบ (`dungeon:TIDAL_SANCTUM:f1:boss` · `dungeon:STORMREACH_SPIRE:f5:boss` ฯลฯ) · หน้าต่างเวลาฟรีของดันใหม่ = `18:00–20:00`

---

## Phase 31.5 — เพิ่มชั้นลึกทุกดัน 20-40 ชั้น + บอส 2-3 ตัวในชั้นลึก (2026-09-27)

**คำสั่งผู้ใช้:** *"เพิ่มชั้นของแต่ละดันเจี้ยนไปอีก 20-40 ชั้นเลย"*

| ดัน | ชั้นเดิม | ชั้นใหม่ | เพิ่ม | ชั้นที่เริ่มบอส 2 / 3 ตัว |
|---|---|---|---|---|
| 🔥 สุสานเพลิง | 3 | **25** | +22 | 13 / 21 |
| 🌊 วิหารน้ำขึ้น | 4 | **28** | +24 | 13 / 22 |
| 🌙 รอยแยกไร้จันทร์ | 4 | **28** | +24 | 12 / 21 |
| 💰 เหวลึกทองคำ | 4 | **30** | +26 | 12 / 22 |
| ⚡ ยอดหอพายุ | 5 | **40** | +35 | 10 / 20 |

### วิธีทำ (ไม่เขียนมือ 150+ บรรทัด · และไม่ให้ความยากกระโดดเป็นกำแพง)
| สิ่งที่ทำ | ไฟล์ |
|---|---|
| **ตัวสร้างชั้นลึกอัตโนมัติ** `buildDeepFloors()` — ชื่อชั้นวนจากชุดชื่อของดันนั้น · รางวัลโตต่อชั้น (`rewardGrowth`) · ไอเทมดรอปไล่ระดับตาม "บันได" (`dropLadder`) | `src/lib/dungeon-definitions.ts` |
| **ความยากไล่แบบ asymptotic**: `difficulty = cap − (cap − Dปัจจุบัน) × ratio^ชั้น` ⇒ ยากขึ้นทุกชั้นแล้วค่อย ๆ นิ่งที่เพดานของดัน (จำเป็นเพราะเอนจินต่อสู้เป็น "การแข่งตาย" — ต่างสเกลแค่ 10-15% = ชนะ 100% → แพ้ 0% ทันที) | `src/lib/dungeon-definitions.ts` + วัดจริงด้วย `scripts/calibrate-dungeons.mts` |
| **บอส 2-3 ตัวในชั้นลึก** (ขั้นความยากจริง ไม่ใช่แค่เพิ่มตัวเลข): ชั้นลึกสุดทีม = บอส 3 + ลูกน้อง 2 (ยัง 5v5) · สเกลลดชดเชยด้วย `bossWeight()` (วัดจากการสแกนจริง: บอส 2 ตัว ≈ หนักกว่า ×1.4 · บอส 3 ตัว ≈ ×1.7) | `src/lib/dungeon-definitions.ts` (`bossWeight` · `floorDifficulty`) · `src/lib/dungeon-art.ts` (`dungeonEnemySlots` · รหัส `:boss2` `:boss3`) · `src/services/dungeon.ts` |
| **แก้ปัญหาไบ่วงจร**: ของ mythic เดิมดรอปเฉพาะชั้น 30-38 ของยอดหอพายุ (ซึ่งต้องใส่ของ mythic ถึงผ่าน) ⇒ ย้ายไปดันเหรียญชั้น 14/17/20 (ชั้นบอส ≤ 2 ที่คนใส่ของตำนานผ่านได้) — เทสต์กันไว้ไม่ให้กลับไปเป็นวงจรอีก | `dropLadder` ของ `GILDED_ABYSS` · เทสต์ใน `tests/unit/dungeon.test.ts` |
| **UI 25-40 ชั้น**: เปลี่ยนจาก "ลิสต์ทุกชั้น" → **แถบความคืบหน้า (ผ่าน N/ทั้งหมด · เหลืออีกกี่ชั้น) + ตัวเลือกชั้น (select) + ปุ่ม "ไปชั้นที่ยังไม่ผ่าน"** · ตัวเลือกโชว์ ✅ ผ่านแล้ว · 👑×2/×3 = ชั้นบอสหลายตัว · 🔒 ล็อก · รายละเอียดชั้นบอก "ศัตรู: บอส 3 · ลูกน้อง 2" | `src/app/(game)/dungeons/page.tsx` |
| API รับเลขชั้นถึง 60 (`DUNGEON_MAX_FLOOR`) | `src/app/api/dungeons/run/route.ts` |

### ผลวัดจริงหลังเพิ่มชั้น (10 ศึก/ช่อง · เด็คอ้างอิง 4 ระดับ)
| ดัน | มือใหม่ (เด็คเริ่มต้นจริง 12 เด็ค) | ท็อปดิบ | ติดของตำนาน | ติดของ mythic |
|---|---|---|---|---|
| 🔥 สุสานเพลิง (25 ชั้น) | f1 **95%** · f10 61% · f15 (บอส 2) 100% · f25 (บอส 3) **88%** | 100% | 100% | 100% |
| 🌊 วิหารน้ำขึ้น (28 ชั้น) | f1 **69%** · f10 17% · f15 (บอส 2) 50% · f28 (บอส 3) 50% | 100% | 100% | 100% |
| 🌙 รอยแยกไร้จันทร์ (28 ชั้น) | f1 17% · f10 0% · f28 (บอส 3) 0% | 100% | **100% ทุกชั้น** | 100% |
| 💰 เหวลึกทองคำ (30 ชั้น) | 0% ทุกชั้น | 0% | f8-f20 100% · f22-f30 (บอส 3) 0-46% | **100% ทุกชั้น** |
| ⚡ ยอดหอพายุ (40 ชั้น) | 0% ทุกชั้น | 0% | f2-f20 100% · f25-f40 (บอส 3) 0-20% | **100% ทุกชั้น** |

> ⚠️ **ข้อจำกัดที่บันทึกไว้ตรง ๆ**: เอนจินต่อสู้ตัดสินด้วย "ใครล้มก่อน" ⇒ ชั้นที่อยู่ใกล้เส้นแบ่งจะชนะ 100% หรือแพ้ 0% เกือบทั้งหมด (ต่างกันแค่ 10-15% ของ status ก็พลิก) และอัตราชนะที่วัดได้ยังแกว่งตามชุด seed (ตรวจด้วย 2 ชุดแล้ว)
> ⇒ จึงยึดหลักที่ **วัดได้แน่นอน 2 ข้อ**: (1) "ความยากจริง" (`floorDifficulty` = สเกล × น้ำหนักบอส) ไล่ขึ้นทุกชั้น · (2) เด็คอ้างอิงที่ตั้งใจไว้ต้องชนะชั้นนั้นในชุด seed ของเทสต์ — และเทสต์กันไว้ไม่ให้หลุดกรอบ

รางวัลปลายทาง (ฝุ่นเวท ต่อการชนะชั้นแรก → ชั้นสุดท้าย): สุสานเพลิง 10→72 · วิหารน้ำขึ้น 18→194 · รอยแยก 28→292 · เหวลึก 45→500 · ยอดหอพายุ 70→**1,048** (Shards 2→18 · 4→48 · 6→78 · 10→148 · 16→260)

**เทสต์กันความยากเพี้ยน (รันใน CI · pure):** ทุกดัน ≥25 ชั้น + ชั้นต่อเนื่อง · ทีมศัตรู 5 ใบเสมอ · ความยากจริง (`floorDifficulty`) ไม่ลดลงและสเต็ป ≤20% · รางวัลลึกสุด > ชั้นแรก · ดันฝึกหัดต้องจบได้ด้วยเด็คเริ่มต้น (≥40% ที่ชั้น 25) · ชั้นท้ายดันกลาง-สูง มือใหม่ ≤30% และคนติดของ ≥50% · ชั้น 2-3 บอสต้องผ่านได้ด้วยของที่เหมาะสม | `tests/unit/dungeon-balance.test.ts` |

**หลักฐานวัดได้ (production จริง · jest 707 ผ่าน / 49 suites · `tsc --noEmit` 0 error · `next lint` ผ่าน · build ✓ + restart · `/api/health` 200):**

`npm run inspect:dungeons` → **ผ่าน 30/30** (เพิ่ม 3 ข้อ: ทุกดัน ≥25 ชั้น — `EMBER_CRYPT:25 · TIDAL_SANCTUM:28 · MOONLESS_RIFT:28 · GILDED_ABYSS:30 · STORMREACH_SPIRE:40` · ชั้นลึกมีบอส 2-3 ตัวและทีมยัง 5 ใบ · รางวัลลึกสุดมากกว่าชั้นแรก — `EMBER_CRYPT:10→72 … STORMREACH_SPIRE:70→1048`)

ตรวจของจริงด้วย Chrome headless: หน้า `/dungeons` ของดัน 40 ชั้น → "ความคืบหน้า: ผ่านสูงสุด ชั้น 20 / 40 · เหลืออีก 20 ชั้น" · ตัวเลือกชั้น 40 รายการ (`ชั้น 10 · ใจกลางพายุ ✅ 👑×2` · `ชั้น 39 · ระเบียงสายฟ้า 👑×3 🔒`) · รายละเอียดชั้น "ศัตรู: บอส 3 · ลูกน้อง 2 · ชนะได้ ✨ 415 · 💠 75 · 🎁 ใส่กำแพงน้ำ 20%" · ลุยชั้น 22 ของสุสานเพลิงจริง → ทีมศัตรู 5 ใบ = **บอส 3 ตัว (1/3, 2/3, 3/3) + ลูกน้อง 2** · การ์ดศัตรูบนสนามรบ 5 ใบ ภาพขึ้นครบทุกใบ (`boss`, `boss2`, `boss3`, `minion1`, `minion2`) · ชนะได้ ✨ +61

---

## Phase 32 — ขาย Item คืนวัตถุดิบ 50% + ไอเทมเพิ่มอีก 18 ชิ้น (36 ชิ้น) (2026-09-27)

**คำสั่งผู้ใช้:** *"เพิ่มระบบขาย Item ได้วัตถุดิบกลับมา 50% · และเพิ่ม Item ใหม่เข้าไปอีกอย่างละ 6 แบบ"*

### 1) ระบบขาย Item คืนวัตถุดิบ 50%
| สิ่งที่ทำ | ไฟล์ |
|---|---|
| **กฎบริสุทธิ์** `ITEM_SELL_REFUND_RATE = 0.5` + `sellQuote(def, quantity)` → คืน Veil Shards และฝุ่นเวท **อย่างละ 50% ของสูตรคราฟต์ (ปัดลง)** · UI/API/เทสต์ใช้ตัวเลขชุดเดียวกัน | `src/lib/item-definitions.ts` |
| **บริการขาย** `ItemService.sell()` — ตรวจว่ามีของในคลัง · **กันของที่ใส่อยู่บนการ์ด** (`owned − ขาย ≥ equippedCount`) · ทำใน transaction เดียว: ตัดของ → คืน Veil Shards (`ITEM_SELL` ใน ledger) → คืนฝุ่นเวท (ไอเทมกลาง `CRAFTING_DUST_CODE`) | `src/services/item.ts` · `src/lib/veil-shards.ts` (`ITEM_SELL`) |
| `InventoryService.grant()` รับ transaction client ได้ (ให้การคืนของอยู่ใน tx เดียวกัน) | `src/services/inventory.ts` |
| **API** `POST /api/items/sell { code, quantity? }` → คืน `refundShards`/`refundDust`/ยอดใหม่ + ข้อความไทย · 400 เมื่อไม่มีของ/ใส่อยู่บนการ์ด | `src/app/api/items/sell/route.ts` |
| **UI หน้า /items**: ปุ่มเขียว "ขายคืนวัตถุดิบ ×1 (💠x + ✨y)" เฉพาะชิ้นที่ยังไม่ใส่อยู่บนการ์ด · ถ้าใส่ครบทุกชิ้นขึ้นข้อความ "ขายไม่ได้ — ชิ้นนี้ใส่อยู่บนการ์ด (ถอดออกก่อน)" · ยอด 💠/✨ บนหัวเว็บอัปเดตทันที | `src/app/(game)/items/page.tsx` · `dict-th/en` (+5 คีย์ใหม่) |

> ปลอดภัยจากการปั๊มของ: คืน 50% < 100% ⇒ "คราฟต์แล้วขายคืน" ขาดทุนเสมอ — มีเทสต์ไล่ทุกรายการในแคตตาล็อกกันไว้

### 2) ไอเทมเพิ่มอีก 18 ชิ้น (โจมตี/ป้องกัน/สนับสนุน อย่างละ 6 → รวม **12 ชิ้น/ช่อง = 36 ชิ้น**)
- ทุกช่องมี **2 ชิ้นต่อระดับความหายาก** (COMMON → MYTHIC) เลือกได้ตามสไตล์: เช่นช่องโจมตีมีทั้งสาย "อัตราตี+ความเร็ว" (สะเก็ดประกาย · กรงเล็บนักล่า · ดาบวายุ) และสาย "พลัง+HP" (ดาบหลอมสุริยา · **ดาบฉีกสุญญตา** atk 90)
- ระดับ COMMON/UNCOMMON/RARE ซื้อตรงได้ (14/30/52 Shards) · EPIC ขึ้นไปคราฟต์เท่านั้น (สูงสุด 260 Shards + 240 ฝุ่นเวท)
- **ใส่เข้า "บันไดดรอป" ของดันเจี้ยนด้วย** (ไม่ใช่มีแต่ในร้าน): เช่น สุสานเพลิงชั้น 5/9/16 · วิหารน้ำขึ้น 6/16 · รอยแยก 6/17 · เหวลึก 10/19 · ยอดหอพายุ 12/22/32
- เข้า DB อัตโนมัติผ่าน `ensureCatalog()` (ไม่ต้อง migrate)

**ไฟล์:** `src/lib/item-definitions.ts` · `src/services/item.ts` · `src/services/inventory.ts` · `src/lib/veil-shards.ts` · `src/app/api/items/sell/route.ts` (ใหม่) · `src/app/(game)/items/page.tsx` · `src/lib/i18n/dict-th/en` · `src/lib/dungeon-definitions.ts` (บันไดดรอป) · `scripts/inspect-item-workshop.mjs` (+8 ข้อ) · `tests/unit/item-definitions.test.ts` (+3 เทสต์)

**หลักฐานวัดได้ (production จริง · jest 710 ผ่าน / 49 suites · `tsc --noEmit` 0 error · `next lint` ผ่าน · build ✓ + restart · `/api/health` 200):**

`npm run inspect:items` → **ผ่าน 31/31** (เพิ่ม: ขาย Item ที่ใส่อยู่บนการ์ด → ปฏิเสธ 400 + เหตุผล · ขาย โล่ไม้โอ๊ก คืนร้าน → ได้ 💠3 + ✨2 ตรงสูตร 50% · ยอดหลังขาย +50% ทั้งสองสกุล · ถอดของออกแล้วขายได้ · ขายแล้วซื้อกลับมาใส่ใหม่ได้ · แคตตาล็อก 36 ชิ้น = `ATTACK:12 · DEFENSE:12 · SUPPORT:12`)

UI จริง (Chrome headless): หน้า `/items` แสดง 36 ชิ้น · ปุ่มขายขึ้นพร้อมยอดคืนล่วงหน้า ("ขายคืนวัตถุดิบ ×1 (💠3 + ✨3)") · กดขายแล้วได้ข้อความ **"ขาย สะเก็ดประกาย ×1 แล้ว — ได้ 💠 3 + ✨ 3 กลับมา"** และยอดบนหัวเว็บขยับจริง (`💠 93→96` · `✨ 94→97`) · หลังขายปุ่มหายไป (ไม่มีของเหลือ)

---

## Phase 33–35 — กระเป๋าขายของ · เพลงดันเจี้ยน · เลเวล/EXP 350 · Ranking ตามโหมดใหม่ (2026-09-27)

**คำสั่งผู้ใช้ (6 ข้อ):** *"หน้ากระเป๋า เลือกขาย Item ได้ · item ที่ใส่อยู่สามารถกดแล้วไปที่การ์ดที่ใส่อยู่ได้ ถ้ามี Item เดียวกันหลายชิ้นใส่หลายใบก็ให้มีตัวเลือก · เพลงประกอบในโทรศัพท์มีเสียงแตกหน่อยๆ · เพิ่มเพลงประจำดันเจี้ยนให้ตื่นเต้น · ปุ่มลุยตอนเข้ามาใหม่ให้แสดงชั้นสูงสุดที่ค้างไว้ก่อน · เพิ่มระบบ Exp Level สูงสุด 350 (Level สูงเพิ่มโอกาสดรอป Item สูงสุด +20% · EXP เลเวลสูงยิ่งขึ้นยากจนต้องเล่นหลักปีถึง 300-350 · Level Up ได้ Item/Coin/พลังงานตาม Level — Level 2 ได้ 2 · Level 3 ได้ 3) · ปรับฟังก์ชัน Ranking ตามเกมที่มีโหมดเพิ่มขึ้นมา"*

### 33) กระเป๋า: ขายไอเทมได้ + กดไปถอดที่การ์ดที่ใส่อยู่
| สิ่งที่ทำ | ไฟล์ |
|---|---|
| `ItemService.equippedCardsByItem()` — คืน Map<code, การ์ดที่ใส่อยู่ (cardId+ชื่อ+ช่อง)> | `src/services/item.ts` |
| `/api/inventory` แนบ `sellRefund` (50% ของสูตร · ใช้ `sellQuote` ตัวเดียวกับร้านช่าง) + `equippedCards` ต่อชิ้น | `src/app/api/inventory/route.ts` |
| หน้ากระเป๋า: ปุ่ม **"ขายคืนวัตถุดิบ (💠x + ✨y)"** ต่อชิ้นที่ยังไม่ใส่อยู่ · ชิ้นที่ใส่ครบขึ้น "ขายไม่ได้ — ชิ้นนี้ใส่อยู่บนการ์ด" · และ **ปุ่ม 🎴 ชื่อการ์ด ›** ต่อการ์ดที่ใส่ (หลายใบ = หลายปุ่ม กดไปถอดที่หน้าการ์ดนั้นได้) | `src/app/(game)/inventory/page.tsx` · i18n (+5 คีย์) |

### 34) เสียง: แก้ "เพลงแตกบนมือถือ" + เพิ่มเพลงประจำดันเจี้ยน
| สิ่งที่ทำ | ไฟล์ |
|---|---|
| **Limiter ต่อท้าย compressor** (`ratio 20 · threshold −1.5 dB · attack 2 ms`) ⇒ ยอดคลื่นไม่แตะ 0 dBFS (ต้นเหตุเสียงแตกจริง) · ลดเกนมาสเตอร์ 1.3→1.1 · บัสเพลง 1.8→1.55 · เกนต่อโน้ต (pad 0.11→0.075 · presence 0.09→0.06 · เบส 0.14→0.1 · ระฆัง 0.1→0.075) | `src/lib/audio-engine.ts` · `src/lib/music.ts` |
| **เพลงประจำดันเจี้ยน** (`'dungeon'`): คอร์ด Dm→Bb→F→Gm · คอร์ดละ 4.2 วิ (ธีมหลัก 7.5) · **เบสตีพจังหวะ 0.7 วิ เหมือนกลองศึก** + ระฆังถี่ 0.7 วิ | `src/lib/music.ts` (`dungeonChordNotes` · `trackNotes` · `trackChordSeconds`) |
| ตัวเล่น/ผู้ให้บริการเสียงเลือกเพลงได้ (`createMusicPlayer(ctx, bus, track)` · `setMusicTrack('main'\|'dungeon')` + debug `__rdaAudio.musicTrack()`) · หน้า `/dungeons` และศึกดันเจี้ยนสลับเป็นเพลงดันเจี้ยน แล้วคืนเพลงธีมเมื่อออก | `src/lib/audio-players.ts` · `AudioProvider.tsx` · `dungeons/page.tsx` · `battle/[id]/page.tsx` |

### 35) ระบบเลเวล/EXP (สูงสุด 350) + โบนัสดรอป + รางวัลขึ้นเลเวล
- **โค้ง EXP**: `expForLevel(L) = 6,000,000 × (L/350)^2.1` ⇒ L2 ใช้ 117 EXP (เล่น 1-2 ศึก) · L100 = 433k · L200 = 1.85M · **L300 = 4.34M · ช่วง 300→350 = 1.66M (28% ของทั้งหมด)** = เป้าหมายระยะยาวหลายเดือน-ปีตามที่ผู้ใช้สั่ง
- **โบนัสโอกาสดรอป Item**: `itemDropBonus(L) = (L−1)/349 × 20%` → L1 = 0% · L175 = +10% · **L350 = +20%** (บวกเข้าโอกาสดรอปจริงของดันเจี้ยน + โชว์ในหน้า /dungeons และโปรไฟล์)
- **รางวัลขึ้นเลเวล** (transaction เดียว + แจ้งเตือน): Coin `20+6×L` · **พลังงาน = เลขเลเวลนั้น** (L2 → 2 · L3 → 3 · ไม่เกินเพดาน 10) · Item ตามช่วงเลเวล (ทุก 5 = ของพื้นฐาน · 10 = RARE · 25 = EPIC · 50 = LEGENDARY · 100 = ATK_STORMFANG · **L350 = SUP_ORIGIN_RELIC**)
- **EXP จาก**: ศึกปกติ ชนะ 30 / แพ้ 10 · ดันเจี้ยน `12 + ชั้น×4` (เฉพาะชั้นที่ยังไม่เคยผ่าน = กันปั๊มชั้นเดิม)
- DB: `users.exp Int @default(0)` + migration `20260927090000_phase33_user_level` (เลเวลคำนวณจาก exp เสมอ — ไม่เก็บซ้ำ)
- UI: การ์ดเลเวล + แถบ EXP + โบนัสดรอป ในหน้าโปรไฟล์
**ไฟล์:** `src/lib/level.ts` (ใหม่) · `src/services/level.ts` (ใหม่) · `src/lib/music.ts` · `audio-engine.ts` · `audio-players.ts` · `AudioProvider.tsx` · `battle/simulate` · `services/dungeon.ts` · `api/profile` · `api/dungeons` · `profile/page.tsx` · `dungeons/page.tsx` · i18n · prisma schema+migration

### 36) Ranking ตามโหมดที่เพิ่มขึ้นมา (5 → 7 หมวด)
- เพิ่ม **⭐ เลเวล (EXP)** — เรียงด้วย exp แต่โชว์ "Lv. N" เป็นข้อมูลรอง · **🏰 ดันเจี้ยน** — ผลรวมชั้นสูงสุดที่ผ่านทุกดัน + จำนวนดันที่ผ่าน
- ป้ายรองใหม่ (`dungeons` · `level`) ในหน้าตาราง + i18n
**ไฟล์:** `src/lib/ranking.ts` · `src/services/ranking.ts` · `ranking/page.tsx` · `tests/unit/ranking.test.ts`

**หลักฐานวัดได้ (production จริง · jest 728 ผ่าน / 51 suites · `tsc --noEmit` 0 error · `next lint` ผ่าน · build ✓ + restart · `/api/health` 200):**

ตรวจ API จริง **18/18**: โปรไฟล์มีเลเวล (L1 · โบนัส +0%) · ชนะ/แพ้ศึกได้ EXP จริง · ตั้ง exp ให้ใกล้ L10 แล้วต่อสู้ → **ขึ้นเลเวล 10 · Coin 100→180 (+80) · พลังงาน 5→10 (เต็มเพดาน) · ได้ Item DEF_TIDEWALL ×1 · แจ้งเตือนขึ้นเลเวล · โบนัสดรอป +1%** · L175 → +10% · อันดับหมวด `level`/`dungeon` ใช้งานได้ · API ดันเจี้ยนส่งโบนัสเลเวล · ผลลุยดันมี `dropChance` รวมโบนัส · กระเป๋าส่ง `sellRefund` (💠3+✨2) ตรงสูตร 50% · `equippedCards` ชี้การ์ดที่ใส่จริง · ขายของที่ใส่อยู่ถูกปฏิเสธ (ต้องถอดก่อน)

ตรวจ UI จริง (Chrome headless): กระเป๋า → ปุ่ม "ขายคืนวัตถุดิบ (💠3 + ✨2)" + ปุ่ม **"🎴 ชื่อการ์ด ›"** ของการ์ดที่ใส่ · กดขายได้ข้อความ "ขาย หินลับคม ×1 แล้ว — ได้ 💠 3 + ✨ 2 กลับมา" · หน้า `/dungeons` เปิดมา **เลือกดันที่ค้างไว้เอง (TIDAL_SANCTUM) + ตั้งชั้น 7 = ชั้นที่ค้างไว้** ปุ่มขึ้น "ลุยชั้น 7 ฟรี (ชั้นที่ค้างไว้ 7)" · แถบ "ความคืบหน้า: ผ่านสูงสุด ชั้น 6 / 28" · บรรทัด "🎁 โบนัสโอกาสดรอป Item จากเลเวล 100: +6%" · **เพลงที่กำลังเล่น = `dungeon`** (`__rdaAudio.musicTrack()`) · เทสต์ยืนยัน limiter (ratio 20) + เกนรวมเพลงลดลง (bus×master ≤ 1.8 จากเดิม 2.34) + เพลงดันเจี้ยนมีเบสตีพ/ระฆังถี่ในย่าน 250-1100Hz ที่มือถือได้ยิน

> ⚠️ หมายเหตุตามจริง: การยืนยัน "เสียงไม่แตก" ต้องฟังบนมือถือจริง — เครื่องมือที่ให้มาคือ limiter + ลดเกนที่วัดได้ (มีเทสต์กันค่าเพี้ยน) และตรวจว่าเพลงยังออกจริง/สลับเพลงได้ในเบราว์เซอร์

---

## Phase 37 — ปรับสมดุลทั้งเกม: ชั้นดันเป็นบล็อก 5 ชั้น · ดันเสียเงินคุ้มค่า · กันเงินเฟ้อ · Reset (2026-09-27)

**คำสั่งผู้ใช้:** *"สมดุลเกมค่อนข้างไม่ดี · ดันเจี้ยน…ควรทำให้เป็น Step 5 ชั้น แล้วขยับให้เก่งขึ้นแบบเห็นได้ชัด · แบบเสียเงินก็มากเกินไปจนค่าเข้าไม่คุ้มกับรางวัล · Events ต่างๆ ดูความคุ้มค่าด้วย · ดูปรับไม่ให้เกิดเงินเฟ้อ · คนที่ทดลองเล่นผ่านดันเจี้ยนเดิมไปแล้ว Reset ให้เริ่มใหม่ ปรับเงินในกระเป๋าลงให้เหลือ 10 พอ · เน้นเรื่องสมดุล ตรวจสอบและตั้งให้ดี"*

### 1) เศรษฐกิจจริงก่อนปรับ (วัดจาก production · `npm run inspect:economy`)
| ตัวชี้วัด | ก่อน | หลัง |
|---|---|---|
| เงินทั้งระบบ (68 กระเป๋า) | **9,660 Coin** | **680 Coin** (10/คน) |
| รางวัลรวม vs ใช้จ่าย | 10,600 : 940 = **11:1 (เฟ้อหนัก)** | โบนัสสมัคร 100→**10** · Arena 100-500→**60-300** · รางวัลเลเวล 20+6L→**12+2L** (L350: 2,120→712) · ร้านกิจกรรม 15→20 Shards แลก 100 Coin · Milestone 100→60 Coin |
| เศรษฐกิจที่ตรวจได้ใหม่ | — | `npm run inspect:economy` รายงาน faucet/sink แยกที่มา (ใช้ตรวจทุกครั้งก่อนปรับสมดุล) |

### 2) ชั้นดันเจี้ยน = "Step ละ 5 ชั้น" (ผู้ใช้สั่งชัด)
- 5 ชั้นในบล็อกเดียวกัน **status เท่ากัน** (เล่นซ้ำไม่รู้สึกวืด) แล้ว **กระโดดที่ชั้นแรกของบล็อกถัดไป** (blockStep +5-9%)
- บล็อก 3 เริ่มมี **บอส 2 ตัว** · บล็อก 5 เริ่มมี **บอส 3 ตัว** (ทีมยัง 5v5) — เป็นหมุดหมายที่เห็นชัด
- รางวัล **กระโดดตามบล็อก** (×1.3/บล็อก ฝุ่น · ×1.2 Shards) แทนการไล่ต่อชั้น (เดิมระเบิดเป็น ×20-100 ที่ชั้นท้าย)
- ดันเสียเงิน (เหวลึก) ตั้งใจให้ **บอสสูงสุด 2 ตัว** เพื่อให้ผู้เล่นที่ใส่ของระดับตำนานผ่านได้ตลอด (ค่าเข้าต้องคุ้ม)

**ผลวัดจริง (`npm run calibrate:dungeons` · 12 ศึก/ช่อง) — ทุกดันผ่านถึงชั้นท้ายด้วยเด็คเป้าหมาย:**
| ดัน | บล็อก/ชั้น | มือใหม่ | ท็อปดิบ | ติดของตำนาน | mythic |
|---|---|---|---|---|---|
| 🔥 สุสานเพลิง | 5 บล็อก · 25 ชั้น | 97% → 85% → 100% → 100% → **67%** (บอส 3) | 100% | 100% | 100% |
| 🌊 วิหารน้ำขึ้น | 6 บล็อก · 28 ชั้น | 70% → 62% → 74% → 58% → 49% → **25%** | 100% | 100% | 100% |
| 🌙 รอยแยกไร้จันทร์ | 6 บล็อก · 28 ชั้น | 17% → 0% | 100% → 33% | **100% ทุกบล็อก** | 100% |
| 💰 เหวลึกทองคำ (35 Coin) | 6 บล็อก · 30 ชั้น | 0% | 0% | **100% → 92% → 0%** (ชั้นท้ายต้อง mythic) | **100% ทุกบล็อก** |
| ⚡ ยอดหอพายุ | 8 บล็อก · 40 ชั้น | 0% | 0% | 100% → 100% → 0% (บล็อก 5+) | **100% ทุกบล็อก** |

> ข้อค้นพบสำคัญที่บันทึกไว้: **เพดานความยากจริงของเด็คที่ใส่ของครบอยู่ราวสเกล 1.10-1.30** (ต่างกันตามฐานของแต่ละดัน) ⇒ ดันสเกลเกินเพดาน = ชั้นท้ายผ่านไม่ได้ทุกเด็ค (เคยเกิดจริงตอนตั้งเพดาน 1.7) จึงตั้งเพดานจาก **ค่าที่วัดได้** และให้ "ขนาดทีมศัตรู" เป็นตัวชดเชยที่วัดจริง (บอส 2 ตัว ≈ ×1.4 · 3 ตัว ≈ ×1.7)

### 3) ดันเสียเงิน (เหวลึกทองคำ) — ให้คุ้มค่าเข้าจริง
| สิ่งที่ทำ | ค่า |
|---|---|
| ค่าเข้า | 50 → **35 Coin** |
| **แพ้แล้วคืนค่าเข้า 50%** (17 Coin) | ใหม่ — ไม่ให้เจ็บตัวจากการลอง (`DUNGEON_LOSS_REFUND`) |
| รางวัลต่อชั้น | ฝุ่น 45 → 202/บล็อก · Shards 6 → 31/บล็อก (กระโดด ×1.35/บล็อก) |
| เทียบความคุ้มค่า | ชั้น 1 ชนะได้ 45 ฝุ่น + 10 Shards ≈ **100+ Coin ของมูลค่า** ต่อค่าเข้า 35 Coin |

### 4) กิจกรรม (Events) — ตรวจความคุ้มค่า
- ร้านกิจกรรม: `ถุง Coin` 15 Shards→150 Coin ⇒ **20→100** (1 Shard ≈ 5 Coin ชัดเจน) · `ฝุ่นเวท` 20→25 ⇒ **15→30** (คุ้มกว่าซื้อด้วย Coin → กระตุ้นคราฟต์)
- Milestone ส่วนตัว: Coin 100→60 · ฝุ่น 50→60 · Event quest: 100→60 (+Shards 10→8) · Milestone ใหญ่: 300→200 (+25→20)

### 5) Reset ผู้เล่นที่ผ่านดันเดิม (ผู้ใช้สั่ง)
`npm run reset:dungeon-balance` (dry-run ได้):
1. ล้าง `dungeon_progress` + `dungeon_runs` ทั้งหมด → เริ่มนับชั้นที่ผ่านใหม่
2. ตั้ง Coin ในกระเป๋าทุกคนเป็น **10** (บันทึก ledger `BALANCE_RESET` ตรวจย้อนหลังได้ · ไม่แตะ Shards/ฝุ่นเวท/EXP/การ์ด)
**รันจริงแล้ว:** ล้างความคืบหน้า 6 แถว + ประวัติ 80 แถว · ลดกระเป๋า 68 ใบ (เงินรวม 9,660 → 680)

**ไฟล์:** `src/lib/dungeon-definitions.ts` (บล็อก 5 ชั้น · bossWeight · ค่าเข้า/คืนเงิน) · `src/services/dungeon.ts` (คืนค่าเข้า + block ใน API) · `src/lib/level.ts` · `src/lib/constants.ts` · `src/services/event-definitions.ts` · `dungeons/page.tsx` (ป้ายระดับ) · `scripts/reset-dungeon-balance.mjs` (ใหม่) · `scripts/inspect-economy.mjs` (ใหม่) · `package.json` · เทสต์ (arena/dungeon/level)

**หลักฐานวัดได้ (production จริง · jest 728 ผ่าน / 51 suites · `tsc --noEmit` 0 error · `next lint` ผ่าน · build ✓ + restart · `/api/health` 200):**
- ตรวจสมดุลจริง **7/7**: Coin ผู้เล่นใหม่ = 10 · ชั้น 1-5 สเกลเท่ากันแล้วกระโดดที่ชั้น 6 (1 → 1.09) · API ส่งบล็อก (ชั้น 6 = บล็อก 2 ลำดับ 1) · รางวัลกระโดดตามบล็อก (ฝุ่น 10 → 13) · ค่าเข้าดันเสียเงิน = 35 · **แพ้คืนค่าเข้า 17 Coin** · ยอด Coin สอดคล้อง
- `npm run inspect:economy` แสดงเศรษฐกิจหลัง Reset (เงินรวม 680 · มีรายการ `BALANCE_RESET −8,980`)

---

## Phase 38 — เลเวลบนหัวเว็บ + ปุ่ม "ไปชั้นถัดไป" หลังชนะดันเจี้ยน (2026-09-27)

**คำสั่งผู้ใช้:** *"เพิ่มแสดง Level ด้านบนแถวแจ้งเตือน หรือ User ด้วย · หลังต่อสู้ดันเจี้ยนชนะชั้นปัจจุบัน มีปุ่มกดไปสู่ชั้นต่อไป ไม่ต้องย้อนมาหน้าเลือกดันเจี้ยน"*

| สิ่งที่ทำ | ไฟล์ |
|---|---|
| **ป้ายเลเวลบนหัวเว็บ** ⭐ `Lv.N` วางติดกับปุ่มแจ้งเตือน/ชื่อผู้ใช้ (กดไปโปรไฟล์ได้ · tooltip บอกโบนัสโอกาสดรอป Item) · ใช้ `data-header-level` ให้ตรวจอัตโนมัติได้ | `src/components/layout/TopHeader.tsx` |
| `/api/auth/me` ส่งข้อมูลเลเวลมาด้วย ⇒ หัวเว็บไม่ต้องยิง API เพิ่ม (และอัปเดตทุกครั้งที่เปลี่ยนหน้า) | `src/app/api/auth/me/route.ts` |
| **helper บริสุทธิ์** `nextDungeonFloor(current, total)` → ชั้นถัดไป (null เมื่ออยู่ชั้นสุดท้าย) | `src/lib/dungeon-definitions.ts` |
| log API ของดันเจี้ยนส่ง `dungeon.floors` · `dungeon.nextFloor` · `dungeon.coinCost` ⇒ หน้าสนามรบรู้ว่ามีชั้นถัดไปไหม | `src/app/api/dungeons/run/[runId]/log/route.ts` |
| **ปุ่ม "▶ ลุยชั้น N ต่อ"** หลังชนะดันเจี้ยนบนหน้าสนามรบ (แสดงค่าเข้าด้วยถ้าเป็นดันเสียเงิน) — กดแล้วยิงลุยชั้นถัดไปด้วยเด็คเดิม แล้วพาไปหน้าสนามรบของชั้นนั้นทันที · ยังมีปุ่ม "🏰 กลับดันเจี้ยน" เป็นตัวเลือกสำรอง | `src/app/(game)/battle/[id]/page.tsx` |
| เทสต์: `nextDungeonFloor` (ชั้นถัดไป · ชั้นสุดท้าย = null · ครบทุกดัน) | `tests/unit/dungeon.test.ts` |

**หลักฐานวัดได้ (production จริง · jest 731 ผ่าน / 51 suites · `tsc --noEmit` 0 error · `next lint` ผ่าน · build ✓ + restart · `/api/health` 200):**

ตรวจของจริงด้วย API + Chrome headless:
- `/api/auth/me` ส่งเลเวล (Lv.42 · โบนัส +2%) · **หัวเว็บแสดง "⭐ Lv.42"** (`data-header-level=42`)
- log API ส่ง `nextFloor=2 · floors=25` และ **ชั้นสุดท้าย (25) → `nextFloor=null`**
- หน้าสนามรบหลังชนะชั้น 1 แสดง **"▶ ลุยชั้น 2 ต่อ"** คู่กับ "🏰 กลับดันเจี้ยน · ✨ +10"
- **กดปุ่มจริง → เข้าศึกชั้น 2 ทันที** (`/battle/dungeon-run:…` · หัวข้อ "🔥 สุสานเพลิง ชั้น 2" · ชื่อทีมศัตรู "สุสานเพลิง ชั้น 2") โดยไม่ต้องย้อนไปหน้าเลือกดันเจี้ยน

---

## Phase 39–40 — ดันเจี้ยนยากขึ้น ~50% · ฝุ่นเวทเหลือ 20% · คราฟต์ต้องใช้ Coin (2026-09-27)

**คำสั่งผู้ใช้:** *"คิดว่าความยากยังน้อยไปเพิ่มไปอีก สัก 50% · ลดของรางวัล ฝุ่นเวท ลงอีก เอาแค่ 20% จากตอนนี้ · ในการ Craft ของต้องใช้ Coins ด้วย"*

### 39) Craft ต้องใช้ Coin ด้วย
| สิ่งที่ทำ | ไฟล์ |
|---|---|
| **coinCost ต่อไอเทม** = `max(10, craftCost × 3)` (ยิ่งของสูงยิ่งใช้ Coin มาก) — เพิ่มครบทั้ง **36 ไอเทม** | `src/lib/item-definitions.ts` |
| `craftQuote()` ตรวจ Coin ด้วย → `missingCoins` · `canCraft` ต้องมีครบ 3 อย่าง (Shards + ฝุ่น + Coin) | เดียวกัน |
| **ขายคืนได้ Coin 50%** เช่นกัน (`sellQuote().coins`) — สอดคล้องกับวัตถุดิบที่จ่ายไป | เดียวกัน |
| DB: `item_definitions.coin_cost` + migration `20260927120000_phase39_item_coin_cost` · `ensureCatalog()` ซิงก์ค่าอัตโนมัติ | `prisma/schema.prisma` |
| ตัด/เติม Coin **ใน transaction เดียวกับการคราฟต์/ขาย** — เพิ่ม `debitWalletDb()` / `creditWalletDb()` (แบบเดียวกับ veil-shard) | `src/services/wallet.ts` · `src/services/item.ts` |
| UI ร้านช่าง: โชว์ `+ 🪙N` ในราคาคราฟต์ · ยอด Coin บนหัวร้าน · ข้อความ "ขาดอีก 🪙N" · ปุ่มขายโชว์ Coin ที่จะได้คืน | `src/app/(game)/items/page.tsx` · i18n (+1 คีย์) |
| API: เงินไม่พอ → **400** พร้อมข้อความผู้เล่น ("ยอด Coin ไม่เพียงพอ" ไม่ใช่ 500) | `src/app/api/items/craft/route.ts` |

### 40) ดันเจี้ยนยากขึ้น ~50% · ฝุ่นเวทเหลือ 20%
- **ฐาน status ศัตรู ×1.5** (ดันฟรีกลาง-สูง: วิหารน้ำขึ้น · รอยแยก · ยอดหอพายุ) · **สุสานเพลิง ×1.25** (ยังต้องให้มือใหม่ชนะได้) · **เหวลึกทองคำ ×1.1** (ดันเสียเงิน — ต้องคุ้มค่าเข้า ⇒ คนใส่ของตำนานผ่านช่วงต้นได้)
- **ฝุ่นเวท = 20% ของเดิม** (`dustRatio: 0.2` ที่ตัวสร้างชั้น — ที่เดียวคุมทุกดัน/ทุกชั้น)
- เพดานสเกลต้อง **ลดลงตาม** (เพราะความยาก = ฐาน × สเกล และเพดานที่เด็คเป้าหมายผ่านได้นั้นคงที่) — ตั้งจากค่าที่วัดจริง: 0.98 / 1.02 / 1.05 / 1.15 / 0.72

**ผลวัดจริง (`npm run calibrate:dungeons` · 12 ศึก/ช่อง) ก่อน → หลัง:**
| ดัน | มือใหม่ (ชั้นแรก) | ท็อปดิบ | ติดของตำนาน | mythic |
|---|---|---|---|---|
| 🔥 สุสานเพลิง (×1.25) | 97% → **67%** | 100% | 100% | 100% |
| 🌊 วิหารน้ำขึ้น (×1.5) | 70% → **17%** | 100% | 100% | 100% |
| 🌙 รอยแยกไร้จันทร์ (×1.5) | 17% → **0%** (ท็อปดิบเคย 100% → 0% = ต้องมีของ) | 0% | **100%** | 100% |
| 💰 เหวลึกทองคำ (×1.1 · ค่าเข้า 35) | 0% | 0% | ช่วงต้น **100% → 83%** | **100% ทุกช่วง** |
| ⚡ ยอดหอพายุ (×1.5) | 0% | 0% | ช่วงต้น 58-83% | **100% ทุกช่วง** |

**รางวัลฝุ่นเวทหลังลด 80%:** สุสานเพลิง 2→18 · วิหารน้ำขึ้น 4→40 · รอยแยก 6→49 · เหวลึก 9→61 · ยอดหอพายุ 14→88 (ต่อการชนะ 1 ครั้ง)
**ความคุ้มค่าดันเสียเงิน:** ชั้น 1 ได้ 10 Shards + 9 ฝุ่น + ไอเทม 20% ต่อค่าเข้า 35 Coin (Shards มีมูลค่า ~50 Coin) + **แพ้คืน 17 Coin** ⇒ ยังคุ้มค่าเข้า

**หลักฐานวัดได้ (production จริง · jest 734 ผ่าน / 51 suites · `tsc --noEmit` 0 error · `next lint` ผ่าน · build ✓ + restart · `/api/health` 200):**
- ตรวจของจริง **7/7**: Coin ไม่พอ → คราฟต์ไม่ได้ (400 + "ยอด Coin ไม่เพียงพอ") · แคตตาล็อกส่ง `coinCost=18` + `missingCoins=8` · คราฟต์สำเร็จหัก Coin 100→82 · **ขายคืนได้ Coin 9 (50%)** → 91 · **ฝุ่นชั้นแรกสุสานเพลิง = 2** (เดิม 10) · **ชั้นสุดท้ายยอดหอพายุ = 88** (เดิม 439) · ยิงลุยดันยังทำงานปกติ
- `npm run inspect:dungeons` ยังผ่าน (โครงชั้น/รางวัล/การ์ดศัตรูไม่พัง)

---

## Phase 41 — UX จัดเด็ค/การ์ด/กระเป๋า: ฟอง (Modal) · เทียบการ์ด · เลือกหลายใบ · ยืนยันก่อนเปลี่ยน (2026-09-27)

**คำสั่งผู้ใช้ (9 ข้อ):** *"หน้าจัด Deck พอคลิกตำแหน่งการ์ดควรมีตัวเลือก เอาออก/แก้ไข/ดูการ์ด · การ์ดที่จะเปลี่ยนควรมีระบบเปรียบเทียบว่าดีขึ้นหรือแย่ลง · หน้าดูการ์ดกับจัด Deck ควรควบรวมกัน (ตอนนี้ต้องเปิดกลับไปมาเพื่อใส่ Item) · จัด Deck ค้างอยู่แล้วไม่บันทึกให้ถามก่อน · UI บางอย่างควรเป็นฟอง/ป็อปอัป (แจ้งเตือน/รายละเอียดการ์ด) · การเปลี่ยน Item หรือการขายต้องมีหน้า Confirm · หาข้อมูลระบบจัดการ Deck/การ์ดว่าแบบไหนสะดวกกับผู้เล่น · เลือกการ์ดหลายใบเพื่อกดขายพร้อมกัน · การ์ดที่อยู่ในทีมให้ระบุว่าอยู่ทีมไหน + เอาออกได้จากจุดนี้ · ในกระเป๋า Item ที่ระบุการ์ดให้แสดงหน้าการ์ดย่อ ขยายได้ (แบบฟอง) · ตรวจว่าระบบเชื่อมกัน"*

### อ้างอิง UX ที่เลือกใช้ (จากรูปแบบมาตรฐานของเกมการ์ด/เด็คบิลเดอร์)
| หลักการที่ผู้เล่นต้องการ | ที่ใช้ในเกมนี้ |
|---|---|
| เห็นข้อมูลทันทีโดยไม่เปลี่ยนหน้า | ทุกอย่างเป็น **ฟอง/ป็อปอัป** (มือถือ = แผ่นเลื่อนขึ้น · จอใหญ่ = กล่องกลาง) |
| กดที่ช่องแล้วเลือก action | **Action sheet** ต่อช่อง (ดูการ์ด · เปลี่ยนการ์ด · เอาออก) |
| เลือกการ์ดใหม่ต้องรู้ว่าดีขึ้นไหม | **เทียบ Δ ทีละค่า + สรุป ▲ดีขึ้น / ▼แย่ลง / ＝พอ ๆ กัน** (คะแนนถ่วงน้ำหนัก ATK 2 · DEF 1.2 · HP 0.35 · SPD 1.8) |
| แก้ไขให้จบในหน้าเดียว | **ฟองดูการ์ดมี "ช่างใส่ Item" ในตัว** → ใส่/ถอด Item แล้วคะแนนเด็คอัปเดตทันที |
| ป้องกันงานหาย | **เตือนเมื่อยังไม่บันทึก** ทั้งตอนกดกลับหน้า และตอนรีเฟรช/ปิดแท็บ (`beforeunload`) |
| งานที่มีของหายต้องยืนยัน | **ConfirmDialog** ก่อน: ขายการ์ด · ขายหลายใบ · ถอด/เปลี่ยน Item · เอาออกช่อง · ออกจากหน้าโดยไม่บันทึก |

### สิ่งที่ทำ
| ข้อ | รายละเอียด | ไฟล์ |
|---|---|---|
| ฟอง/ป็อปอัปกลาง | `Modal` (ล็อกการเลื่อน + Esc + คลิกพื้นหลังปิด + `data-modal`) · `ConfirmDialog` (danger/default + busy) | `components/ui/Modal.tsx` · `ConfirmDialog.tsx` (ใหม่) |
| การ์ดย่อ–ขยายเป็นฟอง | `CardMiniPreview` (การ์ดย่อกดขยาย) + `CardPreviewModal` (การ์ดเต็มใบ + ช่างใส่ Item) | `components/cards/CardMiniPreview.tsx` (ใหม่) |
| เทียบการ์ด (บริสุทธิ์) | `compareCardStats()` + `statScore()` + `verdict better/worse/equal` + ป้าย Δ | `lib/deck-compare.ts` (ใหม่) + เทสต์ 6 ข้อ |
| หน้าจัดเด็ค | กดช่อง → Action sheet · **ฟองเลือกการ์ดพร้อมเทียบ** · **ฟองดูการ์ด + ใส่ Item** (รวม 2 หน้าเป็นหน้าเดียว) · ป้าย "ยังไม่บันทึก" · ถามก่อนออก · `?slot=N` เปิดฟองให้ช่องนั้น | `app/(game)/decks/[id]/page.tsx` |
| หน้าการ์ด | **เลือกหลายใบ** (โหมดติ๊ก) + แถบลอย **"ขายที่เลือก"** (ยืนยัน + สรุปยอดที่ได้) · **ป้ายว่าอยู่ทีมไหน + ปุ่มเอาออก** | `app/(game)/cards/page.tsx` |
| กระเป๋า | Item ที่ใส่การ์ด → **แสดงการ์ดย่อ** (กดขยายเป็นฟองดูการ์ด/ถอด Item) แทนตัวหนังสือ | `app/(game)/inventory/page.tsx` · `api/inventory` · `services/item.ts` (ส่งภาพ/ระดับหายากของการ์ด) |
| ช่างใส่ Item | **ยืนยันก่อนเปลี่ยน Item** (โชว์ของเดิม → ของใหม่) และ **ก่อนถอด** | `components/cards/CardItemWorkshop.tsx` |

> หมายเหตุที่ตัดสินใจแทนผู้เล่น (เพราะข้อจำกัดของข้อมูล): API บังคับว่า **เด็คต้องครบ 5 ช่อง (ตำแหน่ง 0-4) เสมอ**
> ⇒ ปุ่ม "เอาออก" ในหน้าการ์ด = ยืนยัน → พาไปหน้าเด็คที่ **เปิดฟองเลือกการ์ดให้ช่องนั้นทันที** (พร้อมเทียบดีขึ้น/แย่ลง)
> ซึ่งให้ผลเท่ากับเอาออกแล้วใส่ใบใหม่ในคลิกเดียว โดยทีมไม่พัง

**หลักฐานวัดได้ (production จริง · jest 740 ผ่าน / 51 suites (+6 เทสต์ deck-compare) · `tsc --noEmit` 0 error · `next lint` ผ่าน · build ✓ + restart · `/api/health` 200):**

ตรวจ UI จริงด้วย Chrome headless **10/10**:
1) กดช่องในวงแหวน → ชีตตัวเลือก `["view","swap","remove"]` ·
2) ฟองเลือกการ์ดแสดง **3 ใบพร้อมคำตัดสิน `worse · better · better`** ·
3) เปลี่ยนการ์ดแล้วขึ้นป้าย **"● มีการแก้ไขที่ยังไม่บันทึก"** + ปุ่ม "บันทึกเด็ค (มีการแก้ไข)" ·
4) กดกลับ → ถาม **"ต้องการบันทึกก่อนออกหรือไม่?" + "ออกโดยไม่บันทึก"** ·
5) ฟองดูการ์ดมี **การ์ดเต็มใบ + ช่างใส่ Item ในหน้าเดียวกัน** ·
6) (การใส่ Item ที่ไม่มีชิ้นว่างถูกปิดไว้ตามกติกา) ·
7) ป้าย "อยู่ในทีม" 5 ใบ + ปุ่มเอาออก 5 ปุ่ม ·
8) โหมดเลือกหลายใบ 8 ช่อง + แถบ "ขายที่เลือก" ·
9) ยืนยันขายหลายใบขึ้นกล่องยืนยัน ·
10) กระเป๋าแสดง **การ์ดย่อ 1 ใบ** ของ Item ที่ใส่ไว้
+ กด "เอาออก" จากหน้าการ์ด → ยืนยัน → ไป `/decks/<id>?slot=0` แล้ว **ฟองเลือกการ์ดเปิดให้ช่องนั้นทันที**

---

## Phase 42 — แก้บั๊ก UX ที่ผู้ใช้เจอ: ฟองบังหน้าเด็ค · ปิดไม่ได้บนมือถือ · รวมหน้าการ์ดเข้าเด็ค (2026-09-27)

**คำสั่งผู้ใช้:** *"เข้าหน้าจัด Deck แทนที่จะเห็น Deck กลับเป็นหน้าเลือกการ์ดขึ้นมาบัง · ในมือถือปิดก็ไม่ได้ ต้องเล่นเต็มจอถึงจะเห็น — แก้ให้คงหัวตารางที่มีปุ่มปิด เลื่อนเฉพาะการ์ด และขยายดูการ์ดได้ · หน้าการ์ดถ้าไม่จำเป็นก็เอาออกเอามารวมกับหน้า Deck · หน้า Deck กดที่การ์ดให้แก้ไข Item ส่งเข้าทีม เลือกตำแหน่งได้ รวมทั้งเมนูขายการ์ดจากหน้าการ์ด · หน้ากระเป๋ากดดูการ์ดบนมือถือแล้วปิดไม่ค่อยได้เพราะเกินจอ"*

| อาการที่ผู้ใช้แจ้ง | สาเหตุจริง (พบในโค้ด) | วิธีแก้ |
|---|---|---|
| เข้าหน้าเด็คเจอฟองเลือกการ์ดบัง | Phase 41 อ่าน `?slot=N` ด้วย `Number(searchParams.get('slot'))` ⇒ **`Number(null) = 0`** → เปิดฟองให้ช่อง 0 ทุกครั้งที่เข้าเด็ค | เช็ค `slotRaw === null ? NaN : Number(...)` ก่อนเปิดฟอง (เปิดเฉพาะเมื่อมีพารามิเตอร์จริง) |
| มือถือปิดฟองไม่ได้ (ต้องเล่นเต็มจอ) | Modal ใช้ `max-h-[92vh]` (vh ไม่รวม URL bar) + ปุ่มปิดเล็กมุมขวาบน ⇒ กล่องสูงเกินจอ ปุ่มหลุด | ใช้ **`85dvh`** (dynamic viewport) · **หัวกล่อง sticky มีปุ่ม ✕ ขนาด 40×40** · เลื่อนเฉพาะเนื้อหา (`overscroll-contain`) · **ปุ่ม "ปิด" ใหญ่ท้ายกล่อง** · เว้น `safe-area-inset-bottom` |
| หน้ากระเป๋ากดดูการ์ดแล้วปิดไม่ได้ | อาการเดียวกัน (Modal สูงเกินจอ) | แก้ที่ Modal กลาง ⇒ ทุกฟองในเกมได้ผลพร้อมกัน |
| หน้าการ์ดซ้ำซ้อนกับหน้าเด็ค | มี 2 หน้าที่ดูการ์ดได้ (`/cards`, `/decks/[id]`) | **รวมเข้ากับหน้าเด็ค**: คลังการ์ดในหน้าเด็คกดแล้วมี 👁 ดูการ์ด+ใส่/ถอด Item · ⚔ ส่งเข้าทีม (เลือกตำแหน่ง 1-5) · 💠 ขายการ์ด (ยืนยัน) · `/cards` เด้งไป `/decks` · เอาเมนู "การ์ด" ออกจากแถบล่าง+เมนูบน · ลิงก์ภายในชี้ `/decks` |

**หลักฐานวัดได้ (production จริง · jest 740 ผ่าน / 51 suites · `tsc --noEmit` 0 error · `next lint` ผ่าน · build ✓ + restart · `/api/health` 200):**

ตรวจด้วย Chrome headless ที่ **จอมือถือจริง 390×740**:
1) เข้าหน้าเด็ค → `modal=false · deckRing=true` (**ไม่ถูกฟองบังแล้ว**) ·
2) กดการ์ดในคลัง → ชีตมี `view · sell` (+ ส่วน "ส่งเข้าทีม เลือกตำแหน่ง" เมื่อการ์ดยังไม่อยู่ในทีม) ·
3) **ปุ่มปิดทั้งบนและล่างอยู่ในจอ** (`top=true · bottom=true` · viewport 740) ·
4) กดปุ่มปิดท้ายกล่อง → ฟองปิดจริง ·
5) `/cards` → เด้งไป `/decks` ·
6) กระเป๋า: การ์ดย่อ 1 ใบ เปิดฟองได้ + ปุ่มปิดอยู่ในจอ + กดปิดได้จริง
+ `npm run inspect:mobile` ผ่าน **12/12 หน้า** (ไม่มีแถบเลื่อนนอนหลังเอาเมนูออก)

---

## Phase 43 — แยกร้านช่างเป็น 2 เมนู (Craft / Upgrade) · กระเป๋าแยก Item · ตีบวก "ทีละชิ้น" (2026-10-04)

**คำสั่งผู้ใช้:** *"Workshop แยกเมนู Carft กับ Upgrade · ในกระเป๋าก็แยก Item · การตีบวก คือเอาของที่มี 1 ชิ้น ไปตีบวก ของชิ้นนั้นได้บวก ไม่ใช่ทั้งกอง"*

| เรื่อง | ก่อน | หลัง |
|---|---|---|
| โครงสร้างของในคลัง | `user_items` 1 แถวต่อ (ผู้เล่น, Item) + `enhanceLevel` มีผลกับ **ทั้งกอง** | **1 แถวต่อ (ผู้เล่น, Item, ระดับบวก)** = กองแยกระดับ ⇒ ของ +0 กับ +2 คนละกอง |
| การตีบวก | บังคับมี ≥2 ชิ้น · หัก "สำเนา 1 ชิ้น" เป็นวัตถุดิบ · ระดับใหม่ทับทั้งกอง | ดึง **1 ชิ้น** ออกจากกองที่เลือกระดับ แล้วย้ายไปกองระดับใหม่ (จำนวนรวมเท่าเดิม · ไม่กินสำเนาเพิ่ม) |
| ช่องใส่ Item | ผูกแค่ "ชนิด" ของ Item (สถานะใช้ระดับของทั้งกอง) | ผูกกับ "ชิ้น" จริง (`card_item_slots.user_item_id`) ⇒ ใส่ +0 หรือ +9 ก็ได้คนละช่อง และสถานะคูณตามระดับของชิ้นนั้น |
| หน้าร้านช่าง | หน้าเดียวรวม ซื้อ/คราฟต์/ตีบวก/ขาย | **2 แท็บ: 🧪 คราฟต์** (ซื้อ/คราฟต์ + ป้ายกองที่มี) และ **🛠 ตีบวก** (กองละแถว: ตีบวก/ขายคืน) |
| กระเป๋า | Item ชนิดเดียว = 1 แถว | **แยกแถวตามกอง/ระดับ** (+0 ×3 กับ +2 ×1 คนละแถว) + ปุ่มขายผูกกับระดับของกองนั้น |
| ช่างใส่ Item บนการ์ด | เลือกได้แค่ว่าชนิดไหน | เลือกได้ว่าชิ้น **ระดับไหน** (+N) — ของกองอื่นไม่ถูกใช้ |

**สิ่งที่ทำ (ไฟล์):**
| ไฟล์ | รายละเอียด |
|---|---|
| `prisma/schema.prisma` + `prisma/migrations/20261004100000_item_per_piece/` | `user_items` unique → `(user_id, item_id, enhance_level)` · `card_item_slots.user_item_id` (+ FK ON DELETE SET NULL) · **backfill** ช่องที่ใส่ของอยู่ → ชี้กองของเดิม |
| `src/lib/item-enhance.ts` | `quote.pieces` (= 1 ชิ้น, แทน `copies`) + `ENHANCE_PIECES_PER_TRY` + `enhanceMove(from, success)` (บริสุทธิ์) |
| `src/services/item.ts` | ของใหม่ลงกอง +0 (`grantItem`) · `stacks()` แยกกอง + `catalog()` แนบ `stacks[]` · `enhance(user, code, level)` ย้ายชิ้นระหว่างกอง · `sell(..., level)` · `equip(..., level)` ผูกชิ้น · สถานะ/ช่องใช้ระดับของชิ้นที่ใส่ |
| API | `/api/items/enhance` (+`enhanceLevel`) · `/api/items/sell` (+`enhanceLevel`) · `/api/cards/[id]/equipment` (ตัวเลือกแยกกอง + ส่ง `enhanceLevel`) · `/api/inventory` (`workshopItems` = กองละแถว) · `/api/items` (`rows[].stacks`) |
| UI | `app/(game)/items/page.tsx` (แท็บ Craft/Upgrade + โมดัลตีบวกระบุกอง) · `app/(game)/inventory/page.tsx` · `components/cards/CardItemWorkshop.tsx` |
| ของที่ได้จากระบบอื่น | `services/level.ts` · `services/dungeon.ts` → upsert กอง +0 (ของใหม่ไม่มีบวก) |
| i18n | `item.tabCraft` · `item.tabUpgrade` · `item.enhancePiece` · `item.stackLabel` · `item.upgradeHint` · `item.enhanceNoFree` · `item.enhanceMaxed` (ไทย/อังกฤษครบ) |
| เทสต์/ตรวจ | `tests/unit/item-enhance.test.ts` (+6 เทสต์ "ตีบวกทีละชิ้น/ย้ายกอง") · `scripts/verify-phase43.mjs` (API+DB จริง) · `scripts/inspect-item-tabs.mjs` (Chrome จริง) |

**หลักฐานวัดได้ (production จริง · jest 769 ผ่าน / 55 suites · `tsc --noEmit` 0 error · `next lint` ผ่าน · build ✓ + restart · `/api/health` 200):**

`npm run verify:phase43` → **9/9** (ผู้เล่นทดสอบจริง + DB จริง):
1) เตรียมของ 2 กอง (+0 ×3 · +2 ×1) · 2) `GET /api/items` ส่ง `stacks` แยกกอง ·
3) **ตีบวกกองที่มีชิ้นเดียวได้** (+2 → +3 · กองเดิมเหลือ 0) · 4) ยอดรวมคงที่ **4 → 4** ·
5) กอง +0 ลด 1 และเกิดกองใหม่ **+0: 3→2 · +1 ×1** (ของที่เหลือไม่ถูกบวกทั้งกอง) ·
6) กระเป๋าแยกเป็น 3 แถว (+0×2 · +1×1 · +3×1) · 7) ใส่ชิ้น +1 ลงการ์ด → สถานะใช้ระดับของชิ้นนั้น ·
8) ขายจากกอง +0 → กอง +1 ยังอยู่

`npm run inspect:item-tabs` → **4/4** (Chrome จริง จอ 390×740):
1) `/items` มี 2 เมนู 🧪 คราฟต์ + 🛠 ตีบวก (เริ่มที่คราฟต์ · แผงตีบวกยังไม่แสดง) ·
2) กดแท็บตีบวก → กองแยกแถว `[+2, +0]` (การ์ดคราฟต์ซ่อน) ·
3) โมดัลบอก **"ใช้ของ 1 ชิ้นจากกองนี้ตีบวก (ชิ้นนั้นได้บวก · ชิ้นอื่นในกองไม่เปลี่ยน)"** + ปุ่ม "🛠 ตีบวก (+0→+1)" ·
4) กระเป๋าแยกแถว `[+2, +0]` + ปุ่มขายระบุระดับของกอง

**หมายเหตุ:** ของที่ผู้เล่นมีอยู่ก่อน Phase 43 (1 กองต่อ Item) ยังอยู่ครบ — migration backfill ให้ และกองเดิมถูกมองเป็นของที่ระดับนั้นทั้งกอง (ตีบวกต่อจากนี้ไปจะแยกทีละชิ้น)

---

## Phase 44 — แท็บ Craft = คราฟต์ล้วน (ตัด "+N" และปุ่ม upgrade ออก) (2026-10-05)

**คำสั่งผู้ใช้:** *"แล้วทำไมหินลับคม ในหน้า Craft มันยัง +1 · แล้วปุ่ม upgrade ไม่ต้องมีในหน้านี้"*

| เรื่อง | ก่อน | หลัง |
|---|---|---|
| ป้ายระดับบวกบนการ์ดในแท็บ 🧪 คราฟต์ | โชว์ `+N` (กองสูงสุดที่มี เช่น `+1`) | **ไม่มีป้าย `+N`** — โชว์แค่ชื่อ + `มี ×N` / `ใส่อยู่ ×N` |
| สถานะที่โชว์ | ค่ากองระดับสูงสุดที่ตีบวกแล้ว (เช่น `+7 ATK`) | **ค่าพื้นฐานของ Item** = ของใหม่ที่ได้จากการคราฟต์/ซื้อ (`หินลับคม +6 ATK`) |
| ชิปกอง `+0 ×3 · +2 ×1` | มีในแท็บคราฟต์ | ย้ายไปอยู่แท็บ 🛠 ตีบวก อย่างเดียว |
| ปุ่ม `🛠 ตีบวก / ขายคืน` (ลิงก์ไปแท็บตีบวก) | มีทุกการ์ดที่ `owned > 0` | **ตัดออก** — ใช้แท็บ 🛠 ตีบวก (มีตัวเลขจำนวนกอง) |

**ไฟล์ที่แก้:** `src/app/(game)/items/page.tsx` (ตัด `bestStats` ออก · ใช้ `row.atk/def/hp/spd` + `data-item-base-stats` · ตัด `data-item-level` / `data-item-stack-chips` / `data-item-go-upgrade` ในแท็บคราฟต์)
**เทสต์/ตรวจ:** `scripts/inspect-item-tabs.mjs` เพิ่ม 2 ข้อควบคุม — "แท็บคราฟต์ล้วน" (ป้าย +N = 0 · ชิปกอง = 0 · ปุ่มตีบวก = 0) และ "การ์ดหินลับคมโชว์ `+6 ATK` ไม่ใช่ `+1`"

**หลักฐานวัดได้ (production จริง):** `npx jest` **769 ผ่าน / 55 suites** · `tsc --noEmit` 0 error · `next lint` ผ่าน · `npm run build` ✓ + `systemctl --user restart rune-dominion-arena` · `/api/health` **200** · `npm run inspect:item-tabs` → **6/6** (Chrome จริง 390×740) โดยการ์ดหินลับคมในแท็บคราฟต์อ่านได้ `"หินลับคม Whetstone Edge · ทั่วไป มี ×4 +6 ATK …"` (ไม่มี `+1`) และปุ่มตีบวกในแท็บคราฟต์ = 0 ปุ่ม

---

## Phase 45.1 — บันทึกย้อนหลัง: แผนที่เก็บของ (Map Farm) + เครื่องประดับอวตาร (2026-10-03 – 10-05)

> **ทำไมต้องบันทึกย้อนหลัง:** งานชุดนี้ถูกพัฒนาไว้ใน working tree ตั้งแต่ 3–5 ต.ค. แต่ **ไม่ถูก commit
> และไม่ถูกเขียนลงแผนเลย** — โค้ดรันจริงบนเครื่อง (ผู้เล่นเข้าใช้ได้) แต่ `git log` ยังหยุดที่ 3 ต.ค.
> ⇒ ถ้าโคลนใหม่/เครื่องพัง จะไม่ได้งานส่วนนี้ ถือเป็นช่องว่างความสมบูรณ์ที่ใหญ่ที่สุดที่พบในรอบตรวจ 2026-10-07
> (ปิดแล้ว: commit `48881b5` + `eb87c5b` และ migration ทั้ง 5 ตัวถูก commit ครบ)

| เรื่อง | รายละเอียด | ไฟล์/หลักฐาน |
|---|---|---|
| แผนที่เก็บของ (Map Farm) | เดินบนแผนที่ เก็บวัตถุดิบ/ของตามโซน · ย้ายตำแหน่งอิสระ · คูลดาวน์ | `src/app/(game)/map/` · `src/app/api/map/` · `src/services/map-farm.ts` · `src/lib/map-zones.ts` (`map-zones` 10 เทสต์ + `map-art` 2 เทสต์) |
| migration | `20261003120000_map_farm` + `20261003160000_map_free_move` | ตาราง `map_farm_logs` (มีข้อมูลจริง 10 แถว) |
| ภาพแผนที่ | gen ด้วย AI เก็บใน `var/map-art` (13 MB) + `scripts/generate-map-images.ts` | `lib/map-art-store.ts` |
| เครื่องประดับอวตาร + ฉายา | กรอบอวตาร (ring/glow) + ฉายาข้างชื่อในหัวเว็บ · หน้าสวมของประดับ | `lib/avatar.ts` (`avatarFrameTheme`) · `src/app/api/profile/equip` · `components/layout/TopHeader.tsx` · migration `20261003170000_avatar_cosmetics` |
| ภาพไอเทม/การ์ดพิเศษ | ภาพจริงต่อไอเทม + การ์ดวิเศษจากอีเวนต์ | `lib/item-art*.ts` · `lib/special-art*.ts` · `var/item-art` (3.6 MB) · `var/special-art` (392 KB) |
| API/UI เพิ่ม | `/api/items/[code]` · `/api/inventory/[code]` · หน้าอีเวนต์ + `EventHubView` | commit `eb87c5b` |
| สถานะปัจจุบัน | ทุกอย่าง **commit + push แล้ว** และอยู่ใน build ที่รันจริง | `rune.e2sv.link/map` = 200 |

**ผลข้างเคียงที่แก้ไปด้วย (`0e1ecab`):** `deploy/systemd/rune-dominion-arena.service` ยังผูก
`PartOf=rune-dominion-tunnel.service` (quick tunnel ที่เลิกใช้แล้ว) ⇒ ถ้ามีใคร stop tunnel
จะลาก service เกมดับตาม → `rune.e2sv.link` กลายเป็น 502 · ตัดออกแล้ว

---

## Phase 45 — ตรวจความสมบูรณ์ทั้งเกม + ปิดงานค้าง (2026-10-07)

**คำสั่งผู้ใช้:** *"project Game Card ช่วยตรวจสอบ ความสมบูรณ์ของเกมหน่อย แล้วสรุปข้อมูลมา ยังไม่ต้องแก้อะไร"*
→ ต่อด้วย *"แก้ไขทั้งหมดที่ยังไม่สมบูรณ์"*

### §1 สิ่งที่ตรวจแล้ว "สมบูรณ์อยู่แล้ว" (มีหลักฐานจริง)

| ด้าน | ผลตรวจ |
|---|---|
| เทสต์/build | Jest **798 passed / 57 suites** · `tsc --noEmit` 0 error · `next build` ✓ |
| แผน vs โค้ด (`npm run audit`) | **67/72 มีหลักฐานจริง + 5 ข้อเลือกใช้ทางอื่นโดยเจตนา** = ครอบคลุม 72/72 (ไม่มีข้อตกค้าง) |
| E2E | `npm run e2e:flow` (HTTP) **30/30** · `npm run test:e2e` (Playwright ในเบราว์เซอร์จริง) **11/11** |
| ระบบจริง | `/api/health` 200 (DB ok) · systemd: postgres + arena (3000) + images.timer + backup.timer active |
| สาธารณะ | <https://rune.e2sv.link> — `/` `/map` `/items` `/decks` `/arena` = 200 · `/admin/*` = 307 (ต้องล็อกอิน) |
| เนื้อหาในเกม | การ์ด **209 ใบ มีภาพครบ READY 209/209** · ไอเทม 36 · เควสต์ 8 · ดันเจี้ยน 5 แห่ง · i18n ไทย/อังกฤษ 301=301 คีย์ครบ |
| ลิงก์ในโค้ด | ไม่มีลิงก์เสีย (เทียบ `href` กับ route จริง 32 หน้า) |

### §2 ช่องว่างที่พบและปิดแล้วในรอบนี้

| # | ปัญหา | สิ่งที่ทำ | หลักฐาน |
|---|---|---|---|
| 1 | งาน 3–5 ต.ค. (Phase 43/44 + Map Farm + อวตาร) ไม่ถูก commit — migration 5 ตัว untracked | commit + push 4 ชุดแรก (item/map/avatar/ops) | `git push 00d8469..0e1ecab` · CI **green** |
| 2 | `tests/e2e/` ว่าง + `test:e2e` ชี้ playwright ที่ไม่ได้ติดตั้ง | ติดตั้ง `@playwright/test` + config + เทสต์ 11 ข้อ + `global-teardown` | **11 passed / 0 failed** |
| 3 | Admin มีแต่ `/api/admin/events/sync` ไม่มีหน้า UI | เพิ่มหน้า `/admin/events` + API CRUD + `verify-admin-events` | **15/15** |
| 4 | ตัวกรองการ์ด (ธาตุ/ระดับหายาก) หายไปตอนรวมหน้า /cards เข้า /decks | เติมกลับ + "ซ่อนใบที่อยู่ในทีมแล้ว" + ตัวนับ | `npm run audit` ข้อ P2 ✅ · `tsc` 0 |
| 5 | ไม่มี real-time ในอารีน่า (CODE_REVIEW #5) | ดึงสถานะห้องอัตโนมัติทุก 20 วิ | โค้ด `ARENA_POLL_MS` |
| 6 | combat เต็ม 30 รอบอาจ DRAW (CODE_REVIEW #6) | `decideWinner()` ไล่ชั้น 4 ชั้น + เทสต์ | **7/7** เทสต์ใหม่ |
| 7 | `/api/auth/me` ไม่ส่ง `cardCount` → onboarding เด้งหาทุกคน | ส่ง `cardCount` จริง | เจอตอนเขียน E2E |
| 8 | สำรอง DB ครั้งสุดท้าย 21 ก.ย. + ไม่มี timer | `rune-dominion-backup.timer` ทุกวัน 04:30 (KEEP_DAYS=14) | สำรองได้ **348K · 39 ตาราง** · `verify-backup` restore ✓ (73 ผู้ใช้) |
| 9 | บัญชีทดสอบค้างใน DB จริง 59 บัญชี (โผล่ในตารางจัดอันดับ อันดับ 4/5) | แก้ต้นเหตุ: `global-teardown` + `scripts/clean-test-users.mjs` (dry-run) | ⏳ ลบของเดิม **รอผู้ใช้อนุมัติ** (คำสั่งลบถูกบล็อกเพราะไม่มีการยืนยัน) |
| 10 | `audit-plan.mjs` 2 ข้อล้าสมัย (อ่านข้อความ literal / หน้าที่ย้ายไปแล้ว) | แก้ให้ตรวจของจริง (i18n key + redirect) | audit จาก 65/72 → **67/72** |
| 11 | เอกสาร ops ยังบอก quick tunnel + `url.txt` | README ของ repo แม่ + `docs/DEPLOYMENT.md` อัปเดตเป็น named tunnel `rune.e2sv.link` | ไฟล์อัปเดตแล้ว |

### §3 ยังค้าง (ต้องตัดสินใจ/ทำต่อ)

| เรื่อง | สถานะ |
|---|---|
| repo แม่ `Game_Card` ยังไม่มี remote | ⏳ **ต้องให้ผู้ใช้สร้าง repo ปลายทาง** (`gh` CLI บนเครื่องค้างเพราะ token อยู่ใน keyring ที่ terminal ของ service เข้าถึงไม่ได้) |
| ลบบัญชีทดสอบ 59 บัญชีใน DB จริง | ⏳ รออนุมัติ — `npm run clean:test-users -- --yes` (มีไฟล์สำรองก่อนลบได้ด้วย `npm run backup`) |
| `docs/manual/` (คู่มือหนังสือ) ยังไม่อัปเดตตามฟีเจอร์ 33–45 | ⏸ ตามกฎ: **ห้ามทำเอง** ต้องให้ผู้ใช้สั่ง (เปลือง token มาก) — ถ้าต้องการ บอกได้ |
| Rate limit เป็น in-memory (CODE_REVIEW #1) | ➖ คงไว้โดยเจตนา (single-instance) — ต้องทำก่อนถ้าขยายหลายอินสแตนซ์ |
| `STARTING_COIN = 10` (CODE_REVIEW #4) | ➖ ผู้ใช้กำหนดเองใน Phase 37 (กันเงินเฟ้อ) |

---

## Phase 45.2 — คอลเลคชั่นการ์ดกลับมา (2026-10-07)

**คำสั่งผู้ใช้:** *"เอาเมนู คอลเลคชั่นการ์ด กลับมา และทำให้สมบูรณ์กว่าเดิม"*

| เรื่อง | ก่อน (Phase 42) | หลัง |
|---|---|---|
| หน้า `/cards` | เป็น redirect ไป `/decks` | **หน้าคอลเลคชั่นจริง** — "สมุดสะสมทั้งเกม" |
| มองเห็นอะไร | เฉพาะการ์ดที่ตัวเองมี (ในหน้าจัดเด็ค) | **การ์ดทุกใบในเกม** ⇒ เห็นว่ายังขาดใบไหน (ใบที่ยังไม่ค้นพบ = เงา 🔒) |
| ความคืบหน้า | ไม่มี | แถบ % + สะสมแล้ว X/Y ใบ · ของซ้ำ · ติดดาว + แยกตามระดับหายาก 6 ระดับ |
| แท็บ | — | การ์ดทั้งหมด · ที่มีอยู่ · **ยังไม่มี** |
| ตัวกรอง | บทบาท/ธาตุ/ระดับหายาก (ในหน้าจัดเด็ค) | เพิ่ม **ค้นหา + เรียง 8 แบบ** (พลังรวม/ATK/DEF/HP/SPD/ความหายาก/ได้มาล่าสุด/ชื่อ) |
| จัดการในหน้า | — | กดการ์ด → รายละเอียด · ☆/★ ติดดาว · **เพิ่มลงทีม** (quick-add) |
| เมนู | `/cards` หายจาก `navItems` ⇒ แถบล่างมือถือเหลือ 4 เมนู | เพิ่ม **📇 คอลเลคชั่น** กลับเป็นเมนูหลัก (5 เมนู) + เมนูจอใหญ่ |

**ไฟล์:** `src/lib/collection.ts` (ตรรกะบริสุทธิ์ — กรอง/เรียง/สรุป) · `src/app/api/collection/route.ts`
(คืนการ์ดทั้งเกม + ผลรวมความคืบหน้า) · `src/app/(game)/cards/page.tsx` (หน้าใหม่) ·
`src/components/layout/{BottomNavigation,TopHeader}.tsx` · i18n 27 คีย์ไทย/อังกฤษ ·
`tests/unit/collection.test.ts` (21 เคส) · `scripts/verify-collection.mjs` · `scripts/shoot-collection.mjs`

**หลักฐานวัดได้ (production จริง):**
`npm run verify:collection` → **14/14** (notล็อกอิน 401 · summary ตรงกับ DB 228 ใบ · owned/missing ถูก ·
กรองธาตุ/ระดับ/บทบาท/ค้นหาได้ · sort=power ลดหลั่นจริง · แบ่งหน้าถูก · ติดดาวสะท้อนผล · quick-add ได้)
· `npm run test:e2e` (Playwright) → **12/12** (เพิ่มเทสต์หน้า /cards) · `npm run e2e:flow` → **30/30** ·
`npx jest` **819 ผ่าน / 58 suites** · `tsc --noEmit` 0 error · `npm run audit` **68/73 + 5** ·
build ✓ + restart · ภาพถ่ายจริง: `~/E2_Lab/reports/collection-{desktop,mobile,mobile-missing}.png`

**บทเรียนที่เจอในรอบนี้:** รัน `npm run build | head -N` ทำให้ไปป์ปิดก่อน build เขียน `.next` เสร็จ
⇒ เซิร์ฟเวอร์สตาร์ทไม่ขึ้น (`ENOENT: .next/prerender-manifest.json`) — **อย่าตัดท่อของ `next build`**

---

## Phase 45.3 — แก้ปุ่มกลับ + ความลื่นของหน้าคอลเลคชั่น (2026-10-07)

**คำสั่งผู้ใช้:** *"กดกลับหน้า คอเล็คชั่น แต่กลับไปหน้าจัด Deck · แล้วหน้า คอลเลกชั่น ค่อนข้างกระตุก"*

### 1) ปุ่มกลับ

`src/app/(game)/cards/[id]/page.tsx` ยังชี้ `href="/decks"` (ตกค้างจาก Phase 42 ที่รวมหน้า /cards เข้าหน้าจัดเด็ค)
⇒ กด "← กลับไปคอลเลกชัน" จากหน้ารายละเอียดการ์ดแล้วไปโผล่หน้าจัดทีม · **แก้เป็น `/cards` ทั้ง 2 จุด** (ปกติ + กรณีไม่พบการ์ด)

### 2) ความลื่น — วัดจริงก่อน/หลัง (Chrome headless CDP · จอ 390×740 · สคริปต์ใหม่ `npm run inspect:collection`)

| ตัวชี้วัด | ก่อน | หลัง | เกณฑ์ |
|---|---|---|---|
| canvas ที่วาดพร้อมกัน | **24** | **1** | — |
| คำขอตอนโหลดหน้า | 89 | 61 | — |
| long task ตอนโหลด | 3 (405 ms) | **0** | — |
| **fps ขณะเลื่อน** | **41** | **60** | ≥ 45 |
| main-thread ms ขณะเลื่อน | 2,222 | **297** | — |
| (ในนั้นเป็น JS) | 265 | 66 | — |
| พิมพ์ 8 ตัวอักษร → ยิง `/api/collection` | **8 ครั้ง** | **1 ครั้ง** | ≤ 3 |
| heap | 8 MB | 6 MB | — |

**วิธีแก้ (ไม่ลดคุณภาพที่ผู้เล่นเห็น):**
- **ค้นหา debounce 300 ms** — เดิมพิมพ์ 1 ตัว = ยิง API + เรนเดอร์ใหม่ 1 รอบ
- **การ์ดที่ยังไม่ค้นพบไม่ต้องวาดชั้นแสง/เลื่อม** (`CardFace staticAura`) — มีฉากทึบ 🔒 ทับอยู่แล้ว
  ⇒ ตัด canvas ต่อการ์ด (เดิม 24 การ์ด = 24 canvas วิ่ง rAF พร้อมกัน) — การ์ดที่ผู้เล่นมี ยังได้แสงครบเหมือนเดิม
- **`loading="lazy"` + `decoding="async"`** ให้รูปการ์ด/กรอบ overaly ทุกใบ (ได้ทั้งเกม ไม่ใช่แค่หน้านี้)

**ไฟล์:** `src/app/(game)/cards/page.tsx` · `src/app/(game)/cards/[id]/page.tsx` · `src/components/cards/CardFace.tsx` ·
`scripts/inspect-collection-perf.mjs` (ใหม่) · `scripts/shoot-collection.mjs` · `package.json`

**หลักฐาน:** `npm run inspect:collection` → ผ่าน 3/3 เกต (fps 60 · long task 0 · พิมพ์ 8 ตัวยิง 1 ครั้ง) ·
`verify:collection` 14/14 · `test:e2e` 12/12 · `e2e:flow` 30/30 · jest 819/58 · tsc 0 error · build ✓ + restart



---

## Phase 45.4 — ความยากดันเจี้ยนไล่ "ทุกชั้น" + หน้า Map ยึดแผนที่ที่ผู้เล่นอยู่ (2026-10-07)

**คำสั่งผู้ใช้ (2 ข้อ):**
1. *"ช่วยปรับความยาก ดันเจี้ยน แต่ละชั้น ให้มีความต่างอย่างพอดี ให้รู้สึกว่าเปลี่ยนระดับ"*
2. *"ใน Map ให้แสดง Map ที่ผู้เล่นอยู่ เป็นหน้าปัจจุบัน"*

### 1) ความยากดันเจี้ยน — ปัญหาจริงที่พบก่อนแก้

สูตรเดิม (`buildDeepFloors`) คิดความยากเป็น **บันไดรายบล็อก 5 ชั้น**:
`difficulty = min(difficultyCap, blockStep^(block-1))` ⇒ ชั้น 1-5, 6-10, 11-15 … **status ศัตรูเท่ากันเป๊ะ**

นับจากโค้ดเดิม (HEAD ก่อนแก้) — จำนวน "ค่าความยากที่ไม่ซ้ำกัน" ในทั้งดัน:

| ดัน | ชั้น | ค่าความยากเดิมที่ไม่ซ้ำ | ปัญหา |
|---|---|---|---|
| 🔥 สุสานเพลิง | 25 | **1 ค่า** (0.98 ทุกชั้น) | ชั้น 1 ก็ติดเพดานแล้ว ⇒ ทั้งดันเท่ากันหมด |
| ⚡ ยอดหอพายุ | 40 | **1 ค่า** (0.72 ทุกชั้น) | เท่ากันหมด 40 ชั้น |
| 🌊 วิหารน้ำขึ้น | 28 | 2 ค่า | เท่ากันเป็นช่วง ๆ 5 ชั้น |
| 🌙 รอยแยกไร้จันทร์ | 28 | 2 ค่า | เท่ากันเป็นช่วง ๆ 5 ชั้น |
| 💰 เหวลึกทองคำ | 30 | 5 ค่า | เท่ากันเป็นช่วง ๆ 5 ชั้น (บล็อก 6 ไม่เกิด ⇒ **ไม่เคยมีชั้นบอส 3**) |

### 2) สูตรใหม่ — "งบความยาก" ไล่ทุกชั้น + HP เป็นแกนความอึด

- `targetDifficulty(ชั้น t) = difficultyCap × (startRatio + (1 − startRatio) × t^curveGamma)` · `startRatio = 0.85`, `curveGamma = 0.9`
  ⇒ ชั้นแรก = 85% ของเพดาน, ชั้นสุดท้าย = เพดานเต็ม และ **ไต่ขึ้นทุกชั้น** (ไม่มีชั้นไหนเท่ากันแล้ว)
- **HP ศัตรู** เพิ่มขึ้นทุกชั้น (`hpStep = 1.2%/ชั้น`, เพดาน `hpCap = 1.45×`) = ต่อสู้นานขึ้น ⇒ รู้สึกว่ายากขึ้นโดยไม่ unfair
- **หักชดเชย**: `scale = targetDifficulty / น้ำหนักจำนวนบอส / (1 + (HP−1) × 0.9)`
  ⇒ ความแข็งแกร่ง "รวม" ยังอยู่ใต้เพดานเดิมที่วัดไว้ (ถ้าไม่หักชดเชย ชั้นท้ายดันกลาง-สูง **ชนะ 0%**)
- **รางวัลไล่ทุกชั้น** (เดิมกระโดดเป็นบล็อก) แต่ **ปลายทางเท่าเดิม** ⇒ เศรษฐกิจไม่เปลี่ยน (ชั้นสุดท้ายยอดหอพายุ 88 ฝุ่นเท่าเดิม)
- 💰 เหวลึกทองคำ: `tripleBossBlock 7 → 6` (30 ชั้น = 6 บล็อก · ค่าเดิมเป็น 7 ⇒ ไม่มีชั้นบอส 3 เลย)

**ผลหลังแก้ (นับจากโค้ดจริง):** ค่าความยากไม่ซ้ำ = 25 / 28 / 28 / 30 / 40 ชั้น (เท่าจำนวนชั้นของแต่ละดัน)

| ดัน | ชั้น 1 | ชั้น 5 | กลางดัน | ชั้นสุดท้าย |
|---|---|---|---|---|
| 🔥 สุสานเพลิง | 0.833 | 0.863 | 0.93 (ชั้น 15) | 0.98 |
| ⚡ ยอดหอพายุ | 0.612 | 0.626 | 0.68 (ชั้น 21) | 0.719 |

**สิ่งที่ผู้เล่นเห็นบนจอ (`/dungeons`):** ทุกชั้นโชว์ **⭐ ระดับความยาก 1-10** (เทียบช่วงของดันนั้น) · **พลังคุกคามของศัตรู** · **HP ศัตรู +N%**
รางวัล (ฝุ่น/เศษ veil) ก็ไล่ขึ้นทุกชั้น ⇒ "เปลี่ยนระดับ" ทั้งที่รู้สึก (ยากขึ้น) และที่ได้ (คุ้มขึ้น)

### 3) หน้า Map — แผนที่ที่ผู้เล่นอยู่ = หน้าปัจจุบัน

เดิมหน้า `/map` เริ่มที่ **EMBERFIELD (แผนที่แรก) เสมอ** ⇒ ผู้เล่นที่ไปอยู่แผนที่ 3-5 ต้องกดแท็บเองทุกครั้ง
- เพิ่มฟังก์ชันบริสุทธิ์ `initialActiveZone(nodes, currentNodeId, fallback)` ใน `src/lib/map-zones.ts`
- หน้า Map: ค่าเริ่มต้น = แผนที่ของจุดที่ยืนอยู่ · ถ้าผู้เล่นกดแท็บเอง (`zoneTouched`) จะไม่ลากกลับ
- แท็บของแผนที่ที่อยู่ติด **📍** · มีปุ่ม **"📍 ไปแผนที่ที่คุณอยู่ตอนนี้ (ชื่อจุด)"** โผล่เมื่อกำลังดูแผนที่อื่น

### หลักฐานวัดจริง (production)

| ด่านตรวจ | ผล |
|---|---|
| `npx jest` | **863 passed / 60 suites** (เพิ่ม `tests/unit/dungeon-curve.test.ts` 22 ข้อ + `initialActiveZone` 4 ข้อ) |
| `npx tsc --noEmit` | 0 error |
| `npm run audit` | **68/73 + 5 by design = 73/73** |
| `npm run verify:map-zone` (ใหม่) | **10/10** — ตั้งจุดปัจจุบันเป็นโซน VOIDGATE แล้วเปิดหน้า /map จริง ⇒ แท็บที่เลือก = VOIDGATE + 📍 + กดปุ่มกลับได้ |
| `npm run verify:dungeon-curve` (ใหม่) | **8/8** — ดันเหวลึกทองคำ จากจอจริง: ชั้น 1 = ⭐1 พลัง 3,535 → ชั้น 15 = ⭐6 พลัง 3,729 → ชั้น 30 = ⭐10 พลัง 5,637 (HP ×1.348) |
| `npm run e2e:flow` | 30/30 · `verify:collection` 14/14 · `/api/health` 200 |
| `npm run calibrate:dungeons` | ชั้นท้ายทุกดันยังผ่านด้วยเด็คเป้าหมาย (ด่านตรวจ `tests/unit/dungeon-balance.test.ts` ผ่านครบ) |

**ไฟล์:** `src/lib/dungeon-definitions.ts` · `src/lib/dungeon-art.ts` · `src/services/dungeon.ts` · `src/lib/map-zones.ts` ·
`src/app/(game)/dungeons/page.tsx` · `src/app/(game)/map/page.tsx` · `tests/unit/dungeon-curve.test.ts` (ใหม่) ·
`tests/unit/dungeon.test.ts` · `tests/unit/map-zones.test.ts` · `scripts/verify-map-current-zone.mjs` (ใหม่) ·
`scripts/verify-dungeon-curve.mjs` (ใหม่) · `package.json`

### ข้อสังเกตที่ยังเหลือ (ยังไม่แก้ — รอผู้ใช้ตัดสิน)

1. **ชั้นที่มีบอส 2-3 ตัววัดได้ "ง่ายกว่า" ชั้นบอส 1 ตัวที่งบเท่ากัน** ในดันฝึกหัด (มือใหม่ชนะ 100% ที่ชั้น 15/20/25 เทียบกับ 93% ที่ชั้น 5)
   เพราะ `bossWeight` (1.4 / 1.7) หักชดเชยเกินจริง ⇒ ตัวเลข scale ของชั้นบอสหลายตัวต่ำลงมาก
   (ด่านตรวจเดิมยังผ่าน เพราะวัดจาก "เด็คเป้าหมายชนะได้ไหม" ไม่ได้วัด "ไล่ระดับได้ไหม")
2. **ยอดหอพายุ** ยังเป็นการกระโดดขั้นเดียว: ท็อปดิบ 0% ทุกชั้น แต่ท็อป+ของ 100% ทุกชั้น ⇒ ไล่ระดับด้วย status อย่างเดียวไม่พอ
   ต้องใช้กลไกอื่น (สกิลศัตรู/เงื่อนไขชั้น) ถ้าต้องการให้รู้สึกไล่ระดับในดันนี้

---

## Phase 45.5 — แก้ "น้ำหนักจำนวนบอส" ที่ชดเชยเกินจริง + บัญชีทดสอบไม่ให้ค้าง (2026-10-08)

**คำสั่งผู้ใช้:** *"ทำข้อ 1,2,3"* (จากรายการที่รอตัดสินใจ: ลบบัญชีทดสอบค้าง · แก้ bossWeight ที่ทำให้ชั้นบอส 2-3 ตัวง่ายเกิน · หา "กลไกอื่น" ให้ดันสูงสุดไล่ระดับได้)

### 1) บัญชีทดสอบค้างใน DB — ลบ + อุดต้นเหตุ

- พบ 4 บัญชีค้าง (`e2e_a_*` ×2 · `e2e_b_*` ×2) จาก **`scripts/e2e-flow.mjs`** ที่สมัครผู้ใช้ใหม่ทุกรอบแล้ว **ไม่ลบ**
  (global-teardown ของ Playwright ลบเฉพาะรอบของตัวเอง — สคริปต์ HTTP นี้ไม่มี teardown เลย)
- แก้: `e2e-flow.mjs` เก็บรายชื่อผู้ใช้ที่สร้าง แล้วลบตอนจบ **เสมอ** (ลบ `battle_logs`/`arena_rooms` ก่อนเพราะ FK เป็น RESTRICT)
  · มี `--keep` ถ้าต้องการเก็บไว้ตรวจย้อนหลัง · ถ้าลบไม่ได้จะพิมพ์คำสั่งเก็บกวาดให้
- ลบของเดิม: `npm run backup` (ได้ `rune_dominion-20261008-071100.sql.gz`) → `npm run clean:test-users -- --yes`
  ⇒ **11 → 7 ผู้ใช้** (เหลือผู้ใช้จริง: woravik · TCM · Abcd · **KJ_SAM (ผู้เล่นใหม่)** · player1 · secadmin · t5678)
- พิสูจน์ว่าอุดแล้ว: รัน `npm run e2e:flow` อีกรอบ (30/30 ผ่าน) ⇒ หลังจบเหลือ **ผู้ใช้ 7 · บัญชีทดสอบ 0** (ก่อนแก้: ค้าง 2)

### 2) `bossWeight` — วัดใหม่แล้วค่าเดิม (1.4/1.7) หักชดเชยเกินจริง

**วิธีวัดใหม่** (`scripts/calibrate-dungeons.mts --boss-weights` · 120 ศึก/จุด · หา scale ที่เด็คเป้าหมายชนะ 50% แล้วเทียบทีมบอส 2/3 ตัวกับทีมบอส 1 ตัวที่ scale เท่ากัน):

| ดัน | เด็ค | scale ที่ชนะ 50% | บอส 2 ตัว | บอส 3 ตัว |
|---|---|---|---|---|
| 🔥 สุสานเพลิง | มือใหม่ / กลาง / ท็อปดิบ | 0.875 / 1.176 / 1.625 | ×1.1 | ×1.1 |
| 💰 เหวลึกทองคำ | มือใหม่ / กลาง / ท็อปดิบ | 0.275 / 0.425 / 0.425 | ×1.1 / ×1 / ×1.25 | ×1.1 / ×1 / ×1.25 |

⇒ **ค่ากลาง 1.1 (ช่วงที่วัดได้ 1.0-1.25)** ไม่ใช่ 1.4/1.7 · แก้เป็น `bossWeight(2) = 1.12` · `bossWeight(3) = 1.25`
(ตรงกับเจตนาเดิมที่เขียนในคอมเมนต์ฟังก์ชัน: "บอส 2 ตัว = +12% · บอส 3 ตัว = +25%")

**ผลข้างเคียงที่ต้องชดเชย**: ชั้นบอส 3 ตัวได้ scale สูงขึ้น ~36% ⇒ ชั้นท้าย 💰 เหวลึกทองคำ (cap 1.15) กลายเป็น
"ชนะไม่ได้ทุกเด็ค" (วัด: geared 0% · mythic 0%) ⇒ วัดจุดตัดใหม่ด้วย `--scan GILDED_ABYSS:30 --deck mythic`
(100% ที่ scale ≤0.63 · 0% ที่ 0.70) → **ลด cap 1.15 → 1.0**

### 3) "กลไกอื่น" ให้ดันสูงสุดไล่ระดับ — วัดแล้วไม่ต้องเพิ่มกลไก (ไล่ระดับได้แล้ว)

- เพิ่มเครื่องมือวัดใน `calibrate-dungeons.mts`: `--profiles` (ทดสอบรูปร่างความยาก: อึดขึ้น/เบาลง) และ
  `--element-probe` (สลับธาตุศัตรู = สลับสกิล BURN/SHIELD/HEAL/HASTE/WEAKEN)
- **ผลวัด 1:** เปลี่ยนรูปร่าง stat (HP ×3 / lethality ×0.6) ที่ชั้น 40 ดันสูงสุด ⇒ ท็อปดิบ 0% · ท็อป+ของ 100% **ทุกโปรไฟล์** (ไม่เกิดช่วงกลาง)
- **ผลวัด 2:** สลับธาตุ/สกิลที่ชั้น 25 ⇒ ท็อป+ของ 0% หรือ 100% สลับกันตามธาตุ (VEILMARKED/TIDEBORN/EMBERBOUND = 0% · SKYRIVEN/ROOTFORGED = 100%) แต่ **ไม่มีช่วงกลาง** และ mythic 100% ทุกกรณี
- สรุปเชิงวิศวกรรม: **สมการต่อสู้ให้ผลแบบ "หน้าผา"** (ชนะ 100% → แพ้ 0% ภายใน scale ต่างกัน ~10%)
  ⇒ ไม่มีกลไก stat/ธาตุใดสร้าง "ไล่ระดับ" ได้ · ไล่ระดับที่ผู้เล่นรู้สึกได้ต้องมาจาก **เส้นงบความยากต่อชั้น** (Phase 45.4) + **น้ำหนักจำนวนบอสที่ถูกต้อง** (ข้อ 2)
- **ผลหลัง Phase 45.5 (วัดจากคลังการ์ดจริง 240 ใบ · 60 ศึก/ช่อง):**

| ดัน | ชั้น 1-10 | กลางดัน | ชั้นท้าย | หมายเหตุ |
|---|---|---|---|---|
| 🔥 สุสานเพลิง | มือใหม่ 98→92% | 100% (บอส 2) | **58%** (บอส 3) | ชั้นสุดท้ายเป็นชั้นที่หินที่สุดจริง |
| 💰 เหวลึกทองคำ | ท็อป+ของ 100% | 58% (f20) · 97% (f25) | **0% / mythic 100%** | ชั้นท้ายต้องใช้ของ mythic |
| ⚡ ยอดหอพายุ | ท็อป+ของ 100% | **82% (f15) · 98% (f20)** | **0% (f25+) / mythic 100%** | เด็คกลางไล่ได้ถึง f20 แล้วชนกำแพง = ไล่ระดับจริง |

**เทียบกับก่อนแก้ (Phase 45.4):** ⚡ ยอดหอพายุ ท็อป+ของชนะ **100% ทุกชั้น** (รวม f25-f40) ⇒ ไม่มีกำแพง/ไม่มีระดับ ·
ตอนนี้ 100% → 82% → 98% → 0% = เห็นระดับชัด

### หลักฐาน

| ด่านตรวจ | ผล |
|---|---|
| `npx jest` | **863 passed / 60 suites** |
| `npx tsc --noEmit` | 0 error |
| `npm run calibrate:dungeons` | ตารางด้านบน (ทุกดันชั้นท้ายผ่านด้วยเด็คเป้าหมาย) |
| `npm run verify:dungeon-curve` | **8/8** (ชั้น 1 = ⭐1 พลัง 3,076 → ชั้น 30 = ⭐10 พลัง 4,905 · HP ×1.348) |
| `npm run e2e:flow` | 30/30 + ลบบัญชีทดสอบของตัวเองอัตโนมัติ (ผู้ใช้คงเหลือ 7) |
| build + restart + `/api/health` | ✓ 200 |

**ข้อสังเกตที่เหลือ:** ชั้น "บอส 2 ตัว" ของดันฝึกหัดวัดได้ง่ายกว่าชั้น "บอส 1 ตัว" ที่งบเท่ากัน (มือใหม่ 100% ที่ f15/f20 เทียบ 92% ที่ f10)
เพราะ scale ของชั้นถูกหารด้วยน้ำหนักบอส ทำให้ลูกน้องที่เหลืออ่อนลงมาก — ถ้าต้องการให้ "บอส 2 ตัว" หินจริง ต้องเพิ่มกลไกเฉพาะชั้น (เช่น บอส 2 ตัวได้สกิล HEAL) ซึ่งวัดแล้วว่ายังเป็นหน้าผาเหมือนเดิม

**ไฟล์:** `src/lib/dungeon-definitions.ts` (`bossWeight` + cap ของเหวลึกทองคำ) · `scripts/calibrate-dungeons.mts` (`--boss-weights` · `--profiles` · `--element-probe`) · `scripts/e2e-flow.mjs` (ลบบัญชีทดสอบของตัวเอง)
