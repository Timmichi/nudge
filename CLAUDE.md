# CLAUDE.md — Nudge

`nudge-rebuild-spec.md` is the source of truth. If building proves it wrong, the spec is fixed and committed first, then the code. Each feature gets its own spec in `specs/NNN-name/` (spec.md, plan.md, tasks.md) before it is built.

## Stack (pinned; do not change without asking the founder)

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

## Words (one per thing, same on screen, in the API and in code)

| Word | Meaning |
|---|---|
| Friend | Someone the user keeps up with. |
| Bubble | One thing to do for a friend. Every bubble is a nudge or a promise. |
| Nudge | A suggestion nobody has committed to. Only from Nudge or, later, from a friend; you don't make nudges for yourself. Two kinds: just-in-time (from something Nudge noticed; never rerolled) and rhythm (from a friend's rhythm; can be rerolled). Like every bubble it needs an answer by the end of its day. |
| Pips | Five marks per friend showing how the relationship is going. |
| Streak | Days in a row you cleared: every bubble on Today answered, everything due done. Two passes a month cover uncleared days. |
| Promise | A bubble the user committed to. It always has a day. Only you, For a friend, or With a friend. Committing is what makes it a promise. |
| Regular promise | A rule that makes one promise for each date it names ("Basketball, Tuesdays"). You keep the promise it made, never the rule. |
| Rhythm | The longest the user wants to go without catching up with a friend. One per friend. |
| Today | The main page: today's bubbles, a strip of days, and the reminders tray. |
| Coming up | Everything after today as one list: this week by day, later by month, then repeating promises. |
| Reminder | Something to remember, for a plan or just for you. No stars, no streak. Lives in the reminders tray, never a bubble. |
| Note | One per day, plain text, lines can be tagged to a friend. |
| Star | Points earned only by completing a bubble. A running count. Nothing to spend them on yet. |
| Ghost | A friend who is not on Nudge: a soft white shape instead of an avatar. |
| Link | The connection between a user's friend entry and that friend's own account. |

Retired words: idea, reach-out, pick, move, deck, catch-up nudge, timely nudge, kept in a row, weekly streak, Up Next, Later, your nudge. They do not appear in new code. In code a regular promise is `regular`.

## Run and test

- Run the API: TODO (Phase 1)
- Run the iOS app: TODO (Phase 1)
- Backend tests: TODO (Phase 1)
- iOS tests: TODO (Phase 1)
- Regenerate the Swift client from OpenAPI: TODO (Phase 1)

## UI components (Q28)

- Screens are built only from NudgeUI components and its tokens (colours, type, spacing, radii, glass, motion). No one-off colours, fonts or sizes in a screen.
- NudgeUI stays small (about a dozen components). A new component, or a change to how one looks, goes in its own pull request with the code, a preview of every state, screenshots, and the matching component on the board's Components page. Tim approves it before anything uses it.
- Snapshot tests lock each approved component. A failing snapshot is a design change: it needs the same approval, never a re-record to get green.

## Never

- Commit secrets.
- Delete or weaken a failing test to get green.
- Edit the production database by hand.
- Add a surface (web, helper, worker) that is not in section 4 of the spec.
- Call an AI model from the phone.
