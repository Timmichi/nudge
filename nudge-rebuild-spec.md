# Nudge rebuild: spec

Status: REVIEWED by the founder on 2026-11-26 (revision 5). Revision 6 (same day) writes in the founder's later decisions, each in the decision log; it has not yet been read back by the founder. Open questions Q14, Q17, Q18 and Q19 cover later work or details that do not block v1. No feature code is written until the phase 0 exits in section 10 are met.
Final screens: `nudge-v1-final.pen` (one frame per screen, tagged FINAL, V1.1 or REFERENCE, every screen redrawn to one set of rules, with the rules every screen follows at the top). Sources: the redesign board (`nudge-ux-rethink.pen`, sections 20 to 37, kept as history), the field guide "Building a production app with AI in 2026", and a read of `nudge-ai` at `origin/main` (45ea8df4, 2026-11-23).

This is the top-level spec: scope, stack, data, guardrails, build order. Each feature in section 8 later gets its own short spec (30 to 60 lines) in `specs/NNN-name/` before it is built. This file stays the source of truth; if building proves it wrong, this file changes first.

---

## 1. Why rebuild

What the current repo looks like after 20 days (2026-11-04 to 2026-11-23):

| Measure | Now |
|---|---|
| Commits on main | 712 (PR numbers reach #734) |
| Database | 41 models, 72 migrations |
| Backend | 26 route files, about 95 service files |
| Surfaces | iOS app, web app, Mac helper, landing site, avatar worker, bubble site |
| Swift | 328 files in the app, 46 in the helper |
| Tracked files | 8,229, of which about 6,400 are images |
| Local checkout | 129 worktrees, 1,451 branch refs, 574 commits behind main |
| Main branch | 20 "post-merge fail, main may be red, fix forward" alerts on Nov 20 to 21 |
| Incidents | UI tests wrote into the founder's real database (Nov 5); a builder printed session tokens into a transcript (Nov 6) |

Causes, in the guide's terms:

- Features were built before the skeleton, so every safety net was added after something broke.
- There is no single spec. Rules live in about 65 briefs, and one file "overrides everything else".
- Safety relied on instructions to agents, not on checks that software enforces.
- Too many builders ran at once, so main was repaired forward instead of kept green.
- The MVP grew sideways (web app, Mac helper, shop, chat relay) before the core loop was settled.

The redesign also changed the product enough (bubbles, nudge and promise only, ghosts, For and With) that most screens and several tables no longer match.

## 2. The product in three lines

- **Problem:** people lose touch with friends they care about because nothing reminds them at the right moment, and reminder apps feel like chores.
- **Core workflow:** a user can see a few things to do for friends today and mark one done, so that no friendship goes quiet by accident.
- **Core loop:** open Today → do one bubble (text, call, think of them, play, give, help, meet up) → mark it → stars and a celebration.

Everything outside that loop is scope that has to earn its place.

## 3. Words (one per thing, same on screen, in the API and in code)

| Word | Meaning |
|---|---|
| Friend | Someone the user keeps up with. |
| Bubble | One thing to do for a friend. Every bubble is a nudge or a promise. |
| Nudge | A suggestion nobody has committed to. From Nudge or, later, from a friend; you don't make nudges for yourself (Q26). Two kinds from Nudge: a just-in-time nudge and a rhythm nudge. |
| Just-in-time nudge | A nudge from something Nudge noticed ("Noah is sick"). No penalty if it passes, can be dismissed, never rerolled. |
| Rhythm nudge | The nudge that comes from a friend's rhythm. It appears before the rhythm runs out. It can be rerolled. |
| Birthday nudge | A just-in-time nudge Nudge makes from a friend's birthday: a week before and on the day. |
| Promise | A bubble the user committed to. It always has a day. Only you, For a friend, or With a friend. Committing is what makes it a promise. |
| Regular promise | A rule that makes one promise for each date it names ("Basketball, Tuesdays"). You keep the promise it made, never the rule. |
| Rhythm | The longest the user wants to go without catching up with a friend. One per friend. A clock that restarts whenever the user catches up with them. |
| Catching up | Completing any bubble for a friend: a nudge done or a promise kept. Making a promise is not catching up; keeping it is. |
| Pips | Five marks per friend showing how the relationship is going. Changed by kept and broken promises and by the rhythm. |
| Streak | Days in a row with no strike. A strike is a broken promise or a rhythm that ran out. Two strikes a month are forgiven. |
| Later | Every upcoming promise after today, as a strip of days on Today and a running list by day. |
| Note | One per day, plain text, lines can be tagged to a friend. |
| Star | Points earned only by completing a bubble. A running count. Nothing to spend them on yet. |
| Ghost | A friend who is not on Nudge: a soft white shape instead of an avatar. |
| Link | The connection between a user's friend entry and that friend's own account. |

On screen a rhythm is always "due": "Rhythm due Fri", "Rhythm due today", "Rhythm due tonight" (from 6 pm), "Rhythm 9 days overdue"; a promise is "due tonight" on its day. "Runs out" and "ran out" are not used on screen (decided 2026-11-27).

Retired words: idea, reach-out, pick, move, deck, catch-up nudge, timely nudge, kept in a row, weekly streak, your nudge (2026-11-27: what you make is a promise or a reminder). They do not appear in new code. In code a regular promise is `regular`.

## 4. Scope

### In v1 (single player, to 5 to 10 testers)
Sign in, friends, rhythm, birthdays, Today bubbles, nudges from Nudge, your own nudges, promises with a day and an optional time, regular promises, Later, hold and drop, stars and streak, notes, Tell Nudge (type or speak), Show up now with an AI text draft, your avatar picked from the looks that already exist, reminders, account deletion.

### In v1.1 (social, after testers have used v1)
Invite and link, ghosts turning into real avatars, nudge a friend, shared promises (For and With), groups up to six, mute, and sharing your own birthday with linked friends (F2).

### Later, not yet scheduled (noted so they are not lost)
Choosing phone or FaceTime on a call, how a gift gets there, more special events beside birthdays (each an exception like birthdays, F5b), and the star shop and avatar items (Q19).

### Non-goals (not built, not stubbed)
- Android, and a web version of the app.
- The Mac helper and any reading of Messages or WhatsApp.
- In-app chat, read receipts, a feed.
- Shop, closet, buying stars, any payments, and any reward that stars unlock.
- Making an avatar from a photo, and any avatar generation. This comes back as its own feature once the art style is settled.
- Calendar view, widgets, photo moments, recap.
- Scoring or ranking friendships.

## 5. Stack (pinned; an agent may not change these without asking)

| Layer | Choice | Why |
|---|---|---|
| iOS | Swift 6, SwiftUI, iOS 26+, Xcode 27 | The main screen is a gesture-driven, animated glass surface. The guide's own rule sends that case to native. See Q1 and Q2. |
| Backend | TypeScript strict, Node 22, Fastify, one deployable | Your strongest language, so you can review it. Modular monolith, modules by domain. |
| Validation and contract | Zod schemas → OpenAPI → generated Swift client | One contract. A wrong shape fails to compile on both sides. |
| Database | Postgres, Prisma, migrations in the repo | Already known. See Q3 for where it is hosted. |
| Auth | Managed (Supabase Auth), Sign in with Apple first | The guide's rule: do not build auth. The current app hand-rolled it. See Q3. |
| AI | Anthropic API, server-side only, `claude-haiku-5-5` by default | Keys never reach the phone. Per-user daily cap and a monthly cap. |
| Jobs | pg-boss in the same Postgres | Daily nudge generation and reminders. No extra service. |
| Push | APNs directly from the backend | No third-party push service needed at this size. |
| Hosting | Railway for the API (already set up) | Staging and production as separate projects, databases and secrets. |
| Errors and uptime | Sentry on iOS and backend, one external uptime check | From day one, proven with a staged error. |
| Avatars | A fixed set of existing looks, each a small set of images (standing, happy, sad, head) served by URL | No generation in v1. A look is data, so a new art style later is a new set of files, not new code. |
| Assets | Asset catalog for a small fixed set of app art | No image over 200 KB in git. |

### Architecture rules
- **The backend owns every rule.** Ordering, what is due, stars, streaks and misses are pure functions in one `rules/` module with exhaustive unit tests. The phone only draws what it is given. This is the part that survives any future rewrite.
- **Modules by domain:** `friends`, `bubbles`, `notes`, `stars`, `ai`, `avatars`, `auth`. A module talks to another only through its exported functions.
- **Thin client:** one generated API client, one view model per screen, no business rules on the device.
- **Online for writes, cached for reads** in v1. No offline queue.

### What the AI does, sees and costs
- **Three jobs only in v1.** (1) Tell Nudge: turn what the user typed or said into a proposed promise, your nudge, or note. (2) Timely nudges: read note lines that are tagged to a friend and propose a nudge when one is worth acting on. (3) Text drafts: on the Show up now sheet of a Text bubble, draft a short message the user edits before sending. Nothing is ever sent for the user.
- **It proposes, the user decides.** Nothing the AI produces is saved as a promise or changes a note without a tap from the user. Nudges are suggestions already, so they need no confirmation.
- **What it is sent:** the Tell Nudge text, the user's friend names, note lines tagged to a friend, and for a text draft the bubble's what and details. Note lines with no friend tag are never sent to the AI.
- **Where the key lives:** the Anthropic key is a secret on the server. The phone calls our API; our API calls Anthropic. A key inside the app could be pulled out of the app file and used to spend our money.
- **Per-user daily cap:** each user gets a fixed number of AI calls a day (40 today). After that Tell Nudge still works but saves plain text as a note, and the Text sheet shows your details with no draft. This stops one user or one bug from running up the bill.
- **Monthly cap:** a ceiling on calls for the whole app (20,000 today). At the cap, AI features switch off for everyone and the rest of the app keeps working. This is the worst-case bill, chosen in advance.
- **Every call is logged** with its purpose and cost, so the caps are set from real numbers.

## 6. Data model for v1 (9 tables, down from 41)

| Table | Holds |
|---|---|
| `user` | Account, time zone, chosen look, celebration choice. |
| `friend` | Owner, name, rhythm in days, last caught up date, pips (1 to 5, current), optional contact handle, birthday (month and day, optional year), birthday nudges on or off (on by default). |
| `bubble` | Owner, friend, kind (`nudge` or `promise`), nudge type (`rhythm`, `just_in_time`, `birthday` or `yours`), text, details, counts as (`text`, `call`, `think_of_them`, `play`, `give`, `help`, `meet_up`), day (required for a promise, optional for a nudge), state (`open`, `done`, `dismissed`, `missed`), source, regular it came from. |
| `regular` | Owner, friend, text, counts as, pattern (weekly on a weekday, every N weeks, monthly on a date), starts on, ended on. |
| `note` | Owner, date (unique per user per day), title, body. |
| `note_tag` | Note, line range, friend, confirmed or suggested. |
| `star_event` | Owner, friend, bubble, amount, date. A ledger; totals are computed. |
| `pip_change` | Owner, friend, from pips, to pips, reason (`broken_promise`, `rhythm_ran_out`, `overdue_week`, `earned_back`, `new_friend`), bubble (optional), at. Append-only history; the friend row keeps the current pips. Written in the same transaction as the change, only by the rules module. |
| `strike` | Owner, friend, bubble (empty when a rhythm ran out), night, covered by a pass (yes or no). Append-only. Passes left this month and the streak are computed from it. |
| `device` | Owner, APNs token. |
| `ai_call` | Owner, purpose, tokens, cost, result. For caps and debugging. |

v1.1 adds `link`, `invite` and `bubble_person` (bubble, user or friend, `in` or `on`). Nothing in v1 has to change shape for that.

Example bubble as the API returns it:

```json
{
  "id": "bub_01JD…",
  "kind": "promise",
  "friendId": "fr_01JC…",
  "text": "Basketball with Sam",
  "details": "",
  "countsAs": "meet_up",
  "stars": 40,
  "day": "2026-11-24",
  "state": "open",
  "label": "PROMISE · TODAY",
  "place": 1
}
```

## 7. API for v1

REST and JSON under `/v1`. Every route requires a signed-in user and checks ownership on the server. About 20 routes:

- `GET /me`, `PATCH /me`, `DELETE /me`
- `GET /friends`, `POST /friends`, `GET /friends/:id`, `PATCH /friends/:id`, `DELETE /friends/:id`
- `GET /up-next?date=`, `GET /later`
- `POST /bubbles`, `PATCH /bubbles/:id`, `POST /bubbles/:id/done`, `POST /bubbles/:id/dismiss`, `POST /bubbles/:id/promise`, `POST /bubbles/:id/draft` (Text only: returns an AI draft, saves nothing)
- `GET /notes?from=&to=`, `PUT /notes/:date`
- `POST /tell` (text in, proposed promise, your nudge or note out; nothing saved until confirmed)
- `POST /devices`
- `GET /regulars`, `POST /regulars`, `PATCH /regulars/:id`, `POST /regulars/:id/end`, `POST /regulars/:id/skip`
- `GET /looks`
- `GET /health`, `GET /health/ready`

## 8. Features and acceptance criteria

A criterion counts only if a test or a named manual check can prove it. Each feature gets its own spec file before it is built.

Product rules below come from the board. The numbers that do not (time limits, frame rate, the 30-day rule, the streak rule in F7) are my proposals, there so each criterion can be tested. Change them freely.

**F1 Sign in and account**
- A new user signs in with Apple and lands on an empty Today page in under 10 seconds on a physical iPhone.
- A request without a valid session returns 401. A request for another user's row returns 404. Both are tested with two accounts.
- Delete account removes every row for that user within 60 seconds and signs the device out.

**F2 Friends and rhythm**
- Adding a friend needs only a name. The Friends list shows them at once.
- The list is plain rows with no swipe-left actions. Holding a row opens the menu (`Friends/holding-row`). Its "We caught up" makes a done bubble on the spot: the user picks the kind and it earns that kind's stars, within the usual limits.
- "Who needs you" order, in three quiet groups with small headers and counts, no row fill (layout A, section 34): Needs you (rhythm overdue, longest first; then rhythm runs out tonight; then anyone at 1 or 2 pips), then This week (anything due in the next 7 days, soonest first), then Everyone else (fewest pips first, then longest since caught up). A promise more than 7 days away does not move a friend up. Each row shows the pips and one status line.
- The list always opens on "Who needs you"; the last sort picked is not remembered. When the first two groups are empty, the list shows one line ("You're caught up with everyone.") and then everyone.
- Changing rhythm from the ⋯ menu changes when the next rhythm nudge is due, covered by rule tests.
- A friend can have a birthday: month and day, and an optional year so the app can say how old they turn. It comes from the contact picked when adding the friend (the contact picker hands over that one contact and nothing else), or the user types it. Birthday nudges (F5b) are on by default, with one switch in Settings and one on each friend's page.
- In v1.1, linked friends can share their own birthday: one skippable sign-up step (month, day, optional year). Month and day are seen by linked friends unless switched off; the year has its own switch, off by default. A shared birthday fills the friend's entry and stays up to date; if the user typed a different date, they are asked once which to keep; turning sharing off removes it unless the user typed it themselves.

**F3 Today**
- The main page is called Today (it was Up Next; renamed 2026-11-27 because it shows only today's things). Frame names keep `UpNext/` as ids.
- The header has the date on its own line, then the title on a row with the streak chip, the stars chip and settings, centred on the title (header fix 1, section 36). The chips never overlap the date or the title: stars of 10,000 or more show as "12.4k", and the layout is checked at a 365-day streak and 12.4k stars.
- The background (gradient and backdrop circles) is one layer behind the whole screen, header and bottom bar included. No section clips it, so nothing stops at a hard line.
- With 4 or fewer open bubbles for today, they sit in a still cluster with the most important in the middle.
- With 5 or more, four are in view and the next one peeks in half off the right edge (once scrolled, the one behind peeks in on the left). Sideways scrolling is free and weightless, with momentum and soft ends: no wrapping, no page ticks, one soft bump at each end. Starting numbers from the prototype: 55 pt of finger travel per bubble; landing on a bubble is a spring of 0.2 s, damping 0.7. No dot strip. Until someone has scrolled once, the peeking bubble nudges in every few seconds and a "6 more ›" pill sits under the bubbles; every few seconds its glass stretches to read "Swipe to see 6 more" and shrinks back (frame `UpNext/many · hint stretched`). A back-to-the-start button shows only at the end. Until someone has done one hold-and-drop, a hint sits under the header: "Hold a bubble and drag it to an action". Frames: `UpNext/default`, `UpNext/many`.
- Bubbles move with soft physics: they drift toward their place, push each other gently apart, and when one leaves, each of the rest takes the nearest open spot (the most important still goes to the middle).
- Order comes from the server. Given the same data, the order is identical on every load (rule test with 20 fixtures).
- The cluster holds 60 fps or better with 10 bubbles on an iPhone 15, measured in a release build.
- Today shows only today's things. All clear means nothing is left for today; rhythm nudges and your nudges sent to tomorrow with Not today do not block it.
- When the last open bubble of the day is done, the all-clear moment plays once that day (frame `UpNext/all-clear-moment`). After that, Today shows the quiet all-clear state (`UpNext/all-clear`) until local midnight, including when the app is reopened. A new bubble arriving the same day replaces the quiet state; the moment does not play again that day.
- The all-clear moment, in starting numbers to tune in the motion prototype:
  - Bloom (0 to 1.2 s): the friends done today rise in their own bubbles; small bubbles pop into stars.
  - Collapse (1.6 to 1.95 s, ease-in): every bubble, face and star is pulled into the centre.
  - Blink (1.95 to 2.2 s): one star flashes at the centre and is gone.
  - Settle (2.2 to 2.6 s): "All clear for today" fades in.
- Haptics, in starting numbers to tune in the motion prototype:
  - Marking a bubble done plays the system success haptic.
  - The all-clear moment plays one custom Core Haptics pattern, used nowhere else:
    - four soft taps during the bloom, about 90 ms apart, intensity 0.4 rising to 0.7, sharpness 0.3;
    - a continuous swell during the collapse, intensity 0.2 to 0.6 over 300 ms;
    - one sharp tap at the blink, intensity 1.0, sharpness 0.8.
- With Reduce Motion on, the moment is replaced by a fade to the quiet state, and the haptic still plays. The screen must make sense when the system skips the haptic.

**F4 Promises and Later**
- A promise cannot be saved without a day.
- A Help or Meet up promise can cover several days in a row: one bubble, one continuous span, finished on its last day (from nudge-friend-core 2.16). The rules for a span (when it locks, when it breaks, stars per day) are open in Q17.
- The place field shows only for Help and Meet up.
- A promise for a later day shows in the Later strip and not in today's bubbles; on its day it becomes a bubble whose only mark is the maroon time-left ring (F7); the bubble itself has no rim.
- Later lists every upcoming promise as one running list grouped by day, with no week sections. A regular promise shows only its next date.
- Before its day a promise can be moved, edited or called off as often as the user likes, for free. On its day its what, day and counts-as are locked; details and contact stay editable. "I can't make it" on its day counts as broken, so it asks first (decided 2026-11-27): a sheet names the promise and shows what happens, that friend's pip going down and today not counting as cleared (a pass covers it, or with none left the streak starts over), with "Keep the promise" as the main button, plus "Show up now instead" and "Call it off".
- Before its day a promise's menu also offers "Make it a nudge instead": it becomes your nudge for the same day, for free. On its day this is gone.
- Marking it kept awards stars once. A second tap awards nothing.
- If its day ends still open, it becomes `missed` at the user's local midnight. That is a broken promise: one pip off for that friend and one streak strike (F7). It then comes back as a nudge labelled with the day it was promised ("NUDGE · WAS PROMISED TUE"), so it can still be done or promised again; the strike stands. That nudge behaves like your nudge (F5a): no new cost, "Still want this?" on day 3, gone on day 7, and it never counts toward the one-from-Nudge limit. Nothing has to be done with it; doing it, or anything else for that friend, counts as catching up.
- A promise made at least one day before its day and kept earns 50% bonus stars, shown as "Kept your word". A promise made on its own day earns the normal stars.

**F5 Nudges from Nudge**
- A rhythm nudge appears before the rhythm runs out: a quarter of the rhythm early, at least 1 day and at most 7 (weekly: 2 days, every 2 weeks: 3 to 4, monthly: 7). Its label shows the due day ("RHYTHM · DUE FRI").
- If the rhythm runs out with nothing done for that friend, that is one pip off and one streak strike. While it stays overdue, one more pip comes off every 7 days, never below 1. Its label shows how late it is ("RHYTHM · 9 DAYS OVERDUE").
- A rhythm nudge offers Did it, Promise (for any day up to its due day) and Not today (back tomorrow). On its due day Not today is gone. It cannot be dismissed; Pause nudges in the friend menu stops the clock.
- Doing any bubble for that friend, or keeping a promise dated on or before the due day, restarts the clock and removes the rhythm nudge. A promise dated on or before the due day also hides the rhythm nudge until then. The moment it stops covering the rhythm (moved past the due day, called off, made a nudge, or broken), the rhythm nudge comes back, with a one-time line saying why ("Back: dinner moved to Sat"). Moving it past the due day warns first: "Sat won't keep your rhythm with Noah (due Fri). We'll nudge you to catch up before then."
- While a friend's rhythm is in its window, anything that would keep it carries a small "↻ Keeps his rhythm" mark in the rhythm's blue with the rhythm's repeat icon (shape and words, never colour alone): that friend's other bubbles on Today, a promise that covers it (its bubble, its Later chip, and the friend's Friends list line, "Promise Thu · ↻ keeps rhythm"). The rhythm nudge's sheet says "Anything you do for Noah by Fri keeps it"; a promise sheet opened from a rhythm nudge says it covers the rhythm if kept. When something else keeps the rhythm, the rhythm bubble floats off during that done moment and the done pill adds "Rhythm with Noah restarted · next due Dec 8". Board: "Exploring · keeping the rhythm clear", sections 1, 5 and 6.
- A just-in-time nudge is for its day only: its label says so, "TODAY ONLY", and from 6 pm "UNTIL TONIGHT". Like every bubble on Today it needs an answer for the day to be cleared (decided 2026-11-27: one rule, no exception); Dismiss is the one-tap answer. Left unanswered, it leaves the board at midnight and waits in the morning check-in with the rest (Q23 e). It never costs a pip. Whether it has a ring is open (Q24).
- A rhythm nudge can be rerolled: Nudge suggests a different idea for the same catch-up. Just-in-time nudges are never rerolled.
- Reroll is a dice button on the idea in the sheet, showing how many are left (3 per nudge). It opens a menu of ways to reroll (like this, from what Nudge knows, surprise, a challenge) and Tell Nudge what you want. A reroll replaces what, details and counts-as together; the day stays.
- A just-in-time nudge can be dismissed. Dismissing it never costs a pip or the day.
- At most one open nudge from Nudge per friend at any time. Birthday nudges are the exception (F5b), and your own nudges never count toward it. When a just-in-time nudge would open while that friend has a rhythm nudge (or the other way round), they become one bubble: the rhythm nudge's label, due day and ring, with the just-in-time idea and its reason ("he's sick"), like a free reroll. When the just-in-time day ends, the bubble goes back to the rhythm idea.
- A dismissed nudge is never recreated with the same text for that friend within 30 days.
- A just-in-time nudge can have a day. Its label shows it ("NUDGE · THU"). On that day it needs an answer like any bubble; it never costs a pip.
- Dropping a nudge that has a day onto Promise fills that day in on the promise sheet.
- A nudge and a promise use the same sheet and the same fields: what, details, day, place, counts as. The only differences are who made it and whether anyone has committed.
- Counts as is one of seven kinds, from nudge-friend-core 2.19b: Text (anything you send), Call (talking live), Think of them (nothing reaches them, such as prayer), Play (together from different places), Give (a thing changes hands, including a card, a loan or paying), Help (a job you do for them), Meet up (time in the same place). There is no catch-all. Nudge picks by asking in order and taking the first yes: nothing reaches them, a thing changes hands, a job for them, same place, together apart, talking live, anything else sent. When two fit, the one worth more stars wins.
- Nudge fills in counts-as from the what, and fills it in again whenever the what changes, until the user picks a kind; from then on it stays. While Nudge is reading, the line shows "Counts as ···"; before anything is written there is no line. Tapping the kind on the kind line opens a menu of the seven kinds with their stars and a check on the current one; picking one changes only the kind. "Make it a…" in the reroll menu opens the same seven as a submenu and rerolls the idea into the kind picked.
- The Later strip holds promises only. A nudge with a day floats on Today from the moment it exists, so Nudge creates its own dated nudges no more than 2 days ahead.

**F5a Your nudges: REMOVED 2026-11-27 (Q26 a).** What you make yourself is a promise (committed, with stakes) or a reminder (no stakes, can be for a friend); only Nudge makes nudges. The lines below are kept for the record until the final screens and the follow-ons in Q26 are done; none of them is built.
- The sheet has two buttons: Promise (main) and Nudge me. Nudge me saves your nudge: a nudge from you that nobody has committed to. In Tell Nudge, the confirm card shows both buttons; Nudge me is the main one unless a firm day was given.
- Your nudge needs a friend and a day (quick picks: Tomorrow, This weekend, Next week, In a month, Pick a day). The what can be vague ("plan something for his birthday"). Counts as is filled in as for any nudge. A time of day is optional (decided for v1, 2026-11-27); with one, the notification comes at that time, otherwise in the morning summary of its day.
- Before its day it shows in the Later list, lighter than promises, and in the evening-before review (F11). The Later strip stays promises only. It does not move a friend up in Friends.
- On its day it is a bubble labelled "YOUR NUDGE" with no time-left ring. Hold and drop: Promise, Did it, Not today (back tomorrow). The tap menu adds Pick another day, Edit and Let it go.
- Did it earns the kind's stars and counts as catching up, like any bubble. Promise turns it into a promise, and the promise rules apply from then on, including the Kept your word bonus.
- If its day ends untouched, nothing is missed: it stays, labelled "YOUR NUDGE · SINCE TUE". Not today does not restart the count; only Pick another day does, so a nudge cannot be pushed forever. After three days untouched it asks once, "Still want this?", with Keep, Pick a day and Let it go. If that is ignored too, it goes away quietly on day 7, at no cost, and stays in that friend's history as a faint line ("Your nudge · let go · Dec 3").
- It never hides a rhythm nudge; only a promise does, because only a promise is a commitment.
- The first time someone opens the promise sheet and leaves without promising, a one-time tip reads "Not sure yet? Nudge me: no streak, no pressure." An empty Later says "Nothing coming up. Want a nudge about someone?"

**F5b Birthday nudges**
- With a birthday on a friend and birthday nudges on, Nudge makes two just-in-time nudges: 7 days before ("Noah's birthday is Saturday. Plan something?") and on the day ("It's Noah's birthday. Text him?", counts as Text). With a year, the words can say the age ("Noah turns 30").
- Neither costs anything if it passes. Birthday nudges are the exception to one open nudge from Nudge per friend: they show beside a rhythm nudge for the same friend, and doing either restarts the rhythm and removes the rhythm nudge. Other special events may join this exception later, each decided on its own.

**F6 Hold and drop**
- Holding a bubble still for 0.2 s lifts it, with a haptic; moving before that scrolls, and letting go before that is a tap. Holding shows the friend large and still (only their expression changes), the other bubbles shrink, fade and rise, and the top reads "Drag onto an action". The actions are pills only, with no outline on the body and nothing drawn at the feet:
  - a nudge: Did it high on the right beside the friend's head, Promise lower on the right beside their chest with clear space between, Dismiss on the left edge at head height (Not today for a rhythm nudge or your nudge, and nothing there on a rhythm nudge's due day);
  - a promise: Already showed up high on the right, Show up now lower on the right, Call it off on the left edge.
  Nothing covers the friend's face or feet (layout B), and nothing sits low on the right, where a right thumb holding the bubble covers it.
  Frames: `UpNext/holding`, `UpNext/holding-promise`, `UpNext/hold-rhythm`, `UpNext/hold-rhythm-due`.
- Over an action, its pill jumps up into the top bar ("Let go: promise", magenta) and leaves a ghost outline where it was; the others dim. Setting aside (Call it off, Dismiss, Not today) goes dark instead of magenta, and Call it off on a promise's day reads "counts as broken". The held bubble shrinks behind the pills so the words stay readable. Frames: `UpNext/holding-over`, `UpNext/holding-call-off`.
- If a bubble lifts already on an action, nothing happens until it stays there 0.3 s, silently; then the action is live as normal.
- Dropping on Promise opens the promise sheet with the text filled in.
- Dropping a promise on Show up now opens one sheet whose buttons change with the kind (the Show up now group of the final screens): Text has an AI draft to edit or reroll and opens Messages with it; Call starts the call; Help and Meet up offer directions when a place is set; Think of them, Play and Give finish with one button. Nothing is sent for the user. Coming back to Nudge afterwards shows the welcome-back sheet, which asks how it went: "We talked" (in the kind's words) finishes it, "Didn't reach him" keeps it open for later today, "Not yet" just closes.
- A nudge can also be done right away without promising, through Show up now in two places: first in its tap menu (frame `UpNext/menu-nudge`), and as "Show up now instead" on the promise sheet that opens when a nudge is dropped on Promise (frame `Sheet/promise-from-drop`). It opens the same Show up now sheet and welcome-back sheet as a promise; coming back, the kind's done words count as Did it, and "Not yet" leaves it a nudge with nothing at stake. It is not a drag pill: the three pill spots keep one meaning each (done, next step, set aside), and a fourth pill would have to sit low on the left.
- Tapping a bubble opens a menu beside it (the bubble stays bright, the rest dim) with the same actions plus Edit (for a nudge, Show up now first); Edit opens the sheet, which has Save and Cancel and no action buttons. This is also the non-drag way for VoiceOver (frame `UpNext/menu`). A nudge's sheet has no "Promise it" row: its day row only adds, changes or clears the optional day, and the nudge stays a nudge.

**F7 Stars and streak**
- Text 5, call 20, think of them 10, play 20, give 20, help 40, meet up 40. Stars once per friend a day: if two things are done for the same friend, the bigger one earns. At most 400 stars a week, Monday to Sunday in the user's time zone. Past either limit the thing still counts for pips, rhythm and streak, and the screen says so in words ("No stars today · still counts" for the friend limit, "Full week · still counts" for the weekly one), never a bare zero. The stars sheet (tap the stars chip) shows a quiet bar for this week against the 400 and how stars work. Reaching 400 plays a small "Full week ★" moment once that week, so the limit reads as a win, not a wall.
- Stars come only from completing a bubble. No other action awards them.
- A promise made at least one day ahead and kept earns 50% bonus stars (F4).
- Today's count shows done out of what's still for today (decided 2026-11-27): a bubble moved to another day, promised for a later day, called off or dismissed leaves the count instead of counting as done, so "0 of 3" becomes "0 of 2", never "1 of 3". Only doing a bubble moves the first number. The day is still cleared only when everything is answered.
- Every action that can be undone shows a small toast under the strip (34 pt high) with Undo, and a thin ring round the Undo button that runs down while Undo is still possible (decided 2026-11-27; starting number 6 s, to tune in the prototype). First-time hints are quiet grey words in the background under the bubbles, breathing slowly, with no pill and no timer; each stays until you've done what it says (one hold-and-drop, then one change of day), one hint at a time (founder, 2026-11-27: a brief hint went away before he did anything).
- The streak counts days in a row you cleared (decided 2026-11-27): everything due that day done, and every other bubble on Today answered (Did it, Promise, Not today or Let go). Reminders never count toward it. Days with nothing on Today count. A day that isn't cleared is one strike, however many things were left. A broken promise or a rhythm that ran out still takes a pip from that friend. A friend costs at most one strike and one pip a night: a promise dated on the rhythm's due day that breaks as the rhythm runs out counts once, and the morning-after message names both.
- The first two strikes in a calendar month are forgiven; the third resets the streak to 0. The two passes come back on the 1st of each month.
- Each friend has five pips. A broken promise takes one off. Earning one back takes three kept promises with that friend on three separate days. A new friend starts at 3. A nudge done for that friend also counts as one of the three days.
- On Today, a ring around a bubble means time left today, and only bubbles that cost a strike tonight have one: a promise for today and a rhythm nudge on its due day. Their label starts with an hourglass. In the evening the label shows the hours left ("3 H LEFT"). Pips mean the relationship. Faces carry no progress ring.
- The streak shows as a flame chip in the Today header with one shield holding the number of passes left this month (blue with 2 or 1, grey with 0; the grey 0 stays, because the next strike then resets the streak). The flame is one drawn shape whose colour marks the streak: a smaller starter flame for days 0 to 6, orange from 7, gold from 30, blue from 100, violet from 365. Its core is white-hot and its glow is drawn in code, so the art is one image per colour. The flame is always gently alive: a soft glow and a slow, small flicker. After 6pm, while something that costs a strike tonight is still open, the flame and the streak number pulse, slowly fading out and back in, and stop as soon as it's done. With Reduce Motion on nothing moves; an at-risk flame turns orange-red instead. The hourglass is only on bubbles, never on the chip. Tapping the chip opens the streak sheet: the month, passes left and when they reset, best streak, next milestone.
- A calendar month with no pass used shows a thin white-gold ring behind the flame, whatever its colour, with two thin wisps drifting up off it. No badge and no colour change. The ring is what marks the month: it stays with Reduce Motion on, when the wisps do not move.
- The streak going up plays on the flame. When the last thing that would cost a strike tonight is done, the flame swells, hops and leans, and three thin wisps rise off its tip and fade. On the first open after the streak went up, the number ticks up with a small hop. Milestones are 3, 7, 14, 30, 50, 100, 200 and 365 days, then every 100; on a milestone the moment is bigger, with four wisps, and when the milestone changes the colour the flame grows into the new one (day 7: the starter grows into the orange flame). With Reduce Motion on only the number changes.
- The star in the stars chip is the drawn star, with no white outline, always glowing a little, with one small sparkle twinkling now and then. When stars land it glows brighter, grows a little and three sparkles twinkle at its tips, then it settles. Kept your word does the same again on its second beat; there is no third look.
- In v1 a lost streak starts again from 0. Winning it back is v1.1 (F18): a lost streak can be won back within 3 days, in two steps: keep a promise with each friend whose strike broke it that night (strikes earlier in the month were covered by passes and do not count), then pay with stars to restore it (the price is still open, Q17). Never with money. A won-back day has its own mark in the streak calendar, and a month with one is not a perfect month.
- The streak-lost sheet comes up once, over Today, on the first open after the midnight it broke. In v1 it says what happened, keeps the best streak, and offers Start again. (In v1.1 the win-back offer then lives on the streak chip for 3 days.)
- Keeping a promise made at least a day ahead plays a second beat after the done moment: its own gold "Kept your word" pill, a bigger gold star burst, the avatar jumps, a second haptic, then the bonus settles into the star count.
- Pips: 5 Close, 4 Warm, 3 Steady, 2 Needs care, 1 Drifting. The friend's avatar is happy at 4 to 5, standing at 3 and sad at 1 to 2.

**F8 Notes**
- One note per day. Opening Notes on a new day creates today's note titled with the date.
- Typing a friend's name offers a dotted tag; confirming it links that line to the friend. The note's words are never changed by the app.
- A note can be about anything. Lines with no friend tag are stored and searchable, are never sent to the AI, and never produce a bubble.
- A friend's page lists every line tagged to them across all days, newest first.

**F9 Tell Nudge**
- Typing or speaking one clear thing ("dinner with Noah Saturday") creates one proposed promise that the user confirms with one tap. The card always offers both Promise and Nudge me; Promise is the main button only when a firm day was given ("Saturday"), otherwise Nudge me is (F5a).
- Several things in one message produce one review card listing each item. Nothing is saved until confirmed.
- If the AI call fails or the daily cap is reached, the text is kept and the user sees "Saved as a note for now".

**F10 Your avatar and celebration**
- A user picks one look from the existing set and can change it at any time. No image is generated.
- When stars are earned, the celebration the user chose plays once.
- Friends who are not linked show the ghost.

**F11 Reminders**
- The evening before, one notification lists tomorrow's promises and your nudges so the user can review them while moving or calling off is still free.
- On the day, one notification at 6pm only if something that costs a strike tonight is still open. No other reminders: never more than two notifications a day.
- Settings › Notifications gives each reminder its own switch and time. Defaults: the evening-before review at 8pm, the still-open one at 6pm. Turning Nudge off entirely, sounds and banners stay in iOS Settings; the app registers its own notification settings page so iOS Settings links to it.
- Wording is person first: the title names the friend and the thing and fits on one line, because iOS cuts a longer title off; the stakes go in the body; at most one emoji, for the activity; the friend's face as the picture on the right. The streak is not the lead.
- On screen, never "free" or "cost" for what doesn't count against you (decided 2026-11-27): say "won't count against you", or just name the action ("Skip this day", "Call it off", "Dismissed"). Day names are the same three letters everywhere, in the strip and the title ("Fri", "Sat", "Sun"); the date line above the title spells the date.

**F12 Regular promises**
- On the promise sheet the day row offers "Repeats": weekly on a weekday, every N weeks, or monthly on a date.
- Each date becomes its own promise. The next one shows in Later no more than 7 days ahead, and becomes a bubble on its day.
- Keeping one counts as catching up, so no rhythm nudge appears for that friend while the regular promise is being kept.
- Ending a regular promise removes future dates and leaves past ones as they were. Ending is never a miss.
- A regular promise has no end unless one is given ("until Dec 31", "for 4 weeks"). Beyond the 7-day strip it shows once, as the rule ("Call Rachel · every day · next today"), on the friend's page and in Coming up's Repeating section. Changing one date asks "Just this day" or "Every day from now" (Q27).
- A promise can be made for any future date. It shows in Coming up and on the friend's page, joins the strip in its last 7 days, and gets a reminder 7 days before and on the morning, both editable (Q27).
- Coming up (tap the title on Today) lists everything ahead: this week by day, then later by month, then Repeating. A month calendar view is v1.1 (Q27).
- Missing one date is a miss for that date only.
- Skipping a date before its day is free, once in a row. A second skip in a row asks whether to end the regular promise.
- Moving one date onto a day that already has a date of the same regular promise is not allowed; the app offers to skip this date instead.

**v1.1 features (specs written after v1 is in testers' hands):** F13 invite and link, F14 nudge a friend, F15 shared promises For and With (including regular ones), F16 groups up to six, F17 mute, F18 winning back a lost streak (it spends stars; drawn in the V1.1 group of the final screens). The rules are already written on the board in section 29, card "For or With (decided)".

**Linking rules for v1.1 (decided, Q15):**
- An account is identified by its sign-in, which nobody else ever sees. Display names are not unique and do not need to be.
- No usernames, and no lookup by phone number or email. The only way to link is a single-use invite link that expires after 7 days.
- Both people confirm. The person who opens the link sees the inviter's name and avatar; the inviter then sees theirs and confirms it is the right person.
- A link is one shared thing. Either person can remove it, it is removed for both, and nobody is notified.
- A friend's history shows what you did with them (any bubble done), your promises (kept, broken, called off), your own nudges you let go (a faint line, "Your nudge · let go · Dec 3") and rhythms that ran out. Nudges from Nudge that passed or were dismissed are not shown; they are kept only for the 30-day no-repeat rule.
- Your history with a friend (stars together, catch-ups, pips, past promises, tagged notes) belongs to your friend entry and is never touched by unlinking. Linking again reconnects the same entry, so everything is still there.
- On unlink, an open shared promise stays with everyone who was in as an only-you promise, and disappears for anyone it was merely for.
- Removing a friend is different: it deletes your entry and your history with them, after a confirmation.

## 9. Guardrails

Instructions are preferences. Everything below marked **enforced** is checked by software, not by asking an agent nicely.

### The five layers
| Layer | What we do |
|---|---|
| 1 Instructions | One `CLAUDE.md`: stack pins, run and test commands, the words in section 3, the never-list below. Under 150 lines. |
| 2 Approvals | Shell approvals stay on. No skip-permissions mode. Agents cannot read `.env` files. |
| 3 Hooks (**enforced**) | A pre-tool hook blocks: force push, push to main, `prisma migrate reset`, any `DROP` or `TRUNCATE` outside the test database, and any command naming the production database URL. |
| 4 Outside the agent (**enforced**) | Branch protection on main: pull request required, all checks green, no force push. Secret scanning before commit and in CI. The agent's database user cannot connect to production at all. |
| 5 Isolation | Agents run with their own least-privilege credentials. Never the founder's cloud account or production keys. |

### CI gates on every pull request (**enforced**)
- Backend: typecheck, lint, unit tests, integration tests against a throwaway Postgres, dependency audit.
- iOS: build and unit tests. UI smoke tests only through the test backend.
- Contract: the generated Swift client is up to date with the OpenAPI file.
- Size: a warning, not a fail. If hand-written changes exceed 400 lines, CI warns and the PR needs the founder's explicit review before merging. Generated code, migrations, and the generated Swift client do not count toward the 400. The 400 is a heuristic for "still reviewable in one sitting", not an industry standard. A big change is split into stacked PRs (below), not split artificially to get under the number — gaming the count defeats the check.
- Images: fails if an image over 200 KB is added.
- Tests refuse to run unless the database name ends in `_test`. This rule exists because of the Nov 5 incident.

### Working rules
- **One spec, then one task, then one small pull request.** Spec, tasks and code are reviewed together.
- **Stacked PRs are how a big change is split.** One task = one PR, each PR based on the last one, merged bottom-up. Review each small piece in order instead of one giant diff at the end. If a change is legitimately over 400 lines (generated client, migration plus its tests, a large refactor), it is reviewed as an exception with a reason, not failed by CI.
- **One builder at a time until the skeleton is done.** After that, at most two, in different modules.
- **Main is never repaired forward.** A red main is reverted first, then fixed on a branch.
- **You review every diff.** An agent's "done" is not a review.
- **A mistake made twice becomes a check:** a lint rule, a test, a hook or a CI step.
- **A new table, route or dependency needs a line in a spec first.**
- **Done means** the acceptance criteria pass and the flow works on a physical iPhone, not only in the simulator.

### Never
Commit secrets. Delete or weaken a failing test to get green. Edit the production database by hand. Add a surface (web, helper, worker) that is not in section 4. Call an AI model from the phone.

## 10. Build order and exits

| Phase | What | Exit |
|---|---|---|
| 0 Spec and screens | This file reviewed. One clean set of final screens made from the board. Up Next motion prototype tuned on a phone. | Every blocking question in section 13 says DECIDED. Every v1 screen and state has a named frame. The motion numbers are written into the Up Next spec. |
| 1 Skeleton | New repo, CI gates, staging and production, auth, empty database, Sentry, TestFlight from CI, health checks, backup and restore. No features. | A signed-in empty app installs from TestFlight. A staged error shows in Sentry. A restore into staging has been done once. A rollback has been rehearsed. |
| 2 Core loop | F1 to F7, in that order. | You use it daily for a week with your real friends and nothing is lost. |
| 3 The rest of v1 | F8 to F12. | All section 8 criteria pass. |
| 4 Testers | 5 to 10 people. Watch them use it; fix confusion, not requests. | Each tester completes the core loop without help. |
| 5 Social | F13 to F17. | Two testers complete a shared promise end to end on two phones. |

## 11. How we work, feature by feature

### Before the first feature
- **One clean set of final screens.** Done: `nudge-v1-final.pen`. One frame per screen and state, with fixed names such as `UpNext/default` and `UpNext/holding`, all redrawn to the rules card at the top of the file. Each feature spec links the frames it implements.
- **Design values come from the file, not from pixels.** The builder reads the pen.dev file for exact colours, sizes and radii. They are written once to `design/tokens.json`, and Swift constants are generated from it. Pasted screenshots are welcome as extra reference.
- **A flow list per feature**, inside its spec: "tap bubble → sheet → save → Later". This is the storyboard. Nothing creates it automatically.
- **Motion is prototyped, not drawn.** Bubble drift, the spin and the celebration do not show in a screenshot. Up Next is built first as a throwaway prototype on a phone, tuned by feel, and the numbers are then written into its spec.

### The loop for each feature
1. **Spec.** The founder names the next feature. The builder drafts `specs/NNN-name/spec.md` (30 to 60 lines) from this file and the linked frames, and asks about anything unclear. The founder edits and approves.
2. **Tasks.** The builder writes `plan.md` and `tasks.md`. Each task is one visible behaviour, 15 to 30 minutes of work.
3. **Build.** One task per session, started with a short prompt such as "do task 3 of specs/002-friends". The result is one small pull request with a simulator screenshot attached.
4. **Review.** The checks in section 9 run. The founder reads the diff, compares the screenshot with the frame, and merges or comments.
5. **Phone check.** At the end of the feature the founder runs its acceptance criteria on a physical iPhone.

If building shows the spec is wrong, the spec is fixed and committed first, then the tasks are regenerated.

### Reviewing
- The GitHub pull request page is the place of record: diff, check results, screenshot and comments together.
- Neovim is home base for review and editing. The founder uses Neovim; `gh pr diff`, diffview.nvim and octo.nvim cover review in the terminal. Claude Code runs in the terminal, separate from the review tool — the agent writes, the founder reviews its output in Neovim.
- Cursor / VS Code is an optional second tool for a GUI diff on a specific big PR, not a replacement. SwiftUI previews and running on a real iPhone still need Xcode.
- Xcode is needed to build and run on the simulator and the phone, not to edit.

### Tasks and docs
- No Trello or Jira yet. `specs/NNN-name/tasks.md` is the plan. GitHub Issues / GitHub Projects is the board if a visual one is wanted — PRs link to issues automatically, and closing a PR closes the task. A separate board would become a second source of truth that drifts from the specs.
- Technical docs live in the repo, not in a separate wiki. Per feature: `spec.md` (what/why, acceptance criteria), `plan.md` (how, data, API), `tasks.md` (checklist). Plus the OpenAPI file as the contract, the `rules/` tests as executable docs for tricky logic, an ADR only for a big hard-to-reverse decision, and one README / runbook at the repo root (how to run, deploy, roll back).

## 12. What carries over from the current repo

| Keep | Leave behind |
|---|---|
| Bundle ID `so.trynudge.app`, the App Store and TestFlight setup, the domains | The web app |
| The landing site, unchanged | The Mac helper and message sync |
| The Railway deploy pattern: migrate before deploy, `/health/ready` | Shop, star purchases, equipped looks |
| The rule that tests can never reach a real database | Chat, relay, device pairing, user keys |
| AI call logging with daily and monthly caps | Recap, photo moments, moves |
| The starter looks that already exist, as image files | The avatar generation worker and its tables |
| The idea of a regular promise and its date rules | 30 or so of the 41 tables |
| The "one word per thing" habit | The 65 briefs, replaced by this file and `specs/` |

The old app keeps running untouched until the new one reaches the end of phase 3.

## 13. Open questions

Each needs your decision. Nothing here is decided by an agent.

**Q1. iOS in SwiftUI or React Native with Expo?** Decision: DECIDED, option A (2026-11-25)
- A. SwiftUI (recommended). Up Next is a continuously animating, gesture-driven glass surface, and Liquid Glass ships in SwiftUI first. The guide's own rule picks native for that case. Cost: you read Swift less fluently than TypeScript, so reviews are slower. Keeping all rules on the backend limits how much Swift there is to review.
- B. React Native with Expo. One language everywhere and fixes without App Store review. Cost: the glass and the bubble physics would be a copy of a moving target.

**Q2. Minimum iOS version?** Decision: DECIDED, option A (2026-11-25)
- A. iOS 26 only (recommended): real Liquid Glass, no fallback to maintain.
- B. iOS 18 and up with a plainer fallback: more phones, roughly double the visual work.

**Q3. Auth and database hosting?** Decision: DECIDED, option A (2026-11-25)
- A. Supabase for Auth, Postgres and avatar storage; the Fastify API on Railway stays the only thing the phone talks to (recommended). Buys sign-in, backups and file storage.
- B. Keep Railway Postgres and the hand-rolled Apple and Google sign-in. Nothing new to learn, but it is custom auth we own forever.

**Q4. If friends are ghosts until they link, v1 shows only ghosts. Is that acceptable for the first testers?** Decision: DECIDED, option A (2026-11-25)
- A. Yes for phases 2 and 3, and move invite-and-link (F13) to the front of phase 4 so testers get real avatars (recommended).
- B. Pull F13 into v1. Bigger v1, later testers.
- C. Bring back a starter look for unlinked friends. You already said no to this.

**Q5. New repository or clean branch of the old one?** Decision: DECIDED, option A (2026-11-25)
- A. New repository, old one archived read-only (recommended). The old history carries about 6,400 images and 1,400 branches.
- B. Same repository, new root. Keeps issues and history together.

**Q6. Your existing data.** Decision: DECIDED, option A (2026-11-25)
- A. Start empty, and write a one-off import for friends and notes only once F2 and F8 exist (recommended).
- B. Migrate everything. Most of it maps to tables that no longer exist.

**Q7. Avatars in v1.** Decision: DECIDED (2026-11-25). No generation in v1: users pick from the existing starter looks. Making an avatar from a photo is a later feature with its own spec, once the art style is chosen. That spec will need: the style reference, cost per avatar, a dollar cap, and a rule that the photo is deleted after the avatar is made.

**Q8. Timely nudges without message reading.** With the Mac helper gone, timely nudges can only come from notes and dates you enter. Decision: DECIDED, accepted for v1 (2026-11-25).

**Q9. How many builders, and who reviews?** Decision: DECIDED (2026-11-25): one builder at a time through phase 1, two at most after, and you approve every merge.

**Q10. Spec tooling.** Decision: DECIDED (2026-11-25): the GitHub Spec Kit folder layout (`specs/NNN-name/spec.md`, `plan.md`, `tasks.md`) as plain files; adopt its commands only if they help.

**Q11. Actions on the sheet.** Decision: DECIDED (2026-11-25), revised 2026-11-26. Tapping a bubble opens a menu beside it with the same actions as the drops, plus Edit; the edit sheet itself has no action buttons, and editing can never turn a nudge into a promise. Follow-up for the v1.1 spec: a promise from a friend needs "I'm in" and "No need, thanks", which the board currently draws as sheet buttons; decide then whether they become drops.

**Q12. Skipping one date of a regular promise.** Decision: DECIDED, option A (2026-11-25)
- A. Skipping before the day is free, at most once in a row; a second skip in a row asks whether to end it (recommended).
- B. Any skip is a miss.
- C. Skips are always free.

**Q13. A With promise that nobody joined.** Decision: DECIDED, the board's rule stays (2026-11-25). Now that a nudge can carry a day, asking ("Dinner Saturday?") is a nudge and commits nobody. A With promise is the maker's real commitment. The maker takes the miss if the day passes, and can cancel before the day for free.

**Q15. Linking rules.** Decision: DECIDED (2026-11-25): the rules listed under section 8, with invite links only and no usernames.

**Q16. Promise, rhythm and pip rules.** Decision: DECIDED (2026-11-25). Written into F2, F3, F4, F5, F6, F7 and F11. Drawn in section 32 of the board.

**Q14. The later avatar style.** Decision: OPEN. You named B1 and B1.1; I could not find a sheet with those labels, so point me to it when we get to that feature.

**Q17. What is left from sections 33 to 36.** Decision: OPEN. (c) the broken-promise pill shows once on the first open after midnight and leaves after 6 seconds or a tap; (d) the win-back price in stars (300 drawn); (f) the rules for a several-day promise: locked from its first day, broken if not done by midnight after its last day, stars for each day together up to 7 (proposed); (g) the Did it drop uses the kind's words ("We talked", "He has it"). (i) Show up now: founder direction (2026-11-26) is that it opens a sheet that encourages you to text, call, or go and do the thing, and to come back to the app when you're done; picked 2026-11-26: one sheet for each of the seven kinds plus the welcome-back sheet (C), in the final screens' Show up now group; this pulls opening Messages, the phone and Maps from the later list into v1; (j) expression art for the held friend (hopeful, glad, let down).

**Q18. How Nudge makes money.** Decision: OPEN. Founder direction (2026-11-26): free to use and useful for free; paid for things people understand as extra, such as special avatars, notes and AI memory kept past 30 days, and more AI use. Needs: what is free, what is paid, the price, and what the app does when the free AI is used up (today: Tell Nudge saves plain text as a note after the daily cap).

**Q19. Stars: prices and what they buy (v1.1 and later).** Decision: OPEN. The limits are decided (F7). Proposal for later: stars buy only things that cost Nudge nothing per use (avatar items drawn once, the streak restore); anything with a real cost each time (a new AI avatar, more AI) is money, never stars; colours stay free; worn items 300, held props 600, streak restore 300, seasonal 1,000 to 1,500; emotes earned by the friendship, not bought. Avatar items use way 2 (each item drawn once per pose and placed on the avatar), the founder's pick; a throwaway test app, built after phase 0 and capped at 2 to 3 days (founder OK, 2026-11-26), proves whether it looks good before the shop is specced: 6 avatars, 4 poses, 5 items, passing if about 80% of combinations look right without hand fixes.

**Q20. How ignored nudges end and interact.** Decision: DECIDED 2026-11-27, all items, written into F3, F4, F5, F5a, F7 and the friend section. (a) one bubble when a just-in-time and a rhythm nudge meet; (b) a broken promise's comeback nudge behaves like your nudge; (c) only Pick another day restarts your nudge's count; (d) what history shows; (e) the Tell Nudge main button and the tip; (f) the "↻ Keeps his rhythm" mark and wording, promises included; (g) one strike and one pip per friend per night; (h) rule A, a covering promise hides the rhythm nudge and it comes back the moment the promise stops covering; (i) "TODAY ONLY" / "UNTIL TONIGHT" for just-in-time nudges, no ring; (j) the rhythm bubble floats off when something else keeps it; (k) the main page is called Today.

**Q21. The main page: a day you can finish.** Decision: OPEN (2026-11-27). Founder concern: everything sits on Today as equal bubbles, so the day never feels finished. Proposal: only what costs a strike tonight is "Due today" (the finish line, with a count and the all-clear); rhythms coming due, just-in-time nudges and your nudges are "Ideas · no rush" that never block the all-clear; "Coming up" lists the next 7 days to do early or move, replacing the Later strip. Three layouts on the board "Exploring · the main page: due today and coming up": A stacked parts (recommended), B a strip of days, C the finish line first. Also open: your nudges stay their own reminder, scheduled on the phone so they always arrive. Decided 2026-11-27: an optional time of day on promises, your nudges and reminders comes in v1.

**Q22. Kinds that look different, reminders, and two times per plan.** Decision: OPEN (2026-11-27). Founder leans to layout B (a strip of days) and wants every kind to look distinctly different, in priority order, tested against a real week (board "Exploring · your real week, layout B"). Proposals: (a) round bubbles with a face are things with people that count (promise maroon glass, rhythm blue glass, idea from Nudge white glass, birthday gold glass); reminders are flat capsules with an icon and a box to check, no stars or strikes; (b) DECIDED 2026-11-27: reminders "just for me" (gym, church) are allowed; they live in the reminders tray and never count toward the streak; (c) every plan has when it happens (day, optional time) and any number of reminders (default that morning), each with its own day, time and words, shown on its own day as a capsule linked to the plan; (d) inside a day, due tonight leads, then things in time order, then anytime; (e) a daily promise (calling Rachel) shows on each day of the strip for the next 7 days and can be skipped for free from the strip before its day; on the day it counts, and passes cover rough days.

**Q23. When a day ends, and making the daily rule clear.** Decision: (a) to (c) DECIDED 2026-11-27 ("I agree with all these things"); (e) DECIDED 2026-11-27 ("yes to all three"); (d) proposed. Board: "Exploring · ending a day and the morning check-in". (a) the day runs midnight to midnight, but yesterday stays open until you next open the app (up to the end of today): if anything was left, the first screen asks once, "Yesterday: did you call Rachel?", and Yes counts for yesterday, so a call that ran past midnight or a forgotten tap doesn't cost anything (the alternative is Anki's way, the day ending at 4 am); strikes and passes are settled then; (b) Nudge's own bubbles (just-in-time nudges, rhythms entering their window) arrive once, in the morning; one Nudge learns about later in the day waits for tomorrow unless its moment is today, and then it shows marked NEW and never counts against clearing; anything you add yourself counts from when you add it; (c) the rule is taught in three places: one onboarding screen ("Clear your bubbles each day: do it, promise it, or push it"), a "2 left today" count by the date all day (maroon from 6 pm, with the flame's pulse), and the 6 pm notification only when something is left; (d) a pass is used automatically for an uncleared day, one per day, two a month, shown on the shield; at 0 the next uncleared day resets the streak (win-back in v1.1). (e) Details: at midnight unanswered bubbles leave the board and wait in one line at the top ("Friday · 1 to settle", with a Settle button); the streak chip shows a small dot meaning on hold, never broken at midnight; the check-in opens first on the next open (or from the morning summary, "Yesterday: 1 to settle"), one question per leftover: a promise asks "Did you…?" (Yes counts, No breaks it), a rhythm nudge offers "Did it yesterday" or "Move to today"; a just-in-time nudge, whose moment has passed, offers "Did it yesterday" or "Let it go" (decided 2026-11-27); any answer clears it; "Later" tucks it back into the line; it stays open until the end of the next day, and if still open then the day settles itself as not cleared and a pass is used; NEW late bubbles never appear in the check-in; a missed day of a daily or regular promise does not come back as an extra nudge (the next day's is already there).

**Q24. Showing that every bubble needs an answer today.** Decision: partly DECIDED 2026-11-27. Decided: the hard rule is that everything on Today is answered by the end of the day; a time on a bubble ("6 PM") only says when, like a reminder, and missing it doesn't matter; rings that mark only some bubbles are confusing, and a ring on every bubble says nothing; every bubble keeps its stamp on the rim (promise maroon with a handshake, rhythm blue with the repeat arrows, just-in-time blue with a calendar clock). Founder after trying it (2026-11-27): the layout B strip is clear but too small, the Today pill feels tiny, and neither the count nor the ring looks right yet; the faces above the title are liked (a face per friend, a count badge, capped with "+N"). Five new designs on the board "Exploring · the day strip"; founder picked 2 to try (2026-11-27): today is a wide card with today's faces (one per friend, a count badge when a friend has more than one bubble, "+N" past five) and "done / total"; other days show who's on them (two faces, then +N); tapping today's card on today opens "Today's people", each friend with a recognisable face, name, their bubbles and what's left, open ones first; tapping a friend opens their bubble's menu. The separate faces line above the title is off. On another day, today's card contracts to a pill like the others, still showing "done / total", with its faces small and stacked (founder: "it should still show the 1 out of 6 thing, but the faces should contract"). Before that, the prototype tried: the strip of days from layout B (board "the main page"), wide pills, six in view, the name and a number; today's pill says answered out of total ("1 / 3", a check when all are done) and updates as you go; other days say how many bubbles ("–" for none). The line above the title is the date, plus the hours left after 6 pm. To compare in tuning: a ring round today's pill as well; today's friends as small faces above the title (one face per friend, a badge when a friend has more than one bubble, "+N" past six), off by default because the pill already says the count.

**Q25. Moving between days.** Decision: OPEN (2026-11-27). Built in the prototype to try: tap a day in the strip; a sideways swipe anywhere on the board; when a day's bubbles scroll (5 or more), the swipe scrolls them first and at their end goes on to the next day; tap the title for the next 7 days as one list. One-time hints: "Hold a bubble and drag it to an action", then "Swipe sideways for other days". Founder (2026-11-27): changing day was too easy by accident; now the swipe must go 180 pt (founder's number) (the board follows at about half speed), it clicks when it passes the line, the next day's name waits at the edge and goes solid, and only past the line does it change day; the swap is clean, with no ghosts.

**Q26. Your nudges and reminders.** Decision: DECIDED 2026-11-27, option (a): no more your nudges. You make promises (committed, with stakes) and reminders (no stakes, can be for a friend); Nudge makes nudges, and you complete them. Follow-ons, proposed: the sheet's second button "Nudge me" becomes "Remind me" (a reminder for that friend on that day); the one-time tip becomes "Not sure yet? Remind me: no streak, no pressure."; Tell Nudge makes a promise when a day is given, otherwise a reminder; a broken promise's comeback (F4) becomes a nudge from Nudge for one day, answered like any (Did it, Promise, Dismiss), instead of behaving like your nudge.

**Q27. Repeating promises and promises far ahead.** Decision: DECIDED 2026-11-27, (a) to (d) as proposed ("yes to all three, go with your recommendations"); (d) is v1.1. Founder: "what happens with promises … that go on and on … because we only see the first 7 days?", "what happens if you make a promise way far in the future?", "it should be easy … to see what promises you have in the future, maybe even in a calendar view … definitely in a list view". Research: Google Calendar stores a repeating event once as a rule and draws each date from it; Todoist shows only the next date of a repeating task in its lists (all dates only in its calendar layout); Things keeps a template and makes a copy per date, marks copies with a small repeat icon, and asks "Make Exception" or "Update Rule" when one is moved. Proposal: (a) a repeating promise is stored once as a rule (as F12 already says) and has no end unless you give one; each date becomes a promise only when it enters the 7-day strip; beyond that it shows once, as the rule ("Call Rachel · every day · next today"), on the friend's page and in Coming up; changing one date asks "Just this day" or "Every day from now"; (b) a promise can be made for any future date (a wedding in June); it shows in Coming up and on the friend's page, joins the strip in its last 7 days, and gets a reminder 7 days before and the morning of, both editable; it stays free to move or call off until its day; (c) Coming up (tap the title) lists everything ahead: this week by day, then later by month, with repeating rules in their own section; a month calendar view is v1.1; (d) a friend's important dates (wedding, graduation, moving day) give just-in-time nudges like birthdays (F5b), proposed for v1.1.

## 14. Decision log

| Date | Decision | Source |
|---|---|---|
| 2026-11-25 | Q1 SwiftUI. Q2 iOS 26 only. Q3 Supabase for auth, database and storage. Q4 ghosts in v1, invite-and-link first in the social phase. Q5 new repository. Q6 start empty, import friends and notes later. Q8 timely nudges from notes and dates only. Q9 one builder at a time, founder approves every merge. Q10 Spec Kit folder layout as plain files. | Founder: "I agree with everything else", in reply to the recommendations. Reverse any of these by saying so. |
| 2026-11-25 | Regular promises are in v1. | Founder |
| 2026-11-25 | Stars are kept as a count. No rewards or spending for now. | Founder |
| 2026-11-25 | Q7 no avatar generation in v1, users pick from the existing looks. Q12 skipping a regular promise date is free once in a row. Q13 a With promise's maker takes the miss, and can cancel before the day for free. Q15 linking by single-use invite link only, both confirm, unlink removes it for both, history is never touched so linking again shows everything. | Founder: "I like all these ideas." |
| 2026-11-25 | Q11 tapping a bubble only edits it. Hold and drag is the only way to mark done, promise or dismiss. The "Promise it" row is removed from the nudge sheet; the day row only sets the optional day. | Founder |
| 2026-11-25 | The way of working in section 11: clean screen set first, per-feature spec then tasks then small pull requests, motion prototyped on a phone. | Founder: "I like all these ideas." |
| 2026-11-25 | All clear: the bloom moment plays once a day, then the quiet state stays until midnight. Haptic numbers added to F3 as starting values. The collapse-and-blink ending was the founder's idea and is judged in the prototype. | Founder: "I like allclear/moment-bloom", "rest-quiet looks good", "it should only show once and then it should clear for the rest of the day", "you can add haptic numbers yes". |
| 2026-11-25 | Two kinds of nudge: just-in-time and rhythm (was catch-up). Just-in-time nudges have no penalty and no reroll; rhythm nudges can be rerolled and appear before the rhythm runs out. Rings mean time left today; pips mean the relationship. A daily streak replaces the weekly one: a strike is a broken promise or a rhythm that ran out, two strikes a month are forgiven, the third resets it. A broken promise takes one pip; earning it back takes three kept promises on three separate days. Promises are free to change before their day and strict on it. 50% bonus stars for a promise made a day or more ahead and kept. An evening-before review notification. Later is one running list with no week sections. Moving a regular promise date onto another of its dates offers a skip. | Founder, in conversation: "Yes, let's make it daily for the streak", "a rhythm that is running out should count as a streak strike", "I agree with the bonus", "there should also be a reminder the day before". |
| 2026-11-25 | Q16: on its day a promise's what, day and counts-as are locked; a broken promise comes back as a nudge; rhythm nudges have Did it, Promise and Not today, no Dismiss, reroll in the sheet; one pip off per 7 days overdue, never below 1; new friends start at 3 pips; Up Next is today only; Friends sorted by who needs you now, coming up, then fewest pips; notifications are the evening-before review and a 6pm one only if something costs a strike tonight. | Founder: "Yes, all this makes sense to me." |
| 2026-11-26 | A promise bubble has no rim of its own; the maroon time-left ring is its only mark. Notification times live in Settings › Notifications, each reminder with its own switch and time, defaults 8pm and 6pm. | Founder: "8 pm in settings is a good baseline", "get rid of the inner red ring … and just keep the red ring on the outside". |
| 2026-11-26 | Reroll is the dice on the idea with a menu (B). Friends list is layout A: Needs you includes 1–2 pips, This week means 7 days, it always opens on Who needs you. Notifications are person first, short titles, streak not the lead. Streak chip combines shields and milestone colours, with a gold halo for a perfect month. A lost streak can be won back and the calendar shows it. Kept your word plays a second beat. Bubbles that cost a strike tonight get an hourglass. | Founder: "I especially like the B dropdown menu", "let's go with friends/list A", "let's go with the first, the person first", "a combination of 2 and 3 for streak … I especially love the gold halo", "you should be able to win back the streak", "I love the celebration … kept your word". Hourglass: founder suggested it, I agreed. |
| 2026-11-26 | Counts-as is one of the seven kinds from 2.19b, filled in by Nudge from the what until the user picks it, shown as "···" while reading. Winning back a lost streak is a kept promise with that friend, then stars, never money. The Friends empty line is "You're caught up with everyone." | Founder: "these are the ones that we should use", "I agree with the behavior of what nudge picks counts as", "Agreed with paying with stars not money", "nobody needs you right now is honestly such a brutal response". |
| 2026-11-26 | The kind picker is a menu from the kind line, not a sheet. A streak at risk is shown by the flame flickering after 6pm, not by an hourglass on the chip. | Founder: "should we just be updating the dropdown menu", "it should just start flickering". |
| 2026-11-26 | Up Next header is fix 1: date on its own line, title and chips on one row, never overlapping. Streak and stars stay separate pills. The background runs behind the whole screen and is never clipped. | Founder: "let's go with header fix 1", "not the biggest fan of the combined pill look", "the background should just keep expanding the whole way". |
| 2026-11-26 | Streak chip shows passes as one shield with the number left inside. Winning back a streak broken by two strikes on one night needs a kept promise with each friend. A call earns a flat 20 stars. | Founder: "B, each friend, and flat 20 for calls". |
| 2026-11-26 | Help and Meet up promises can cover several days in v1. Opening apps from a bubble is later and noted. Place shows only for Help and Meet up. | Founder: "I think several days. Opening apps can come later as a later feature, but we should note it." |
| 2026-11-26 | Final screens approved as the set to build: `nudge-v1-final.pen`, made from sections 30 to 37, older versions dropped. Winning back a streak moves to v1.1. Star limits (once per friend a day, 100 a day) and the "still counts" wording approved. Avatar items: way 2, tested first. | Founder: "Any adjustments I've made over each of the sections in Pen.dev, everything that I've approved, those are the ones we should go with. Anything old or out of date should not be kept.", "let's move it to 1.1", "The two limits look good to me", "Do number 2". |
| 2026-11-26 | "We caught up" on a friend's row makes a done bubble on the spot. The throwaway avatar test app is built after phase 0. Every final screen redrawn to the rules card after the founder found the older ones broken. | Founder: "I agree with the we caught up creating a done bubble on the spot", "after phase zero, we can build the throwaway avatar test". |
| 2026-11-26 | The redrawn final screens in `nudge-v1-final.pen` are approved. | Founder: "yeah its good enough". |
| 2026-11-26 | `pip_change` and `strike` tables added to the v1 data model. Spec marked reviewed. | Founder: "yes add both tables, mark it reviewed, and commit". |
| 2026-11-26 | From the motion prototype: four bubbles in view with a fifth peeking, free scrolling with soft ends and no dots, soft physics with least-movement refills. Hold and drop is pills only (no body outline, nothing at the feet), different actions for a promise (Show up now, Already showed up, Call it off) and a nudge, the hovered action jumps to the top with a ghost, a 0.6 s silent hold when lifting on an action, and tap opens a menu beside the bubble. | Founder, in conversation: "Let's go with jumps up", "the holding to keep should exist … 0.6 seconds", "I like the tap B", "just stick with the pill, no circle, no thing on the bottom". |
| 2026-11-26 | Show up now opens a sheet that sends you to text, call or go do it, and welcomes you back. A one-time hold-and-drag hint for new users. After 6pm the flame and streak number pulse instead of flickering; the flame always glows gently; the star loses its white outline and glows. | Founder: "it should open a tab for you and encourage u to either text or call or go and do the thing and come back", "sure on c", "after 6pm the flame + streak number should pulse fading in and out". |
| 2026-11-25 | A nudge can have an optional day. Committing, not the day, is what makes a promise. Nudges from Nudge, nudges from a friend and promises share one sheet and one set of fields. | Founder pushed back on "a nudge never has a day"; I agree and withdrew it. |
| 2026-11-26 | Hold still 0.2 s to lift (moving sooner scrolls, letting go sooner taps); 0.3 s on an action before it is live; 55 pt of finger travel per bubble; landing spring 0.2 s, damping 0.7. The "N more" pill stretches into "Swipe to see N more". | Founder, tuned on the phone: "hold before it lifts should be 0.2 seconds so it doesnt feel heavy", "Finger travel per bubble: 55 points", "Settle speed: 0.20 seconds", "Settle bounce: 0.70". |
| 2026-11-26 | Holding layout B: the actions sit beside the friend on the right, never on the face. Show up now: the seven kinds plus the welcome-back sheet. AI text drafts move into v1 (out of the non-goals). | Founder: "I think B is the best, for sure", "Yes, text can draft messages with AI. I think we should allow that option", "I'm aligned on everything else". |
| 2026-11-26 | Your own nudges: a "Nudge me" button and a "YOUR NUDGE" label; a time of day comes in v1.1. Stars: at most 400 a week from Monday, still once per friend a day, with a "Full week" moment. Perfect month: slightly more embers, no gold halo. | Founder: "there should be a Nudge Me button. There should be a Your Nudge label, and we can do time later in 1.1, but we should definitely do it", "400 a week, starting fresh every Monday", "The perfect month should be just slightly more embers". The Full week moment was my proposal, accepted with "aligned on everything else". |
| 2026-11-26 | Birthdays: on the friend (month, day, optional year), from the contact or typed; birthday nudges on by default, a week before and on the day; they show beside a rhythm nudge for the same friend, the one exception to one nudge per friend. Sharing your own birthday is v1.1. | Founder: "I'm for the birthday design", "birthday nudges should be automatically enabled", "Optional is fine for that", "Agreed on showing both … birthdays and maybe special events are an exception to that rule". |
| 2026-11-26 | Holding: Did it (Already showed up on a promise) moves high on the right beside the head, Promise (Show up now) lower on the right, with clear space between. Picked 2026-11-26 (Did it on top, Claude's recommendation). | Founder: "if I'm a right-hand user and I hold onto the bubble, the Didit is hidden behind my thumb … I need it to be on the right-hand side but above Promise, or actually, maybe Promise is above Didit still. I'm not sure which one is better". |
| 2026-11-26 | Did it above Promise on the right is picked. A nudge gets Show up now first in its tap menu and "Show up now instead" on the promise sheet, with no fourth drag pill (research: hidden gestures are found far less than visible options; doing it now beats planning it). Your nudge, ignored after "Still want this?", goes away quietly on day 7 and stays as a faint line in the friend's history. | Founder: "Sounds good for all of them … go with the recommended. Let's not overengineer anything for the ignored Nudge, quietly going away on day 7. It should show up in the friends history." |
| 2026-11-27 | Streak flame and star art: five flame colours (starter 0–6 days, orange 7+, gold 30+, blue 100+, violet 365+), one shape. Perfect month is a white-gold ring with two wisps, no badge, replacing "slightly more embers". The streak going up plays when tonight's last at-risk thing is done (three wisps) and as a small hop on the next first open; milestones 3, 7, 14, 30, 50, 100, 200, 365, then every 100, with four wisps and the colour change. The grey shield at 0 stays. The star glows at rest and glows brighter when stars land; kept your word repeats that. Art from Higgsfield, cut out and kept as an asset catalog under 200 KB per image; the masters stay outside git. | Founder, reviewing the icon sheets: "it should be a blue flame", "should there be a small flame when they first start", "at resting, it could look like the star is an earned star", "Agreed with everything", "ok good enough". Ring over embers: founder asked whether the halo was clear enough and agreed to a visible ring after I pointed out embers are too small to read at header size. |
| 2026-11-27 | Q20 (a) to (c): a just-in-time nudge and a rhythm nudge for the same friend become one bubble; a broken promise's comeback nudge behaves like your nudge and need not be done; only Pick another day restarts your nudge's count. | Founder: "Yes, on A", "It should behave like a nudge, and they don't have to remember to keep the streak going", "Yeah, only pick another day should start it over". |
| 2026-11-27 | At most one strike and one pip per friend per night. | Founder: "it should be at most one strike and one pip for a friend in one day. There shouldn't be more than that." |
| 2026-11-27 | Q20 (d) to (k) and the birth-year switch: history rule; Tell Nudge main button by firm day; tip when leaving the promise sheet; the "↻ Keeps his rhythm" mark in the rhythm's blue and icon (research: similarity groups things that share colour and shape; WCAG 1.4.1, never colour alone), shown on promises too; rule A for covering promises; "TODAY ONLY" labels; the rhythm bubble floats off; Up Next renamed Today; the birth year shared only by its own switch. | Founder: "I'm overall aligned", "with all the other changes that you mentioned, I agree with them as long as it helps someone navigate the app easily and understand it", "remember that promises can also count towards it". |
| 2026-11-27 | On screen, rhythms and promises are "due" ("Rhythm due tonight", "Rhythm 9 days overdue"); "runs out" is not used. | Founder: "Yeah, we should just use 'do'" (due). |
| 2026-11-27 | An optional time of day on promises, your nudges and reminders moves from v1.1 into v1. | Founder: "time of day should come in in V1." |
| 2026-11-27 | The streak counts days you cleared (everything due done, every other bubble answered); reminders stay out of it; stars stay visible. | Founder: "maybe it should be based on clearing a day", "It also needs to be clear that the bubbles have to be cleared on that day ... Otherwise, it breaks their streak", "I agree that reminders should stay out of the streak", "I think stars should still show. They're very motivational." |
| 2026-11-27 | The morning check-in as drawn (opens first, Later, open until the end of the next day, settles itself as not cleared). A missed day of a daily or regular promise does not come back as an extra nudge. Reminders just for you live in the tray and never count. | Founder: "yes to all three". |
| 2026-11-27 | Every bubble needs an answer to clear the day, just-in-time nudges included (Dismiss answers it). Calling off a promise on its day asks first and shows the pip and the streak. On screen, no "free" or "cost": "won't count against you". Short day names in the title. | Founder: "I agree with the just-in-time thing: every bubble should need an answer.", "there should be some sort of 'Are you sure you want to do this?'", "I don't like that we're using cost as the word free", "we could just write Sun". |
| 2026-11-27 | Everything on Today must be answered by the end of the day (the hard rule); a time on a bubble only says when. Rings on only some bubbles go. Calling off names the streak. Day names are three letters everywhere. Changing day by swiping needs more distance, a click and the next day's name. | Founder: "everything needs to be done by the end of the day. That's the hard rule", "That time is more of just a reminder of when they need to do it by", "the ring doesn't really make sense", "Calling off the promise should mention the streak", "it should feel harder or more resistant". |
| 2026-11-27 | Your nudges are dropped (Q26 a): you make promises and reminders, Nudge makes nudges. Stamps stay on every bubble. The strip of days from layout B with today's count is tried in the prototype. A haptic rumble whenever the streak changes, bigger on a milestone. Swipe to change day: 180 pt. Swipe up on the reminders tray opens it. | Founder: "they should just be reminders, like you said", "you're forgetting the badges", "we could use what we have in … layouts B: the strip of days", "haptic feedback or some sort of fire rumbling thing whenever the streak changes", "around 180 points", "swipe up on the reminders tab to open it". |
| 2026-11-27 | Moving, promising for later, calling off or dismissing a bubble takes it out of today's count (0 of 3 becomes 0 of 2). Undo shows a ring that runs down (6 s to start). The streak plays a fire sound with its rumble. First-time hints are brief glass chips at the top, not a chip on the bubble. Bubbles follow a day swipe at 0.35. | Founder: "It should be one of two … It shouldn't be one for three", "there should be a timer for how long we can undo … like a ring around the pill", "there should also be sound, like a fire sound", "I don't really like the hold and drag thing … Maybe it should show up briefly at the top", "I did 0.35x". |
| 2026-11-27 | Dismiss also takes a bubble out of today's count: the count is only what you did for someone. Today's card contracts on other days. The toast is small again, its countdown on the Undo button. A day swipe gives one click, not two. "Drag onto an action" is plain words with a drag icon, not a pill. The milestone sound has no chime. | Founder: "today's count should only count if you did something for that person", "it shouldn't stay at that size", "the notification pill looks a little too big", "it shouldn't be double haptic feedback", "I'm not sure if it should be plain text as opposed to a pill", "get rid of the little ding … It sounds kind of corny". |
| 2026-11-27 | Q27: a regular promise is one rule with no end unless given, shown once beyond the 7-day strip; promises can be made for any future date, with reminders 7 days before and on the morning; Coming up is a list in v1, a calendar in v1.1; friends' important dates give nudges in v1.1. In the check-in a just-in-time nudge offers "Let it go" instead of "Move to today". | Founder: "yes to all three, go with your recommendations". |
