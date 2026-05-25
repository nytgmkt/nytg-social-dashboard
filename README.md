# NYTG · Social Media Dashboard

แดชบอร์ดวิเคราะห์ Social Media สำหรับ Nan Yang Textile Group  
ไฟล์เดียว (`index.html`) ทำงานบน Browser ทั้งหมด — ไม่ต้องมี server, ไม่ต้องติดตั้งอะไร

---

## ฟีเจอร์

| หมวด | รายละเอียด |
|---|---|
| **Executive Summary** | ภาพรวม Category, Monthly Volume, KPIs |
| **Content Performance** | Trend Reach/Interactions, Best Category, Post Table |
| **Category Content** | YoY Category Mix, Avg %Reach, Total Reach/Interactions |
| **Brand Performance** | เปรียบเทียบ %Reach/%Engagement ระหว่าง Brand |
| **Leads & CRM** | Pipeline Funnel, BU Destination, Contact Person, Product Category |
| **Ad Spending** | Spend by Ad, Cost per Result, Reach, Messaging Contacts |

**Data Management**
- `🔄 Load New Data` — ล้างข้อมูลเดิมแล้วโหลด CSV ใหม่ทั้งหมด
- `➕ Add Month` — Merge เดือนใหม่เข้ากับเดิม (deduplicate ด้วย Permalink)
- ข้อมูลทุกอย่างบันทึกใน **localStorage** — refresh หน้าแล้วยังอยู่ครบ
- รองรับ **2 แบรนด์** แยกกัน: **NYTG** และ **NIC** (localStorage แยก key)

---

## วิธีใช้งานเบื้องต้น

### 1. ดู Sample Data
กด **▶ Sample Data** ในแถบ Navbar — ข้อมูลตัวอย่าง Jan 2025 – Apr 2026 จะโหลดทันที

### 2. Upload ข้อมูลจริง

#### Social Posts CSV
กด **🔄 Load New Data** แล้วเลือกไฟล์ CSV ที่ Export มาจาก Facebook Business Suite

ไฟล์ต้องมี column เหล่านี้:

| Column | ตัวอย่าง |
|---|---|
| `Permalink` | `https://www.facebook.com/...` |
| `Publish time` | `1/6/2026 10:30` (M/D/YYYY) |
| `Post type` | Videos, Reels, Single Photo, Photos, Photo Album |
| `Category 1` / `Category 2` / `Category 3` | Innovation, Sustainability, VI |
| `Brand` | Dry-Tech, Smoothness, Elitech360 |
| `%Reach` | 86 |
| `%Engagement Rate` | 5.3 |
| `Reach` | 67409 |
| `View` | 78558 |
| `Reactions, comments and shares` | 4185 |
| `Lead Count` | 3 |
| `Average Seconds viewed` | 1.5 |

> รองรับทั้ง CSV (`,`) และ TSV (Tab) รองรับ quoted fields และ multiline

#### Leads & CRM CSV
ไปที่ Tab **Leads & CRM** → กด **📋 Upload Leads CSV**

Column ที่ต้องการ: `Date`, `Customer / Account Name`, `Channel`, `Product Category`,  
`Stage`, `Contact Person`, `Est. value`, `Notes`, `Bounce`

ค่า `Stage` ที่รองรับ: `Qualify Lead` · `Land to Yuedpao` · `Land to Vel` · `Land to TC` · `Closed - Not Interested`

#### Ad Spending Excel (.xlsx)
ไปที่ Tab **Ad Spending** → กด **📊 Upload Ads Excel**

Export ตรงจาก Facebook Ads Manager — column ที่ใช้: `Ad name`, `Amount spent (THB)`,  
`Results`, `Result indicator`, `Cost per results`, `Reach`, `Impressions`,  
`Total messaging contacts`, `Views`, `Clicks (all)`, `CPM (...)`

---

## Filter การแสดงผล

```
[Brand: NYTG | NIC]  →  [Year: All | 2025 | 2026]  →  [Period: All | Jan 2025 | Feb 2025 | ...]
```

- **Brand** — สลับระหว่าง NYTG และ NIC (localStorage แยกกัน)
- **Year** — กรอง All Years / 2025 / 2026
- **Period** — กรองรายเดือน (จะแสดงเฉพาะเดือนที่มีข้อมูล)
- Tab **Leads & CRM** และ **Ad Spending** ไม่ขึ้นกับ Year/Period filter

---

## Tech Stack

```
HTML5 + CSS3 + Vanilla JS   — ไม่มี framework, ไม่ต้อง build
Chart.js 4.4.1              — กราฟทุกชนิด
SheetJS (xlsx) 0.18.5       — อ่านไฟล์ Excel (.xlsx)
localStorage                — เก็บข้อมูลฝั่ง Browser
Google Fonts                — Syne, DM Sans, DM Mono
```

---

## Deploy ขึ้น GitHub Pages

ทำตาม **6 ขั้น** นี้ตามลำดับ:

---

### ขั้นที่ 1 — Merge branch ลง `main`

branch ปัจจุบันคือ `claude/wizardly-hypatia-SpsnU`  
GitHub Pages จำเป็นต้องให้ไฟล์อยู่ใน branch `main` (หรือ `gh-pages`)

ไปที่ https://github.com/nytgmkt/nytg-social-dashboard แล้ว:

1. คลิก **Pull requests** → **New pull request**
2. Base: `main` ← Compare: `claude/wizardly-hypatia-SpsnU`
3. คลิก **Create pull request** → **Merge pull request** → **Confirm merge**

> ถ้าต้องการข้าม PR ทำด้วย command line แทน:
> ```bash
> git checkout main
> git merge claude/wizardly-hypatia-SpsnU
> git push origin main
> ```

---

### ขั้นที่ 2 — เปิด GitHub Pages Settings

1. ไปที่ repo: `https://github.com/nytgmkt/nytg-social-dashboard`
2. คลิก **Settings** (tab บนสุด)
3. เลื่อนลงเมนูซ้ายมือ → คลิก **Pages**

---

### ขั้นที่ 3 — เลือก Source

ในหน้า Pages ให้ตั้งค่าดังนี้:

| ฟิลด์ | ค่าที่ต้องเลือก |
|---|---|
| **Source** | `Deploy from a branch` |
| **Branch** | `main` |
| **Folder** | `/ (root)` |

คลิก **Save**

---

### ขั้นที่ 4 — รอ Deploy (1–3 นาที)

GitHub จะรัน Actions ให้อัตโนมัติ ดูสถานะได้ที่:

```
https://github.com/nytgmkt/nytg-social-dashboard/actions
```

รอจนเห็น ✅ สีเขียว

---

### ขั้นที่ 5 — เข้าใช้งาน

URL ของ dashboard จะเป็น:

```
https://nytgmkt.github.io/nytg-social-dashboard/
```

> URL นี้เปิดได้จากทุกอุปกรณ์ที่มี Internet โดยไม่ต้องติดตั้งอะไรเพิ่ม

---

### ขั้นที่ 6 — อัปเดตข้อมูลครั้งต่อไป

เมื่อมีข้อมูลใหม่ **ไม่ต้อง deploy ใหม่** — เปิดหน้าเว็บแล้วใช้ปุ่ม upload ได้เลย:

```
🔄 Load New Data   →  ล้างข้อมูลเดิม แล้วโหลด CSV ใหม่ทั้งหมด
➕ Add Month       →  เพิ่มเดือนใหม่เข้ากับข้อมูลที่มีอยู่
```

ข้อมูลจะเก็บใน localStorage ของ Browser นั้น ๆ  
ถ้าต้องการให้ทีมทุกคนเห็นข้อมูลเดียวกัน ให้แต่ละคน Upload CSV เอง

---

## โครงสร้างไฟล์

```
nytg-social-dashboard/
└── index.html    ← ไฟล์เดียว ทำงานได้สมบูรณ์
```

---

## หมายเหตุ Privacy

- ข้อมูลทั้งหมดเก็บใน **localStorage ของ Browser เครื่องนั้นเท่านั้น**
- ไม่มีการส่งข้อมูลไปยัง server ใด ๆ
- GitHub Pages serve เฉพาะ HTML ไฟล์ — ไม่มี backend
- ข้อมูล Leads (ชื่อลูกค้า, เบอร์โทร) **ไม่ถูกอัปโหลดที่ใด**
