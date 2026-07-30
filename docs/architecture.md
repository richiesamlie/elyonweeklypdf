# Architecture — Elyon Weekly Program PDF Hub

## Overview

A Google Apps Script web application that connects to Google Sheets to export weekly program schedules as PDFs and edit subject materials directly from the browser. Designed for Elyon Primary School teachers.

```
┌─────────────────────────────────────────────────────────────────┐
│                        PWA Wrapper (GitHub Pages)               │
│  index.html ── iframe ──► GAS Web App (script.google.com)      │
│  manifest.json                                                  │
│  sw.js                                                          │
└─────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────┐
│                    Google Apps Script Backend (Code.gs)          │
│                                                                 │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────────────┐  │
│  │ scanParent   │  │ getWeeksFor  │  │ createWeeklyPdfZip   │  │
│  │ Folder()     │  │ Level()      │  │ ()                   │  │
│  │              │  │              │  │                      │  │
│  │ ~0.5s*       │  │ ~1s*         │  │ Parallel PDF export  │  │
│  │ Folder scan  │  │ Tab scan     │  │ via fetchAll()       │  │
│  └──────┬───────┘  └──────┬───────┘  └──────────┬───────────┘  │
│         │                 │                      │              │
│  ┌──────┴───────┐  ┌──────┴───────┐  ┌──────────┴───────────┐  │
│  │ createNew    │  │ deleteWeek   │  │ getClassSheetData    │  │
│  │ WeekTab()    │  │ Tab()        │  │ saveClassSheetData   │  │
│  │              │  │              │  │ ()                   │  │
│  │ LockService  │  │ Dry-run      │  │                      │  │
│  │ protected    │  │ validation   │  │ Editor CRUD          │  │
│  └──────────────┘  └──────────────┘  └──────────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────┐
│                       Google Services                           │
│                                                                 │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────────────┐  │
│  │ Google Drive  │  │ Google       │  │ CacheService         │  │
│  │              │  │ Sheets       │  │ PropertiesService    │  │
│  │ Folder scan  │  │ Tab CRUD     │  │ LockService          │  │
│  │ File list    │  │ Cell R/W     │  │ UserCache (10 min)   │  │
│  └──────────────┘  └──────────────┘  └──────────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
```

---

## Tech Stack

| Layer | Technology | Purpose |
|-------|-----------|---------|
| **Frontend** | HTML/CSS/JS (single file) | All UI in `WeekSelector.html` |
| **Backend** | Google Apps Script (`Code.gs`) | Server-side logic, API |
| **Data** | Google Sheets | Source of truth for schedules |
| **Storage** | Google Drive | Folder structure for grade levels |
| **Caching** | CacheService | 10-min user cache for level data |
| **Concurrency** | LockService | Prevent race conditions on tab creation |
| **PWA** | Static HTML + Service Worker | Installable wrapper on GitHub Pages |

---

## Data Model

### Google Drive Folder Structure

```
Parent Folder (configurable)
├── P1/
│   ├── 1A Weekly Program
│   ├── 1B Weekly Program
│   └── 1C Weekly Program
├── P2/
│   ├── 2A Weekly Program
│   ├── 2B Weekly Program
│   └── ...
├── P3/
├── P4/
├── P5/
└── P6/
```

### Spreadsheet Structure (per class)

Each spreadsheet has multiple sheet tabs (one per week):

```
┌─────────────────────────────────────────────────────────────┐
│  Tab: "03-07 Aug"                                           │
│                                                             │
│  Row 10 (D10): Grade (e.g. "5")                             │
│  Row 11 (D11): Week name (e.g. "03-07 Aug")                │
│  Row 14-26: Subject rows                                    │
│    Col B: Subject name (e.g. "English", "Mathematics")      │
│    Col D: Materials                                         │
│    Col E: Notes                                             │
│  Row 29-30: Homeroom teacher                                │
│  Row 33-34: Teacher name                                    │
│  H1: Last-edited stamp (hidden, "email|isoDate")            │
└─────────────────────────────────────────────────────────────┘
```

---

## Data Flow

### 1. App Startup (approximately 0.5s in observed use)

```
Browser ──google.script.run──► scanParentFolder()
                                    │
                                    ▼
                              DriveApp.getFolderById()
                                    │
                                    ▼
                              Returns: level folders (P1-P6)
                                    │
                                    ▼
                              Render grade level chips
                              Auto-select first level
```

### 2. Level Selection (approximately 1s first load; immediate on client cache hit)

```
Click grade chip ──► selectGradeLevel(levelId)
                          │
                          ├─ Cache hit? → renderLevelData(cached)
                          │
                          └─ Cache miss? ──google.script.run──► getWeeksForLevel()
                                                                  │
                                                                  ▼
                                                          Drive: scan files
                                                          Sheets: read all tabs
                                                          Build union week list
                                                          Build availability map
                                                                  │
                                                                  ▼
                                                          CacheService.put(10 min)
                                                                  │
                                                                  ▼
                                                          renderLevelData()
```

If a grade has no week tabs in any spreadsheet, `getWeeksForLevel()` returns an
error rather than a normal empty result. The editor's visual empty state applies
when the selected class has no matching week after a successful level load.

### 3. PDF Export (parallel)

```
Click Export ──► generateExport()
                      │
                      ▼
              createWeeklyPdfZip()
                      │
                      ├─ Validate: week exists in all selected classes
                      │
                      ├─ Build PDF export URLs (Google Sheets export API)
                      │   - Print area: cols A-E, rows 1-40
                       │   - Scale: fit to width (intended to improve horizontal centering)
                      │   - Margins: 0.25" top/bottom, 0.5" left/right
                      │
                      ├─ UrlFetchApp.fetchAll() ← PARALLEL, not sequential
                      │
                      ├─ 1 class → return single PDF (base64)
                      │  >1 class → Utilities.zip() → return ZIP (base64)
                      │
                      ▼
              Browser receives base64 → data URI → auto-download
```

### 4. Editor: Load & Save

```
Load:
  selectClass ──► loadClassContent()
                      │
                      └─► getClassSheetData(spreadsheetId, weekName)
                              │
                              ├─ Read cells: D10 (grade), D11 (date)
                              ├─ Read rows 14-26: subject name, materials, notes
                              ├─ Read H1: last-edited stamp
                              │
                              └─► renderEditorForm(data)

Save:
  Click Save ──► saveContent()
                      │
                      ├─ Build updates[] from pendingDirty (only changed rows)
                      │
                      └─► saveClassSheetData(spreadsheetId, weekName, updates)
                              │
                              ├─ Write to cells (rows 14-26, cols D-E)
                              ├─ Write stamp to H1 (email|isoDate)
                              └─ SpreadsheetApp.flush()
```

---

## Key Design Decisions

### 1. Instant-Load Architecture

**Problem:** Opening all spreadsheets at startup is slow (~5-10s).

**Solution:** Two-phase loading:
- **Phase 1 (0.5s):** Scan parent folder for subfolder *names* only (P1, P2, P3…). No spreadsheets opened.
- **Phase 2 (1s, on-demand):** When user clicks a grade, open *only that grade's* spreadsheets. Cache for 10 minutes.

**Result:** App feels instant. Teacher sees grade chips in <0.5s.

### 2. Union Week Scan

**Problem:** Different classes might have different week tabs (some created, some not).

**Solution:** Scan ALL spreadsheets in a grade, build a *union* of all week tabs. Normalize names with `normalizeWeekKey_()` to handle variations:
- `03-07 Aug` vs `3-7 Aug` vs `03-07 August` → all normalize to `37aug`

**Result:** Dropdown shows all weeks that exist in *any* class. Availability map tracks which classes have which weeks.

### 3. Parallel PDF Export

**Problem:** Exporting 4 classes sequentially takes ~4s each = ~16s total.

**Solution:** `UrlFetchApp.fetchAll()` executes all HTTP requests in parallel.

**Result:** 4 classes export in ~4s (same as 1 class), not ~16s.

### 4. Fuzzy Tab Finder (3-tier)

**Problem:** Teachers type tab names inconsistently (`03-07 Aug` vs `3-7 aug`).

**Solution:** `findSheetByName_()` tries 3 matching tiers:
1. Exact match
2. Trimmed + case-insensitive
3. Normalized (leading zeros, hyphens, month abbreviations)

If no tier matches, the function returns `null`.

**Result:** Handles all common variations without requiring exact input.

### 5. LockService for Tab Creation

**Problem:** Two teachers creating a tab simultaneously could duplicate tabs.

**Solution:** `LockService.getDocumentLock()` with 30s timeout. Pre-flight check for existing tabs before creation.

**Result:** Only one teacher can create tabs at a time per grade level.

### 6. Partial Save (Dirty Tracking)

**Problem:** Saving all subjects when only 1 changed is wasteful.

**Solution:** `markDirty()` tracks which rows changed vs server state. `saveContent()` only sends dirty rows. `renderSaveBar_()` shows count of unsaved changes.

**Result:** Minimal writes to Google Sheets.

---

## Performance Characteristics

| Operation | Time | Notes |
|-----------|------|-------|
| App startup | approximately 0.5s* | Folder scan only |
| Level selection (first) | approximately 1s* | Opens spreadsheets, reads tabs |
| Level selection (cached) | immediate* | Client-side cache |
| PDF export (1 class) | approximately 2-4s* | Google Sheets export API |
| PDF export (4 classes) | approximately 4s* | Parallel via fetchAll() |
| Load editor data | approximately 1-2s* | Opens 1 spreadsheet |
| Save editor data | approximately 1s* | Writes to 1 spreadsheet |

\* Approximate observations, not service-level targets. Actual time depends on Apps Script,
Drive, Sheets, network, and account load.

---

## Caching Strategy

| Cache | Scope | TTL | Purpose |
|-------|-------|-----|---------|
| `window.LEVEL_CACHE` | Client (browser) | Session | Immediate client-side level switching |
| `CacheService.getUserCache` | Server (GAS) | 10 min | Avoid re-scanning sheets |
| `PropertiesService` | Server (GAS) | Persistent | Store folder URL config |

Cache invalidation:
- Manual: "🔄 Refresh" button clears server cache
- Automatic: `createNewWeekTab()` and `deleteWeekTab()` clear cache after mutation

---

## Security Model

| Layer | Mechanism | Enforced By |
|-------|-----------|-------------|
| Authentication | @elyon.sch.id only | GAS deploy settings (not code) |
| Execution identity | Requests run as the signed-in user | "Execute as: User accessing" |
| Admin actions | Shared password gate | `verifyPassword()` in code |
| Data isolation | Per-user cache; Drive/Sheets permissions still apply | `CacheService.getUserCache()` |

**Note:** The admin password is intentionally implemented as a shared client-visible gate.
It is not a separate authorization boundary between already-authorized school users.
Primary access control comes from the GAS deployment restriction and Google Drive/Sheets
permissions.

---

## PDF Export Configuration

```javascript
// Tunable constants in Code.gs
var MAX_PRINT_ROW = 40;    // Last row to include in PDF
var MIN_PRINT_ROW = 34;    // Minimum rows (template padding)
var MAX_PRINT_COL = 5;     // Column E
var PRINT_COL_START = 1;   // Column A

// Export URL parameters
size=A4                     // Paper size
portrait=true               // Orientation
scale=2                     // Fit to width; centering depends on sheet print layout
top_margin=0.25             // inches
bottom_margin=0.25
left_margin=0.5
right_margin=0.5
```

---

## File Structure (GAS Project)

```
Google Apps Script Project
├── Code.gs                    ← Backend (663 lines)
│   ├── doGet()                ← Entry point, serves HTML
│   ├── scanParentFolder()     ← Phase 1: folder scan
│   ├── getWeeksForLevel()     ← Phase 2: tab scan + cache
│   ├── createWeeklyPdfZip()   ← PDF export
│   ├── createNewWeekTab()     ← Tab creation (locked)
│   ├── deleteWeekTab()        ← Tab deletion (validated)
│   ├── getClassSheetData()    ← Editor: read
│   ├── saveClassSheetData()   ← Editor: write
│   ├── findSheetByName_()     ← 3-tier fuzzy matcher
│   └── normalizeWeekKey_()    ← Week name normalizer
│
├── WeekSelector.html          ← Frontend (current line count may change)
│   ├── CSS                    ← Design system
│   ├── HTML                   ← 3 modes: Exporter, Editor, Guide
│   └── JS                     ← Client logic, state management
│
└── Editor.html.legacy-backup  ← Archived, not served
```

---

## PWA Wrapper (GitHub Pages)

```
elyonweeklypdf/
├── index.html       ← iframe wrapper + install prompt
├── manifest.json    ← PWA manifest
├── sw.js            ← service worker (cache shell assets)
├── icon.svg         ← app icon
└── README.md
```

The PWA wrapper is a thin shell that:
1. Embeds the GAS app via `<iframe>`
2. Provides "Add to Home Screen" installability
3. Caches the wrapper shell for offline loading

The GAS application itself is not offline-capable. The iframe still requires a
network connection, Google authentication, and access to Google Drive/Sheets.

---

## Spreadsheet Template Layout

```
     A          B              C              D              E
  ┌──────────────────────────────────────────────────────────────┐
1 │ (hidden row - H1 stores last-edited stamp)                   │
  │                                                              │
10│                   [Grade]                                    │
11│                   [Week Name]                                │
  │                                                              │
14│              English         [Materials]  [Notes]            │
15│              Mathematics     [Materials]  [Notes]            │
16│              Science         [Materials]  [Notes]            │
  │              ...                                             │
26│              [Last Subject]  [Materials]  [Notes]            │
  │                                                              │
29│              [Homeroom]                                     │
30│              [Homeroom Name]                                 │
  │                                                              │
33│              [Teacher]                                      │
34│              [Teacher Name]                                  │
  └──────────────────────────────────────────────────────────────┘
```

PDF export range: `A1:E40` (rows 1-40, columns A-E)

---

## GAS Limits & Quotas

| Limit | Value | Impact |
|-------|-------|--------|
| Execution time | 6 min per execution* | Bulk export can time out if the workload is too large |
| UrlFetchApp.fetchAll | 50 requests/call | Max 50 classes per export |
| CacheService | 100KB/key, 1MB/user | Level cache fits easily |
| PropertiesService | 500KB/script | Folder URL storage |
| Daily execution quota | ~90 min/day (consumer) | Sufficient for small teams |
| Concurrent executions | 30/user | No issue for typical usage |

\* Verify current quotas against the official Apps Script quotas page before relying
on a plan-specific value. Limits can vary by account type and Google may change them.
