# Xaw House Channel Manager — Plan (VPS edition)

## Context
Xaw House (Chiang Mai, 6 rooms, each listed as its own room type with 1 unit) sells on **Agoda** and **Booking.com** with no channel manager. Today the owner by hand:
1. closes the room on the other platform after every booking
2. messages each guest in the platform inbox before 12:00 on check-in day with a self check-in template
3. adds a card to the Trello board "XAW House Bookings" (one list per room)

Goal: automate all three for the two platforms with **own software and no channel-manager subscription**, running on **one small paid VPS**. Facebook and other direct bookings stay manual.

### Decisions made
- Neither platform gives an API to non-partner properties, and Agoda has no calendar sync. So the program drives a **real Chrome on the VPS** in which the owner has logged into Agoda YCS and the Booking.com extranet. It **acts automatically and reports afterwards** in Telegram, with a screenshot.
- **VPS only**, with no Cloudflare Workers: about 2 GB RAM, x86, Ubuntu 24.04, in Singapore or Thailand (about $6–12/month).
- Alerts and control go through **Telegram**, with an alert on **every** booking.
- Stack: **TypeScript / Node.js 22 + Playwright** (best browser tooling), SQLite.
- **Accepted risks:**
  - Automated extranet use may breach platform terms.
  - Logging in from a data-centre IP brings more "verify it's you" prompts.
  - Site redesigns break the clicking code until it's fixed.
  - Every automated step falls back to a one-tap manual Telegram task.

## How it works
```
Hostel Gmail ─IMAP IDLE──► ┌──────────────── VPS (one Node service + Chrome) ─────────────────┐
                           │ mail watcher → parser → bookings (SQLite)                         │
                           │ closures engine: what should be closed vs what's confirmed closed │
                           │ browser runner (one Chrome profile, logged into both extranets)   │──► Agoda YCS / Booking.com extranet
                           │ Trello sync · Telegram bot (long polling) · scheduler             │──► Trello API, Telegram API
                           └───────────────────────────────────────────────────────────────────┘
Owner's phone/laptop ──Cloudflare Tunnel + Access (free)──► noVNC view of that Chrome (for logins and 2FA codes)
```
1. **Booking email arrives in the hostel Gmail** (new, modified or cancelled). The program reads that Gmail through IMAP with a **Gmail app password** (owner's choice) and never through the Gmail web page; it only reads and never changes or deletes mail. IMAP IDLE picks up new mail within seconds. It's parsed into one stay per room, the Trello card is created, moved or archived, and a Telegram alert is sent. If an email lacks details, the runner opens the reservation page by ID and reads them.
2. **Room closing works the same for both platforms.** The rooms that should be closed on platform P are the nights booked through the *other* platform. The engine compares that with the nights confirmed closed and makes close/open tasks for the difference. The browser runner works through the tasks one at a time:
   - open the calendar for the room
   - read the current state
   - close or open the nights
   - **re-read to verify**
   - take a screenshot
   - send Telegram: "✅ Closed Room 3 on Agoda, nights 24–25 Oct"

   After 3 failed attempts, or while logged out, the task becomes a **manual Telegram task** with a link and a Done button, reminded every 30 minutes (silent 22:00–07:00).
3. **Cancellations and changes work in both directions.**
   - An Agoda cancellation reopens the nights on Booking.com, and a Booking.com cancellation reopens them on Agoda. Each platform restores its own availability.
   - A date change closes the new nights and reopens the dropped ones. A room change does the same across both rooms.
   - Nights still covered by another active booking stay closed.
   - Telegram: "↩️ Agoda #123456 cancelled – reopened Room 3 on Booking.com, nights 24–25 Oct" plus a screenshot.
   - The Trello card gets a `CANCELLED` label and is archived.
   - A failed reopen becomes a manual task. Booking.com's "cancelled – invalid card" emails count as cancellations.
4. **Never undo the owner's own closures** (e.g. for FB bookings). A night that was already closed before the runner acted is stored as `already_closed` and is never reopened by the program. Only nights the program itself closed (`closed_by_us`) are reopened on cancellation or date change.
4. **Check-in messages:** a mode per platform, `platform` | `agent` | `manual`.
   - **Booking.com:** default `platform`, using the built-in Template scheduler ("day of arrival"), set once in the extranet. Switch to `agent` if it sends after 12:00.
   - **Agoda:** default `agent`. At 09:00 the runner opens each arriving booking in YCS, sends the filled-in template, and confirms with a screenshot. If that fails, the ready-to-paste text and a Sent button go to Telegram, with an 11:00 reminder.
   - **Agoda chat removes URLs, phone numbers and emails**, so the template gets tested there.
5. **09:00 Bangkok summary:** today's arrivals and departures, check-in message status, pending tasks, and session health.
6. **Session health:** every 30 minutes the runner loads a light extranet page, which also keeps the session alive. If logged out → Telegram "🔐 Agoda login needed" with the remote-screen link, and new tasks go manual until the login is back.
7. **Safety nets and conflict alerts:**
   - the Booking.com per-room iCal export (no login needed) is checked every 30 minutes; Booking.com nights with no matching email → alert
   - two OTA stays overlapping in a room → urgent alert
   - a new booking overlapping an existing Trello card in that room (e.g. an FB booking) → urgent alert

### Trello conventions (from the current board)
- Lists = rooms: `Room 1 | Junior Suite (Daily)` … `Room 6 | Small Double Room`. The room → list ID map comes from `scripts/trello-setup.ts`.
- Cards follow the owner's style: `Room 3 | John Smith (Agoda)` / `(Booking)`.
  - **start = check-in, due = check-out**, sorted by check-in
  - a coloured `Agoda` / `Booking.com` label
  - the description holds the ref, guests, price and an extranet link
- A modification updates the card or moves it to another room list. A cancellation adds a `CANCELLED` label and archives the card.
- The program only edits its own cards (card ID stored). It reads room lists only for the overlap check, and never touches FB cards or the due-complete tick.

## Key rules
- **Dates:** all dates are Bangkok `YYYY-MM-DD` with check-out exclusive; the service runs with `TZ=Asia/Bangkok`. Tasks and messages talk in **nights** ("close nights Fri 24 – Sat 25, check-out Sun 26").
- **Emails:**
  - Each email's IMAP UID and Message-ID are stored, so duplicates are ignored, and the raw email is saved before parsing.
  - A parse failure → Telegram alert, and `/reparse <id>` after a fix. Every failure becomes a new test fixture.
  - Events older than the stay's last update are ignored. A cancellation is final.
  - A multi-room booking becomes one stay per room. An unknown room name → urgent alert, never a silent action.
  - Only allowlisted senders are processed (Agoda and Booking.com domains, plus the owner's address for backfill forwards). Both extranets should send email in **English**.
- **Browser:**
  - Real Google Chrome is started with its own profile (`--user-data-dir`, `--remote-debugging-port` on localhost only) on a virtual display. Playwright attaches with `connectOverCDP`, so the owner sees the same window in noVNC.
  - One action runs at a time, with human-like pacing and no unnecessary page loads.
  - Page code lives in two "page object" modules (`agoda-ycs.ts`, `booking-extranet.ts`), so a redesign means fixing a single file.
  - Every action keeps a screenshot (removed after 30 days). `DRY_RUN=1` goes through all steps without saving.
- **Security:**
  - The VPS has only SSH (key-only) open; there are no inbound web ports.
  - noVNC is reached only through Cloudflare Tunnel + Access (email one-time code, owner only), with a VNC password as a second lock.
  - Secrets live in `.env` (chmod 600).
  - The Telegram bot only answers the owner's chat ID and HTML-escapes guest names.
- **Telegram:** long polling with grammY. Outgoing messages go through an outbox table with a dedupe key and retries.
- **Scheduler** (in-process): task runner on demand plus every minute; export check every 30 minutes; session check every 30 minutes; daily 09:00 / 11:00; 03:00 cleanup and SQLite backup (7 kept locally, plus weekly VPS snapshots). Emails and screenshots are deleted after 90 / 30 days.

## Project layout (new private repo)
```
package.json  tsconfig.json  vitest.config.ts  .env.example  .gitattributes (*.eml -text)
src/main.ts                       starts mail watcher, bot, scheduler, runner; graceful shutdown
src/config.ts                     rooms (Agoda/Booking.com names + IDs, Trello list IDs), times, modes, allowlist
src/db/{schema.sql,db.ts,repo.ts} better-sqlite3, migrations on start
src/mail/{imap-watcher,classify,parse-agoda,parse-booking,text}.ts      imapflow + postal-mime
src/domain/{types,apply-event,closures,dates,export-reconcile}.ts       pure logic, heavily tested
src/browser/{chrome,session,runner,agoda-ycs,booking-extranet}.ts       CDP attach, health, queue, page objects
src/integrations/{telegram,trello,booking-export}.ts
src/jobs/{scheduler,daily,cleanup}.ts
src/templates/checkin.ts          owner's template with {guest} {room} {checkout}… placeholders
deploy/setup.sh                   Ubuntu packages: Node 22, Google Chrome, Xvfb, x11vnc, noVNC, cloudflared
deploy/systemd/{xvfb,chrome,x11vnc,novnc,xaw}.service
scripts/{trello-setup,redact-eml,try-action}.ts   try-action: run one close/open in dry-run from the CLI
test/fixtures/{email/agoda,email/booking,ical,html}/   test/*.test.ts
README.md                         owner setup and run guide
```

## SQLite schema (`schema.sql`)
- `emails` — uid, message_id UNIQUE, from, subject, raw, status (received / parsed / ignored / stale / failed), attempts, error
- `bookings` — one row per OTA room-stay:
  - source (agoda / booking), ref, stay_no, status
  - room_id (nullable), check_in, check_out, guest_name, guests, price_text
  - last_event_at, version
  - trello_card_id, trello_synced_version, trello_managed, backfilled
  - checkin_msg_status
- `closures` — PRIMARY KEY (platform, room_id, night); state (closed_by_us / already_closed / manual_done); confirmed_at
- `tasks` — platform, room_id, action (close / open / message), first_night, last_night, booking_id, status (pending / running / done / manual / superseded), attempts, last_error, screenshot, tg_message_id, last_reminded_at
- `export_state` — per room: last nights seen, missing streak
- `outbox` — dedupe_key UNIQUE, payload, attempts, next_attempt_at
- `kv` — daily-job dates, session status, alert throttles

## Configuration
- **`config.ts`:** room map, check-in modes, times, quiet hours, reminder interval, allowlist.
- **`.env`:**
  - Telegram: `TELEGRAM_BOT_TOKEN`, `OWNER_CHAT_ID`
  - Trello: `TRELLO_KEY`, `TRELLO_TOKEN`, `TRELLO_BOARD_ID`
  - Email: `IMAP_HOST`, `IMAP_USER`, `IMAP_PASS` (Gmail app password)
  - Calendar: `BOOKING_EXPORT_URLS`
  - Browser: `CHROME_CDP_URL`, `DRY_RUN`, `SCREEN_URL`
- **Staging:** run locally on the Windows dev PC with `DRY_RUN=1`, a copy of the Trello board and a test Telegram bot, reading fixtures or a test mailbox.

## Phases (each ends with a check)
0. **Setup and probes.**
   - **Owner:**
     - buy the VPS and add an SSH key
     - create a Telegram bot (BotFather) and a Trello Power-Up key and token
     - turn on 2-Step Verification on the hostel Gmail and create an app password (Google Account → Security → App passwords)
     - set both extranets to English
     - collect at least 2 real `.eml` samples each of Agoda and Booking.com new / modified / cancelled
     - provide the check-in template and the Booking.com per-room iCal export links
     - check whether YCS has Manage → Automation
   - **I:** write and run `deploy/setup.sh` (Chrome, Xvfb, noVNC, cloudflared tunnel + Access policy). The owner logs into both extranets through noVNC.
   - *Check:* the sessions survive 48 h with the 30-minute health ping; the owner shows the exact clicks for closing one room-night and sending a message on each extranet, and I save HTML snapshots as fixtures; the emails contain room, dates and ref.
1. **Mail watcher, parsers, bookings, Telegram alerts.**
   - *Check:* every fixture parses to its snapshot; scenario tests pass (new → modify → cancel, out of order, duplicate, multi-room, unknown room); a test email leads to an alert.
2. **Trello sync.**
   - *Check:* on the staging board, cards are created, updated, moved and cancelled + archived; overlap with an FB card raises an alert; FB cards are never modified.
3. **Closures engine + manual Telegram tasks.** The system is fully usable even before browser automation exists.
   - *Check:* table tests for new / date change / room change / cancel before and after done / `already_closed` protection; Done buttons and reminders work, including quiet hours.
4. **Browser runner**, Booking.com first, then Agoda.
   - session check
   - close/open of a night range with re-read verification
   - screenshots
   - fallback to manual after 3 failures or a logout
   - *Check:* `try-action` dry-run on a far-future date for one room; then a live close and reopen of that far-future night on each platform, confirmed by eye in the extranet; a forced logout leads to a "login needed" alert and manual tasks.
5. **Check-in messages + daily jobs.** Set up the Booking.com Template scheduler; the Agoda agent sends at 09:00.
   - *Check:* a real test message in Agoda chat shows what gets redacted; check what time Booking.com's "day of arrival" message actually sends; simulated clock runs produce exactly one summary per day.
6. **Booking.com export safety check.**
   - *Check:* fixtures cover an unknown booking, missing nights over 2 polls, and a fetch failure.
7. **Go-live:**
   - optional backfill: forward confirmation emails for existing future bookings; these are marked `already_closed`, with no new Trello cards
   - **1–2 weeks shadow mode**: alerts tagged `[SHADOW]`, the runner in `DRY_RUN`, and the owner keeps the manual routine
   - then switch to live, one platform at a time

## Verification
- **Unit and integration tests (Vitest)** on Node:
  - parsers against redacted `.eml` fixtures
  - table tests for the pure domain modules (dates, closures, apply-event, export-reconcile)
  - Telegram and Trello clients with an injected fake `fetch`
  - SQLite in-memory per test
  - page objects tested with Playwright against saved HTML snapshots of extranet pages, to catch selector mistakes offline
- **Local runs:** `npm run dev` with `DRY_RUN=1`, a test Telegram bot and a staging Trello board; `scripts/try-action.ts` for single browser actions.
- **On the VPS:** `journalctl -u xaw -f` for logs; `/status` in Telegram shows session health, last email, last export check, pending tasks and errors. Shadow-mode reports are compared with what the owner did by hand.

## Known limits
- If either extranet changes its pages or asks for a new login, automation pauses for that platform. Tasks then become one-tap manual tasks until it's fixed or the owner logs in again.
- FB and direct bookings remain fully manual: a Trello card plus closing the room on both platforms. The program protects those closures and warns on overlaps.
