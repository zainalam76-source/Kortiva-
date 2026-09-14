# Kortiva

A multi-tenant booking & operations platform for padel facilities — an embeddable booking widget for each facility's own website, a full admin panel for staff/owners, and a portable player identity with a verified match history as a byproduct of real bookings.

Built to the spec in `../BUILD_INSTRUCTIONS.md`. Every Launch-Critical acceptance criterion in that document (Section 10) has been implemented and tested against a live database — see "What's been verified" below.

## Project structure

```
kortiva/
  server/    Node.js + Express + TypeScript + PostgreSQL (Prisma) API
  admin/     React + Vite admin panel (facility staff + platform admin)
  widget/    React + Vite embeddable booking widget (players)
  docs/      OPEN_QUESTIONS.md, DEPLOYMENT.md
  .github/workflows/ci.yml   Type-check + test + build all three apps on every push/PR
```

Deploying? See [docs/DEPLOYMENT.md](docs/DEPLOYMENT.md) — a self-hosted Docker path (`docker-compose.prod.yml`, Dockerfiles + nginx configs for all three apps, all verified locally) and a managed-platform path (Railway/Render/Fly).

## Quick start

### 1. Database

You need a local PostgreSQL instance. Two options:

**Docker (recommended if you have it):**
```bash
docker compose up -d
```

**No Docker?** Run a portable Postgres with no install/admin rights required — download the
[EDB Windows binaries zip](https://www.enterprisedb.com/download-postgresql-binaries), extract it,
then:
```bash
cd server
./.pgportable/pgsql/bin/initdb.exe -D ./.pgportable/data -U kortiva --pwfile=<(echo kortiva)
./.pgportable/pgsql/bin/pg_ctl.exe -D ./.pgportable/data -l ./.pgportable/logfile.txt start
./.pgportable/pgsql/bin/createdb.exe -U kortiva -h localhost kortiva
```

### 2. API server

```bash
cd server
cp .env.example .env      # adjust DATABASE_URL if needed
openssl rand -hex 32      # paste the output into CONNECTOR_ENCRYPTION_KEY in .env — needed for facilities to connect their own WhatsApp/email (Integrations page); everything else works without it
npm install
npx prisma migrate dev    # creates all tables
npm run seed               # demo facility, staff, courts, player, discount codes
npm run dev                 # http://localhost:4000
```

Seed credentials:
| Role | Email | Password |
|---|---|---|
| Platform Admin | admin@kortiva.app | ChangeMe123! |
| Facility Owner | owner@dhapadel.example | Owner123! |
| Manager | manager@dhapadel.example | Manager123! |
| Staff | staff@dhapadel.example | Staff123! |
| Player | player@example.com | Player123! |

### 3. Admin panel

```bash
cd admin
cp .env.example .env
npm install
npm run dev    # http://localhost:5173
```
Facility staff sign in at `/`. Platform admin (onboarding new facilities) signs in at `/platform-admin`.

### 4. Booking widget

```bash
cd widget
cp .env.example .env
npm install
npm run dev    # http://localhost:5174
```
Visit `http://localhost:5174/book/dha-padel-club` to see the live widget, or `/help` for the player help page.

To actually embed it on a facility's website, copy the snippet from **Settings & Policies** in the admin panel — it's a `<script>` tag that injects a branded iframe, no visible Kortiva domain in the page URL.

## Automated tests

```bash
cd server
npm test
```

148 tests as of the current build (started at 40; see below for how it grew) — unit + integration against a real Postgres database, no mocks — covering exactly the acceptance criteria in Section 10 of the spec: multi-tenant isolation, booking concurrency (including a Prisma P2034 write-conflict edge case the tests themselves surfaced and got fixed — see below), the full deposit-deadline/grace-buffer state machine, all three refund-policy scenarios (now with owner-configurable thresholds, not just the percentage), weather-closure cancellation + refund, discount-code validation/expiry/comp-reporting/campaign-targeting, the payments/accounting audit pass, Section 6's Open Match / Looking to Play module, facility timezone/operating-hours correctness, equipment/rental stock enforcement/pricing/returns-with-refund-and-email, and the in-depth reporting/customer-analytics layer. Integration tests create their own tagged fixtures and clean up after themselves in FK-safe order (`tests/helpers.ts`) so repeat runs against the dev database don't accumulate junk.

Two real bugs were caught and fixed while writing these tests, not staged for demonstration:
- The `@@unique([courtId, startTime])` constraint blocked ever re-booking a slot after an earlier booking there was cancelled — fixed with a partial unique index scoped to `PENDING`/`CONFIRMED` bookings only (see the `partial_unique_active_booking` migration).
- A concurrent-but-not-identical booking overlap (Postgres Serializable transaction conflict, Prisma code `P2034`) surfaced as a raw 500 instead of a clean 409 — `errorHandler.ts` now maps it the same way as the exact-duplicate case (`P2002`).

## Payments & accounting — a dedicated audit pass

A second pass specifically on the money paths (`server/src/services/payment.service.ts`) found and fixed real gaps, all now pinned by `tests/integration/paymentsAccounting.test.ts` (46 tests total, up from 40):

- **Walk-ins can now capture payment in the same step they're created** (`POST .../bookings/walk-in` accepts an optional `payment` field) — previously a walk-in booking and recording its payment were two separate calls, so a busy staff member could create the booking, get pulled away, and never record the cash that was actually collected.
- **Check-in has a real "collect the remaining balance" flow.** `checkIn()` used to just reject with a generic "still unpaid" — it now accepts an optional `settlement` (amount + payment method) and records it atomically with the check-in, and the admin Calendar page prompts for exactly that when check-in is refused, naming the exact shortfall (`error.details.balanceDue`), not just a vague message.
- **Fixed a real accounting bug in extensions**: `extendBooking` hardcoded the payment gateway to `BANK_TRANSFER` regardless of how the customer actually paid, and unconditionally marked the *entire* booking `FULLY_PAID` — so if the original deposit was never fully settled, that debt silently vanished the moment someone paid for a 30-minute extension. Fixed by centralizing payment-state computation into one function (`syncBookingPaymentState`) that recomputes from actual confirmed payments vs. price every time, so an old unpaid balance correctly "carries forward" instead of disappearing.
- **Same bug, same fix, in equipment rentals** — attaching a rental to a booking increases its price but now correctly re-checks payment state, so a booking can't stay marked `FULLY_PAID` while quietly owing for a racket it never paid for.
- **Every confirmed payment now triggers a notification** (`notifyPaymentReceived`, logged to `NotificationLog`, sent via WhatsApp once that's configured) naming the amount, method, and any remaining balance — so accounting is never "silent."
- **A new overdue-payment flag**: a 15-minute sweep (`paymentFlags.job.ts`) finds any booking whose slot has already ended with money still owed and flags it exactly once (deduped), and a new **Outstanding Balances** report/admin page (`Payments → Outstanding Balances`) lists every unpaid booking with a running total, so a missed payment surfaces on its own rather than needing someone to notice.
- **Caught one more bug in the admin UI itself** while testing this in the browser (not just via the test suite): the walk-in "collect payment now" amount field defaulted to empty rather than the estimated price, so leaving it blank — the natural thing to do — silently created the booking as UNPAID even with the checkbox ticked. Fixed by defaulting to the estimated price at submit time, shown as the field's placeholder.

## What's been verified

Tested live against a running Postgres database (not just read from code):

- Two facilities onboarded, fully isolated (Facility B's owner gets a 403 reading Facility A's data)
- 5 concurrent booking requests for the identical slot → exactly 1 succeeds, 4 get a clean 409 conflict
- Deposit calculated correctly from facility settings (% and deadline-before-slot)
- Owner free-play code → Rs 0, auto-confirmed, still shows as "comped" in the daily report (not just missing revenue)
- Invalid/unknown discount code rejected outright, never silently charges full price
- Staff walk-in booking at 30-minute granularity; customer booking rejected below 60 minutes
- Weather closure cancels every booking in its window and credits the paid amount to each player's wallet
- Customer cancellation < 2h before slot → no refund, even when other rules would otherwise apply
- Daily report reconciles revenue, refunds, deposits vs. full payments, and comped bookings
- End-to-end widget flow: browse availability → sign up → reserve → submit bank-transfer proof → appears in the admin's Payments queue

## Fast-Follow modules (Section 5)

All six are built and tested:

1. **WhatsApp automation** — booking reminder (30 min before), thank-you (after the game), owner new-booking alert. Every trigger fires and is logged to `NotificationLog`; actually sends via Meta's Graph API once `WHATSAPP_ACCESS_TOKEN`/`WHATSAPP_PHONE_NUMBER_ID` are set, "stubbed" until then.
2. **Booking extension** — 30-minute increments, paid in full immediately, blocked if another booking already follows (admin Calendar page → "+30 min" on a confirmed booking).
3. **Waitlist auto-notify** — join the waitlist for a full day from the widget; the moment a matching booking is cancelled (customer or staff — not a weather closure, since that slot is genuinely still blocked), everyone waiting is notified. Verified live: join → cancel → status flips to `NOTIFIED` in the same request cycle.
4. **Equipment/racket rental** — simple per-facility inventory; staff attach a rental to a booking and its cost is added straight to that booking's price.
5. **Loyalty/wallet credit** — set a facility-wide loyalty % in Settings; every booking that becomes fully paid credits that percentage to the player's wallet, reusing the same wallet as refunds. Players can spend any wallet balance toward a new booking's deposit at checkout ("Use it").
6. **Birthday free-session trigger** — a daily sweep finds players with a birthday today who've opted into marketing and actually played at a given facility before, and issues them a personal, single-use, 100%-off code valid for 14 days. Verified idempotent (won't double-issue the same year) and single-use (a second redemption attempt is rejected).

## Section 6 — Open Match & Looking to Play

The Phase 2 module from the updated build instructions, built end-to-end and covered by 13 new integration tests (`openMatch.test.ts`, `lookingToPlay.test.ts`), bringing the suite to 68 tests total.

- **Open Match** — at checkout, a player can tick "Open Match — let others join," set how many players are needed (including themselves), and choose Split the cost (share recalculates live as each new unpaid player joins: `price / playersJoined`) or I'll cover it (joiners play free). The match appears under the widget's "Open Matches" tab for other players at that facility, with a live joined-count and each joiner's current share, until it fills up or the payment deadline passes.
- **Never over-collects, never claws back**: a joiner who has already paid keeps their locked-in share even if the group later grows; `payOpenMatchShare` caps what it collects at the booking's true remaining balance and settles a joiner for free if the booking is already fully funded — so total collected can never exceed the booking price. Verified with a dedicated test asserting payments sum stays exactly at the booking price even when a share is "paid" after the booker already covered the balance.
- **Deadline behavior is its own sweep** (`openMatch.job.ts`, every minute), deliberately separate from the generic deposit-deadline sweep: if the *booker's own* share is unpaid at the deadline, the whole match voids and any joiners who paid are refunded to wallet automatically; if only a *joiner's* share is unpaid, just that slot expires and the shortfall becomes a normal outstanding balance on the booker's account — the booking is never cancelled over one no-show joiner.
- **Concurrency-safe**: two players claiming the last open slot simultaneously — only one succeeds (advisory-locked transaction, verified with a `Promise.all` race test asserting exactly one 201 and one 409).
- **Looking to Play** — a player with no group yet can ask to be notified ("Looking to Play"), with an optional time window (defaults to the next 24 hours) and skill level. The moment a matching Open Match opens up at that facility — whether it already existed or someone books a new one after — they get an in-app alert and can join straight from Open Matches. Requests expire automatically once their window passes (15-minute sweep) and unmatched ones surface in the admin Daily Report as an "Unmet demand" signal.
- **Admin visibility**: an "Open Match (need N)" badge on the Calendar's bookings list, and the Reports page's new "Unmet demand — Looking to Play" card.
- Verified live end-to-end in the browser: booked an Open Match (split, 2 players needed) → confirmed the booker's deposit from the admin Payments queue → the Calendar badge appeared correctly → a second player account browsed Open Matches, joined, and paid their share (bank transfer → correctly shown as pending confirmation, not falsely "paid") → match auto-closed at 2/2 joined. Separately, a player filed a Looking to Play request → a matching Open Match was booked by someone else → the request correctly showed "1 match found."
- One real bug caught live and fixed: the "Pay your share" success toast said "Share paid — you're all set" even for a bank-transfer payment that was actually still `PENDING` confirmation — misleading, unlike the main checkout flow's correct "Proof submitted" copy. Fixed to check the payment's actual status and show the right message, and added the same receipt-link/instructions UI the main checkout uses for bank transfer instead of submitting blind.
- Design decisions not specified in the spec (facility-scoped vs. platform-wide matching, share-locking, booker-vs-joiner deadline handling) are recorded in `docs/OPEN_QUESTIONS.md` (#9–#11).

## Timezone, operating hours, and facility address

A platform serving facilities (and their players) across timezones can't treat every court time as a naive UTC timestamp — this pass made facility timezone the actual source of truth end-to-end, fixing several real bugs along the way.

- **Facility timezone is authoritative** — every court time, in the widget and the admin panel, is always shown in the FACILITY's own local time, never converted to the viewer's. This was a deliberate product decision: a padel court is a physical place, so "6pm" means 6pm there whether the player booking it is in Lahore or London — converting to the viewer's timezone would risk someone showing up hours off. `Facility.timezone` (IANA, e.g. `Asia/Karachi`) drives it, set once during onboarding (auto-derived from the facility's address, or manual).
- **Operating hours** — `Facility.openTime`/`closeTime` (facility-local `HH:mm`, wraps past midnight if `closeTime <= openTime`) now actually constrain which slots show as bookable, instead of a fixed midnight-to-midnight grid.
- **Real bug fixed**: the availability endpoints computed a day's range as naive UTC midnight (`` `${date}T00:00:00.000Z` ``) and had no "is this slot already in the past" check at all — reproducible live as a 6am slot showing bookable at 2pm local time. Fixed with facility-timezone-aware day bounds (`server/src/utils/timezone.ts`, built on `Intl.DateTimeFormat` alone — no new date library) and a genuine "past" slot status. The admin dashboard's "today" stats and the Calendar's booking-list query had the identical bug and got the same fix.
- **12h/24h time format** is a separate, per-viewer preference (defaults from the browser's own locale, remembered after a manual toggle) — independent in the admin panel (per staff member) and the widget (per player).
- **Facility address**: Google Places Autocomplete when configured (`GOOGLE_PLACES_API_KEY`, same "stub until real credentials exist" pattern as the payment gateways), falling back to plain manual entry otherwise. Selecting an address auto-fills coordinates and, via the offline `tz-lookup` package (not Google's billed Time Zone API), the facility's timezone — always still manually overridable.
- Verified live end-to-end: set a facility's timezone to `Asia/Karachi` via Settings (through the real "not configured" Places fallback, since no API key is set in dev) → the widget correctly showed "Times shown are DHA Padel Club's local time (Asia/Karachi)" with already-passed slots correctly greyed out and excluded → booked a slot, confirmed the 12h/24h toggle instantly reformats every displayed time → confirmed the admin Calendar, defaulted to the facility's own "today," showed the exact same booking at the exact same local time.
- 15 new tests (`src/utils/__tests__/timezone.test.ts`, plus new cases in `availability.test.ts`) lock in the DST-correctness of the zone math and the past/operating-hours slot filtering, bringing the suite to 85 tests total.
- Design decisions (viewer-relative vs. facility-local display, per-viewer time format, Places vs. manual address) are recorded in `docs/OPEN_QUESTIONS.md` (#13).

## Discount code campaign targeting

Discount codes started out as a flat percent-off or free-play grant with an expiry date and a global use cap — enough for a simple corporate code, but not enough for the kind of targeted campaigns a facility actually runs ("20% off Wednesdays," "fill the dead 2–4pm slot," "one free session per new player"). This pass layered a set of independent, AND-ed targeting rules on top of the existing discount types.

- **New optional restrictions on `DiscountCode`**, all combinable: `daysOfWeek` (facility-local weekday, e.g. Wednesdays only), `startTimeOfDay`/`endTimeOfDay` (an off-peak booking-start window, e.g. 14:00–16:00), `courtIds` (restrict to specific courts), `minDurationMinutes` (require a minimum booking length), `validFrom` (in addition to the existing `expiryDate`, so a code can be scheduled to start in the future), and `perPlayerLimit` (a per-player redemption cap alongside the existing facility-wide `maxUses`). Every rule is optional and empty/null means "unrestricted" on that dimension.
- **One shared validator, not two.** Both the live checkout discount-code preview (widget) and the actual booking-creation code path call the same `validateDiscountCode()` (`server/src/services/discount.service.ts`), so a code can never validate in the preview and then fail (or vice versa) at checkout — the exact "duplicated validation logic drifts apart" bug class already hit once this session with payment status handling.
- All day-of-week and time-of-day checks are resolved in the **facility's own local time** (`dayOfWeekInZone`/`timeOfDayInZone` in `server/src/utils/timezone.ts`), consistent with the rest of the platform treating facility-local time as authoritative.
- The admin Discounts page exposes all of this as a real form — day-of-week chip toggles, an off-peak time-window pair, a court multi-select, a minimum-duration field, valid-from/expiry dates, and separate total-uses / per-player-limit fields — plus a human-readable summary line per code (e.g. "20% off · Wednesdays only · 14:00–16:00 · Court 2 · no expiry · 1 use/player · used 0 time(s)") so staff can see a campaign's rules at a glance without opening the edit form.
- Verified live end-to-end in the browser: created a code restricted to Wednesdays, 14:00–16:00, and Court 2 — attempting it on a Thursday was correctly rejected with "This code is only valid on Wednesday," and applying it on Wednesday at 3pm on Court 2 correctly showed "20% off applied" and priced the booking at Rs 3,600 (20% off the base rate).
- 7 new tests in `discountCodes.test.ts` (day-of-week, off-peak window, court restriction, minimum duration, not-yet-active `validFrom`, per-player limit, and the live-preview endpoint honoring the same rules), bringing that file to 11 tests and the suite to 92 total.
- Recorded in `docs/OPEN_QUESTIONS.md` (#14), along with the field list and the decision to consolidate validation into `discount.service.ts`.

## Equipment & rentals rework

Equipment rental started as "one flat item name, one flat price, staff-only, added after the fact" — no way to model a racket's skill level, a ball's condition, a free house racket, or a per-game rate, and (found while rebuilding it) no real stock enforcement at all. This pass turned it into an actual revenue-driving catalog, usable by customers themselves at checkout, not just staff.

- **Groups scoped by sport** (`EquipmentGroup`: name + `sport`) — "Rackets," "Balls," anything else. Kortiva is a padel platform and a facility is always exactly one sport (courts always inherit their facility's `sport`, never their own), so in practice this just means every group at a facility is that facility's own sport — see the padel-only correction below.
- **Per-item level/condition tags, descriptions, a free tier, and per-game pricing**: `Equipment` gained `level` (free-text — "Beginner"/"Pro" for a racket, "New"/"Lightly used (1–3 games)" for balls), `description`, `isFreeTier` (an included racket that always rents at Rs 0 but is still tracked for stock), and an optional `perGameRate` alongside the existing flat `rentalPrice`, so a player who only needs a racket for part of their session isn't stuck paying the full-booking rate.
- **Real, transactional stock enforcement**, not just a number shown on a page: adding equipment to a booking now locks each item (the same `pg_advisory_xact_lock` pattern already used for the court itself) and checks how much is genuinely free for that time window — summed the same way as the court double-booking check — inside the exact same transaction that creates the booking. If stock runs out, the whole booking fails atomically; the court slot is never claimed while the equipment silently drops.
- **Customers can now rent at checkout**, not just staff after the fact: an optional "Need a racket or balls?" step appears once the court and time are already chosen — visual cards, one-tap add with an instant quantity stepper, a running total that updates immediately, and items with no stock left for that slot shown clearly disabled rather than hidden or selectable-then-failing. The identical picker (same catalog endpoint, same stock/price rules) is embedded in the staff walk-in form too, with a search box for speed at the counter — one real implementation behind both surfaces, not two that could drift apart.
- **Real bug found and fixed via live browser testing**: `admin/src/components/Modal.tsx` had no height cap or scroll, so any modal taller than the viewport — which the walk-in form now is, with the equipment picker embedded — clipped its bottom content, including the Create/Cancel buttons, completely unreachable with no way to scroll to them. Fixed generally (`max-h-[90vh] overflow-y-auto`) so every current and future modal in the admin app is protected, not just this one.
- Verified live end-to-end: added a Pro racket and a can of new balls to a widget checkout, watched the running total update from Rs 4,500 to Rs 5,950 before reserving, confirmed the server-recorded booking priced and itemized both lines correctly and set the deposit off the true total; separately created a walk-in booking with a ball rental from the admin Calendar, confirming the identical picker, pricing, and payment-collection flow; and confirmed a tennis court's checkout shows only its own tennis catalog, never the padel one.
- 6 new tests in `equipment.test.ts` (levels/pricing/catalog listing, free-tier zero-pricing, per-game pricing math, atomic stock enforcement across overlapping slots, walk-in creation with equipment, and the public catalog's active-only/stock-aware filtering), bringing the suite to 98 tests total.
- Recorded in `docs/OPEN_QUESTIONS.md` (#15), along with the decision to exclude equipment from discount-code percentages and the shared-catalog-function reasoning.

## Padel-only correction, shorter forms, billing breakdown, and in-depth reports

A follow-up pass on top of the equipment rework above, covering four things found by live-testing it.

- **Real bug fixed**: `POST /facilities/:id/courts` hardcoded every new court's sport to `"padel"` regardless of the facility's own `sport` — harmless only by coincidence, since every facility on the platform happens to be padel (the real spec, Section 1, is explicitly padel-only, not multi-sport-per-facility). Fixed so a court's sport is always derived from its facility, never independently set. The Equipment page's sport-selector UI (from the rework above) was removed for the same reason — a facility is always exactly one sport, so a selector implying otherwise was actively misleading. An earlier test tennis court used to verify multi-sport catalog isolation was reverted.
- **Shorter forms, not longer scrollbars**: the equipment picker added to checkout and the walk-in form made both noticeably longer. Both now show it collapsed behind a "+ Need a racket or balls? (optional)" toggle that expands inline (with its own capped, scrolling picker) only when clicked — most bookings skip equipment entirely, so the common case stays short.
- **Court fee vs. equipment fee, wherever a total is shown**: the widget's payment screen, `MyBookings`, the admin Calendar's booking list, and Outstanding Balances all now show the split (e.g. "Rs 5,950 (court Rs 4,500 + equipment Rs 850)") instead of one undifferentiated number whenever a booking has equipment attached.
- **In-depth reports and a real customer roster**, replacing what used to be a single daily-totals card and a bare name/phone/email list:
  - Revenue per court and revenue by day of week, for any date range, using actual collected payments (split proportionally between court and equipment on a partially-paid booking) — the day-of-week view feeds directly into the day-restricted discount codes from the campaign-targeting pass above.
  - New vs. returning customer counts per range, computed from whether a player's first-ever booking falls inside it.
  - An automatic customer tier per player — New / Occasional / Regular / VIP by lifetime visit count, plus a separate Lapsed flag — with freeform manual tags layered on top for anything the automatic tiers don't capture.
  - Per-customer play-pattern insights: total visits, a day-of-week histogram, and a month-by-month "which day did they favor that month" breakdown, aimed squarely at "when should I send this customer an offer."
- Verified live: confirmed the collapsed equipment sections expand correctly and keep both forms short; confirmed the billing breakdown renders correctly across all four surfaces above with real bookings; confirmed the Reports page's revenue-per-court and day-of-week bars against known fixture data, and the Customers page's tier badges, tag add/remove, and per-customer insights modal (day-of-week histogram + monthly best-day breakdown) against a real customer's booking history.
- 3 new tests in `analytics.test.ts` (revenue per court/day-of-week/new-vs-returning for a date range, the customer roster with tiers and manual tags, and per-customer day-of-week/monthly insights), bringing the suite to 101 tests total.
- Recorded in `docs/OPEN_QUESTIONS.md` (#16).

## Itemized equipment and a return/refund mechanism

A live-testing follow-up on the point above: the court-fee-vs-equipment-fee split answered "how much," but not "which item" — and there was no way to undo an equipment line once added.

- Every place that shows a booking's money (the payments ledger, pending confirmations, outstanding balances, the admin Calendar's booking list) now itemizes the actual equipment — "🎒 Pro racket, New balls ×1" — not just a dollar split, via a shared `summarizeEquipmentLines()` helper so the format can't drift between endpoints.
- **A real return/refund mechanism**: each equipment line on an active booking has its own "Return" button. Removing one computes how much of what's already been collected was paid specifically toward that item and refunds exactly that — as wallet credit for a registered player (the same path every other refund uses), or a recorded `Refund` row for a manager to settle by hand on a guest walk-in (no wallet to credit). If nothing had been paid toward the item yet, no refund is issued — the price and balance owed just drop. Stock frees up immediately since availability is always computed live, never a separate counter.
- Adding equipment to a booking that isn't PENDING or CONFIRMED is now refused outright — a leftover gap where a cancelled booking could still have rentals attached.
- Verified live: added a paid racket to a walk-in, returned it, and confirmed via the database that the original payment record stays untouched while a separate refund row correctly brings the booking back to fully paid at its new, lower price; confirmed the Payments page's pending and ledger sections now name the exact item behind every equipment-inclusive amount.
- 4 new tests in `equipment.test.ts` (full refund on a paid item, no refund when the item was never paid for, a guest booking's refund without wallet credit, and the cancelled-booking guard), bringing the suite to 105 tests total.
- Recorded in `docs/OPEN_QUESTIONS.md` (#17).

## Real bug: `window.confirm()` doing nothing, plus "reject as unavailable" with email

Reported directly by the user: the new Return button (and Cancel, and No-show) appeared to do nothing when clicked. The cause — every one of those actions was gated behind a native `window.confirm()` dialog, and native JS dialogs can be silently suppressed by the browser/environment; a suppressed `confirm()` returns `false`, so `window.confirm(...) && mutate()` just quietly does nothing, with no error and no visible feedback. This was directly reproduced in this session's own testing tooling.

- **Every `window.confirm()` in the admin app replaced with a real in-app `Modal`** — Calendar's Return/No-show/Cancel, Weather's Block-courts. None of these can be silently suppressed the way `confirm()` was; this is a general fix, not specific to equipment.
- **Equipment return now asks why**: "Customer changed their mind" (silent, refund only) or "Item isn't available" (refund + email the customer). A new `EMAIL` notification channel (`notification.service.ts`) follows the exact same stub-until-real-credentials pattern already used for WhatsApp and the payment gateways — every rejection fires and logs, and only actually sends once `RESEND_API_KEY`/`EMAIL_FROM_ADDRESS` are configured. Only fires for a registered player with an email on file; a guest walk-in has none, so staff are expected to tell them directly.
- Verified live: clicked Return with no workaround this time, confirmed the modal actually appears and works, chose "Item isn't available," and confirmed via the database that a `NotificationLog` row was created with the correct player email, message (item name + refund amount), and `STUBBED` status.
- 2 new tests in `equipment.test.ts`, bringing the suite to 107 tests total.
- Recorded in `docs/OPEN_QUESTIONS.md` (#18).

## Cancellation & refund thresholds are now fully owner-editable

The spec is explicit that every cancellation-policy number must be settable per facility, but only the partial-refund percentage actually was — the "2 hours before the slot" cutoff and "1 hour grace" window were hardcoded constants, and a no-show's refund was hardcoded to always be 0% with no way to change it (the code that would have applied a configurable no-show refund existed but its result was silently discarded). Settings' own copy stated these as fixed facts rather than something an owner could change. Reported directly: "cancelation and refund amount should be set by the owner so should be editable."

- `CancellationPolicy` now has four owner-configurable fields — the no-refund cutoff (hours before the slot), the full-refund grace window (minutes since booking), the partial-refund percentage, and a no-show refund percentage (0% by default, but a real setting now) — all editable from Settings, with a live example sentence that updates as the owner types.
- **The widget matches, not just the admin panel**: both `GET /players/me/bookings` and the public facility endpoint now return a fully-resolved policy, and the widget's cancel-booking and leave-Open-Match confirmation modals interpolate the facility's real numbers instead of hardcoding "2 hours"/"50%" — showing a customer the wrong number would have been actively misleading once these became per-facility. Static help articles were softened to describe the shape of the rule with illustrative defaults rather than asserting fixed numbers as platform fact.
- 7 new tests (a custom policy actually changes a real cancellation's outcome, a no-show refund is issued once configured above 0%, the public endpoint exposes the resolved policy, plus unit coverage), bringing the suite to 113 tests total.
- Verified live: changed the demo facility's cutoff to 3h and no-show refund to 20% from Settings, saved, and confirmed the widget's cancel modal for a real booking immediately reflected the new 3-hour number.
- Recorded in `docs/OPEN_QUESTIONS.md` (#19).

## Platform Admin panel: Modules & Billing, Facilities, Sports, Settings & Support

A dedicated internal control surface for the Platform Admin role (Section 3), separate from each facility's own admin panel — five top-level nav sections, its own login/session/route tree (`/platform-admin/...`), and a full billing engine underneath.

- **Modules & Billing**: a Platform-Admin-managed `Module` registry (add a module, offer it to any facility, no code deploy — same philosophy as the Sport catalog below) and a per-facility `FacilityModule` toggle with a live, counted consequence preview before it takes effect ("1 equipment rental is attached to upcoming bookings", not a bare "Are you sure?"). Billing statements are generated per facility per cycle, itemized by module + a commission line; a facility on the Payment/Accounting Service module nets automatically (no separate invoice), everyone else gets a real itemized statement with mark-paid/waive and a retry-then-flag sweep for missed payments.
- **Facilities**: real server-side filtering (market/sport/status/module) and a full drill-down per facility — modules, billing history, usage stats, staff, and a commission-override control that's always visible on the record with who set it and when, never a silent exception. Suspend/reactivate shows real, counted consequences (upcoming bookings, pending statements) before confirming.
- **Sports**: a Platform-Admin-managed sport catalog, seeded with Padel (what's actually live today).
- **Settings & Support**: platform-wide defaults (commission %, per-court cap, new-facility deposit/cancellation defaults, billing schedule) that a new facility starts with; Platform Admin's own staff accounts (add/deactivate, can't deactivate yourself); and a dispute/audit tool that pulls a booking's or billing statement's full history (payments, refunds, audit log) by ID.
- Platform revenue and gross facility booking revenue are computed and shown as two separate, currency-labeled numbers on the Dashboard — never summed or shown ambiguously close together, per the spec's own nav standard.
- The build prompt referenced a commission model ("6% Pakistan graduated model, capped per court") and several BUILD_INSTRUCTIONS.md sections (Sport Management, Match Score Recording, Tournaments) that don't actually exist anywhere in this repo — every price and threshold was built as a configurable value with an honest placeholder default rather than a guessed number. Full detail in `docs/OPEN_QUESTIONS.md` (#20).
- 11 new integration tests, bringing the suite to 124 tests total.
- Verified live as `admin@kortiva.app`: real dashboard numbers, a module-toggle consequence preview computed from actual in-progress bookings, a suspend preview, a commission-override save rejected without a note and accepted with one, and an audit lookup returning a real booking's full history.

## Marketing: facility-owned WhatsApp/email connectors, and real campaign sending

WhatsApp and email were previously 100% platform-wide (one shared credential pair for every facility) and automated-trigger-only — nothing let a facility send an ad-hoc message to their own customers. Asked directly for a connector a facility can attach their own API credentials to, plus a real campaign scenario built on top.

- **Integrations** (new nav page): a facility connects their own WhatsApp Business API or Resend email account. Credentials are AES-256-GCM encrypted before they're ever stored — the first secret-at-rest pattern in this codebase — and only a masked preview (`•••• 8f21`) is ever shown back. Saving runs a live connection test immediately. A facility's own connector wins over Kortiva's shared one everywhere, not just for campaigns — it also switches their existing automated reminders/receipts to send from their own number, once connected.
- **Campaigns** (new nav page): pick an audience by tier (New/Occasional/Regular/VIP), tag, or lapsed status, write a WhatsApp or email message, and send. Consent is enforced server-side as a hard filter — a campaign can never reach a customer who hasn't opted in, no matter what's requested, and the composer shows the honest gap ("31 of 42 matching customers have opted in"). Every campaign gets its own detail page showing exactly who received it and whether it delivered, not just a count.
- Both features are gated by the Platform Admin module registry (`whatsapp_automation`/`marketing_services`) — an unentitled facility sees a clear upsell-framed locked state, not a dead link.
- 9 new integration tests, bringing the suite to 133 tests total.
- Verified live: enabled Marketing Services from Platform Admin, watched the facility's nav unlock immediately, built a real audience from actual customer data, sent a campaign that correctly failed with a clear "no connector configured" reason (nothing's wired up platform-wide either), and connected a deliberately-fake Resend key that correctly failed its live test with Resend's own rejection reason.
- Recorded in `docs/OPEN_QUESTIONS.md` (#21).

## Five gaps found from actually using the Platform Admin panel: owner login editing, flexible pricing, billing approval, player insights, tickets

Reported directly after live-testing #20: owner email/password couldn't be edited anywhere, commission/module pricing was percent-only, billing statements went straight to PENDING/PAID with no review step and no facility-facing view at all, no platform-wide player data existed, and there was no support-ticket path between a facility and Kortiva.

- **Editing**: Settings now exposes facility name/logo/contact fields it already accepted server-side but never rendered; a new "My Account" card lets any staff member (owner included) change their own login email/password (current-password-verified) — `PATCH /staff/:id` explicitly refuses to touch an OWNER row, and nothing else existed for self-service. Platform Admin gained the matching support-side actions: edit a facility's core details, and reset a locked-out owner's login (explicitly audited).
- **Flexible pricing**: the commission override and a module's negotiated price can now each be either a flat amount or a percentage of gross revenue — previously percent-only, and module pricing had no UI to set at all despite the field existing.
- **Billing approval**: statements now generate as a `DRAFT` — invisible to the facility, editable line-by-line by Platform Admin, recomputing the total server-side — and only become real (`PENDING`/`PAID`, and for invoiced facilities, emailed and visible) once explicitly approved. A brand-new facility-side Billing page shows their own approved statements; a "Generate drafts" button was added since the API already existed with no way to trigger it from the UI.
- **Player insights**: a new Dashboard section — cross-facility engagement tiers, a market breakdown, a top-15-by-spend table, and a genuinely platform-only signal (how many players use more than one Kortiva facility).
- **Support tickets**: a facility can raise a ticket and thread messages with Kortiva from Help → My Tickets; Platform Admin triages every facility's tickets in one place from Settings & Support. Replying auto-advances status; a facility replying to a resolved ticket reopens it automatically.
- 18 new integration tests, bringing the suite to 142 tests total.
- Verified live: edited the facility's own details and changed a password via My Account; generated real draft statements for all 6 facilities, edited and approved one, and watched it immediately appear on the facility's new Billing page; raised a support ticket, replied as Platform Admin, and watched the reply and status change land on the facility side; confirmed the Dashboard's new Players section with real cross-facility numbers.
- Recorded in `docs/OPEN_QUESTIONS.md` (#22).

## Widget Link Generation & Management

Every embeddable widget in the product now runs through a shared, unified link system instead of a permanent slug-based URL with no way to revoke it.

- A facility's booking widget link is generated automatically the moment the facility is onboarded — zero manual step — and an unguessable token (`crypto.randomBytes(24)`), never a predictable id.
- **Regenerate control**: an owner can invalidate and reissue a link with an explicit "old embeds will stop working immediately" warning before confirming. Regenerating never mutates a token in place — it revokes the old row and creates a fresh one, so a stale embed gets a real, permanent 410, not a silent break or a silent rotation.
- View/load counts are tracked server-side (the widget app's new `/w/:token` resolver counts every load) and visible on both the new facility-side Widget Links page and nested into the Platform Admin facility drill-down, plus a platform-wide adoption card on the Dashboard.
- The schema (`ownerType`/`widgetType`) is already shaped for tournament registration/schedule/countdown widgets — see below for why Tournaments itself wasn't built in this pass.
- Also corrected the Tournaments module's billing type to per-transaction (tournament frequency varies too much facility-to-facility for a flat monthly fee to be fair).
- 6 new integration tests, bringing the suite to 148 tests total.
- Verified live: generated a real snippet, navigated the actual widget app to its token URL and watched it resolve to the real booking page with the view count incrementing, then regenerated the link and confirmed the old token now shows a clear "no longer active" message while the new one works.
- Recorded in `docs/OPEN_QUESTIONS.md` (#23) — which also documents the deliberate decision **not** to build Tournaments or Match Score Recording in this pass (a full real-time bracket/live-scoring system with its own reporting pipeline, sized for a dedicated build).

## Known gaps — see `docs/OPEN_QUESTIONS.md`

The short version: JazzCash, Easypaisa, and card payments are wired to the interface those providers' sandbox APIs expect, but return "not configured" until real merchant credentials exist (that's a business onboarding step, not a code gap). Bank transfer — the payment method the spec says to build first — is fully live. WhatsApp similarly needs Meta Business verification before it actually sends anything, but every trigger, message, and log entry is already built and tested in stub mode. Email (currently just the "your equipment isn't available" notice) follows the same pattern via Resend — falls back to logging only until `RESEND_API_KEY`/`EMAIL_FROM_ADDRESS` are set. Google Places (facility address autocomplete) follows the same pattern — falls back to manual entry until `GOOGLE_PLACES_API_KEY` is set.
