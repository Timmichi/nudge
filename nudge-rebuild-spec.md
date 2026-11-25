# Nudge rebuild: spec for review

Status: DRAFT, revision 4 after founder feedback on 2026-11-25. One question remains open (Q14, the later avatar style); it does not block anything in v1. No feature code is written until the phase 0 exits in section 10 are met.
Sources: the redesign board (`nudge-ux-rethink.pen`, sections 20 to 29), the field guide "Building a production app with AI in 2026", and a read of `nudge-ai` at `origin/main` (45ea8df4, 2026-11-23).

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
- **Core loop:** open Up Next → do one bubble (text, call, help, meet up) → mark it → stars and a celebration.

Everything outside that loop is scope that has to earn its place.

## 3. Words (one per thing, same on screen, in the API and in code)

| Word | Meaning |
|---|---|
| Friend | Someone the user keeps up with. |
| Bubble | One thing to do for a friend. Every bubble is a nudge or a promise. |
| Nudge | A suggestion nobody has committed to. From Nudge (catch-up or timely) or, later, from a friend. It can have a day, which means "when", never a deadline. No penalty if it passes. |
| Promise | A bubble the user committed to. It always has a day. Only you, For a friend, or With a friend. Committing is what makes it a promise. |
| Regular promise | A rule that makes one promise for each date it names ("Basketball, Tuesdays"). You keep the promise it made, never the rule. |
| Rhythm | The longest the user wants to go without catching up with a friend. One per friend. |
| Later | The strip of days after today that hold promises. |
| Note | One per day, plain text, lines can be tagged to a friend. |
| Star | Points earned only by completing a bubble. A running count. Nothing to spend them on yet. |
| Ghost | A friend who is not on Nudge: a soft white shape instead of an avatar. |
| Link | The connection between a user's friend entry and that friend's own account. |

Retired words: idea, reach-out, pick, move, deck. They do not appear in new code. In code a regular promise is `regular`.

## 4. Scope

### In v1 (single player, to 5 to 10 testers)
Sign in, friends, rhythm, Up Next bubbles, nudges from Nudge, promises with a day, regular promises, Later, hold and drop, stars and streak, notes, Tell Nudge (type or speak), your avatar picked from the looks that already exist, reminders, account deletion.

### In v1.1 (social, after testers have used v1)
Invite and link, ghosts turning into real avatars, nudge a friend, shared promises (For and With), groups up to six, mute.

### Non-goals (not built, not stubbed)
- Android, and a web version of the app.
- The Mac helper and any reading of Messages or WhatsApp.
- In-app chat, read receipts, a feed.
- Shop, closet, buying stars, any payments, and any reward that stars unlock.
- Making an avatar from a photo, and any avatar generation. This comes back as its own feature once the art style is settled.
- Calendar view, widgets, photo moments, recap.
- AI-written message drafts.
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
- **Two jobs only in v1.** (1) Tell Nudge: turn what the user typed or said into a proposed promise or note. (2) Timely nudges: read note lines that are tagged to a friend and propose a nudge when one is worth acting on.
- **It proposes, the user decides.** Nothing the AI produces is saved as a promise or changes a note without a tap from the user. Nudges are suggestions already, so they need no confirmation.
- **What it is sent:** the Tell Nudge text, the user's friend names, and note lines tagged to a friend. Note lines with no friend tag are never sent to the AI.
- **Where the key lives:** the Anthropic key is a secret on the server. The phone calls our API; our API calls Anthropic. A key inside the app could be pulled out of the app file and used to spend our money.
- **Per-user daily cap:** each user gets a fixed number of AI calls a day (40 today). After that Tell Nudge still works but saves plain text as a note. This stops one user or one bug from running up the bill.
- **Monthly cap:** a ceiling on calls for the whole app (20,000 today). At the cap, AI features switch off for everyone and the rest of the app keeps working. This is the worst-case bill, chosen in advance.
- **Every call is logged** with its purpose and cost, so the caps are set from real numbers.

## 6. Data model for v1 (9 tables, down from 41)

| Table | Holds |
|---|---|
| `user` | Account, time zone, chosen look, celebration choice. |
| `friend` | Owner, name, rhythm in days, last caught up date, optional contact handle. |
| `bubble` | Owner, friend, kind (`nudge` or `promise`), nudge type (`catch_up` or `timely`), text, details, counts as (`text`, `call`, `help`, `meet_up`), day (required for a promise, optional for a nudge), state (`open`, `done`, `dismissed`, `missed`), source, regular it came from. |
| `regular` | Owner, friend, text, counts as, pattern (weekly on a weekday, every N weeks, monthly on a date), starts on, ended on. |
| `note` | Owner, date (unique per user per day), title, body. |
| `note_tag` | Note, line range, friend, confirmed or suggested. |
| `star_event` | Owner, friend, bubble, amount, date. A ledger; totals are computed. |
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
- `POST /bubbles`, `PATCH /bubbles/:id`, `POST /bubbles/:id/done`, `POST /bubbles/:id/dismiss`, `POST /bubbles/:id/promise`
- `GET /notes?from=&to=`, `PUT /notes/:date`
- `POST /tell` (text in, proposed promise or note out; nothing saved until confirmed)
- `POST /devices`
- `GET /regulars`, `POST /regulars`, `PATCH /regulars/:id`, `POST /regulars/:id/end`, `POST /regulars/:id/skip`
- `GET /looks`
- `GET /health`, `GET /health/ready`

## 8. Features and acceptance criteria

A criterion counts only if a test or a named manual check can prove it. Each feature gets its own spec file before it is built.

Product rules below come from the board. The numbers that do not (time limits, frame rate, the 30-day rule, the streak rule in F7) are my proposals, there so each criterion can be tested. Change them freely.

**F1 Sign in and account**
- A new user signs in with Apple and lands on an empty Up Next in under 10 seconds on a physical iPhone.
- A request without a valid session returns 401. A request for another user's row returns 404. Both are tested with two accounts.
- Delete account removes every row for that user within 60 seconds and signs the device out.

**F2 Friends and rhythm**
- Adding a friend needs only a name. The Friends list shows them at once.
- The list is plain rows with no swipe-left actions. Holding a row opens the menu from section 27 of the board.
- Changing rhythm from the ⋯ menu changes when the next catch-up nudge is due, covered by rule tests.

**F3 Up Next**
- With 5 or fewer open bubbles for today, all are visible with the most important nearest the middle.
- With more than 5, a swipe spins to the next set, and the dot strip shows at most 9 dots.
- Order comes from the server. Given the same data, the order is identical on every load (rule test with 20 fixtures).
- The cluster holds 60 fps or better with 10 bubbles on an iPhone 15, measured in a release build.

**F4 Promises and Later**
- A promise cannot be saved without a day.
- A promise for a later day shows in the Later strip and not in today's bubbles; on its day it becomes a bubble with a thin maroon rim.
- Marking it kept awards stars once. A second tap awards nothing.
- If its day ends still open, it becomes `missed` at the user's local midnight and "kept in a row" for that friend resets.

**F5 Nudges from Nudge**
- A catch-up nudge appears when today minus last caught up is greater than or equal to the friend's rhythm.
- At most one open nudge per friend at any time.
- A dismissed nudge is never recreated with the same text for that friend within 30 days.
- A nudge can have a day. Its label shows it ("NUDGE · THU"). At the end of that day an untouched nudge leaves quietly: no miss, no effect on any streak.
- Dropping a nudge that has a day onto Promise fills that day in on the promise sheet.
- A nudge and a promise use the same sheet and the same fields: what, details, day, place, counts as. The only differences are who made it and whether anyone has committed.
- The Later strip holds promises only. A nudge with a day floats in Up Next from the moment it exists, so Nudge creates its own dated nudges no more than 2 days ahead.

**F6 Hold and drop**
- Holding a bubble shows the friend large and three drops: Did it, Promise, Dismiss.
- Dropping on Promise opens the promise sheet with the text filled in.
- The other bubbles stay faintly visible while holding.
- Hold and drag is the only way to mark a bubble done, promise it or dismiss it on Up Next. Tapping a bubble opens its sheet for editing only; the sheet has Save and Cancel and no action buttons. A nudge's sheet has no "Promise it" row: its day row only adds, changes or clears the optional day, and the nudge stays a nudge.

**F7 Stars and streak**
- Text 5, call 20, help 40, meet up 40. One scoring event per friend per day.
- Stars come only from completing a bubble. No other action awards them.
- The weekly streak holds if every friend whose rhythm came due that week was caught up with.

**F8 Notes**
- One note per day. Opening Notes on a new day creates today's note titled with the date.
- Typing a friend's name offers a dotted tag; confirming it links that line to the friend. The note's words are never changed by the app.
- A note can be about anything. Lines with no friend tag are stored and searchable, are never sent to the AI, and never produce a bubble.
- A friend's page lists every line tagged to them across all days, newest first.

**F9 Tell Nudge**
- Typing or speaking one clear thing ("dinner with Noah Saturday") creates one proposed promise that the user confirms with one tap.
- Several things in one message produce one review card listing each item. Nothing is saved until confirmed.
- If the AI call fails or the daily cap is reached, the text is kept and the user sees "Saved as a note for now".

**F10 Your avatar and celebration**
- A user picks one look from the existing set and can change it at any time. No image is generated.
- When stars are earned, the celebration the user chose plays once.
- Friends who are not linked show the ghost.

**F11 Reminders**
- A promise sends one notification on its day at the user's chosen time. Completing it earlier cancels the notification.

**F12 Regular promises**
- On the promise sheet the day row offers "Repeats": weekly on a weekday, every N weeks, or monthly on a date.
- Each date becomes its own promise. The next one shows in Later no more than 7 days ahead, and becomes a bubble on its day.
- Keeping one counts as catching up, so no catch-up nudge appears for that friend while the regular promise is being kept.
- Ending a regular promise removes future dates and leaves past ones as they were. Ending is never a miss.
- Missing one date is a miss for that date only.
- Skipping a date before its day is free, once in a row. A second skip in a row asks whether to end the regular promise.

**v1.1 features (specs written after v1 is in testers' hands):** F13 invite and link, F14 nudge a friend, F15 shared promises For and With (including regular ones), F16 groups up to six, F17 mute. The rules are already written on the board in section 29, card "For or With (decided)".

**Linking rules for v1.1 (decided, Q15):**
- An account is identified by its sign-in, which nobody else ever sees. Display names are not unique and do not need to be.
- No usernames, and no lookup by phone number or email. The only way to link is a single-use invite link that expires after 7 days.
- Both people confirm. The person who opens the link sees the inviter's name and avatar; the inviter then sees theirs and confirms it is the right person.
- A link is one shared thing. Either person can remove it, it is removed for both, and nobody is notified.
- Your history with a friend (stars together, catch-ups, kept in a row, past promises, tagged notes) belongs to your friend entry and is never touched by unlinking. Linking again reconnects the same entry, so everything is still there.
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
- **One clean set of final screens.** The board has 29 sections and most are superseded, so a builder pointed at it would copy the wrong thing. The set is one frame per screen and state, with fixed names such as `UpNext/default` and `UpNext/holding`. Each feature spec links the frames it implements.
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

**Q11. Actions on the sheet.** Decision: DECIDED (2026-11-25). Tapping a bubble opens the sheet to edit it and nothing else, and editing can never turn a nudge into a promise. Did it, Promise and Dismiss exist only as hold-and-drag on Up Next. Follow-up for the v1.1 spec: a promise from a friend needs "I'm in" and "No need, thanks", which the board currently draws as sheet buttons; decide then whether they become drops.

**Q12. Skipping one date of a regular promise.** Decision: DECIDED, option A (2026-11-25)
- A. Skipping before the day is free, at most once in a row; a second skip in a row asks whether to end it (recommended).
- B. Any skip is a miss.
- C. Skips are always free.

**Q13. A With promise that nobody joined.** Decision: DECIDED, the board's rule stays (2026-11-25). Now that a nudge can carry a day, asking ("Dinner Saturday?") is a nudge and commits nobody. A With promise is the maker's real commitment. The maker takes the miss if the day passes, and can cancel before the day for free.

**Q15. Linking rules.** Decision: DECIDED (2026-11-25): the rules listed under section 8, with invite links only and no usernames.

**Q14. The later avatar style.** Decision: OPEN. You named B1 and B1.1; I could not find a sheet with those labels, so point me to it when we get to that feature.

## 14. Decision log

| Date | Decision | Source |
|---|---|---|
| 2026-11-25 | Q1 SwiftUI. Q2 iOS 26 only. Q3 Supabase for auth, database and storage. Q4 ghosts in v1, invite-and-link first in the social phase. Q5 new repository. Q6 start empty, import friends and notes later. Q8 timely nudges from notes and dates only. Q9 one builder at a time, founder approves every merge. Q10 Spec Kit folder layout as plain files. | Founder: "I agree with everything else", in reply to the recommendations. Reverse any of these by saying so. |
| 2026-11-25 | Regular promises are in v1. | Founder |
| 2026-11-25 | Stars are kept as a count. No rewards or spending for now. | Founder |
| 2026-11-25 | Q7 no avatar generation in v1, users pick from the existing looks. Q12 skipping a regular promise date is free once in a row. Q13 a With promise's maker takes the miss, and can cancel before the day for free. Q15 linking by single-use invite link only, both confirm, unlink removes it for both, history is never touched so linking again shows everything. | Founder: "I like all these ideas." |
| 2026-11-25 | Q11 tapping a bubble only edits it. Hold and drag is the only way to mark done, promise or dismiss. The "Promise it" row is removed from the nudge sheet; the day row only sets the optional day. | Founder |
| 2026-11-25 | The way of working in section 11: clean screen set first, per-feature spec then tasks then small pull requests, motion prototyped on a phone. | Founder: "I like all these ideas." |
| 2026-11-25 | A nudge can have an optional day. Committing, not the day, is what makes a promise. Nudges from Nudge, nudges from a friend and promises share one sheet and one set of fields. | Founder pushed back on "a nudge never has a day"; I agree and withdrew it. |
