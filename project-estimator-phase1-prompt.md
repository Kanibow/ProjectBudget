# Project Estimator (Phase 1) — Build Instructions Prompt (Markdown)

> ใช้ไฟล์นี้เป็น “prompt instructions” เพื่อสั่ง AI Agent ให้สร้างโปรเจกต์เว็บแอป Phase 1 (Comparable-based Estimator) ตามสเปกที่กำหนด

## 0) Role / Goal

คุณคือ Senior Full-stack Engineer สร้างเว็บแอป “Project Estimator (Phase 1)”  
**Stack:** React (Vite + TypeScript) + Node.js (Express + TypeScript) + PostgreSQL + Google Sheets API  
**Focus:** Sync ข้อมูลโครงการจาก Google Sheets เข้า PostgreSQL และทำหน้า Estimate โดยเทียบโครงการที่ “คล้ายกัน” (comparable projects) + สถิติเบื้องต้น (p50/p80)

**Constraints**
- รัน local ได้ด้วย `docker compose`
- ถ้าไม่มี credential ของ Google ให้ใช้ mock data เพื่อ demo ได้
- TypeScript ทั้งฝั่ง client/server
- มี README ที่รันได้จริง
- โค้ดอ่านง่าย มี validation, error handling, และ logging พื้นฐาน

---

## 1) Monorepo Structure

สร้าง repo โครงสร้างนี้:

```
project-estimator/
  apps/
    web/         # React Vite TS
    api/         # Node Express TS
  packages/
    shared/      # shared types (zod schemas, DTO)
  infra/
    docker-compose.yml
  README.md
```

**Workspace**
- ใช้ `pnpm` (แนะนำ) และตั้ง workspace scripts ให้รันพร้อมกันได้
- ต้องมีคำสั่ง:
  - `pnpm dev` รัน web+api
  - `pnpm db:up` เปิด Postgres
  - `pnpm migrate` รัน migration
  - `pnpm lint` / `pnpm test` (ถ้าทำได้)

---

## 2) Database (Postgres) + Migration

เลือก ORM/Query tool อย่างใดอย่างหนึ่ง (**Prisma** หรือ **Drizzle**) แล้วทำให้ครบ พร้อม migration

### Tables (Minimum)

#### 1) `projects`
- `id` uuid pk
- `project_code` text unique
- `project_name` text
- `project_type` text (e.g. `WIND`, `SOLAR`, ...)
- `budget_million_thb` numeric (เก็บหน่วย “ล้านบาท” เพื่อเทียบง่าย)
- `duration_days` int
- `start_date` date nullable
- `end_date` date nullable
- `created_at`, `updated_at`

#### 2) `engineering_costs`
- `id` uuid pk
- `project_id` fk -> projects
- `level` text (e.g. `JUNIOR`, `MID`, `SENIOR`, `LEAD`, `PM`)
- `headcount` int
- `man_days` numeric
- `rate_per_day_thb` numeric
- `amount_thb` numeric

**Rule**
- คิดค่า `amount_thb = man_days * rate_per_day_thb` ถ้าไม่ได้ส่ง amount มา
- ต้องรองรับ upsert ด้วย `project_code`

#### 3) `sync_runs`
- `id` uuid pk
- `status` text (`SUCCESS`/`FAIL`)
- `started_at`, `finished_at`
- `message` text nullable
- `row_count` int

---

## 3) Google Sheets Ingestion (Phase 1)

สร้าง service ใน API:
- อ่านข้อมูลจาก Google Sheets (service account)
- รองรับ column mapping (ชื่อไทย/อังกฤษ → canonical fields)
- ทำ endpoint `POST /api/sync/google-sheets` เพื่อ sync
- เก็บผล run ลงตาราง `sync_runs`

**Offline / Mock mode**
- ถ้าไม่มี credential ให้ใช้ไฟล์ mock `apps/api/mock/sheets.json` แล้ว sync จาก mock แทน เพื่อให้ demo ได้

### Sheet Input Format (MVP)

สมมติใน Google Sheets มี 2 tabs:

**Tab: `Projects`**
- `project_code`
- `project_name`
- `project_type`
- `budget_million_thb`
- `duration_days`

**Tab: `EngineeringCosts`**
- `project_code`
- `level`
- `headcount`
- `man_days`
- `rate_per_day_thb`

> ถ้าข้อมูลจริงต่าง ให้ทำ mapping layer ในไฟล์ config เช่น `apps/api/src/config/sheets.mapping.ts`

---

## 4) API Requirements

ทำ Express API พร้อม validation (Zod) และ error handling + logging

### Endpoints

#### 1) `GET /api/health`
- คืนสถานะ ok

#### 2) `POST /api/sync/google-sheets`
- body:
```json
{ "source": "google" | "mock" }
```
- returns: sync run result (status, rowCount, message)

#### 3) `POST /api/estimate`
รับ input:
```json
{
  "projectType": "WIND",
  "budgetMillionTHB": 10,
  "targetDurationDays": 180,
  "topK": 8
}
```

คืนผล:
```json
{
  "input": { "...": "..." },
  "comparableProjects": [
    {
      "projectCode": "WIND-2024-01",
      "projectName": "...",
      "budgetMillionTHB": 9.5,
      "durationDays": 200,
      "engineeringAmountTHB": 1200000,
      "engineeringManDays": 240,
      "levels": [
        { "level": "SENIOR", "headcount": 1, "manDays": 40, "amountTHB": 400000 }
      ],
      "similarity": 0.82
    }
  ],
  "stats": {
    "engineeringCostPerMillionTHB": { "p50": 120000, "p80": 160000 },
    "manDaysPerMillionTHB": { "p50": 22, "p80": 30 }
  },
  "estimate": {
    "engineeringAmountTHB": { "p50": 1200000, "p80": 1600000 },
    "engineeringManDays": { "p50": 220, "p80": 300 },
    "suggestedStaffing": [
      { "level": "JUNIOR", "headcount": 2 },
      { "level": "MID", "headcount": 1 },
      { "level": "SENIOR", "headcount": 1 }
    ],
    "notes": [
      "Based on 6 comparable WIND projects within ±30% budget"
    ]
  }
}
```

### Comparable Logic (Phase 1)

Filter โครงการคล้ายกันจากฐานข้อมูล:
- `project_type` เท่ากัน
- งบใกล้เคียง: อยู่ในช่วง ±30% ของ budget input
- ระยะเวลาใกล้เคียง: อยู่ในช่วง ±40% ของ `targetDurationDays` (ถ้า input ใส่มา)

Similarity score แบบง่าย:
- `budgetDiff = abs(budget - inputBudget) / inputBudget`
- `durationDiff = abs(duration - inputDuration) / inputDuration`
- `score = 1 - clamp(0..1, 0.6*budgetDiff + 0.4*durationDiff)`

เลือก topK ตาม score สูงสุด

### Stats (Phase 1)

คำนวณ p50/p80 ของ:
- `engineeringAmountTHB / budgetMillionTHB`
- `engineeringManDays / budgetMillionTHB`

> ใช้ percentile แบบง่าย (sort แล้ว pick index) ก็พอใน Phase 1

### Suggested Staffing (Heuristic)

- หา “สัดส่วนระดับวิศวกร” เฉลี่ยจาก comparable set (เช่น JUNIOR 30%, MID 40%, SENIOR 20%, LEAD/PM 10%)
- คูณกับ `engineeringManDays` ที่ estimate แล้วแปลงเป็น headcount:
  - utilization default = `0.75`
  - `availableDaysPerPerson = targetDurationDays * utilization`
  - `headcount(level) = ceil(manDays(level) / availableDaysPerPerson)`

---

## 5) Web App (React)

สร้าง UI 3 หน้า (React Router)

### 1) Dashboard
- ปุ่ม “Sync from Google” และ “Sync from Mock”
- แสดง sync run ล่าสุด (status, row_count, time)

### 2) Estimator
- ฟอร์ม:
  - `projectType` (dropdown)
  - `budget` (ล้านบาท)
  - `targetDurationDays`
  - `topK`
- ปุ่ม “Estimate”
- แสดงผล 3 ส่วน:
  1) **Comparable projects table** (budget, duration, eng cost, similarity)
  2) **Stats cards** (p50/p80 cost per million, man-days per million)
  3) **Suggested staffing** แยกตาม level + engineering cost estimate (p50/p80)

### 3) Projects
- ตารางโปรเจกต์ทั้งหมด + filter by type
- click row แล้วเห็น breakdown engineering costs

**UI Tech**
- ใช้ Tailwind + (optional) shadcn/ui
- โทนสว่าง อ่านง่าย
- จัด layout ให้ดูเป็น dashboard

---

## 6) DevOps / DX

- มี `infra/docker-compose.yml` สำหรับ Postgres
- มี `.env.example` ทั้ง web และ api
  - `DATABASE_URL`
  - `GOOGLE_SHEETS_ID`
  - `GOOGLE_SERVICE_ACCOUNT_JSON` (optional; ถ้าไม่ใส่ให้ใช้ mock)

**README ต้องมี**
- วิธี run local ทีละ step
- วิธีใช้ mock data
- วิธีใส่ Google credential (service account)
- ตัวอย่างรูปแบบ mapping column

---

## 7) Acceptance Criteria (ต้องผ่าน)

- `pnpm db:up` เปิด DB ได้
- `pnpm migrate` สร้างตารางได้
- `pnpm dev` เปิด web+api ได้
- กด Sync (mock) แล้วข้อมูลเข้า DB
- กรอก Estimator แล้วได้:
  - comparable projects
  - p50/p80 stats
  - suggested staffing
  - แสดงผลบน UI ถูกต้อง

---

## 8) Deliverables

- โค้ดทั้ง repo ตามโครงสร้าง
- README
- ไฟล์ mock data (`apps/api/mock/sheets.json`)
- Migration scripts
- Mapping config (`apps/api/src/config/sheets.mapping.ts`)

---

## 9) Optional (Nice-to-have)

- เพิ่ม endpoint `GET /api/projects` และ `GET /api/projects/:code`
- เพิ่ม basic tests สำหรับ estimate logic
- เพิ่ม OpenAPI/Swagger
- เพิ่ม logging ด้วย pino/winston

---

## 10) Implementation Order (ถ้าต้องทำทีละขั้น)

1) Setup monorepo + docker + migration  
2) Implement sync mock → DB  
3) Implement estimate endpoint + test  
4) Build web pages + integrate  
5) Add Google Sheets auth + mapping layer  

---

## 11) Notes / Guardrails

- Phase 1 ห้ามใช้ LLM มาเดาตัวเลขเอง: ตัวเลขต้องมาจาก SQL + สูตรที่ตรวจสอบได้
- เตรียมพื้นที่สำหรับ Phase 2: เก็บ `estimate_requests`/`estimate_results` ได้ในอนาคต
- ต้องระบุเหตุผลของ comparable set ใน response notes (เพื่อ audit)

---

**End of prompt**  
นำข้อความทั้งหมดนี้ไปวางให้ AI Agent แบบตรง ๆ ได้เลย
