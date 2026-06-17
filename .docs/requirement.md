# NumeroScope — Requirements Specification

> **Project**: NumeroScope  
> **Tech Stack**: Vue 3 (Frontend) + ASP.NET Core Web API (Backend)  
> **Target**: Web application with login, data entry, auto-calculation, preview, and PDF export

---

## Table of Contents

1. [Project Overview](#1-project-overview)
2. [Number Sequence Logic](#2-number-sequence-logic)
3. [System Architecture](#3-system-architecture)
4. [Authentication — Login Page](#4-authentication--login-page)
5. [Data Entry Grid (Main Table)](#5-data-entry-grid-main-table)
6. [Master Config — Editable Headers](#6-master-config--editable-headers)
7. [Auto-Calculation Rules](#7-auto-calculation-rules)
8. [Preview Page — Monthly Calendar](#8-preview-page--monthly-calendar)
9. [PDF Export](#9-pdf-export)
10. [Recommended Free Deployment](#10-recommended-free-deployment)
11. [Appendix — Sequence Rotation Examples](#11-appendix--sequence-rotation-examples)

---

## 1. Project Overview

NumeroScope is a numerology-based web application. Users log in, enter data into a structured grid, and the system auto-calculates derived values. The results are displayed as a 12-month calendar preview and can be exported as PDF.

### Core Workflow

```
Login → Data Entry Grid → Auto Calculation → Monthly Calendar Preview → PDF Export
```

---

## 2. Number Sequence Logic

One of the core concepts is a **cyclic number sequence**. The base (canonical) sequence is:

```
[1, 2, 3, 4, 7, 5, 8, 6]
```

Depending on which **starting number** is chosen, the sequence rotates cyclically:

| Starting Number | Resulting Sequence |
|-----------------|-------------------|
| 1               | 1, 2, 3, 4, 7, 5, 8, 6 |
| 2               | 2, 3, 4, 7, 5, 8, 6, 1 |
| 3               | 3, 4, 7, 5, 8, 6, 1, 2 |
| 4               | 4, 7, 5, 8, 6, 1, 2, 3 |
| 5               | 5, 8, 6, 1, 2, 3, 4, 7 |
| 6               | 6, 1, 2, 3, 4, 7, 5, 8 |
| 7               | 7, 5, 8, 6, 1, 2, 3, 4 |
| 8               | 8, 6, 1, 2, 3, 4, 7, 5 |

**Algorithm**: Given `start` (1–8), find its index in the base sequence `[1,2,3,4,7,5,8,6]`, then rotate left by that index.

---

## 3. System Architecture

```
┌─────────────────────────────────────────────────────┐
│                    Frontend (Vue 3)                   │
│  ┌──────────┐  ┌──────────┐  ┌────────┐  ┌────────┐ │
│  │  Login    │  │  Data     │  │Preview │  │ Master  │ │
│  │  Page     │  │  Entry    │  │Monthly │  │ Config  │ │
│  │           │  │  Grid     │  │Calendar│  │ Page    │ │
│  └──────────┘  └──────────┘  └────────┘  └────────┘ │
│                      │                              │
│              Axios HTTP Calls                        │
└──────────────────────┬──────────────────────────────┘
                       │
┌──────────────────────▼──────────────────────────────┐
│                  Backend (ASP.NET Core)               │
│  ┌──────────┐  ┌──────────┐  ┌────────────────────┐ │
│  │ Auth     │  │ Grid     │  │ PDF Generation     │ │
│  │ (JWT)    │  │ CRUD API │  │ (QuestPDF / wkhtml)│ │
│  └──────────┘  └──────────┘  └────────────────────┘ │
│                      │                              │
│              Entity Framework Core                   │
└──────────────────────┬──────────────────────────────┘
                       │
              ┌────────▼────────┐
              │   Database      │
              │ (PostgreSQL /   │
              │  SQL Server)    │
              └─────────────────┘
```

### 3.1 Tech Stack Details

| Layer        | Technology                         |
|-------------|-----------------------------------|
| Frontend    | Vue 3 + Composition API + Vite    |
| State Mgmt  | Pinia                             |
| Routing     | Vue Router                        |
| HTTP Client | Axios                             |
| Backend     | ASP.NET Core 8 / .NET 9 Web API   |
| Auth        | JWT (Identity + Bearer Token)     |
| ORM         | Entity Framework Core             |
| PDF         | QuestPDF (C#) or jsPDF (client)   |
| Database    | PostgreSQL (production) / SQLite (dev) |
| CSS         | Tailwind CSS or PrimeVue          |

---

## 4. Authentication — Login Page

### Page: `/login`

### Elements
- Username / Email input
- Password input
- Login button
- (Optional) Register link for first-time admin setup

### Behavior
1. User submits credentials
2. Backend validates against stored user records
3. On success, returns a **JWT token** (with expiry)
4. Frontend stores token in `localStorage` (or httpOnly cookie)
5. Protected routes redirect to `/login` if no valid token exists
6. (Optional) Session timeout after inactivity

### Security Requirements
- Passwords hashed with bcrypt / ASP.NET Identity
- JWT includes role claims
- HTTPS only in production

---

## 5. Data Entry Grid (Main Table)

### Page: `/grid`

### Grid Layout — 7 Rows × 8 Columns

The grid is a fixed 8×7 table with the following structure:

|     | Col 1 | Col 2 | Col 3 | Col 4 | Col 5 | Col 6 | Col 7 | Col 8 |
|-----|-------|-------|-------|-------|-------|-------|-------|-------|
| **A** | 1     | 2     | 3     | 4     | 7     | 5     | 8     | 6     |
| **B** | input | input | input | input | input | input | input | input |
| **C** | auto  | auto  | auto  | auto  | auto  | auto  | auto  | auto  |
| **D** | input | input | input | input | input | input | input | input |
| **E** | input | input | input | input | input | input | input | input |
| **F** | input | input | input | input | input | input | input | input |
| **G** | input | input | input | input | input | input | input | input |

### Row Details

| Row | Type         | Description                                            |
|-----|-------------|--------------------------------------------------------|
| A   | **Fixed**   | Always `[1, 2, 3, 4, 7, 5, 8, 6]`. Cannot be edited. |
| B   | **User Input** | Free text / number entry per cell                   |
| C   | **Auto-calc**  | `C[i] = Concat(A[i], B[i])` — see §7                |
| D   | **User Input** | Free input per cell                                   |
| E   | **User Input** | Free input per cell                                   |
| F   | **User Input** | Free input per cell                                   |
| G   | **User Input** | Free input per cell                                   |

### Visual Behavior
- Row A cells are **read-only**, visually distinct (greyed out)
- Row C cells are **auto-calculated**, read-only, visually distinct (light blue / green)
- Rows B, D, E, F, G are editable input fields
- All cells are single-line text/number inputs
- The grid should be responsive (scrollable on mobile)

### Constraints
- Column headers (Col 1–8) are **editable** via Master Config (§6)
- Row labels (A–G) are **fixed**
- Grid data must auto-save or have a "Save" button

---

## 6. Master Config — Editable Headers

### Page: `/config` (accessible only to admin)

### Configurable Items
1. **Column Header Labels** — Rename "Col 1" through "Col 8" to custom names (e.g., "House", "Health", "Wealth", etc.)
2. **Starting Number** — Select the starting number (1–8) that defines the rotated sequence used elsewhere in the app

### Persistence
- Config values stored in the database
- Loaded on app startup / when accessing the grid
- Changes to headers reflect immediately on the Data Entry Grid page

---

## 7. Auto-Calculation Rules

### Row C Calculation

For each column position `i` (1 to 8):

```
C[i] = String(A[i]) + String(B[i])
```

| A[i] | B[i] | C[i] |
|------|------|------|
| 1    | 3    | "13" |
| 2    | 7    | "27" |
| 3    | 1    | "31" |
| 4    | 5    | "45" |

- This is **string concatenation**, not numeric addition
- Updates **instantly** when any cell in Row B changes (reactive)

### Row H (Calendar Input)

Row H (not shown in the standard grid) is a special row that holds the **eight final values** used to populate the calendar. The origin of Row H values is determined by the domain logic (to be defined in Phase 2).

---

## 8. Preview Page — Monthly Calendar

### Page: `/preview`

### Layout — 12 Pages, 1 Month Per Page

The preview shows a **full-year calendar**, one month per view / page.

### Calendar Grid Structure

Each month displays a standard calendar grid:

```
┌──────────────────────────────────────────────┐
│            January 2026                       │
├──────┬──────┬──────┬──────┬──────┬──────┬────┤
│ Sun  │ Mon  │ Tue  │ Wed  │ Thu  │ Fri  │ Sat│
├──────┼──────┼──────┼──────┼──────┼──────┼────┤
│      │      │      │  1   │  2   │  3   │  4 │
│      │      │      │ val  │ val  │ val  │val │
├──────┼──────┼──────┼──────┼──────┼──────┼────┤
│  5   │  6   │  7   │  8   │ ...  │ ...  │ ...│
│ val  │ val  │ val  │ val  │      │      │    │
├──────┼──────┼──────┼──────┼──────┼──────┼────┤
│ ...  │      │      │      │      │      │    │
└──────┴──────┴──────┴──────┴──────┴──────┴────┘
```

### Mapping Final Values to Calendar

- Each of the **8 final values** (from the grid) is assigned to specific **date patterns** within a month
- The exact mapping logic (e.g., value assigned to Day 1, Day 2, etc. or assigned by week patterns) will be refined in Phase 2
- Values that appear on dates are displayed inside each day cell

### Navigation
- **Prev / Next** buttons to switch between months
- **Dropdown** to jump to a specific month
- **Year selector** (if year is configurable)

### Preview Features
- Clean, printable layout
- Values displayed within each date cell
- Color coding or badges for different value types (optional)

---

## 9. PDF Export

### Trigger
- A **"Export PDF"** button on the Preview page
- (Optional) Export from individual month view or entire year

### Output
- **Single PDF file** containing all 12 months (12 pages)
- Each page = one month calendar
- Format: A4 or Letter, portrait or landscape (configurable)
- Includes: Year, Month name, Day numbers, and the assigned values

### Generation Approach (Frontend)

**Option A — Client-side with html2pdf / jsPDF**
- Vue component captures the calendar DOM
- Converts to PDF using `html2canvas` + `jsPDF`
- Pros: No server load
- Cons: Less precise layout control

**Option B — Server-side with QuestPDF (C#)**
- Backend generates the PDF using QuestPDF library
- Frontend sends grid data to backend API
- Backend returns the PDF file as a download
- Pros: Pixel-perfect, consistent output
- Cons: Slightly more backend complexity

> **Recommendation**: Option B (server-side) for production-quality PDFs.

---

## 10. Recommended Free Deployment

### Frontend (Vue 3 + Vite)

| Platform     | Free Tier Highlights                                  |
|-------------|------------------------------------------------------|
| **Vercel**  | ∞ bandwidth, 100 GB/month, 6000 build min/month      |
| **Netlify** | 100 GB bandwidth, 300 build min/month                |
| **Cloudflare Pages** | ∞ bandwidth, 500 builds/month, global CDN    |
| **GitHub Pages** | Free for public repos, 1 GB storage, 100 GB/month |

> **Recommendation**: **Vercel** or **Cloudflare Pages** — easy Git-based deploy, generous free tier, global CDN.

### Backend (ASP.NET Core Web API)

| Platform        | Free Tier Highlights                                     |
|----------------|---------------------------------------------------------|
| **Azure App Service (F1)** | 1 GB memory, 60 min/day compute (resets daily) |
| **Railway**     | $5 credit/month (~500 hours runtime), 500 MB RAM        |
| **Render**      | 750 hours/month, 512 MB RAM, spins down after inactivity |
| **Oracle Cloud (Always Free)** | 2 x AMD VMs, 24 GB RAM total (more setup effort) |

> **Recommendation**: **Azure App Service (F1 Free)** for simplest setup with ASP.NET Core.  
> **Alternative**: **Render** — easier to set up, 750 hrs/month is enough for demo/PoC.

### Database

| Platform         | Free Tier Highlights                                  |
|-----------------|------------------------------------------------------|
| **Supabase (PostgreSQL)** | 500 MB database, 2 GB RAM, forever free       |
| **Neon (PostgreSQL)** | 0.5 GB storage, 100 hrs/month compute            |
| **SQLite** (app-only) | No server needed, single file (dev/demo only)     |

> **Recommendation**: **Supabase PostgreSQL** for production-like setup (free, scalable, hosted).

### Recommended Architecture (Fully Free)

```
GitHub Repository (code)
    │
    ├── Frontend → Vercel (free) — connected via Git deploy
    │
    ├── Backend  → Render.com or Azure App Service Free (ASP.NET Core)
    │
    └── Database → Supabase (PostgreSQL) — managed, free tier
```

### CI/CD
- GitHub Actions for building and testing
- Automatic deploy on push to `main` branch (via Vercel / Render integrations)

---

## 11. Appendix — Sequence Rotation Examples

### Base Sequence (Canonical)
```
Index:   0  1  2  3  4  5  6  7
Value:   1  2  3  4  7  5  8  6
```

### Rotated Sequence by Starting Number

```
Start=1 → [1, 2, 3, 4, 7, 5, 8, 6]  (no rotation)
Start=2 → [2, 3, 4, 7, 5, 8, 6, 1]
Start=3 → [3, 4, 7, 5, 8, 6, 1, 2]
Start=4 → [4, 7, 5, 8, 6, 1, 2, 3]
Start=5 → [5, 8, 6, 1, 2, 3, 4, 7]
Start=6 → [6, 1, 2, 3, 4, 7, 5, 8]
Start=7 → [7, 5, 8, 6, 1, 2, 3, 4]
Start=8 → [8, 6, 1, 2, 3, 4, 7, 5]
```

### Rotation Algorithm (JavaScript)

```js
function rotateSequence(start) {
  const base = [1, 2, 3, 4, 7, 5, 8, 6];
  const idx = base.indexOf(start);
  if (idx === -1) return base;
  return [...base.slice(idx), ...base.slice(0, idx)];
}
```

### Rotation Algorithm (C#)

```csharp
int[] RotateSequence(int start) {
    int[] baseSeq = new int[] { 1, 2, 3, 4, 7, 5, 8, 6 };
    int idx = Array.IndexOf(baseSeq, start);
    if (idx < 0) return baseSeq;
    return baseSeq.Skip(idx).Concat(baseSeq.Take(idx)).ToArray();
}
```

---

## Roadmap / Phases

| Phase | Features                                              |
|-------|------------------------------------------------------|
| **1** | Login, Grid Entry (Rows A–G), Row C auto-calc        |
| **2** | Master Config (headers, starting number), Save/Load  |
| **3** | Calendar Preview (12 months), Final value mapping    |
| **4** | PDF Export                                           |
| **5** | Deployment, Polish, Testing                          |

---

*Document version 1.0  
Last updated: 2026-06-17*
