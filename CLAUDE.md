# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Repository state

There is no code yet. The only content is the requirements spec `Спецификация_требований_Сервис_бронирования_столов_v2.docx` (Russian, v2). It is the source of truth for the project, and requirements are cited by number (`FR.1`–`FR.62`). When code is added, update this file with real build, test and run commands. Do not invent them.

The `~$…docx` file is a Word lock file. Ignore it and never commit it.

## Project

A restaurant table booking web service, built as a student MVP by a 5-person team. The MVP deadline is 2026-11-25. Guests search for free tables and book them. Staff manage halls, tables, bookings and seating, and see analytics.

Out of scope for v1: payments, real SMS/e-mail (notifications are mock only), POS or calendar integration, mobile app, and multi-language support.

## Mandated stack (spec §2.4)

- Frontend: React SPA written in **JavaScript** (not TypeScript).
- Backend: .NET REST API that exchanges JSON.
- Database: PostgreSQL, with schema changes done through migrations.
- Auth: a JWT stored in an **HttpOnly cookie**, plus role checks on every request on the server. Sessions expire after 12 h. Logout invalidates the token. After 5 failed logins, login is locked for 15 min. Passwords are stored as salted hashes.
- Deployment: Docker for the backend and the DB, plus a run instructions document. Secrets come from environment variables.

## Roles

| Role | How they log in | Can do |
|---|---|---|
| Guest | Phone + password | Self-registers. While logged out, can only search and view the hall map. |
| Хостес (hostess) | E-mail + password | Works with bookings |
| Администратор (admin) | E-mail + password | Edits the directory, hall map, schedule, rules, staff accounts and audit log |
| Руководитель (manager) | E-mail + password | Everything an admin can do, plus analytics and report export |

- A guest can see and act only on their own bookings (FR.62).
- Staff accounts are tied to one restaurant.
- Hostess, admin and manager can all create a booking on behalf of a guest.

## Core invariants that span multiple layers

- **No double bookings, enforced in the DB (FR.26).** Use a constraint that rejects overlapping intervals of active bookings for the same table. In Postgres this means an exclusion constraint on `tstzrange` (btree_gist). It must account for the buffer between bookings. A check in application code alone is not enough.
  - Create, reschedule and cancel must each run inside a transaction that re-checks availability (FR.25).
  - This must hold up under a load test of 50+ concurrent booking requests.
- **Active statuses:** Ожидает подтверждения, Подтверждена, Гость пришёл.
- **Status transitions (Table 2):**
  - Pending → Confirmed
  - Pending or Confirmed → Arrived, Cancelled by guest, Cancelled by restaurant (reason required), or No-show (only after the start time)
  - Arrived → Completed
  - "Опаздывает" (late) is only a UI highlight, not a status. Only staff change statuses.
  - A walk-in creates a booking that is already in the Arrived status.
- **Time:** store everything in UTC and display it in the restaurant's own timezone, handling DST through the tz database.
  - The schedule is a weekly table (`restaurant_weekly_hours`) plus date exceptions (`restaurant_schedule_exceptions`). An exception overrides the weekly schedule for its date.
  - A booking must fit within opening hours.
- **Per-restaurant booking rules (FR.16):**

  | Rule | Default |
  |---|---|
  | Slot step | 30 min |
  | Booking duration | 2 h |
  | Buffer between bookings | 15 min |
  | How far ahead guests can book | 30 days |
  | Minimum lead time | 1 h |
  | Late grace period | 20 min |
  | Reminder | 3 h before |
  | Auto-confirm | Toggle |

- **Limits:**
  - At most 3 active bookings per phone number per day.
  - Party size is 1–20.
  - Phone numbers use the format `+7XXXXXXXXXX`.
  - Booking wishes are at most 500 chars.
- **Search fallbacks:**
  - If no table is free, suggest up to 3 earlier and 3 later free slots on the same day (FR.21).
  - If no table has enough capacity, show the restaurant phone number.
- **Deleting a table or hall** is blocked while it has future active bookings (FR.9).
- **Notifications (FR.43–45):** keep them behind a separate interface. The mock implementation writes to a notifications table and to the server log, so a real provider can be plugged in later.
- **Audit log (FR.46–48):** record every booking action and every admin change: who did it, when, the action and the object. Entries cannot be edited or deleted through the UI. Keep them for at least 1 year.
- **Personal data (152-ФЗ):**
  - Guests give consent at registration.
  - Exported reports (XLSX/CSV) must not contain guest phones or e-mails (FR.57).
  - Guest data is anonymized 1 year after the last visit.
- **Hall map (FR.13):** refreshes automatically at least every 3 s with no page reload.

## Non-functional targets

| Target | Limit |
|---|---|
| Free-table search | ≤ 1 s |
| API requests, p95 | ≤ 1 s |
| Guest booking | ≤ 4 steps and ≤ 1 min |
| Main hostess actions | ≤ 2 taps, tablet-first layout |
| Minimum screen width | 360 px |

- The UI is in Russian only.
- Dates are shown as `ДД.ММ.ГГГГ` and times as 24h `ЧЧ:ММ`.
