# Build Script: Dealer Acquisition MVP (Buy Cars From Private Sellers)

Version 1.0, 26 November 2026. Written to be pasted into an AI coding tool (Claude Code, Cursor, Lovable, Bolt, v0 or similar) together with the mockups exported from the "Acquisition flow mockups" canvas.

---

## 0. Instructions to the AI coding tool

Read this whole file before writing any code. Then follow these rules for the entire build:

1. Build exactly what this file describes. Do not add features, screens, fields, statuses or integrations that are not listed here. If something is ambiguous, choose the simplest option that satisfies the acceptance criteria in section 14, and write the decision into `DECISIONS.md`.
2. Match the mockups for layout, copy and order of fields. Where this file and a mockup differ, this file wins.
3. Build in the phases in section 15, in order. Finish and test each phase before starting the next.
4. There is no AI chat anywhere in this product. Sellers use forms. Staff use screens. AI is used only inside the photo check (section 8.3), and only to flag things for a person.
5. No price is ever sent to a seller automatically. A person sets every range and every final price.
6. Every change to a lead is written to the audit log (section 11). If an action does not write an audit event, it is not finished.
7. Never log, print or send personal information to any third party except the services named in section 8, and only the fields each one needs.
8. All copy is Australian English. All times are shown in the dealership time zone (default `Australia/Melbourne`) and stored in UTC.
9. Every integration in section 8 must have a working manual fallback, so the product works end to end with no paid APIs connected.

---

## 1. What this product does

A small dealership uses this to buy cars from private sellers.

1. A seller fills in a short form on their phone: rego, a few questions, six photos.
2. The system looks up the car, checks the photos and gathers a valuation.
3. A buyer at the dealership reviews everything on one screen and sets a price range.
4. The seller gets the range by SMS and books a time to bring the car in.
5. At the inspection, staff complete checks, agree a final price and both people sign on a tablet.
6. A deal pack is produced. Everything that happened is in the activity log.

**Out of scope for version 1:** AI chat, automatic offers, payments (the system only records the payment method), multiple dealerships or rooftops, wholesale or auction workflows, integration with a DMS (stock comes from a CSV upload), customer accounts or passwords for sellers.

---

## 2. Default technical stack

Use this stack unless the person running the build says otherwise. If you substitute anything, keep the same behaviour.

| Concern | Default |
|---|---|
| Framework | Next.js (App Router) with TypeScript, strict mode |
| Database | PostgreSQL (Supabase) |
| File storage | Supabase Storage, private buckets only |
| Staff authentication | Supabase Auth, email and password, with optional TOTP two-factor |
| SMS | Twilio Programmable Messaging |
| Scheduled jobs | One job endpoint called every 5 minutes by a cron (Vercel Cron or Supabase scheduled function) |
| PDF generation | Server-side HTML to PDF (for example Playwright or a PDF library) |
| Styling | Tailwind CSS using the design tokens in section 3 |
| Hosting | Vercel or similar, HTTPS only |

Environment variables (never commit values): `DATABASE_URL`, `SUPABASE_URL`, `SUPABASE_SERVICE_ROLE_KEY`, `SUPABASE_ANON_KEY`, `TWILIO_ACCOUNT_SID`, `TWILIO_AUTH_TOKEN`, `TWILIO_FROM` (number or alphanumeric sender), `APP_BASE_URL`, `CRON_SECRET`, and optional `VEHICLE_LOOKUP_API_KEY`, `VALUATION_API_KEY`, `PHOTO_CHECK_API_KEY`, `PPSR_PROVIDER_API_KEY`, `EV_SIGNAL_API_KEY`.

---

## 3. Design tokens

Match the mockups using these exact values.

| Token | Value |
|---|---|
| Font, body | Inter 400, 500, 600 |
| Font, headings | Space Grotesk 700 |
| Text | `#10151C` |
| Muted text | `#4B5563` |
| Page background (desk) | `#F6F7F9` |
| Surface | `#FFFFFF` |
| Border, light | `#E5E7EB` |
| Border, input | `#9CA3AF` |
| Accent (configurable) | `#1F5EFF` |
| Accent dark (text on tint) | `#1747C2` |
| Accent tint | `#F5F8FF` |
| Success | `#0F7B4D` |
| Warning | `#B45309`, tint `#FFF4E5`, text on tint `#92400E` |
| Radius | inputs and buttons 10px, cards 12px (phone) and 16px (desk) |
| Spacing scale | 4, 8, 12, 16, 24, 32 px only |

Sizes: phone screens are designed at 390 px wide with 24 px side padding. Inputs are 48 px tall on phone and 44 px on desk. Primary buttons are 52 px tall on phone. Every tap target is at least 44 px. Text contrast must meet WCAG 2.1 AA (4.5:1).

---

## 4. Settings

One settings record, editable by managers only (screen D3).

| Setting | Type | Default |
|---|---|---|
| `dealership_name` | text, required | "Your Dealership" |
| `dealership_phone` | text, required | none |
| `privacy_policy_url` | URL, required | none |
| `time_zone` | IANA zone | `Australia/Melbourne` |
| `response_window_hours` | integer 1 to 72 | 4 |
| `accent_colour` | hex | `#1F5EFF` |
| `booking_days` | set of weekdays | Mon, Tue, Wed, Thu, Fri, Sat |
| `booking_times` | list of HH:MM | 09:00, 10:30, 12:00, 13:30, 15:00, 16:30 |
| `booking_duration_minutes` | integer | 30 |
| `booking_capacity_per_slot` | integer | 1 |
| `sms_window_start` / `sms_window_end` | HH:MM | 08:00 / 20:00 |
| `sms_days` | set of weekdays | Mon to Sat |
| `follow_up_days` | list of integers | 1, 4, 10 |
| `follow_up_send_time` | HH:MM | 10:00 |
| `auto_lost_after_days` | integer | 14 |
| `retention_days_not_bought` | integer | 90 |
| `retention_years_bought` | integer | 7 (confirm with your state regulator) |
| `odometer_tolerance_km` | integer | 1000 |
| `odometer_tolerance_percent` | number | 2 |
| `contract_template` | rich text, required before the first purchase | none (see section 10) |

---

## 5. Data model

Use these tables and columns. Add `id uuid primary key default gen_random_uuid()`, `created_at timestamptz not null default now()` and `updated_at timestamptz not null default now()` to every table unless stated. Use Row Level Security so only authenticated staff can read or write anything, and sellers reach their own lead only through a valid token handled on the server.

### 5.1 `staff_users`
`auth_user_id` (uuid, unique), `display_name` (text, required), `email` (text, unique), `role` (enum: `buyer`, `manager`), `active` (boolean, default true).

### 5.2 `leads`
One row per seller enquiry.

| Column | Type | Rules |
|---|---|---|
| `reference` | text, unique | Format `ACQ-YYMMDD-NNN`, generated |
| `status` | enum | `draft`, `new`, `range_sent`, `booked`, `needs_person`, `bought`, `lost` (see section 6) |
| `lost_reason` | enum, nullable | `seller_declined`, `opted_out`, `no_response`, `sold_elsewhere`, `price_gap`, `failed_checks`, `not_needed`, `other` |
| `lost_note` | text, nullable | Required when `lost_reason = other` |
| `needs_person_reason` | text, nullable | Required when status is `needs_person` |
| `previous_status` | enum, nullable | Status before `needs_person`, used by "Return to queue" |
| `likely_recon` | integer dollars, nullable | Entered by staff in the "Do we need it?" panel |
| `source` | enum | `website`, `staff_entry` |
| `assigned_to` | fk `staff_users`, nullable | |
| `seller_first_name` | text | 1 to 40 characters |
| `seller_mobile` | text | Stored in E.164 (`+614XXXXXXXX`). Must be an Australian mobile |
| `sms_consent` | boolean | Must be true to submit the public form |
| `sms_consent_at` | timestamptz | |
| `sms_consent_text` | text | Exact consent sentence shown, stored verbatim |
| `sms_consent_method` | enum | `web_checkbox`, `verbal_staff_recorded` |
| `opted_out_at` | timestamptz, nullable | Set when the seller replies STOP |
| `rego` | text | Uppercase, spaces removed, 1 to 9 alphanumeric characters |
| `rego_state` | enum | `VIC`, `NSW`, `QLD`, `SA`, `WA`, `TAS`, `ACT`, `NT` |
| `vin` | text, nullable | 17 characters. Set by lookup or by staff at inspection only |
| `year` | integer | 1980 to current year plus 1 |
| `make`, `model` | text | Required |
| `variant`, `colour`, `body_type`, `transmission` | text, nullable | |
| `fuel_type` | enum | `petrol`, `diesel`, `hybrid`, `plug_in_hybrid`, `electric`, `other` |
| `vehicle_source` | enum | `lookup`, `seller_entered`, `staff_entered` |
| `odometer_stated_km` | integer | 0 to 999,999 |
| `odometer_confirmed_km` | integer, nullable | Set at inspection or by staff edit |
| `logbook_history` | enum | `full`, `partial`, `none` |
| `money_owing` | enum | `yes`, `no`, `not_sure` |
| `damage_scratches_dents`, `damage_windscreen`, `damage_warning_lights`, `damage_interior` | boolean | Default false |
| `seller_token_hash` | text | SHA-256 of the seller link token. Never store the raw token |
| `seller_token_expires_at` | timestamptz | 30 days after creation, extended by each SMS sent |
| `auto_follow_ups_enabled` | boolean | Default true |
| `follow_ups_paused_until` | timestamptz, nullable | Set by "Remind me later" |
| `last_activity_at` | timestamptz | Updated by every audit event |
| `submitted_at` | timestamptz, nullable | When the seller finished step 3 |
| `anonymised_at` | timestamptz, nullable | Set by the retention job |

### 5.3 `photos`
`lead_id` (fk), `slot` (enum: `driver_side`, `front`, `back`, `dash_odometer`, `passenger_side`, `tyres`), `storage_path` (text), `width`, `height` (integers), `uploaded_at`. Unique on (`lead_id`, `slot`); uploading again replaces the file and writes an audit event.

### 5.4 `photo_checks`
One row per lead, rerun replaces it. `status` (`pending`, `complete`, `failed`, `manual`), `plate_read` (text, nullable), `plate_result` (`matches`, `mismatch`, `unreadable`), `odometer_read_km` (integer, nullable), `odometer_result` (`close`, `check`, `unreadable`), `damage_flags` (jsonb array of `{slot, description, approx_size_cm}`), `provider` (text), `checked_at`.

### 5.5 `valuations`
`lead_id`, `market_value` (integer dollars), `comparable_sales_60d` (integer, nullable), `source` (`api` or `manual`), `provider` (text), `valued_at`, `entered_by` (fk staff, nullable). Keep history; the desk shows the newest.

### 5.6 `ev_signals`
Only for `fuel_type = electric`. `lead_id`, `value_trend_6m` (text, for example "Softening", "Stable", "Firming"), `battery_note` (text), `provider` (text, default "MotoIQ"), `retrieved_at`. Vehicle details only are sent to the provider, never seller details.

### 5.7 `ppsr_checks`
`lead_id`, `result` (`clear`, `finance_owing`, `stolen`, `written_off`, `other`), `certificate_number` (text, required), `certificate_path` (text, required, PDF in storage), `searched_at`, `recorded_by` (fk staff), `method` (`manual` or `provider_api`), `note` (text, nullable).

### 5.8 `ranges`
`lead_id`, `low` and `high` (integer dollars), `team_note` (text, nullable, never sent), `state` (`draft` or `sent`), `set_by` (fk staff), `sent_at` (nullable). Only one draft per lead at a time. Sent ranges are never edited; a new range creates a new row.

### 5.9 `bookings`
`lead_id`, `starts_at` (timestamptz), `duration_minutes`, `status` (`confirmed`, `cancelled`, `completed`, `no_show`), `cancelled_by` (`seller` or `staff`, nullable). A slot is full when confirmed bookings for that `starts_at` reach `booking_capacity_per_slot`.

### 5.10 `reminders`
`lead_id`, `kind` (`staff_task` or `seller_sms`), `title` (text, required for staff tasks), `due_at`, `assigned_to` (fk staff, nullable for seller SMS), `status` (`open`, `done`, `cancelled`), `completed_at`, `created_by` (`staff` id or `system`).

### 5.11 `messages`
Every SMS in or out. `lead_id`, `direction` (`outbound` or `inbound`), `template_key` (nullable for inbound), `body` (text), `status` (`queued`, `sent`, `delivered`, `failed`, `received`), `provider_message_id`, `scheduled_for` (timestamptz, nullable), `sent_at`.

### 5.12 `deals`
Created when staff start the offer screen. `lead_id` (unique), `check_licence_matches`, `check_ppsr_clear`, `check_vin_matches`, `check_both_keys`, `check_inspected_driven` (booleans, default false), `battery_health_percent` (integer 0 to 100, nullable, required for electric), `inspection_notes` (text), `final_price` (integer dollars), `out_of_range_reason` (text, nullable), `payment_method` (`bank_transfer_on_signing` or `bank_transfer_next_business_day`), `seller_signature_path`, `seller_signed_at`, `buyer_signature_path`, `buyer_signed_at`, `buyer_signed_by` (fk staff), `completed_at`, `deal_pack_path`.

### 5.13 `stock_items`
Replaced in full by each CSV upload (screen D4). `stock_number`, `year`, `make`, `model`, `variant`, `body_type`, `fuel_type`, `price` (integer), `status` (`in_stock` or `sold`), `days_in_stock` (integer, for in stock), `sold_date` (date, nullable), `days_to_sell` (integer, nullable), `uploaded_at`.

### 5.14 `audit_events`
Append only. See section 11.

---

## 6. Lead statuses

| Status | Meaning | Shown in tab |
|---|---|---|
| `draft` | Seller started but has not finished step 3 | Not shown (visible only via search) |
| `new` | Seller submitted, no range sent yet | New |
| `range_sent` | A range has been sent | Range sent |
| `booked` | Seller has a confirmed booking | Booked |
| `needs_person` | Staff flagged it for personal attention | Needs a person |
| `bought` | Deal completed | Bought |
| `lost` | Ended without a purchase | Lost |

Allowed transitions. Anything not listed is rejected with an error and no change.

| From | To | Trigger |
|---|---|---|
| `draft` | `new` | Seller submits step 3 with 6 photos |
| `new`, `needs_person` | `range_sent` | Staff sends a range |
| `range_sent` | `range_sent` | Staff sends a revised range |
| `range_sent` | `booked` | Seller or staff confirms a booking |
| `booked` | `range_sent` | Booking cancelled by seller or staff |
| `new`, `range_sent`, `booked` | `needs_person` | "Hand to a person" with a reason |
| `needs_person` | previous status | "Return to queue" (stored previous status) |
| `booked` | `bought` | "Complete purchase" succeeds |
| any except `bought` | `lost` | "Mark as lost" with a reason, seller "No thanks", STOP reply, or auto rule |
| `lost` | `new` | Manager only, "Reopen", requires a note |

Side effects:
- Entering `needs_person`, `bought` or `lost` stops all automatic follow-ups and cancels queued follow-up SMS.
- STOP from the seller sets `opted_out_at`, cancels every queued outbound SMS and sets status `lost` with reason `opted_out` unless the status is `booked` or `bought`, in which case status is unchanged and staff are shown a warning banner "Seller opted out of SMS. Contact by phone only."

---

## 7. Screens

Routes use these paths. Copy in quotes is exact.

### 7.1 Seller screens (public, phone first)

All seller screens share:
- Header: dealership name on the left, step label on the right, 3-segment progress bar (screens S1 to S3).
- Footer link: "Prefer to talk? Call [dealership_phone]" as a `tel:` link.
- Session: on first load of S1, create a `draft` lead and set an HttpOnly, Secure, SameSite=Lax cookie holding the seller token (32 random bytes, URL-safe base64). The same token is used in SMS links as `/s/{token}`.
- Rate limit: 10 form submissions per IP per hour.

**S1: Rego lookup** (`/sell`) — mockup "Seller 1"
- Fields: "Number plate" (text, uppercase on blur, spaces removed on save), "State" (select, default from settings time zone state).
- On valid plate and state, call the vehicle lookup (section 8.1). While waiting show "Looking up your car…" for up to 8 seconds.
- Found: show the "We found this car" card with year, make, model, variant, colour. Buttons "Yes, that is it" and "Not my car".
- "Not my car" or lookup not found or timed out: show manual fields "Year", "Make", "Model", "Variant (optional)" and "Fuel type". Set `vehicle_source = seller_entered`.
- "Continue" is disabled until a car is confirmed or the manual fields are valid.
- Audit: `lead.created`, `vehicle.lookup_succeeded` or `vehicle.lookup_failed`, `vehicle.confirmed_by_seller` or `vehicle.entered_by_seller`.

**S2: A few questions** (`/sell/details`) — mockup "Seller 2"
- "Kilometres on the odometer": integer, required, digits only, displayed with thousands separators.
- "Logbook service history?": Full, Partial, None. Required.
- "Any money owing on it?": Yes, No, Not sure. Required.
- "Anything we should know? Tick any that apply.": four checkboxes, in this order, mapped to these columns: "Scratches or dents" (`damage_scratches_dents`), "Windscreen chips or cracks" (`damage_windscreen`), "Warning lights on the dash" (`damage_warning_lights`), "Interior wear or smells" (`damage_interior`). None ticked is valid.
- A divider, then the heading "Where should we send your range?"
- "First name": required, 1 to 40 characters.
- "Mobile": required, Australian mobile, normalised to E.164.
- Consent checkbox, unticked by default, required: "I am happy for [dealership_name] to contact me by SMS about selling this car. Reply STOP to opt out at any time." Store the exact rendered sentence in `sms_consent_text`.
- Privacy line with link to `privacy_policy_url`.
- "Next: photos" saves and goes to S3. Errors show beside the field in text, not colour alone.
- Audit: `lead.details_saved`, `consent.given`.

**S3: Six quick photos** (`/sell/photos`) — mockup "Seller 3"
- Six tiles in this order: Driver's side, Front (plate showing), Back (plate showing), Dash and odometer, Passenger side, Tyre tread.
- Primary button opens the camera (`<input type="file" accept="image/*" capture="environment">`) for the next empty slot, labelled "Take photo: [slot label]", where slot labels are "driver's side", "front of the car", "back of the car", "dash and odometer", "passenger side", "tyre tread". The next empty tile has an accent border. Secondary "Upload from my gallery" opens the picker without capture.
- Tapping a done tile lets the seller retake it.
- Accept JPEG, PNG, HEIC and WebP up to 15 MB. On the server: convert to JPEG, resize so the long edge is at most 2000 px, and **strip all EXIF metadata, including GPS location**.
- Show "[n] of 6 done". When all 6 are uploaded, the primary button becomes "Send to our buyer".
- On submit: status `draft` to `new`, set `submitted_at`, run the photo check, the valuation and (for electric cars) the EV signal in the background, send SMS `ack_received`, then go to S4.
- Audit: `photo.uploaded` (per slot), `lead.submitted`.

**S4: Done** (`/sell/done`) — mockup "Seller 4"
- Heading "Thanks, [first name]. Our buyer is on it."
- Text "You will get a price range by SMS within [response_window_hours] hours. It is based on your photos, and the final price is confirmed when we see the car."
- Card "What happens next" with three numbered steps: "Our buyer reviews your car against today's market." / "You get a range and a link to book a time to bring it in." / "Bring your rego papers, both keys and your licence. If we agree a price, you can sign on the spot."
- Small print "No obligation. Reply STOP to any message and we will not contact you again."
- Footer "Questions in the meantime? Call [dealership_phone]". No other buttons.

**S5: Book a time** (`/s/{token}`) — mockup "Seller 5"
- Valid token and status `range_sent`: show the latest sent range card, "Pick a day" (next 4 dates that are booking days and have at least one free slot), "Pick a time on [weekday] (about [booking_duration_minutes] minutes)" (booking times, full or past slots disabled and struck through, nothing preselected), the "Please bring" box, and buttons:
  - "Confirm [Day date], [time]": creates the booking, status to `booked`, sends SMS `booking_confirmed`, schedules SMS `booking_reminder` for 17:00 local the day before (or immediately if that time has passed and the booking is more than 2 hours away).
  - "Remind me later": sets `follow_ups_paused_until` to now plus 7 days, cancels queued follow-ups, queues SMS `seller_requested_reminder` for that date at `follow_up_send_time`, shows "No problem. We will send you a reminder on [date]."
  - "No thanks": confirmation dialog "Are you sure? We will not contact you about this car again." On confirm, status `lost` with reason `seller_declined`, cancel queued SMS, show "Thanks for letting us know."
- Status `booked`: show the booking with "Change time" (returns to slot picking and cancels the old booking on confirm) and "Cancel booking" (status back to `range_sent`).
- Invalid or expired token, or status `bought` or `lost`: show "This link has expired. Please call [dealership_phone]."
- Audit: `booking.created`, `booking.changed`, `booking.cancelled`, `seller.remind_later`, `seller.declined`.

### 7.2 Staff screens (authenticated)

**D1: Buying desk** (`/desk`) — mockup "Buyer desk"

Header:
- "Buying desk" title. A "Sample data" badge appears only when the seed data from section 13 is loaded.
- Search box "Search rego or name": matches rego (normalised), seller first name or reference, across all statuses, results replace the list.
- "Add seller": opens a form with the S1 and S2 fields for phone or walk-in leads. Consent is recorded with the checkbox "Seller agreed to SMS contact during this call" and stored with `sms_consent_method = verbal_staff_recorded`. Photos are optional for staff-entered leads. The lead goes straight to `new` with `source = staff_entry`.
- "Signed in as [name]" with a menu holding "Settings" (managers only), "Upload stock list" and "Sign out".

Status tabs: New, Range sent, Booked, Needs a person, Bought, Lost, each with a live count. Default tab is New.

Seller list (left column): cards sorted newest first by `submitted_at` (Booked tab sorts by booking time, soonest first). Each card shows year make model, "[first name], [odometer] km, [relative time]" and one status line, chosen in this order: "Opted out of SMS" (warning), "[n] thing(s) to check" (warning) when the photo check has any `check` or `mismatch` result or damage flags, "Waiting on [n] photos" for staff entries without photos, "Reminder due [date]" if an open reminder exists, else "All checks clear" (success). Selecting a card opens the record on the right. Selected card has a 2 px accent border.

Record header:
- Title, status badge, and "[STATE] [rego]. [first name], via [source], [date] at [time]. Assigned to [name]." An assignee select appears on click of the name.
- Buttons: "Set reminder", "Hand to a person" (or "Return to queue" when status is `needs_person`), "Mark as lost". When status is `booked`, show a primary button "Start offer and sign" that opens O1. When status is `lost`, managers see "Reopen".
- "Hand to a person" opens a dialog with a required reason. "Mark as lost" opens a dialog with the lost reason select and a note field (required for Other).

Photos row: the six slots in order. A slot with a damage flag has a warning border and a "Check" tag. Clicking opens a full-size viewer with the flag description, and left and right arrows. Missing slots show "Not provided".

Checks panel: rows exactly as the mockup, with results in a fixed second column so values line up:
- "Plate matches rego": Matches (success), Mismatch (warning), Unreadable (muted).
- "Odometer in photo [n] km": Close (success) when the difference from the stated reading is within `odometer_tolerance_km` or `odometer_tolerance_percent`, whichever is larger; otherwise Check (warning).
- One row per damage flag: "Possible [description], about [n] cm": Check (warning).
- "PPSR (finance, stolen, written off)": Not run (muted), Clear (success), or the problem in warning colour.
- Buttons "Run PPSR" and "Record result" (section 8.4). If the photo check failed, show "Photo check unavailable. Review the photos yourself." and a "Retry" button.

Value panel: "Market value" (newest valuation, or "Add value" button opening a manual entry dialog with amount and source text), "Valued" (date), "Comparable sales, 60 days". For electric cars, the "EV signal from MotoIQ" box with value trend and battery note; hidden for other fuel types.

"Do we need it?" panel, calculated from `stock_items` where make and model match and fuel type matches: "Similar on the lot" (count in stock), "Days in stock, similar" (average, rounded), "Sold, last 90 days" (count), "Average days to sell" (rounded), "Likely recon" ("$[n]" from a manual field staff can edit, or "Not set"). Footer: "From the stock list, updated [date, time]". With no stock list uploaded: "Upload a stock list to see this" with a link to D4.

Seller and car details panel:
- Read view by default with an "Edit" button. Edit view as the mockup: First name, Mobile, Rego and state, Odometer (km), Logbook history, Money owing, plus fuel type and damage checkboxes.
- VIN is shown read only with "From rego lookup, not editable" (or "Added at inspection").
- SMS consent is shown read only: "Given [date, time]. Only the seller can change it." or "Opted out [date]".
- "Cancel" discards changes. "Save changes" validates with the same rules as S1 and S2, saves, and writes one audit event per changed field with before and after values. Changing rego or state reruns the vehicle lookup and asks staff to confirm the result. Changing the odometer here sets `odometer_confirmed_km` and reruns the odometer comparison.

Set the range panel:
- "From" and "To": whole dollars, required, From greater than 0, To greater than or equal to From. If To is more than 20 percent above From, show a warning "That is a wide range. Sellers tend to anchor on the top number." but allow sending.
- "Note for the team (not sent)".
- "SMS preview" shows the exact `range_sent` text with values filled in, updating as staff type.
- "Save draft" saves without sending. "Send range to [first name]" is disabled with an explanation when the seller has opted out, when consent is missing, or when the status does not allow it. Sending: creates a sent range, sends SMS `range_sent`, status to `range_sent`, schedules automatic follow-ups (section 9).
- Previously sent ranges are listed below as "Sent [date]: $[low] to $[high] by [name]".

Reminders panel:
- "Add reminder": dialog with title (required), due date and time (required, in the future) and assignee (default current user).
- Each open reminder shows title, "[day date], [time]. [assignee]." and buttons "Edit" and "Done". Done reminders are hidden behind "Show completed".
- "Automatic follow-ups" row with an On or Off switch, the follow-up schedule in words, and the stop conditions. Turning it off cancels queued follow-ups; turning it on reschedules from now.
- A read-only row "Delete photos and details if not bought", "Automatic, [date] ([retention_days_not_bought] days)", calculated from `last_activity_at`.
- A badge with the number of reminders due today appears in the header next to the user name, and due reminders are listed at the top of the seller list.

Activity log panel:
- Table with columns "Time", "Who", "What happened", newest first, 20 rows with "Show more".
- "Who" is the staff display name, "Seller" or "System".
- "Export CSV" downloads every event for this lead (section 11.3).
- Managers also get "Export all activity" in Settings for a date range.

**O1: Offer and sign** (`/desk/leads/{id}/offer`), tablet landscape — mockup "Offer and sign"
- Opens only for status `booked`. Creates the `deals` row on first open. "Back to buying desk" returns without losing data (all fields autosave on blur).
- Two cards side by side: "1. Inspection checks" with a counter "[n] of 5 done", and "2. Offer and signatures".
- Inspection checks, five checkboxes in this order, mapped to these columns: "Seller's licence matches the name on the rego papers" (`check_licence_matches`), "PPSR certificate saved, no finance, not stolen or written off" (`check_ppsr_clear`), "VIN on the car matches the rego papers" (`check_vin_matches`), "Both keys received" (`check_both_keys`), "Car inspected and test driven" (`check_inspected_driven`). Each tick and untick records who and when. "PPSR certificate saved…" can only be ticked when a `ppsr_checks` row exists with result `clear`; otherwise it is disabled with "Record a clear PPSR result first".
- "Odometer confirmed (km)": required. "Battery health (EVs only)": shown and required only when fuel type is electric, 0 to 100. "Inspection notes": optional. Ask for the VIN here if it is missing; a VIN is required to complete.
- "Final price": required whole dollars. If outside the latest sent range, show "Outside the range sent ($[low] to $[high]). Add a reason." and require `out_of_range_reason`.
- "Payment": select with the two options in section 5.12.
- Two signature boxes captured with a canvas (pointer events, touch friendly). Seller signs first. "Clear" resets a signature that has not been confirmed. Each signature is saved as a PNG with the signer name and time. The buyer signature records the signed-in staff user.
- "Save and finish later" saves and returns to the desk.
- "Complete purchase" is disabled until all five checks are ticked, every required field is valid and both signatures exist. On click: confirmation dialog "Complete the purchase of [car] for $[price]?". On confirm: set `completed_at`, status to `bought`, generate the deal pack (section 10), cancel all queued SMS and open reminders, send SMS `purchase_complete`, show the deal pack link.

**D3: Settings** (`/desk/settings`, managers): every field in section 4, staff user list (invite by email, set role, deactivate), and "Export all activity".

**D4: Upload stock list** (`/desk/stock`): CSV upload with the header row `stock_number,year,make,model,variant,body_type,fuel_type,price,status,days_in_stock,sold_date,days_to_sell`. Validate every row, reject the whole file if any required column is missing, show a preview of the first 10 rows and the row count, then "Replace stock list". Provide a downloadable template file with this exact header and two example rows.

---

## 8. Integrations

Every integration is an adapter behind an interface. Ship the manual adapter first; the API adapter is enabled only when its environment variable is set. All calls time out after 10 seconds and failures never block the seller.

### 8.1 Vehicle lookup
`lookupVehicle(rego, state) -> { year, make, model, variant, colour, body_type, transmission, fuel_type, vin } | null`
- API adapter: a licensed Australian vehicle data provider (for example AutoGrab). Send rego and state only.
- Manual adapter: returns null, so the seller enters details.

### 8.2 Valuation
`getValuation(vehicle, odometer_km) -> { market_value, comparable_sales_60d, provider } | null`
- API adapter: a valuation provider (for example AutoGrab). Send vehicle details and odometer only.
- Manual adapter: staff enter a value with "Add value".

### 8.3 Photo check (the only AI in the product)
`checkPhotos(photos, rego, odometer_stated_km) -> photo_checks row`
- API adapter: a vision-capable AI model on a business plan whose terms exclude training on your data. Send only the six images and the rego to compare against. Prompt it to return strict JSON: `plate_read`, `odometer_read_km`, `damage_flags[]` with slot, short description and approximate size in centimetres. Validate the JSON against a schema; on any failure set status `failed`.
- Compute `plate_result` and `odometer_result` in your own code from the returned readings. The model never decides a result.
- Every output is labelled "possible" in the interface. Nothing from this check is ever shown to the seller.
- Manual adapter: status `manual`, staff review the photos themselves.

### 8.4 PPSR
- Manual adapter (default): "Run PPSR" copies the VIN to the clipboard and opens `https://www.ppsr.gov.au` in a new tab. "Record result" opens a dialog: result (required), certificate number (required), certificate PDF upload (required, PDF up to 5 MB), note. If the VIN is unknown, the dialog asks for it first.
- Provider adapter (optional): a PPSR reseller API that returns the result and certificate. Store the same fields with `method = provider_api`.
- Any result other than `clear` sets the Checks row to warning, adds a banner "PPSR shows [result]. Do not buy until this is resolved." and blocks the PPSR checkbox in O1.

### 8.5 EV signal
`getEvSignal(vehicle, odometer_km) -> { value_trend_6m, battery_note } | null`, electric vehicles only, provider MotoIQ. Send vehicle details and odometer only. Manual adapter hides the box.

### 8.6 SMS
- Twilio. Outbound messages are queued in `messages` and sent by the job runner (section 9) only inside the SMS window and days. Messages created outside the window are scheduled for the next window start. `booking_reminder` and `booking_confirmed` are exempt from the day rule but not the hour rule.
- Inbound webhook verifies the Twilio signature, matches the lead by mobile number (most recent open lead), stores the message, writes an audit event, and if the body is STOP, STOP ALL, UNSUBSCRIBE, CANCEL, END or QUIT (case insensitive, trimmed) applies the opt-out in section 6. Any other inbound message stops automatic follow-ups for that lead, creates a staff reminder "Reply from [first name]: [first 80 characters]" due now, and shows the message in the activity log.
- Every outbound message ends with "[dealership_name]. Reply STOP to opt out." except the opt-out confirmation.

---

## 9. Job runner, reminders and follow-ups

A single endpoint `POST /api/jobs/run`, protected by `CRON_SECRET`, called every 5 minutes. Each run, in this order, and safely repeatable (use row locks so two runs never send the same message twice):

1. Send queued outbound SMS whose `scheduled_for` is due and inside the window.
2. Automatic follow-ups: for each lead with status `range_sent`, `auto_follow_ups_enabled = true`, not opted out, not paused, and no inbound message since the range was sent, queue `follow_up_1`, `follow_up_2` and `follow_up_3` on the days in `follow_up_days` after `sent_at`, at `follow_up_send_time`. Never queue the same template twice for the same range.
3. Auto lost: leads in `range_sent` with no inbound message, no booking and `sent_at` older than `auto_lost_after_days` move to `lost` with reason `no_response`.
4. Booking reminders and no-shows: bookings starting within 24 hours have `booking_reminder` queued if not already; confirmed bookings more than 2 hours in the past with no deal started create a staff reminder "Did [first name] arrive?" with buttons in the desk to mark Completed or No show.
5. Staff reminders: reminders that became due since the last run are surfaced in the desk badge. (Email notification is optional and off by default.)
6. Retention (section 12.3).

Every action the job takes writes an audit event with actor `System`.

### 9.1 SMS templates

Use these exactly. Square-bracket values are filled at send time. `[car]` is "[year] [make] [model]", `[short car]` is "[make] [model]" (for example "Tesla Model 3"), `[link]` is `APP_BASE_URL/s/[token]`, `[sign-off]` is "[dealership_name]. Reply STOP to opt out." Keep every message under 320 characters after filling; if it would be longer, shorten `[car]` to `[short car]`.

| Key | Sent when | Text |
|---|---|---|
| `ack_received` | Seller submits S3 | "Hi [first name], thanks for sending through your [short car]. Our buyer will text you a price range within [response_window_hours] hours. [sign-off]" |
| `range_sent` | Staff sends a range | "Hi [first name], thanks for sending through your [short car]. Our buyer [staff first name] has looked at the photos and we would be looking at $[low] to $[high], subject to seeing the car. Book a time: [link]. [sign-off]" |
| `follow_up_1` | Day 1 | "Hi [first name], just checking you got our range of $[low] to $[high] for your [short car]. Book a time that suits: [link]. [sign-off]" |
| `follow_up_2` | Day 4 | "Hi [first name], our range for your [short car] still stands, subject to seeing the car. Happy to answer any questions on [dealership_phone], or book here: [link]. [sign-off]" |
| `follow_up_3` | Day 10 | "Hi [first name], last message from us about your [short car]. If you would still like to sell, book here: [link] or call [dealership_phone]. [sign-off]" |
| `booking_confirmed` | Booking created or changed | "Booked: [weekday] [date] at [time] at [dealership_name]. Please bring your rego papers, both keys and your licence. Change or cancel: [link]. [sign-off]" |
| `booking_reminder` | 17:00 the day before | "Reminder: we will see you [weekday] at [time] with your [short car]. Bring your rego papers, both keys and your licence. Change or cancel: [link]. [sign-off]" |
| `seller_requested_reminder` | Date chosen by "Remind me later" | "Hi [first name], here is the reminder you asked for. Our range for your [short car] was $[low] to $[high], subject to seeing the car. Book a time: [link]. [sign-off]" |
| `booking_cancelled_by_staff` | Staff cancels a booking | "Hi [first name], we need to move your appointment on [weekday] at [time]. Please pick a new time: [link] or call [dealership_phone]. [sign-off]" |
| `purchase_complete` | Deal completed | "Thanks [first name], the purchase of your [short car] is complete. Payment: [payment method in words]. Questions? Call [dealership_phone]. [dealership_name]." |
| `opt_out_confirmation` | STOP received | "You have been unsubscribed and will not receive more messages from [dealership_name] about this car." |

The SMS preview in the desk renders the `range_sent` template with the same function used to send it, so the preview and the message can never differ.

---

## 10. Deal pack

Generated as one PDF on "Complete purchase" and stored privately. Contents, in order:
1. Cover: dealership name, reference, car, rego, VIN, final price, payment method, completion date and time.
2. Contract of sale, filled from the deal and lead. **The contract wording is supplied by the dealership.** Store it as a template in Settings with merge fields `{{seller_first_name}}`, `{{car}}`, `{{rego}}`, `{{vin}}`, `{{odometer}}`, `{{final_price}}`, `{{payment_method}}`, `{{date}}`. Do not write contract terms yourself. If no template is set, block "Complete purchase" with "Add your contract template in Settings first."
3. Both signatures with names and timestamps.
4. Inspection checks with who ticked each and when.
5. The PPSR certificate (attached PDF pages).
6. The six photos and any damage flags.
7. The full activity log for the lead.

The PDF footer on every page: reference, "Generated [date, time]", page "n of N".

---

## 11. Audit log

### 11.1 Rules
- Table `audit_events` is append only. The application database role has INSERT and SELECT only; no UPDATE or DELETE grants. The only exception is the retention job, which runs as a separate role allowed to UPDATE the `before`, `after` and `summary` columns to redact personal information.
- Every write to a lead, its photos, checks, valuations, ranges, bookings, reminders, messages or deal happens in the same database transaction as its audit event. If the audit insert fails, the change fails.

### 11.2 Columns
`id`, `lead_id` (nullable for settings and user events), `occurred_at` (timestamptz, UTC), `actor_type` (`seller`, `staff`, `system`), `actor_id` (staff user id, nullable), `actor_label` (display name at the time, "Seller" or "System"), `event_type` (text, from the list below), `summary` (one plain sentence for the desk, for example "Changed odometer from 48,200 to 48,410"), `before` (jsonb, nullable), `after` (jsonb, nullable), `ip_hash` (SHA-256 of IP with a server secret, seller events only).

Event types: `lead.created`, `lead.details_saved`, `lead.submitted`, `lead.field_changed`, `lead.assigned`, `lead.status_changed`, `lead.handed_to_person`, `lead.returned_to_queue`, `lead.marked_lost`, `lead.reopened`, `lead.anonymised`, `consent.given`, `consent.opted_out`, `vehicle.lookup_succeeded`, `vehicle.lookup_failed`, `vehicle.confirmed_by_seller`, `vehicle.entered_by_seller`, `photo.uploaded`, `photo.replaced`, `photo.check_completed`, `photo.check_failed`, `valuation.retrieved`, `valuation.entered`, `ev_signal.retrieved`, `ppsr.recorded`, `range.draft_saved`, `range.sent`, `sms.queued`, `sms.sent`, `sms.failed`, `sms.received`, `sms.cancelled`, `follow_ups.enabled`, `follow_ups.disabled`, `seller.remind_later`, `seller.declined`, `booking.created`, `booking.changed`, `booking.cancelled`, `booking.completed`, `booking.no_show`, `reminder.created`, `reminder.edited`, `reminder.completed`, `deal.started`, `deal.field_changed`, `deal.check_ticked`, `deal.check_unticked`, `deal.signed_seller`, `deal.signed_buyer`, `deal.signature_cleared`, `deal.completed`, `deal_pack.generated`, `export.csv_downloaded`, `stock.uploaded`, `settings.changed`, `staff.invited`, `staff.role_changed`, `staff.deactivated`.

### 11.3 CSV export
Columns, in order: `occurred_at_local`, `actor`, `event_type`, `summary`, `before`, `after`. UTF-8 with header row. File name `activity-[reference]-[YYYYMMDD].csv`. Exporting writes `export.csv_downloaded`.

---

## 12. Privacy and security

### 12.1 Data minimisation
- Collect only the fields in section 5. Do not collect surnames, email addresses, home addresses, licence numbers, dates of birth or bank details anywhere in the product. Licence and rego papers are checked in person at inspection; only the tick and the staff member are recorded.
- Third parties receive only what section 8 lists. Never send seller name or mobile to any provider except Twilio.

### 12.2 Security
- HTTPS only, HSTS on. Secure, HttpOnly cookies. CSRF protection on all state-changing requests.
- Storage buckets are private. Images and PDFs are served through signed URLs that expire after 10 minutes, generated only for authenticated staff, or for the seller's own photos during S3.
- Seller tokens: stored hashed, expire as in section 5.2, single lead only.
- Staff: minimum 12-character passwords, lock after 10 failed attempts for 15 minutes, optional TOTP. Managers can deactivate staff immediately.
- Never write personal information into application logs, error tracking or analytics. Use the lead reference instead.
- Validate every input on the server, not just in the browser.

### 12.3 Retention (job step 6)
- Leads not `bought` with `last_activity_at` older than `retention_days_not_bought`: delete all photos and the PPSR certificate from storage, set `seller_first_name` to "Removed", `seller_mobile` to null, clear message bodies, redact personal values in that lead's audit `before` and `after` fields and summaries, set `anonymised_at`, write `lead.anonymised`. Keep vehicle details and status for reporting.
- `bought` leads are kept for `retention_years_bought`, then anonymised the same way except the deal pack, which managers must export or delete manually; the job creates a manager reminder 30 days before.

### 12.4 Before going live
Show this checklist on the Settings screen until a manager ticks each item:
- "Privacy policy updated to cover this form, SMS and photo checks"
- "Contract template reviewed by our lawyer or industry association"
- "AI photo check provider is on a business plan that does not train on our data" (or photo check set to manual)
- "SMS sender and opt-out tested"
- "Staff trained: a person sets every range and price"

---

## 13. Seed data

Provide `npm run seed` that creates, clearly marked as sample data:
- Settings with dealership name "Sample Motors" and phone "03 0000 0000".
- Two staff users: Jordan (buyer) and Alex (manager), with passwords printed once in the terminal.
- Leads matching the mockups: 2021 Tesla Model 3 Long Range (Sam, new, scratch flag), 2019 Toyota Hilux SR5 (Priya, new, all clear), 2020 Mazda CX-5 Maxx Sport (Tom, staff entry, waiting on 2 photos), plus 5 range sent, 2 booked, 1 needs a person, 8 bought and 4 lost, with realistic audit events.
- Placeholder photos (plain coloured images with the slot name), never real number plates.
- A stock list of 30 cars including one similar Model 3.
- Seed mobiles use the reserved test range `+61491570xxx` so no real person is ever messaged.

When seed data exists, the desk shows the "Sample data" badge and outbound SMS are written to the log but never sent.

---

## 14. Acceptance criteria

The build is complete only when every item passes. Write an automated test for each item marked (T).

Seller flow
1. A seller can go from S1 to S4 on a 390 px wide phone with no horizontal scrolling. (T)
2. With no vehicle lookup configured, the seller can enter the car manually and finish. (T)
3. The consent box is unticked by default and the form cannot be submitted without it. (T)
4. Uploaded photos have no EXIF or GPS data when downloaded from storage. (T)
5. S4 shows the configured response window in hours.
6. A range SMS link opens S5 with the latest sent range, and a full slot cannot be booked. (T)
7. "Remind me later" pauses follow-ups for 7 days and queues one reminder SMS. (T)
8. "No thanks" sets status lost with reason seller_declined and cancels queued SMS. (T)
9. An expired or unknown token shows the expired message and no lead data. (T)

Desk
10. Each status tab count matches the number of leads in that status. (T)
11. Search finds a lead by rego with or without spaces, and by first name. (T)
12. Values in the Checks, Value and Do we need it panels line up in a single column.
13. Editing a field and saving writes one `lead.field_changed` event per field with before and after. (T)
14. "Cancel" in edit mode discards all changes and writes no event. (T)
15. A range cannot be sent to an opted-out seller, and the button explains why. (T)
16. The SMS preview text exactly matches the SMS that is sent. (T)
17. Follow-ups are sent on days 1, 4 and 10 at the configured time, never outside the SMS window, and stop after any inbound reply, booking or opt-out. (T)
18. A lead with no response is moved to lost after `auto_lost_after_days`. (T)
19. Reminders can be added, edited and marked done, and due reminders show in the header badge. (T)
20. The activity log shows every action from this list in order, and the CSV export contains the same events. (T)
21. No UPDATE or DELETE on `audit_events` succeeds from the application role. (T)
22. The Do we need it panel calculates from the uploaded stock list and shows the upload time. (T)
23. With every optional integration switched off, a lead can go from seller form to bought. (T)

Offer and sign
24. O1 cannot be opened unless the lead is booked. (T)
25. The PPSR check cannot be ticked without a recorded clear PPSR result. (T)
26. Battery health is shown and required only for electric vehicles. (T)
27. A final price outside the sent range requires a reason. (T)
28. "Complete purchase" stays disabled until all checks, required fields and both signatures are done. (T)
29. Completing a purchase sets status bought, cancels queued SMS and open reminders, and produces a deal pack PDF with every section in section 10. (T)

Privacy and security
30. STOP, in any capitalisation, opts the seller out within one job run and sends no further messages. (T)
31. Leads past the retention period lose their photos and personal details and keep their audit events in redacted form. (T)
32. No personal information appears in server logs during a full end-to-end test run.
33. Every page meets WCAG 2.1 AA contrast and can be completed with a keyboard.

---

## 15. Build order

Finish each phase, run its tests and update `DECISIONS.md` before moving on.

1. **Foundations:** project setup, design tokens, database schema with migrations, Row Level Security, audit log helper used by every write, staff authentication, settings screen.
2. **Seller flow:** S1 to S4 with manual vehicle entry, photo upload with EXIF stripping, seller token, consent capture.
3. **Desk basics:** status tabs, search, seller list, record header, photos, details read and edit with save and cancel, activity log with CSV export, Add seller.
4. **Range and SMS:** range panel with preview, Twilio send and inbound webhook, STOP handling, job runner with the SMS window.
5. **Booking and reminders:** S5, booking slots, reminders panel, automatic follow-ups, auto lost, no-show prompt.
6. **Checks and value:** stock CSV upload and Do we need it panel, manual valuation, manual PPSR with certificate upload, then each optional API adapter behind its environment variable, then the AI photo check.
7. **Offer and sign:** O1, signature capture, contract template, deal pack PDF.
8. **Privacy and hardening:** retention job, go-live checklist, rate limits, accessibility pass, full acceptance test run with seed data.

---

## 16. Never do these

- Never send a price, range or offer to a seller that a staff member did not set.
- Never let the AI photo check change a status, a price or anything the seller sees.
- Never message a seller who has not given consent or who has opted out.
- Never store or send a seller's surname, email, address, licence number, date of birth or bank details.
- Never keep GPS or other metadata in uploaded photos.
- Never edit or delete audit events, except the retention redaction.
- Never write the contract terms yourself.
- Never use real people's details or real number plates in seed or test data.
