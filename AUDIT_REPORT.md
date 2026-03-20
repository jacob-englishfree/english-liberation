# Comprehensive Audit Report
**Date:** 2026-03-17
**Audited Apps:** ehg-academy (Admin Dashboard), english-liberation (Student App)

---

## CRITICAL Issues

### [C1] Supabase Anon Key Exposed in Client-Side HTML
- **App:** english-liberation (index.html, line 444)
- **Status:** ❌ Broken/Error
- **Detail:** The Supabase anon key is hardcoded directly in the HTML source: `const SUPABASE_KEY='eyJhbG...'`. While anon keys are designed for client-side use, there are no Row Level Security (RLS) policies visible in the codebase to verify data is properly protected. If RLS is not enabled on all tables, any user can read/write all data.

### [C2] Curriculum Names Mismatch Between Admin and Student App
- **App:** Both
- **Status:** ❌ Broken/Error
- **Detail:** The admin app's `assignments/page.tsx` uses curriculum presets: `"정규반"`, `"프리패스구문"`, `"파이널코드V1"`, `"파이널코드V2"`. The student app's `index.html` uses completely different names: `"프리패스 어법/구문"`, `"이미지 독해"`, `"제로 코드"`, `"일관적 코드"`, `"파이널 코드"`, `"정규반(기출제로원)"`, `"써먹는단어"`, `"내신의기술"`. These will never match when cross-referencing data between apps. A student who selects "프리패스 어법/구문" in the survey will not be matched by the admin homework distribution system that uses "프리패스구문".

### [C3] `toISOString()` Used for Date Calculations — Timezone Bug Risk
- **App:** ehg-academy
- **Status:** ⚠️ Warning (works but has issues)
- **Detail:** Multiple files use `new Date().toISOString().split("T")[0]` to get today's date string. `toISOString()` returns UTC time, which means after midnight KST (UTC+9), until 9am KST, the date will be *yesterday* in UTC. Affected files:
  - `src/app/dashboard/page.tsx` (line 79, 91) — dashboard today stats
  - `src/app/dashboard/students/page.tsx` (line 43) — D-day calculation
  - `src/app/dashboard/attendance/page.tsx` (line 202) — settlement date
  - `src/app/dashboard/payments/page.tsx` (line 73, 82) — payment date
  - `src/app/dashboard/schedule/page.tsx` (line 30, 45, 98, 106) — schedule dates
  - `src/app/dashboard/assignments/page.tsx` (line 72, 148, 336) — assignment dates
  - `src/components/StudentDetailModal.tsx` (line 158, 171, 255) — homework/warning dates

  **Note:** `homework/page.tsx` and `weekly-reports/page.tsx` correctly use a local `toDateStr()` helper. The inconsistency is itself a bug — some pages show correct local dates while others show UTC dates.

---

## HIGH Severity Issues

### [H1] UI Design Inconsistency — Several Pages Not Using Toss Design System
- **App:** ehg-academy
- **Status:** ⚠️ Warning
- **Detail:** Several pages use `border border-gray-200` + `rounded-xl` (old style) instead of `shadow-[0_1px_3px_rgba(0,0,0,0.04)]` + `rounded-2xl` (Toss style). Also some use `bg-blue-500` instead of `bg-[#3182F6]`. Affected pages:
  - `students/page.tsx` — uses `border border-gray-200`, `bg-blue-500`, `rounded-xl` throughout (tables, modals, buttons)
  - `salary/page.tsx` — uses `border border-gray-200`, `bg-blue-500`, `rounded-xl` for all cards
  - `payments/page.tsx` — uses `border border-gray-200`, `rounded-xl`, `bg-blue-500` buttons, also uses `border border-red-300` / `border border-emerald-300` stat cards
  - `settings/page.tsx` — uses `border border-gray-200`, `bg-blue-500` buttons
  - `schedule/page.tsx` — uses `border border-gray-200`, `bg-blue-500` buttons
  - `assignments/page.tsx` — uses `border border-gray-200`, `bg-blue-500` buttons

  Pages that DO follow Toss design correctly: `dashboard/page.tsx`, `homework/page.tsx`, `tests/page.tsx`, `weekly-reports/page.tsx`, `zoom-survey/page.tsx`, `zoom-session/page.tsx`, `attendance/page.tsx`, `Sidebar.tsx`

### [H2] Login Page Uses Dark Theme — Inconsistent with Light App
- **App:** ehg-academy
- **Status:** ⚠️ Warning
- **Detail:** `login/page.tsx` uses `bg-gray-950`, `bg-gray-900`, `bg-gray-800`, `border-gray-700`, `text-white` — a full dark theme. The entire rest of the app is light-themed Toss design. This creates a jarring visual transition when logging in.

### [H3] `console.error` Left in Production Code
- **App:** ehg-academy
- **Status:** ⚠️ Warning
- **Detail:** `zoom-survey/page.tsx` line 270: `console.error("Supabase error:", error);` left in production. Should use a proper error handling pattern or at minimum be removed.

### [H4] Unused Imports in homework/page.tsx
- **App:** ehg-academy
- **Status:** ⚠️ Warning
- **Detail:** `homework/page.tsx` line 8 imports `Warning` and `StudentNote` types from `@/types/database`, but neither is used anywhere in the file. The `Warning` type is only used indirectly via the `addHomeworkWarning` function which constructs a warning object inline.

### [H5] Missing Error Handling on Supabase Calls
- **App:** ehg-academy
- **Status:** ⚠️ Warning
- **Detail:** Most Supabase queries lack error checking. The pattern across files is:
  ```
  const { data } = await supabase.from("table").select("*");
  ```
  Without checking `error`. If Supabase returns an error, `data` will be `null`, which is silently handled by `(data || [])`, but the user gets no feedback that data failed to load. Affected in critical data-loading functions:
  - `dashboard/page.tsx` `fetchDashboard()` — no try/catch, no error check
  - `students/page.tsx` `fetchData()` — no error handling
  - `attendance/page.tsx` `fetchAttendance()`, `fetchAllAttendance()` — no error handling
  - `settings/page.tsx` `fetchData()` — no error handling
  - `schedule/page.tsx` `fetchSchedules()` — no error handling
  - `assignments/page.tsx` `fetchData()` — no error handling

### [H6] Homework Status "예정" Inserted but Never Used in Homework Grid
- **App:** ehg-academy
- **Status:** ⚠️ Warning
- **Detail:** `assignments/page.tsx` lines 270-279 insert homework records with `status: "예정"` when distributing assignments. However, `homework/page.tsx` only recognizes `"ㅇ"`, `"x"`, and `null` as valid statuses. The "예정" status will display as an empty cell (same as null), making it invisible to teachers. This data is being written but never meaningfully consumed.

---

## MEDIUM Severity Issues

### [M1] getWeekStart() in zoom-survey/page.tsx Returns ISO String (UTC)
- **App:** ehg-academy
- **Status:** ⚠️ Warning
- **Detail:** `getWeekStart()` (line 37-44) creates a Monday date but then returns `monday.toISOString()` which converts to UTC. This is used to filter surveys by `created_at >= weekStart`. Since `created_at` in Supabase is stored in UTC, this mostly works, but the Monday boundary could be off by hours depending on timezone.

### [M2] Teacher Filtering May Leak Data for Non-Directors
- **App:** ehg-academy
- **Status:** ✅ Working correctly
- **Detail:** Checked all pages — non-director teachers are correctly filtered by `session.teacherId` in: `dashboard/page.tsx` (line 112-113), `homework/page.tsx` (line 439-441), `tests/page.tsx` (line 342-344), `weekly-reports/page.tsx` (line 527-529), `students/page.tsx` (line 102-104), `zoom-survey/page.tsx` (line 241-243). Director-only pages (`salary`, `payments`, `settings`) properly redirect non-directors.

### [M3] zoom/page.tsx Tab Imports — All 3 Components Working
- **App:** ehg-academy
- **Status:** ✅ Working correctly
- **Detail:** `zoom/page.tsx` dynamically imports all 3 tabs: `ZoomSessionContent`, `TestsContent`, `ZoomSurveyContent`. All three source files export the named components correctly. Tab switching works as expected.

### [M4] Homework Click (ㅇ/x) Save to Supabase
- **App:** ehg-academy
- **Status:** ✅ Working correctly
- **Detail:** `homework/page.tsx` `handleCellClick()` (left click = toggle ㅇ) and `handleCellContext()` (right click = toggle x) both call `applyStatus()` which correctly inserts/updates/deletes homework records in Supabase with proper try/catch error handling.

### [M5] Test Score Entry
- **App:** ehg-academy
- **Status:** ✅ Working correctly
- **Detail:** `tests/page.tsx` `handleScoreSave()` properly inserts or updates test scores, handles retest flow, and auto-navigates to next student. Has proper validation (0-100 range).

### [M6] Weekly Report Save and Copy
- **App:** ehg-academy
- **Status:** ✅ Working correctly
- **Detail:** `weekly-reports/page.tsx` implements auto-save on blur (`handleTextareaBlur`), explicit draft save (`handleDraftSave`), and save+copy (`handleSaveAndCopy`). Clipboard copy generates formatted text with `getFormattedText()`. beforeunload warning for unsaved changes is implemented.

### [M7] AI Feedback API
- **App:** ehg-academy
- **Status:** ✅ Working correctly
- **Detail:** Both `/api/feedback/route.ts` and `/api/quick-feedback/route.ts` correctly:
  - Check for `ANTHROPIC_API_KEY` env var
  - Make proper POST to `https://api.anthropic.com/v1/messages`
  - Use model `claude-sonnet-4-20250514`
  - Include proper headers (`x-api-key`, `anthropic-version`)
  - Handle errors with try/catch and proper HTTP status codes

### [M8] Student Status Dropdown Save
- **App:** ehg-academy
- **Status:** ✅ Working correctly
- **Detail:** `students/page.tsx` line 353-358: status dropdown `onChange` handler calls `supabase.from("students").update({ status: newStatus })` and updates local state. Also works in `StudentDetailModal.tsx` line 334-339.

### [M9] Attendance Self-Edit for Teachers
- **App:** ehg-academy
- **Status:** ✅ Working correctly
- **Detail:** `attendance/page.tsx` line 154: `const canEdit = director || selectedTeacher?.id === session?.teacherId;` — teachers can edit their own attendance. The calendar input fields are conditionally rendered based on `canEdit`.

### [M10] `useCallback` Missing Dependencies
- **App:** ehg-academy
- **Status:** ⚠️ Warning
- **Detail:** Several `useCallback` hooks have empty or incomplete dependency arrays:
  - `students/page.tsx` line 108: `fetchData` uses `director` and `session` but dependency is `[]`
  - `salary/page.tsx` line 93: `fetchTeachers` dependency is `[]` but uses `selectedTeacher`
  - `attendance/page.tsx` line 53: `fetchTeachers` uses `director` and `session?.teacherId` but dependency is `[director, session?.teacherId]` (correct here)

  The build passes without errors because these are suppressed by eslint-disable comments in some files, but they could cause stale closure bugs.

### [M11] Password Stored and Displayed in Plaintext
- **App:** ehg-academy
- **Status:** ⚠️ Warning
- **Detail:** `settings/page.tsx` line 269: `<span className="text-gray-400 font-mono">{t.password}</span>` — passwords are displayed in plaintext on the settings page. `login/page.tsx` also compares passwords in plaintext by querying Supabase with `.eq("password", password)`. No hashing is used.

---

## LOW Severity Issues

### [L1] `toDateStr()` vs `toISOString().split("T")[0]` Inconsistency
- **App:** ehg-academy
- **Status:** ⚠️ Warning
- **Detail:** The codebase has two date formatting approaches: `toDateStr(d)` (local timezone, used in homework/weekly-reports) and `new Date().toISOString().split("T")[0]` (UTC, used elsewhere). This creates potential date mismatches between pages.

### [L2] No `console.log` in Production Code
- **App:** ehg-academy
- **Status:** ✅ Working correctly
- **Detail:** No `console.log` statements found in any source file. One `console.error` found (see H3).

### [L3] No TypeScript `any` Usage
- **App:** ehg-academy
- **Status:** ✅ Working correctly
- **Detail:** No `: any` type annotations found in any source file.

### [L4] Build Check
- **App:** ehg-academy
- **Status:** ✅ Working correctly
- **Detail:** `npx next build` completes successfully with zero errors and zero warnings. All 22 routes compile and generate correctly. TypeScript check passes.

### [L5] Mobile Responsiveness
- **App:** ehg-academy
- **Status:** ✅ Working correctly
- **Detail:** Sidebar has mobile overlay mode with slide-in animation. Layout uses `hidden md:flex` for desktop sidebar. Hamburger menu triggers mobile sidebar. Grid layouts use responsive breakpoints (`grid-cols-2 lg:grid-cols-3 xl:grid-cols-6`).

### [L6] Toast Component
- **App:** ehg-academy
- **Status:** ✅ Working correctly
- **Detail:** `Toast.tsx` implements a proper toast notification system with success/error/info variants, auto-dismiss after 3 seconds, centered bottom positioning, and Toss-style design with shadow cards.

---

## Student App (english-liberation) Findings

### [S1] Supabase Connection
- **App:** english-liberation
- **Status:** ✅ Working correctly
- **Detail:** Uses CDN-loaded `@supabase/supabase-js@2`. Creates client with `window.supabase.createClient(SUPABASE_URL, SUPABASE_KEY)`. Falls back to `null` if library not loaded, and checks `if(supa)` before DB operations.

### [S2] Zoom Survey Flow (4 Steps)
- **App:** english-liberation
- **Status:** ✅ Working correctly
- **Detail:** Survey flow has 4 steps:
  1. Select curriculum + teacher (with validation)
  2. Select week number (1-14 range per teacher)
  3. Select day + time (fetches availability from `teacher_schedule` table)
  4. Enter name/school/grade + confirmation
  Each step validates before proceeding. Final submission inserts to `zoom_surveys` table.

### [S3] Teacher Schedule Availability
- **App:** english-liberation
- **Status:** ✅ Working correctly
- **Detail:** `fetchSurveyAvail()` fetches from `teacher_schedule` table and `zoom_surveys` to show unavailable days (marked with cross) and capacity counts per day. Handles weekend edge case (if today is Sat/Sun, shows next week's schedule).

### [S4] OG Image
- **App:** english-liberation
- **Status:** ✅ Working correctly
- **Detail:** `og-image.png` exists (1200x630 PNG). Meta tag references `https://yhg-app.vercel.app/og-image.png`.

### [S5] Teacher Names — Potential Staleness
- **App:** english-liberation
- **Status:** ⚠️ Warning
- **Detail:** Teacher names are HARDCODED in `index.html` line 1918:
  ```javascript
  const teachers=['윤성T','현서T','민중T','주원조교','희수조교','희선실장'];
  ```
  And in `teacherWeeks` on line 1912. These are NOT fetched from the Supabase `teachers` table. If teachers change (new hire, departure, name change), the student app will show stale teacher names. The admin app fetches teachers dynamically from Supabase, so they will be out of sync.

### [S6] Curriculum Names Mismatch (see C2)
- **App:** english-liberation
- **Status:** ❌ Broken/Error
- **Detail:** Student app uses: `['프리패스 어법/구문','이미지 독해','제로 코드','일관적 코드','파이널 코드']` plus `['정규반(기출제로원)']` plus `['써먹는단어','내신의기술']`. Admin app assignments presets use: `['정규반','프리패스구문','파이널코드V1','파이널코드V2']`. These will never match.

### [S7] JS Syntax Validity
- **App:** english-liberation
- **Status:** ⚠️ Warning
- **Detail:** Cannot extract standalone JS for `node --check` as the script is embedded inline in the HTML with template literal rendering functions. However, the build on the admin side succeeds, and the student app is a static HTML file loaded by browser. No syntax errors are apparent from code review.

---

## Summary by Severity

| Severity | Count | Items |
|----------|-------|-------|
| CRITICAL | 3 | C1, C2, C3 |
| HIGH | 6 | H1, H2, H3, H4, H5, H6 |
| MEDIUM | 11 | M1-M11 |
| LOW | 6 | L1-L6 |
| Student App | 7 | S1-S7 |

### Top Priority Fixes:
1. **C2/S6:** Align curriculum names between admin and student apps
2. **C3/L1:** Replace all `toISOString().split("T")[0]` with local `toDateStr()` helper
3. **H1:** Migrate students/salary/payments/settings/schedule/assignments pages to Toss design (shadow cards, #3182F6 blue, rounded-2xl, border-none)
4. **S5:** Fetch teacher names dynamically from Supabase in student app instead of hardcoding
5. **H5:** Add error handling/user feedback on failed Supabase queries
6. **M11:** Hash passwords instead of storing in plaintext
