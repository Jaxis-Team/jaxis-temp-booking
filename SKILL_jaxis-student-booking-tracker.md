---
name: jaxis-student-booking-tracker
description: "Use this skill when the user uploads two CSVs for a JAXIS one-on-one student (a class history file and a Pass/points file) and wants to add new class bookings, update attendance, track points, or build/update that student's HTML tracking dashboard. Trigger on phrases like \"book more classes for [student]\", \"update her points\", \"mark [date] as completed\", or when two CSVs matching the history+pass pattern are uploaded together."
---

# JAXIS Student Booking & Points Tracker

## ⚠️ ALWAYS DO FIRST — Credentials & Repo
**Never ask the user for the token or repo URL.** Always read them from:
`/mnt/project/steps_of_build_student_booking_tracker`

That file contains:
- GitHub repo URL: `github.com/Jaxis-Team/jaxis-temp-booking`
- Personal Access Token: (read from file directly)

Clone command:
```
git clone https://Jaxis-Team:<TOKEN>@github.com/Jaxis-Team/jaxis-temp-booking.git
```

---

## Input files (always come as a pair)
1. **Class history CSV** — columns: `#, Class, Time, Attendance, Points, Reservation Time, Pass, Instructor, Dependency`. Sorted **descending by date** (most future date at top, oldest history at bottom).
2. **Pass CSV** — columns: `#, Pass, Payment Status, Usage, Type, Status, Expiration`. Each row is one purchased pass; `Usage` is `used / total`. Only one pass is normally `Active` at a time — that's the one new bookings deduct from.

---

## Key concepts

- **Points are deducted at booking time, not attendance time.** Marking a class "Yes" (attended) later does NOT change the points balance — it's already spent.
- **A "term" is a batch of classes booked together.** There are often two batches to book at once:
  - **Old term leftover** — classes from the previous term that were never booked, needs of a smaller set.
  - **New term** — the new upcoming term's full class set (commonly ~20 classes).
  Keep these two batches visibly separate everywhere (tables, dashboard sections, download export).
- **Points per class**: confirm with the user (commonly 6 pts), don't assume without checking their history pattern.
- **Instructor**: stays the same for a given student unless told otherwise — check history file for the existing instructor and reuse it.
- **Attendance values**: `Yes`, `Upcoming`, `待補課` (pending makeup). Never invent new values without confirming.
- **"Took off" a class = 待補課 + 補課去 button.** If the user says a student *took off / took a day off / skipped / didn't come / 請假 / 沒來* for a class, this AUTOMATICALLY means: set that row's `Attendance` to `待補課`, and in the dashboard render that row's status cell as a clickable **補課去** button (not a plain badge) linking to the makeup form. No confirmation needed — this mapping is fixed. Points do NOT change (already deducted at booking).
- **Makeup classes**: free makeups do NOT deduct points. Note the makeup date in the `Dependency` column of the original missed class row. Do not add a separate row for the makeup unless instructed.
- **Parent name**: the CSV filename often contains the parent's Chinese name — use it in the dashboard header.

---

## Workflow: adding new bookings

1. Read both CSVs.
2. Identify the currently **Active** pass and its remaining points.
3. Get from the user: list of new booking dates/times, split by old-term-leftover vs new-term.
4. **Confirm before writing anything:**
   - Instructor to use
   - Points per class × number of classes = total deduction
   - Which pass to deduct from, and resulting new balance
5. Build new history rows: `Attendance = "Upcoming"`, `Points = <per-class points>`, `Reservation Time = today`, `Pass = <active pass name>`, same `Instructor`.
6. Sort new rows descending by date, prepend to existing rows, **renumber the `#` column for the whole file sequentially**.
7. Update the Pass CSV: increase `Usage` used-count by the total deduction (e.g. `216 / 500` → `360 / 500`).
8. Output both CSVs with the **same filenames** as the originals — preserve original column order, quoting style, and encoding (UTF-8, handles Chinese names).

## Workflow: marking attendance

- User says "mark [date] as completed" → find that row by date+time in the history CSV, change `Attendance` from `Upcoming` to `Yes`. No points change.
- Don't touch the Pass CSV for attendance changes.

## Workflow: marking 待補課 (pending makeup)

- Triggered whenever the user says a student **took off / skipped / didn't attend / 請假 / 沒來** a class — this automatically means 待補課.
- Change `Attendance` from `Upcoming` (or `Yes`) → `待補課` for that class.
- In the dashboard, that row's status cell must render as a clickable **補課去** button, NOT a plain badge (see Dashboard spec below).
- When the makeup is completed, note the makeup date/time in the `Dependency` column of the original missed row (e.g. `已於 7/28 18:45 補課`).
- Do NOT write `（免費）` or similar in the note — keep it clean.
- No points change for free makeups.

---

## Dashboard (HTML) — Design Spec

Single self-contained HTML file per student. Always follow this design:

### Layout (top to bottom)
1. **Header**: Student first name only as `<h1>` (no Chinese name in title). Sub-line: 家長：[Chinese parent name] | 老師：[instructor name].
2. **Action buttons row**: Zoom classroom link + 補課登記 + 我要請假 — all inline.
3. **課卡購買 card**: Payment buttons grid (see below).
4. **Pass warning banner** (if no active pass or pass fully used).
5. **Term schedule card(s)**: one per booking batch, never merged.
6. **Download CSV button** at the bottom.
7. **Footer**: manual tracking disclaimer + updated date.

### Fonts
- Font stack: `"Times New Roman", Georgia, "PingFang TC", "Microsoft JhengHei", sans-serif`
- Times New Roman handles English; PingFang TC / JhengHei handle Chinese automatically via fallback — do NOT use Songti or any serif CJK font.

### Colors
- Background: `#F5EFE1` (light beige)
- Cards: `#fff` with `box-shadow: 0 1px 4px rgba(0,0,0,.06)`
- Zoom button: `#2D8CFF`
- 補課登記 button: `#7A5CB0` (purple)
- 我要請假 button: `#C77D3E` (orange)
- Payment buttons (points cards): `#2E7D4B` (dark green)
- Payment button (一對一課程40堂): `#F2B90C` (yellow), text `#4A3800`
- Attendance badge — Yes: green (`#E4F5E9` / `#1E7A44`)
- Attendance badge — 待補課: render as a clickable **補課去** button, NOT a plain badge. Red button (`#B02234` background, white text), label `補課去 →`, links to the makeup form `https://docs.google.com/forms/d/1WT4dLtC7vDA-d0vtCzwMAeuwIaJ9HvNEKE21Nsaa1jw/viewform` with `target="_blank"`. Wire the render logic so ANY 待補課 row becomes this button automatically.
- Makeup note text: `#B02234`, font-size 11px

### Action buttons (always present)
- 🎥 **進入教室 Enter Classroom** → student's personal Zoom link (read from `/mnt/project/ZOOM_classroom_link_for_1-1_student`)
- 📝 **補課登記** → `https://docs.google.com/forms/d/1WT4dLtC7vDA-d0vtCzwMAeuwIaJ9HvNEKE21Nsaa1jw/viewform`
- 🙋 **我要請假** → `https://forms.gle/vriQ4zbMYuq4w1N48`

### 課卡購買 payment buttons
Always include all 7, in this order, with these Stripe URLs:

| Button | URL |
|--------|-----|
| 500點 | https://book.stripe.com/aFa6oI95m61Z9I4fgk8AE06 |
| 250點 | https://book.stripe.com/bJe7sMbdu1LJ9I4fgk8AE0c |
| 200點 | https://book.stripe.com/fZucN6epGbmj5rO6JO8AE0d |
| 100點 | https://book.stripe.com/eVq9AUchy7633jGb048AE0e |
| 50點  | https://book.stripe.com/00wfZidlC61ZaM85FK8AE0f |
| 20點  | https://book.stripe.com/8x2bJ2bdubmjaM85FK8AE0g |
| 一對一課程40堂 | https://book.stripe.com/7sY6oIchyeyvbQc5FK8AE0b |

---

## Deploying to GitHub Pages

1. Read token from `/mnt/project/steps_of_build_student_booking_tracker` — never ask the user.
2. `git clone https://Jaxis-Team:<TOKEN>@github.com/Jaxis-Team/jaxis-temp-booking.git`
3. Copy dashboard as `<firstname-lowercase>.html` to repo root.
4. `git config user.email "pa@jaxis.com" && git config user.name "Peggy PA"`
5. `git add`, `commit`, `push` to `main`.
6. Live URL: `https://jaxis-team.github.io/jaxis-temp-booking/<firstname>.html`
7. GitHub Pages already enabled — no setup needed.
8. This sandbox cannot reach `*.github.io` — ask user to confirm the link loads.

---

## Notes
- Manual tracking system — not connected to any live booking platform. State this in dashboard footer.
- Each student gets their own CSV pair + dashboard file; handle one student per conversation.
- Do NOT include `（免費）` in any makeup notes — just state the date.
- Always read Zoom links from `/mnt/project/ZOOM_classroom_link_for_1-1_student` — never hardcode from memory.
