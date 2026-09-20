# Rune Dominion Arena
## Game Design Document + Technical, UI/UX, Art, Narrative และ LiveOps Prompts

**เวอร์ชัน:** 0.1  
**ประเภท:** Fantasy Trading Card Game / Auto Battle / Competitive Arena  
**แพลตฟอร์ม:** Mobile-first Web App  
**ภาษา UI หลัก:** ไทย  
**สถานะ:** MVP Planning

---

# 1. ภาพรวมเกม

## 1.1 Elevator Pitch

Rune Dominion Arena คือเกมการ์ดแฟนตาซีที่ผู้เล่นค้นพบการ์ดผ่านการเลือก “รูน” จากตารางอักขระขนาด 100×100 ผู้เล่นจะนำลำดับรูนไปถอดรหัสเพื่อค้นหาการ์ดเฉพาะตัว สะสมการ์ด จัดทีม 5 ใบ ต่อสู้แบบ Auto Battle และแข่งขันใน Arena ระยะเวลา 24 ชั่วโมง

ทุกลำดับรูนถูกแปลงเป็น Seed แบบกำหนดผลลัพธ์ตายตัว หากผู้เล่นสองคนใช้ลำดับเดียวกัน จะได้รับการ์ดใบเดียวกันจากฐานข้อมูล โดยไม่สร้างการ์ดหรือภาพใหม่ซ้ำ

## 1.2 Core Loop

```text
เลือก Rune
  → ถอดรหัส Rune
  → ค้นพบ/รับการ์ด
  → จัดทีม 5 ใบ
  → ต่อสู้ Auto Battle
  → รับ Coin / EXP / วัตถุดิบ
  → เปิดหรือเข้าห้อง Arena
  → ปรับเด็คและค้นหา Rune ต่อ
```

## 1.3 ข้อจำกัดด้านเศรษฐกิจ

- Coin เป็นสกุลเงินในเกมเท่านั้น
- Coin ได้จากการเล่น ภารกิจ รางวัล และกิจกรรม
- Coin ห้ามแลกเป็นเงินจริง
- Coin ห้ามถอนออกเป็นเงินจริง
- Coin ห้ามโอนระหว่างผู้เล่น
- ทุกธุรกรรม Coin ต้องบันทึกใน Coin Ledger
- ใช้จำนวนเต็มเท่านั้น ห้ามใช้ Float กับสกุลเงิน

---

# 2. โลกของเกม: Aetherra

## 2.1 Lore หลัก

ก่อนที่แผ่นดินจะมีชื่อ ก่อนที่ทะเลจะรู้จักกระแสน้ำ มีแสงเส้นหนึ่งพาดผ่านท้องฟ้าของ Aetherra และทิ้งร่องรอยเป็นอักขระที่ไม่มีผู้ใดอ่านออก

ผู้คนเรียกมันว่า “อักษรแรกเริ่ม” ร่องรอยเหล่านั้นไม่ได้บันทึกเพียงคำพูด แต่เก็บเศษเสี้ยวของผู้กล้า สัตว์อสูร ปราการที่สูญหาย และคำสาบานซึ่งไม่เคยเสื่อมสลายไว้ภายใน

เมื่ออักษรแตกกระจายไปทั่วดินแดน ผู้ที่สัมผัสลำดับรูนได้ถูกเรียกว่า Rune Seeker พวกเขาสามารถปลุก Echo หรือร่องรอยที่หลับใหล ให้กลับมาปรากฏในรูปของการ์ด

แต่ยิ่งมีผู้ค้นพบมากขึ้น ความจริงหนึ่งก็เริ่มชัดเจน: อักษรแรกเริ่มไม่ได้แตกสลายเพราะอุบัติเหตุ และบางส่วนของมันอาจกำลังพยายามเรียกสิ่งที่ถูกผนึกไว้ให้ตื่นขึ้นอีกครั้ง

## 2.2 กลุ่มธาตุหลัก

| กลุ่ม | ธาตุ | แนวคิด | บทบาทโดยรวม |
|---|---|---|---|
| Emberbound | Fire | การเปลี่ยนแปลงและการต่อสู้ | นักรบ ช่างตีอาวุธ |
| Tideborn | Water | ความทรงจำและการปรับตัว | ผู้รักษา นักปราชญ์ |
| Skyriven | Wind | เสรีภาพ ข่าวสาร ความเร็ว | ผู้ส่งสาร นักล่า |
| Rootforged | Earth | การปกป้องและความอดทน | ผู้พิทักษ์ ช่างแกะสลัก |
| Dawnsworn | Light | ความหวังและคำมั่นสัญญา | ผู้เยียวยา นักดาราศาสตร์ |
| Veilmarked | Shadow | ความลับและความจริงที่ซ่อนอยู่ | ผู้สังเกตการณ์ นักเวท |

---

# 3. ระบบค้นหาการ์ดด้วย Rune

## 3.1 Rune Grid

- ตาราง Rune ขนาด 100×100
- รวมตำแหน่ง Rune 10,000 จุด
- ผู้เล่นเลือก Rune จำนวน 8–16 จุดต่อการค้นหา 1 ครั้ง
- Rune แต่ละตำแหน่งแทนด้วยเลข `0–9999`
- UI ต้องใช้ Canvas หรือ Virtualized Grid
- ห้าม Render ปุ่ม DOM จำนวน 10,000 ปุ่มพร้อมกัน

## 3.2 Discovery Flow

```text
ผู้เล่นเลือกลำดับ Rune
  → Client ส่ง runeSequence ไปยัง Server
  → Server ตรวจสอบรูปแบบและพลังค้นหา
  → Server สร้าง Canonical String
  → Server Hash ด้วย SHA-256 + Server Pepper
  → ค้นหา CardDefinition จาก canonicalSeedHash
  → พบ: คืนข้อมูลการ์ดเดิม
  → ไม่พบ: สร้างการ์ดใหม่ + สร้างงาน AI Image
```

## 3.3 Canonical String

ตัวอย่าง Rune Sequence:

```json
[182, 9931, 2045, 18, 5031, 744, 8800, 19]
```

Canonical String:

```text
version=1|runes=0182,9931,2045,0018,5031,0744,8800,0019
```

Hash:

```text
SHA-256(canonicalString + SERVER_PEPPER)
```

## 3.4 กติกาสำคัญ

- ห้ามใช้ `Math.random()` เพื่อสร้างข้อมูลการ์ด
- การสร้าง Card Metadata ต้องทำจาก Deterministic PRNG
- Seed เดิมต้องคืนการ์ดเดิมเสมอ
- `canonicalSeedHash` ต้องเป็น Unique Constraint
- Server Pepper ห้ามเปิดเผยแก่ Client
- ทุก Discovery Request ต้อง Idempotent
- ใช้ Database Transaction เพื่อป้องกัน Race Condition
- ภาพ AI สร้างครั้งเดียวเฉพาะการ์ดใหม่
- ระหว่างภาพยังไม่พร้อม ให้ใช้ Elemental Placeholder

---

# 4. ระบบการ์ด

## 4.1 ธาตุ

| ธาตุ | จุดเด่น | ชนะทาง |
|---|---|---|
| Fire | ดาเมจ, Burn, ทำลายโล่ | Wind |
| Water | Heal, Cleanse, ลดพลัง | Fire |
| Wind | ความเร็ว, หลบ, โจมตีหลัง | Earth |
| Earth | เกราะ, HP, ลด AoE | Water |
| Light | Shield, Heal, สนับสนุน | Shadow |
| Shadow | Debuff, Curse, Control | Light |

## 4.2 บทบาทการ์ด

- `TANK`: รับความเสียหาย ปกป้องทีม
- `FIGHTER`: โจมตีระยะประชิด สมดุล
- `RANGER`: ดาเมจระยะไกล โจมตีแนวกลาง/หลัง
- `MAGE`: AoE, Debuff, Control
- `SUPPORT`: Heal, Shield, Buff, Cleanse
- `ASSASSIN`: ความเร็วสูง โจมตีแนวหลัง

## 4.3 ระดับความหายาก

| ระดับ | โอกาส | แนวทาง |
|---|---:|---|
| Common | 55% | การ์ดพื้นฐาน เข้าใจง่าย |
| Uncommon | 25% | มีจุดเด่นเล็กน้อย |
| Rare | 12% | มีสกิลเฉพาะ |
| Epic | 6% | เป็นแกนทีมได้ |
| Legendary | 1.8% | ความสามารถโดดเด่น |
| Mythic | 0.2% | มีเอกลักษณ์สูงและข้อจำกัด |

## 4.4 CardDefinition Fields

```ts
type CardDefinition = {
  id: string;
  canonicalSeedHash: string;
  generationVersion: number;
  name: string;
  lore: string;
  element: "FIRE" | "WATER" | "WIND" | "EARTH" | "LIGHT" | "SHADOW";
  role: "TANK" | "FIGHTER" | "RANGER" | "MAGE" | "SUPPORT" | "ASSASSIN";
  rarity: "COMMON" | "UNCOMMON" | "RARE" | "EPIC" | "LEGENDARY" | "MYTHIC";
  attack: number;
  defense: number;
  health: number;
  speed: number;
  manaCost: number;
  skillName: string;
  skillType: string;
  skillPower: number;
  skillDescription: string;
  imagePrompt: string;
  imageUrl?: string;
  imageStatus: "PENDING" | "READY" | "FAILED" | "REJECTED";
  discoveredByUserId: string;
  discoveredAt: Date;
};
```

---

# 5. ระบบทีมและ Deck Builder

## 5.1 ทีม 5 ใบ

| ตำแหน่ง | จำนวน | Role ที่เหมาะ |
|---|---:|---|
| Frontline | 2 | Tank, Fighter |
| Midline | 2 | Fighter, Ranger, Mage |
| Backline | 1 | Mage, Support, Ranger |

## 5.2 กติกาเด็ค

- เด็คมีการ์ด 5 ใบ
- ห้ามใช้การ์ดเดียวกันซ้ำในเด็ค
- จำกัดธาตุเดิมไม่เกิน 3 ใบ
- แสดง Team Power รวม
- บางห้อง Arena อาจกำหนด Team Power Cap
- มีระบบบันทึกเด็คหลายชุด
- รองรับ Drag & Drop บน Desktop
- รองรับ Tap-to-Place บน Mobile

---

# 6. ระบบ Auto Battle 5v5

## 6.1 กติกาการต่อสู้

- ต่อสู้แบบ Turn-Based Auto Battle
- ทั้งสองฝ่ายมี 5 การ์ด
- สูงสุด 30 รอบ
- Mana เริ่มต้น 0 และสูงสุด 100
- ทุกเทิร์นได้รับ Mana 20
- หาก Mana ถึงค่า Skill Cost จะใช้ Skill
- หาก Mana ไม่ถึง จะใช้ Basic Attack
- ผู้ชนะคือฝ่ายที่กำจัดอีกฝ่ายได้หมด
- หากครบ 30 รอบ ให้เปรียบเทียบผลรวม HP% ของยูนิตที่รอด
- หากเท่ากัน ให้ถือว่า Draw

## 6.2 สูตรดาเมจ

```text
baseDamage = attacker.ATK × skillMultiplier
mitigation = 100 / (100 + defender.DEF)
elementMultiplier = 1.15 หากได้เปรียบธาตุ
elementMultiplier = 0.90 หากเสียเปรียบธาตุ
elementMultiplier = 1.00 หากเป็นกลาง
variance = 0.95 ถึง 1.05 จาก Deterministic RNG

finalDamage = max(1, floor(baseDamage × mitigation × elementMultiplier × variance))
```

## 6.3 Status Effects

| สถานะ | ผล |
|---|---|
| BURN | เสีย 5% Max HP เป็นเวลา 2 รอบ |
| SHIELD | ลดความเสียหาย 25% เป็นเวลา 2 รอบ |
| HASTE | เพิ่ม Speed 20% เป็นเวลา 2 รอบ |
| WEAKEN | ลด Attack 20% เป็นเวลา 2 รอบ |
| HEAL | ฟื้น HP แต่ไม่เกิน Max HP |

## 6.4 Deterministic Battle

```text
battleSeed = SHA-256(
  battleId +
  teamA +
  teamB +
  combatVersion +
  serverSideSecret
)
```

- Combat Engine ต้องเป็น Pure Function
- ห้ามเรียก Database ใน Combat Engine
- ห้ามใช้ `Math.random()`
- ต้องบันทึก Battle Log เพื่อ Replay ได้
- Server เป็นผู้คำนวณผลเสมอ

---

# 7. Arena Room 24 ชั่วโมง

## 7.1 รูปแบบ

- ผู้เล่นเปิดห้อง Arena ได้ด้วย Coin ในเกม
- ห้องมีอายุ 24 ชั่วโมง
- เจ้าของห้องตั้งทีมป้องกัน 5 ใบ
- ผู้เล่นอื่นจ่าย Coin ค่าเข้าร่วมเพื่อท้าทาย
- ผู้ชนะการต่อสู้ขึ้นเป็น Champion ปัจจุบัน
- เมื่อครบ 24 ชั่วโมง Champion อันดับ 1 รับรางวัล
- รางวัลเป็น Coin, XP, Material และ Cosmetic ภายในเกม

## 7.2 ค่าเริ่มต้น MVP

| รายการ | ค่า |
|---|---:|
| ค่าเปิดห้อง | 30 Coin |
| ค่าเข้าห้อง | 10 Coin |
| รางวัลฐาน | 100 Coin |
| โบนัสต่อผู้เข้าร่วม | 5 Coin |
| รางวัลสูงสุด | 500 Coin |
| ผู้เข้าร่วมสูงสุด | 100 คน |
| ระยะเวลาห้อง | 24 ชั่วโมง |
| Cooldown เปิดห้อง | 5 นาที |
| Daily Entry Cap | 20 ครั้ง |

## 7.3 สูตรรางวัล

```text
reward = min(100 + (uniqueParticipants × 5), 500)
```

## 7.4 ข้อกำหนดความปลอดภัย

- ทุกการใช้ Coin ต้องผ่าน Transaction
- มี Idempotency Key สำหรับการจ่าย Coin
- เก็บ `balanceBefore` และ `balanceAfter`
- ป้องกัน Double Click และ Network Retry
- Settlement Job ต้อง Idempotent
- จำกัด Rate Limit, Daily Cap และตรวจ Alt-account Farming

---

# 8. Technology Stack

| ส่วน | เทคโนโลยี |
|---|---|
| Frontend | Next.js App Router, TypeScript |
| Styling | Tailwind CSS |
| UI Components | shadcn/ui หรือระบบ Component ที่แก้ไขได้ |
| State | Zustand + TanStack Query |
| Form | React Hook Form + Zod |
| Backend | Next.js Route Handlers หรือ NestJS |
| Database | PostgreSQL |
| ORM | Prisma |
| Authentication | Auth.js / Supabase Auth |
| Cache / Queue | Redis + BullMQ |
| Image Storage | S3-Compatible Storage |
| Test | Vitest/Jest + Playwright |
| Deployment | Docker Compose |

---

# 9. หน้าจอ UI/UX หลัก

## 9.1 Routes

```text
/
 /discover
 /cards
 /cards/[id]
 /decks
 /battle/[id]
 /arena
 /arena/create
 /arena/[id]
 /wallet
 /quests
 /profile
 /events/[eventId]
```

## 9.2 หน้าจอหลัก

| หน้าจอ | เป้าหมาย |
|---|---|
| Home Dashboard | แสดง Core Loop และ CTA หลัก |
| Rune Discovery | เลือก Rune 8–16 จุด |
| Card Reveal | เปิดเผยการ์ดใหม่หรือการ์ดเดิม |
| Card Collection | ค้นหา กรอง และดูการ์ด |
| Card Detail | ดูสเตตัส สกิล Lore และภาพ |
| Deck Builder | จัดทีม 5 ใบ |
| Battle | ดู Auto Battle และ Replay |
| Arena List | ค้นหาห้องแข่งขัน |
| Arena Room Detail | ดู Champion, Leaderboard, รางวัล |
| Wallet | ดู Coin และประวัติธุรกรรม |
| Quests | ภารกิจรายวันและรายสัปดาห์ |
| Profile | สถิติและการตั้งค่า |

## 9.3 Design Tokens

```css
:root {
  --bg-primary: #0B1020;
  --bg-surface: #141B2D;
  --gold: #D7A84B;
  --cyan: #53D5E5;
  --text-primary: #F4F1E8;
  --text-secondary: #A9B1C6;

  --fire: #E85D32;
  --water: #3BA7E8;
  --wind: #55D6B1;
  --earth: #9B7A3E;
  --light: #F2CD68;
  --shadow: #8061D8;

  --rarity-common: #77808D;
  --rarity-uncommon: #57A66A;
  --rarity-rare: #4B91E8;
  --rarity-epic: #9B67D4;
  --rarity-legendary: #E49B3D;
  --rarity-mythic: #E35B5B;
}
```

## 9.4 UX Rules สำคัญ

- Mobile-first
- Tap Target ขั้นต่ำ 44×44px
- รองรับ `prefers-reduced-motion`
- ไม่ใช้สีเพียงอย่างเดียวเพื่อสื่อสถานะ
- Coin และ Discovery Energy ต้องมี Confirmation Modal ก่อนใช้
- แสดงยอดก่อนและหลังใช้ Currency ทุกครั้ง
- Rune Grid ใช้ Canvas/Virtualization
- การ์ดใหม่ต้องแสดง “ผู้ค้นพบคนแรก”
- การ์ดเดิมต้องแสดง “การ์ดที่ถูกค้นพบแล้ว”
- AI Image Pending ต้องไม่ขัดขวางการเล่น

---

# 10. AI Image Generation Pipeline

## 10.1 หลักการ

- สร้างภาพเฉพาะ CardDefinition ใหม่
- การ์ดเดิมใช้ภาพเดิมจาก Database/Storage
- สร้างภาพแบบ Async Queue
- ห้าม Block API Discovery ระหว่างสร้างภาพ
- มี Placeholder ตามธาตุและระดับการ์ด
- Retry แบบ Exponential Backoff สูงสุด 3 ครั้ง
- หากล้มเหลว ให้สถานะ `FAILED`
- Admin สามารถ Requeue ได้
- ภาพต้องผ่าน Content Moderation ก่อนเผยแพร่

## 10.2 Prompt Template

```text
Original high-fantasy trading card illustration of "{cardName}", a {rarity} {element}-aligned {role}.

Visual concept: {visualTraits}.
Scene: {environment}.
Mood: {mood}.

Dynamic cinematic composition, clear character silhouette, detailed character design, dramatic elemental magic, rich material textures, painterly digital illustration, original fantasy world, collectible card art, vertical 2:3 composition.

No text, no letters, no logo, no watermark, no signature, no card border, no UI, no copyrighted character, no recognizable franchise, no gore, no explicit content.
```

## 10.3 Negative Prompt

```text
text, letters, numbers, typography, logo, watermark, signature, card frame, card border, user interface, low resolution, blurry, pixelated, duplicate character, extra arms, extra legs, malformed hands, distorted anatomy, distorted face, cropped head, celebrity likeness, copyrighted character, recognizable franchise character, explicit content, nudity, gore, dismemberment
```

## 10.4 Element Art Direction

| ธาตุ | สี | Motif |
|---|---|---|
| Fire | Crimson, Orange, Gold | เปลวไฟ เถ้า หินภูเขาไฟ |
| Water | Blue, Cyan, Silver | คลื่น หมอก คริสตัลน้ำ |
| Wind | Teal, Mint, White | เมฆ ขนนก ลมหมุน |
| Earth | Moss, Ochre, Bronze | หิน รากไม้ ผลึก |
| Light | Ivory, Gold, Amber | รัศมี แสงอรุณ รูนเรขาคณิต |
| Shadow | Indigo, Violet, Silver | หมอกเงา รอยแยก ดวงจันทร์ |

---

# 11. Brand Identity

## 11.1 ชื่อเกม

```text
Rune Dominion Arena
```

## 11.2 บุคลิกแบรนด์

- Ancient Mystery
- Premium Fantasy
- Strategic Competition
- Elemental Magic
- Discovery
- Collectible
- Tactical
- Hopeful Darkness

## 11.3 Concept โลโก้

### Concept A: Rune Gate

สัญลักษณ์เป็นรูนเรขาคณิต 3 ชิ้น ประกอบเป็นประตูพลังงาน มีแกนแสงตรงกลาง สื่อถึงการค้นพบและการเปิดทางสู่ Echo

### Concept B: Arena Sigil

ตราวงกลมที่แบ่งเป็น 6 ส่วนแทนธาตุ ล้อมแกนรูนกลาง สื่อถึงสนามแข่งขันและกลยุทธ์

### Concept C: Discovery Prism

แท่งปริซึมรูนเปิดออก ปล่อยลำแสง สื่อถึงการถอดรหัสและค้นพบการ์ด

## 11.4 Prompt Logo Exploration

```text
Minimal original fantasy game emblem, abstract rune gate formed from three interlocking geometric rune shards, a small luminous core at the center, balanced vertical symmetry, premium game identity, midnight navy background, muted antique gold and arcane cyan glow, clean vector-like shapes, high readability at small app icon size.

No text, no letters, no numbers, no watermark, no existing franchise symbols.
```

---

# 12. ตัวอย่างการ์ดหลัก

## 12.1 เคล ผู้พิทักษ์เถ้าถ่าน

| Field | Value |
|---|---|
| English Name | Kael, Ashen Vanguard |
| Element | Fire |
| Rarity | Rare |
| Role | Fighter |
| Skill | คมดาบเถ้าร้อน |
| Effect | โจมตีแนวหน้าและติด Burn |

**Lore:** เคลไม่เคยสาบานว่าจะชนะ เขาสาบานเพียงว่าจะยืนอยู่ตรงเดิมจนกว่าเปลวไฟสุดท้ายจะดับลง

## 12.2 เนริสซา ผู้เก็บรักษากระแสน้ำ

| Field | Value |
|---|---|
| English Name | Nerissa, Keeper of Tides |
| Element | Water |
| Rarity | Epic |
| Role | Support |
| Skill | วังวนแห่งความทรงจำ |
| Effect | Heal หลายเป้าหมายและล้าง Weaken |

**Lore:** เนริสซาเชื่อว่าน้ำจำได้ทุกสิ่ง แม้แต่คำสัญญาที่เจ้าของพยายามลืม

## 12.3 บรอมม์ มหึมาผูกพันราก

| Field | Value |
|---|---|
| English Name | Bromm, Rootbound Colossus |
| Element | Earth |
| Rarity | Legendary |
| Role | Tank |
| Skill | ป้อมปราการรากลึก |
| Effect | สร้าง Shield ทั้งทีมและดึงการโจมตี |

## 12.4 ออเรเลีย คำสาบานแห่งอรุณ

| Field | Value |
|---|---|
| English Name | Aurelia, Dawn's Oath |
| Element | Light |
| Rarity | Epic |
| Role | Support |
| Skill | โคมอรุณนิรันดร์ |
| Effect | Heal เป้าหมายที่ HP ต่ำสุดและให้ Shield |

## 12.5 มอร์โรว์ ประตูไร้จันทร์

| Field | Value |
|---|---|
| English Name | Morrow, Veil of the Moonless Gate |
| Element | Shadow |
| Rarity | Legendary |
| Role | Mage |
| Skill | รอยแยกไร้จันทร์ |
| Effect | AoE Damage และติด Weaken |

---

# 13. Seasonal Event 01: Call of the Moonless Gate

## 13.1 ภาพรวม

**ชื่อไทย:** เสียงเรียกจากประตูไร้จันทร์  
**ระยะเวลา:** 14 วัน  
**Grace Period:** 24 ชั่วโมง  
**ธีม:** รอยแยกธาตุ Shadow ปรากฏขึ้นทั่ว Aetherra  
**Event Currency:** Veil Shards  

## 13.2 Event Loop

```text
ทำ Quest
  → รับ Veil Shards
  → เข้า Boss Raid
  → สร้าง Damage / Event Points
  → รับ Milestone Rewards
  → ปลดล็อก Story Chapters
  → ซื้อ Cosmetic / วัตถุดิบจาก Event Shop
```

## 13.3 Boss: Veil of the Moonless Gate

| Phase | HP | ชื่อ |
|---|---:|---|
| 1 | 100–76% | The Rift Warden |
| 2 | 75–51% | Reflected Knight |
| 3 | 50–26% | Moonless Veil |
| 4 | 25–0% | Echo of Morrow |

## 13.4 Boss Mechanics

| กลไก | ผล |
|---|---|
| Veil Shield | ลด Damage ที่ได้รับ 40% |
| Rune Fracture | ทุก 3 Turn ทำให้ธาตุหนึ่งรับ Damage เพิ่ม |
| Moonless Mark | เป้าหมายรับ Damage เพิ่ม 20% |
| Eclipse Pulse | AoE Shadow Damage ทุก 4 Turn |
| Rift Hunger | HP ต่ำกว่า 30% เพิ่มความเร็วบอส |
| Ember Break | Fire ทำ Damage ต่อ Shield เพิ่ม |
| Gale Shift | Wind หลบการโจมตีเป้าหมายเดี่ยวได้เพิ่ม |
| Rooted Guard | Earth ลด AoE Damage ให้ทีม |
| Tide Cleanse | Water ล้าง Moonless Mark |
| Dawn Resonance | Light ลด Veil Shield ได้ดีขึ้น |

## 13.5 Raid Entry

| รายการ | ค่า |
|---|---:|
| ค่าเข้า Raid | 10 Veil Shards |
| Daily Raid Cap | 10 ครั้ง |
| แพ้ | ได้ Participation Reward |
| ชนะ | ได้ Clear Bonus |
| โบนัสทีม 4 ธาตุขึ้นไป | +10% Event Points |

## 13.6 Personal Milestones

| คะแนน | รางวัล |
|---:|---|
| 2,000 | 100 Coin |
| 5,000 | Crafting Dust 50 |
| 10,000 | Event Card: ผู้สังเกตการณ์รอยแยก |
| 20,000 | Avatar Frame: ม่านไร้จันทร์ |
| 35,000 | Shadow Card Art Variant |
| 50,000 | Event Card: เซลิน ผู้ผนึกอรุณ |
| 75,000 | Title: ผู้พิชิตประตูไร้จันทร์ |

## 13.7 Community Milestones

| Community Damage | รางวัล/เนื้อหา |
|---:|---|
| 1,000,000 | Story Chapter 2 |
| 5,000,000 | Veil Shards 30 |
| 10,000,000 | Story Chapter 3 + Shop Item |
| 25,000,000 | Cosmetic Banner |
| 50,000,000 | Chapter Finale + Crafting Dust |

---

# 14. Prompt หลักสำหรับ Development Agent

```text
คุณคือ Senior Full-Stack Game Developer และ Game Systems Designer

ช่วยสร้าง MVP สำหรับเกมชื่อ “Rune Dominion Arena” เป็นเกมการ์ดแฟนตาซีบนเว็บ รองรับมือถือเป็นหลัก โดยใช้ TypeScript ทั้ง Frontend และ Backend

เป้าหมาย:
1) ผู้เล่นกดรูนจากตาราง 100x100 เพื่อค้นพบการ์ด
2) ลำดับรูนถูกแปลงเป็น deterministic seed
3) seed เดียวกันต้องได้การ์ดใบเดิมเสมอ
4) หากการ์ดเคยถูกค้นพบแล้ว ให้ดึงจากฐานข้อมูล ห้ามสร้างซ้ำ
5) ผู้เล่นสะสมการ์ด จัดทีม 5 ใบ และแข่ง Auto Battle
6) มีห้องแข่งขัน 24 ชั่วโมง และระบบอันดับ
7) รางวัลเป็นเหรียญในเกมเท่านั้น ไม่มีเงินจริง ไม่มีถอนเงิน ไม่มีการแลกเปลี่ยนเหรียญระหว่างผู้เล่น

Tech Stack:
- Next.js App Router
- TypeScript
- Tailwind CSS
- PostgreSQL
- Prisma
- Auth.js หรือ Supabase Auth
- Redis + BullMQ
- S3-compatible Storage
- Zod
- Vitest/Jest
- Playwright
- Docker Compose

Business Rules:
- ห้ามใช้ random ฝั่ง client สำหรับข้อมูลการ์ด
- ทุกการคำนวณสำคัญทำฝั่ง server
- SHA-256 จาก canonical rune sequence + GAME_RULE_VERSION + server-side pepper
- canonical_seed_hash ต้อง unique
- Server Pepper ห้ามส่งให้ client
- Discovery ต้อง idempotent
- ใช้ transaction และ unique constraint ป้องกัน Race Condition
- AI image เป็น async job และสร้างเฉพาะ CardDefinition ใหม่
- ใช้ original fantasy world เท่านั้น ห้ามใช้ทรัพย์สินจากแฟรนไชส์อื่น

Deliverables:
A. Architecture document
B. Prisma schema
C. Database migrations และ seed script
D. API endpoints พร้อม examples
E. Pure combat engine พร้อม tests
F. Frontend screens
G. Docker Compose และ .env.example
H. README วิธีรัน
I. Sample data
J. Security และ Anti-cheat checklist

ทำงานตาม Phase:
Phase 1: Architecture, Database Schema, API Contract
Phase 2: Card Discovery และ Collection
Phase 3: Deck Builder และ Battle Simulator
Phase 4: Arena Room และ Coin Ledger
Phase 5: AI Image Queue, Tests, Logging, Deployment

ก่อนเขียนโค้ด ให้สรุป assumptions และคำถามที่จำเป็นเท่านั้น
จากนั้นเริ่มทำทีละ Phase
```

---

# 15. Prompt สำหรับ UI/UX Agent

```text
คุณคือ Senior Product Designer และ UI/UX Designer เชี่ยวชาญเกมมือถือและเว็บแอป

ออกแบบ UI/UX สำหรับเกมการ์ดแฟนตาซีชื่อ Rune Dominion Arena เป็น Mobile-first Responsive Web App รองรับ 360px, 390px, 768px และ Desktop 1440px

Core Loop:
Discover Rune > Receive Card > Build Team > Battle > Earn Coin > Discover Again

Requirements:
- Dark fantasy premium แต่ต้องอ่านง่าย
- UI ภาษาไทย
- Dark Mode เป็นธีมหลัก
- ใช้สีธาตุ Fire, Water, Wind, Earth, Light, Shadow
- มี Home, Rune Discovery, Card Reveal, Collection, Card Detail, Deck Builder, Battle, Arena, Wallet, Quests, Profile
- Rune Grid 100x100 ต้องใช้ Canvas หรือ Virtualized Grid
- ผู้เล่นต้องเห็น Currency ก่อนใช้และหลังใช้ทุกครั้ง
- ทุกการใช้ Coin หรือ Discovery Energy ต้องมี Confirmation Modal
- รองรับ Accessibility, Keyboard Navigation, Touch Interaction และ Reduced Motion
- ห้ามใช้ทรัพย์สินหรือภาพลักษณ์จากแฟรนไชส์อื่น

สำหรับทุกหน้าจอ ให้ส่ง:
1. เป้าหมายผู้ใช้
2. Information Architecture
3. Wireframe Description
4. UI Components
5. CTA หลักและรอง
6. Loading/Empty/Error/Success State
7. Microcopy ภาษาไทย
8. Responsive Rules
9. Accessibility Notes
10. Edge Cases
```

---

# 16. Prompt สำหรับ Frontend Agent

```text
คุณคือ Senior Frontend Engineer

สร้าง Frontend UI สำหรับ Rune Dominion Arena โดยใช้:
- Next.js App Router
- TypeScript strict mode
- Tailwind CSS
- shadcn/ui
- Lucide Icons
- Zustand
- TanStack Query
- React Hook Form + Zod
- Framer Motion โดยรองรับ prefers-reduced-motion

Requirements:
- Mobile-first
- UI ภาษาไทย
- แยกข้อความไว้ใน i18n-ready dictionary
- มี Mock API Layer และ Mock Data ก่อนต่อ Backend
- ห้ามใช้ภาพจากแฟรนไชส์ที่มีลิขสิทธิ์
- Placeholder Art ใช้ Abstract Elemental Gradient
- Rune Grid ใช้ Canvas หรือ Virtualization
- Confirmation Modal สำหรับ Coin / Discovery Energy
- ป้องกัน Double Submit
- Semantic HTML, ARIA, Focus State, Contrast และ Tap Target 44px

Routes:
- /
- /discover
- /cards
- /cards/[id]
- /decks
- /battle/[id]
- /arena
- /arena/create
- /arena/[id]
- /wallet
- /quests
- /profile

Components:
- AppShell
- TopHeader
- BottomNavigation
- CoinBalance
- DiscoveryEnergy
- ElementBadge
- RarityBadge
- CardThumbnail
- RuneCanvas
- RuneSequenceTray
- DiscoverConfirmModal
- CardRevealModal
- DeckFormationBoard
- TeamPowerMeter
- BattleField
- BattleLogPanel
- RoomCard
- ArenaLeaderboard
- CoinTransactionList
- QuestCard
- ConfirmCurrencyModal
- Toast
- EmptyState
- ErrorState
- LoadingSkeleton

ส่งมอบ:
1. Folder Structure
2. Design Tokens
3. TypeScript Interfaces
4. หน้า Home, Discover, Collection, Deck Builder, Arena, Wallet
5. Mock Data: 30 Cards, 3 Decks, 10 Arena Rooms, 30 Coin Transactions
6. README
7. จุดเชื่อมต่อ Backend API
```

---

# 17. Prompt สำหรับ Audio Agent

```text
คุณคือ Senior Game Audio Designer และ Sound Director

ออกแบบ Audio Direction สำหรับเกม Rune Dominion Arena

เป้าหมาย:
- สื่อความลึกลับของ Rune โบราณ
- ให้การค้นพบการ์ดรู้สึกพิเศษ
- การต่อสู้มีพลังแต่ไม่รบกวน
- รองรับ Mobile Web
- มี Toggle สำหรับ Music, SFX, Ambience และ Reduce Intense Effects

ออกแบบเสียง:
1. Global UI
2. Rune Discovery
3. Card Reveal แยกตาม Rarity
4. Deck Builder
5. Battle
6. Arena
7. Event Raid

ให้ส่ง:
- Audio Pillars
- ตารางรายการเสียง
- File Naming
- Duration
- Loudness Target
- Variations
- Priority Rules
- Export Format: OGG + MP3 fallback
- Accessibility Audio Guidelines
```

---

# 18. Prompt Seasonal Event Agent

```text
คุณคือ Senior LiveOps Game Designer, Backend Engineer และ Frontend Game Developer

ออกแบบ Seasonal Event “Call of the Moonless Gate” ระยะเวลา 14 วัน สำหรับ Rune Dominion Arena

Event Features:
- Veil Shards เป็น Event Currency
- Daily / Weekly Event Quests
- Boss Raid 5v1 หรือ 5v5
- Personal Milestones
- Community Milestones
- Story Chapters
- Event Shop
- Reward Claim
- Grace Period 24 ชั่วโมง

Rules:
- Event Status: UPCOMING, ACTIVE, GRACE_PERIOD, ENDED
- ใช้เวลา Server-side เท่านั้น
- Currency และ Reward ต้องผ่าน Ledger
- ทุก Claim/Purchase/Raid ต้อง Idempotent
- ใช้ Integer Currency เท่านั้น
- มี Daily Cap, Rate Limit และ Anti-bot Signals
- Raid ใช้ Deterministic Combat Engine
- เก็บ Battle Log และ Replay ได้

Deliverables:
1. Event GDD
2. Prisma Schema
3. API Endpoints
4. Backend Transactions
5. Raid Combat Extension
6. Scheduler Jobs
7. UI Specification
8. Quest/Reward System
9. Test Plan
10. Admin Tools
```

---

# 19. MVP Roadmap

| Phase | งานหลัก |
|---|---|
| MVP 1 | Auth, Rune Discovery, Seed, Card Database, Placeholder Art |
| MVP 2 | Collection, Card Detail, Deck Builder |
| MVP 3 | Auto Battle, Battle Log, Replay |
| MVP 4 | Coin Ledger, Quests, Arena Rooms |
| MVP 5 | AI Image Queue, Moderation, Admin Tools |
| Beta | Seasonal Event, Balancing, Anti-cheat, Monitoring |
| Launch | Seasons, Cosmetics, Story Content, LiveOps |

---

# 20. Security และ Anti-Cheat Checklist

- [x] Server-side validation ทุก Currency Action (✅ Phase 10 — Zod schema + ตรวจสิทธิ์/เงื่อนไขฝั่ง server ทุก endpoint)
- [x] Server-side Battle Calculation (✅ Phase 4 — deterministic simulateBattle + seed จาก SERVER_PEPPER)
- [x] Idempotency Key สำหรับ Discovery, Payment, Claim, Purchase และ Challenge (✅ discovery/wallet/arenaChallenge มี unique idempotencyKey, quest claim ตรวจ claimed ซ้ำ — Purchase ยังไม่มีระบบร้านค้า)
- [x] Database Transaction สำหรับ Currency (✅ wallet debit/credit ใช้ prisma.$transaction)
- [x] Unique Constraint สำหรับ Seed และ Ownership (✅ canonicalSeedHash @unique + @@unique([userId, cardId]))
- [x] Rate Limit ต่อ User/IP/Device (✅ Phase 10 — sliding-window + middleware burst; ปรับผ่าน RATE_LIMIT_<SCOPE>_LIMIT)
- [x] ตรวจ Bot Pattern เช่น Action เร็วผิดปกติ (✅ Phase 10 — FAST_ACTIONS + UNIFORM_CADENCE → 429 + SecurityEvent)
- [x] Daily Cap สำหรับ Discovery, Arena Entry และ Raid (✅ Discovery = energy 5/วัน (lazy refill), Arena Entry = 20 ครั้ง/วัน — Raid ยังไม่มีระบบ)
- [x] Audit Log สำหรับ Admin Action (✅ Phase 9 admin_action_logs + Phase 10 security_events)
- [x] Battle Replay Verification (✅ Phase 10 — เก็บ teams snapshot + seed → re-simulate เทียบผล; TAMPERED → SecurityEvent)
- [x] ห้ามเชื่อข้อมูล Stat, Damage หรือ Reward จาก Client (✅ server คำนวณทั้งหมด — client ส่งได้แค่ตัวตน/ID)
- [ ] Image Moderation และ File Validation (⏳ N/A ตอนนี้ — ไม่มี file upload จากผู้เล่น ภาพมาจาก AI API ฝั่ง server; ทำตอนเปิดให้อัปโหลด)
- [x] จำกัดความยาวชื่อห้องและกรองคำไม่เหมาะสม (✅ Phase 10 — Zod max 60 ตัวอักษร + profanity filter ไทย/อังกฤษ)
- [x] รองรับ Retry โดยไม่หัก Currency ซ้ำ (✅ idempotencyKey ใน WalletService.debit/credit + idempotent claim/join/challenge)

---

# 21. Microcopy ภาษาไทย

## Rune Discovery

- “เลือกรูน 8–16 ตำแหน่งเพื่อเริ่มถอดรหัส”
- “ใช้พลังค้นหา 1 หน่วยเพื่อถอดรหัสชุดรูนนี้?”
- “หากมีผู้ค้นพบชุดนี้แล้ว คุณจะได้รับการ์ดใบเดิม”
- “ถอดรหัสรูน”
- “กลับไปแก้ไข”
- “พลังค้นหาไม่เพียงพอ”
- “กำลังอ่านบันทึกแห่งรูน...”

## Card Reveal

- “ผู้ค้นพบคนแรก”
- “การ์ดที่ถูกค้นพบแล้ว”
- “ภาพอัญเชิญกำลังถูกวาดลงในบันทึกแห่งรูน”
- “การ์ดใบนี้ใช้งานได้ทันที”
- “เพิ่มลงทีม”
- “ค้นหารูนต่อ”

## Arena

- “ใช้ Coin 10 เหรียญเพื่อเข้าท้าทายห้องนี้?”
- “ยอดคงเหลือหลังทำรายการ”
- “ค่าเข้าร่วมใช้ภายในเกมเท่านั้น”
- “ห้องนี้จะสิ้นสุดใน”
- “ผู้ครองบัลลังก์ปัจจุบัน”
- “รางวัลโดยประมาณ”
- “รางวัลสุดท้ายขึ้นกับจำนวนผู้เข้าร่วมภายใต้เพดานที่กำหนด”

## Wallet

- “Coin เป็นสกุลเงินภายในเกม ไม่สามารถโอน แลก หรือถอนเป็นเงินจริงได้”
- “ได้รับ Coin”
- “ใช้ Coin”
- “ประวัติธุรกรรม”

---

# 22. สถานะเอกสาร

เอกสารนี้เป็นฐานสำหรับเริ่มพัฒนา Rune Dominion Arena โดยแนะนำให้เริ่มจาก:

1. ระบบ Seed และ Card Discovery
2. Collection และ Deck Builder
3. Combat Engine แบบ Deterministic
4. Coin Ledger และ Arena
5. AI Image Generation
6. Seasonal Event และ LiveOps