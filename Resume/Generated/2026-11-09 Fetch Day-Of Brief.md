# Fetch — Day-of brief (panel, 2026-11-09)

- **Created**: 2026-11-09, 9:45 ET
- **Company**: Fetch (Fetch Rewards)
- **Schedule (ET)**: 2:30 Omar Al Masri, hiring manager. 3:30 Ricky Witherspoon, iOS systems design plus AI integration. 4:30 break. 5:00 Brandon Erbschloe, coding, Swift concurrency. One Zoom link for all three, in Liz Shadd's 2026-11-04 confirmation email.
- **Sources**: [[2026-11-03 Fetch Interview Prep - Round 2]], [[2026-11-05 Fetch STAR Stories]], [[Isolation Review Matrix]], [[2026-10-30 Fetch Interview Prep]], and Fetch's own `/d/dl/Interview Guide - iOS.pdf`, which describes each panel session and had not been folded into the round 2 notes.
- **Purpose**: one page to work from today. It lists what is ready, what is still open, and the cards for the parts of Fetch's guide the earlier notes do not cover.

## State this morning

| Item | State |
|---|---|
| `nonisolated` and Sendable definitions | Written. Matrix sections 7, 9, 13. Clean answers repeated below. |
| STAR stories | Seven, with results filled in (note edited 2026-11-09 9:28). In note order: 1 thank-you screen, 2 accessibility, 3 Builder.io, 4 CLEAR migration, 5 ground-up rewrite, 6 binary size, 7 principal scope. No product or design conflict story. |
| Guide topics for Omar with no prepared answer | AI strategy, risks and guardrails. A time AI hurt a team. Coaching an engineer through an architectural tradeoff. Future growth. |
| Receipt pipeline point of view | Was a to-do. Drafted below. |
| System design, lists and pagination | Deep notes exist in the round 2 note. |
| System design, scenario discussion and AI follow-up | Not covered before today. Cards below. |
| Practice app `/t/fp/chatClient1` (2026-11-05, three SPM modules) | 31 tests pass today on Xcode 26.5, iPhone 17 Pro simulator. Not mentioned in the vault notes until now. |
| Practice app `/t/fp/infinitelist` | 23 tests pass today on the `mentions` branch. The repo is currently checked out on `mentions`; the round 2 note says to start a live session from `main`. |
| Combine | In Fetch's guide for the concurrency session. Covered only in the round 1 note. |

## What Fetch's guide says each session is

**Hiring manager, 60 minutes.** "Your experience as a senior technical leader": architecture influence, mentoring, helping teams adopt modern practices "including the responsible use of AI". Topics: technical ownership, scaling through mentorship, judgment, "AI strategy, risks, and guardrails". Example asks: decisions that impacted multiple teams; coaching engineers through architectural tradeoffs; times when AI helped or hurt a team; technically challenging problems; mistakes and recovery; debugging production issues; future growth. The receipt-scanning revamp came from the recruiter, not the guide.

**System Design, "Chart Problem", 60 minutes.** Two parts plus one follow-up.

1. Scenario discussion of your own experience: modularization (packages, frameworks); architecture patterns (MVVM, TCA, Clean); offline-first and sync models; integrating ML or LLM components (Core ML, Vision, NLP); platform constraints (app size, battery, privacy, background execution limits).
2. Exercise: "You'll be shown a complex mobile screen and asked to design it end-to-end", verbally, in pseudocode, or in Xcode. Areas: data models; boundaries and modules; data flow and state; networking and caching; navigation and screen coordination; transforming backend data into UI-ready state.
3. One AI follow-up, "not an AI-focused interview". Example prompts: AI autocomplete or ranking with offline support and caching; on-device versus server inference; user override and fallback when AI is uncertain or wrong; how AI features interact with pagination, caching, or recommendations. Also possible: model size, production monitoring, fallback when slow or unavailable, testing AI-driven experiences, model updates and drift.

The guide's title says "Chart", the recruiter said UI, display, pagination. The screen is unknown until it is shown.

**Swift Concurrency, 60 minutes.** The guide describes a discussion ("exploratory and discussion-based, may include whiteboarding or light code sketching"); the recruiter described a build (chat app with mentions, AI tools). Be ready for both. Guide focus areas: actors (isolation, reentrancy, pitfalls); async/await (structured concurrency, task hierarchy, error handling); Combine (publishers, subscribers, operators, backpressure); legacy interop with `withCheckedContinuation`; Sendable and data races. Guide follow-ups: an API client with retries in Combine and database writes in async/await; how to test an actor; wrap a completion-handler call safely.

The guide also lists a 30-minute Product or Design partner session that is not on today's schedule. Its typical prompt, "a feature was technically possible but had major tradeoffs, how did you handle it", may surface with Omar.

## Session 1: Omar

### AI strategy, risks, guardrails: material on record

Facts only, from the engagement notes and last week's practice runs. The framing is yours.

- **Evaluation before adoption (Walmart, 2024 to 2026).** Working groups for the 120 plus iOS org. Most Xcode plugins were found deficient for iOS work. Builder.io was evaluated hands-on against written criteria, then adopted by management for new view work, with 80 plus engineers trained live.
- **Agentic tools as accelerators for judgment (CLEAR, 2026).** Claude Code and Codex for codebase analysis, proposals, code and tests, working with a six-engineer team.
- **Spec-first with agents (FirefighterSpot, 2026).** 238 numbered specs written before implementation; 433 tests; shipped 1.0 and 1.0.1.
- **A concrete case of AI output being wrong (2026-11-04 practice).** A one-paragraph prompt produced a competent model layer and four real bugs: id-derived cursors, swallowed errors with no retry, a `.task(id:)` that cancelled fetches mid-flight, and a scroll anchor that snapped back on every prepend. Three view bugs in the sample were invisible to the build, the tests, and a screenshot, and were found only by scrolling by hand.
- **Guardrails those cases support.** The compiler is the only validator of strict concurrency. Tests are deterministic and written with the code. Behaviour a tool cannot verify is named as unverified and checked by a person. Each constraint in the prompt exists because its absence produced a specific bug.
- **Routy, only if AI-directed delivery comes up.** An agent-directed demo. Not production, not hands-on code.
- **Fetch's own vocabulary.** The guide points at Anthropic's 4D framework for AI fluency: delegation, description, discernment, diligence.

### The 4D framework against the record

| D | What it means | Your evidence on record |
|---|---|---|
| Delegation | Deciding whether and how to use AI for a task, from the goal and the tool's strengths and limits | Walmart: Xcode plugins rejected as deficient, Builder.io adopted for view work. Routy: agents trusted on the backend, the iOS client and globe reviewed by you. |
| Description | Telling the AI what output you want, how to approach the work, and how to behave | FirefighterSpot: 238 specs before implementation. Last week: the one-paragraph prompt gave four bugs, and each added sentence prevents one. |
| Discernment | Judging the output, the reasoning behind it, and how the tool behaved | Three view bugs passed the build, the tests, and a screenshot, and were found by scrolling by hand. The follow-live fault was diagnosed from logged scroll geometry. |
| Diligence | Owning what you ship: tool choice, honesty about AI's role, and verifying what you put your name on | Routy is always described as agent-directed. The prompt requires the tool to report what it could not verify. FirefighterSpot shipped with 433 tests. |

### Receipt pipeline point of view (draft)

One tradeoff per stage. The public record on Fetch's pipeline is thin: a March 2023 press release says document-understanding models read receipt text and map line items to a product catalog, and does not say where the models run. Everything below is general reasoning, not knowledge of Fetch's system.

- **Capture.** Gate quality on the device before upload: edges found, blur, glare, is this a receipt. Feedback is instant and bad uploads never leave the phone. The cost is model size, battery, and behaviour that varies by device. VisionKit's document camera gives edge detection and perspective correction with little control over a long-receipt flow; a custom AVFoundation capture with Vision requests gives control and a maintenance bill. iOS 26 adds `DetectLensSmudgeRequest`, a cheap gate for hazy frames.
- **Stitch.** Stitching on the device gives the user a preview of the whole receipt and a single upload. Stitching on the server keeps the algorithm changeable without an app release and runs on uniform hardware. Thermal paper is low texture, so pixel registration is fragile; merging per-frame OCR on overlapping text lines is an alternative worth asking about.
- **Parse.** On-device OCR (`RecognizeTextRequest`, or iOS 26 `RecognizeDocumentsRequest`, which Apple documents for receipts) can show a provisional total in a second. The server stays authoritative for line items, catalog matching, duplicates, and fraud. The cost of a provisional result is the correction when the server disagrees, so the UI has to make provisional look provisional.

### Five questions for Omar

1. What does the revamp change: capture, stitching, parsing, or the whole flow?
2. Where does the pipeline hurt most today, and how is that measured?
3. What runs on the device now, and what is expected to move?
4. How does the iOS team interface with the ML team: SDK, model drops, API contract?
5. What number says the revamp worked?

## Session 2: Ricky

### Scenario discussion: one item of your own per topic

- **Modularization.** Walmart: platform and feature modules shared across apps for multiple markets, inner-sourced. FirefighterSpot: 16 SPM modules behind a thin app shell, one per feature, protocol-first services.
- **Architecture patterns.** Walmart checkout: unidirectional MVVM-C in Swift and Combine; the one-paragraph version is at the end of the round 1 note. FirefighterSpot: ViewModel plus ActionHandler. TCA: the 2.0 beta work, plus the Routy client with TCA paged loading, which was agent-directed and is described that way.
- **Offline-first and sync.** FirefighterSpot: every action writes local SwiftData first; persisted pending-upload queue; Supabase Realtime for live updates; last-write-wins on update timestamps; soft-delete tombstones; a schema-version flag that forces a reset; offline join against a placeholder incident reconciled later. Contrast: Walmart checkout had no local database because the server owned the rules.
- **ML or LLM components.** On record: VisionKit scanning and CoreImage generation for QR transfer in FirefighterSpot. No Core ML or on-device model work appears in the engagement notes. If there is some, have it ready; if not, say so and reason from the constraints.
- **Platform constraints.** App size: the Walmart binary-size targets, story 6. Background execution: the FirefighterSpot PAR timer, computed from wall-clock time, recovered on scene-phase change, backed by a local notification. Privacy: the CLEAR app reads the user's calendar to detect flights. Battery: no item on record.

### Exercise order

Clarify requirements. Data models. Module boundaries. API and pagination. Data flow and state. Caching and offline. Navigation. Backend data to UI-ready state. Performance. Testing. Observability. The guide's list of areas matches this order except that it names navigation and the backend-to-UI transform explicitly; give each its own minute.

If the screen is a chart rather than a list, the same frame applies with three substitutions: the server aggregates and the client requests by range and resolution; cache per range and resolution with stale-while-revalidate; downsample to the pixel width before rendering and keep gesture state out of the data model.

### The AI follow-up: an answer shape

1. **Where inference runs.** On device: private, works offline, no per-call cost, limited by model size, battery, and the slowest supported phone, and updated only as fast as you can ship models. Server: larger models, updated at will, one behaviour for every user, at the price of latency, cost, and a network dependency. Common split: small on-device model for instant, private, or offline cases; server for quality.
2. **AI output is a suggestion with a deterministic fallback.** The feature must work when the model is slow, absent, or wrong: a timeout, then the non-AI ranking or plain prefix search. The user can always override, and the override is logged as signal.
3. **Caching.** Cache AI results keyed by input and model version, so a model update invalidates them. Offline, serve the last cached ranking and mark it stale. With pagination, the ranking must be stable across pages: the server issues a cursor tied to a ranking snapshot, or pages shift under the user.
4. **Monitoring.** Latency percentiles, fallback rate, override and acceptance rate, per model version, with a remote kill switch and staged rollout.
5. **Testing.** The model sits behind a seam. Tests inject fixed outputs, including slow, failing, empty, and low-confidence ones, and assert the UI's behaviour. Model quality is evaluated separately from the app.
6. **Lifecycle.** Models downloaded on demand rather than bundled, versioned, with drift watched through acceptance rate.

## Session 3: Brandon

### The two definitions

- **`nonisolated`**: removes a declaration from the isolation it would inherit. It runs on the caller, cannot touch isolated state, and the compiler checks that. Callers need no `await` unless it is also `async`.
- **Sendable**: a type whose values can be safely shared across isolation domains. It is what lets a value cross a boundary. It says nothing about where code runs.
- One is about code and where it runs. The other is about values and whether they may cross. `nonisolated` uses the word Sendable in one place only: a `nonisolated let` must have a Sendable type.

### `nonisolated`, short form

`nonisolated` separates a declaration from the isolation it would otherwise inherit. It gives up access to that isolation's state, and in exchange it runs wherever the caller already is, with no hop.

### Sendable versus `sending`

Sendable is a property of a type: its values can be shared across isolation domains without a data race. A type earns it in one of four ways: value semantics, immutability, isolation, or internal synchronization. `sending` is a property of one parameter or result: a non-Sendable value may cross once, because the compiler proves the sender never touches it again.

### The guide's discussion questions, one line each

- **What actors solve versus locks and queues.** The compiler enforces that isolated state is touched only from inside the actor; a lock or queue is a convention the compiler cannot check. The cost is that access from outside is asynchronous, and every `await` inside the actor is a reentrancy window where state may have changed. A `Mutex` is still right for small synchronous critical sections.
- **async/await versus Combine.** async/await for one-shot work, sequential steps, structured cancellation, and error propagation by `throw`. Combine for values over time that need operators: debounce, combineLatest, throttling, and demand-based backpressure. `AsyncStream` covers simple streams with a buffering policy instead of demand.
- **Wrapping a completion handler.** `withCheckedThrowingContinuation`, resume exactly once on every path; never resuming leaks the task forever, resuming twice traps. Add `withTaskCancellationHandler` to cancel the underlying call.
- **Testing an actor.** Await its methods, inject its dependencies and clock, hold a dependency open on a continuation to force an interleaving, and assert state after release. No sleeps.
- **Retries in Combine, writes in async/await.** A publisher pipeline with `retry` or a `catch` plus delay for the network call, bridged with `.values` or a continuation into an async function that awaits an actor-owned store.

### The two practice apps take opposite positions on Sendable protocols

Expect to be asked why, whichever one is on screen.

- **`infinitelist`**: the client is a struct of closures and no protocol is Sendable. The module is default MainActor, where a Sendable protocol cannot take an isolated conformance, so its requirements would have to be `nonisolated` and lose access to main-actor state.
- **`chatClient1`**: `MessageService` and `MemberService` are `protocol …: Sendable`. They live in `ChatInterface`, a module without default MainActor, so there is no isolated conformance to break. The conformers are actors, which are Sendable already. The code comment gives the reason: one instance is shared between the main-actor view model and background components.
- The sentence that covers both: put Sendable on a protocol when its instances are shared across isolation domains and its conformers can honour that; leave it off when the conformers are meant to live in one domain.

### Where each term appears in `chatClient1`

- **Actors**: `MockMessageService`, `MockMemberService`, `DemoActivitySimulator`. They own mutable state that must be serialized.
- **Sendable**: `Member`, `Message`, `MessagePage`, `MessageDraft` and their ids, because they cross from the service actors to the main actor. `SimulatedLatency`, because its presets are shared global constants.
- **`nonisolated`**: `LoadState`, `ScrollDistances`, `MessageWindow` and its nested types in `ChatUI`, opting plain value types out of the module's default MainActor isolation so they are not tied to main.
- **Main actor by default**: `ChatViewModel` and the views. No annotations; it is the `ChatUI` target setting.

### How stale mention results are stopped, in each app

- **`chatClient1`** (`ChatViewModel.draftChanged`): each keystroke cancels the stored search task and starts a new one. The task assigns results only if `!Task.isCancelled`. That check is sufficient because the cancel and the assignment both run on the main actor with no suspension between the check and the assignment. The mock also stops early, since its simulated latency throws on cancellation.
- **`infinitelist`, `mentions` branch**: a result is kept only if the draft's current token still has the queried prefix, which covers out-of-order replies without relying on cancellation.
- **Neither app debounces.** The design list in the round 2 note included a debounce on an injected clock. It is the likeliest "now add" request: sleep on an injected `any Clock<Duration>` at the top of the search task, so the next keystroke's cancel removes the request before it is sent, and drive it in tests with a hand-rolled manual clock since the project takes no third-party packages.

### First five minutes of the session

1. Ask: build from scratch or extend something, and is the target default MainActor with the approachable-concurrency flags?
2. State the scope limits from the brief: no in-flight request dedupe, no user-to-message index, user list assumed large so search is a server call returning one page.
3. Say the plan before prompting: models, service seam, view model, tests, then view.
4. Tell the tool the working method: write everything, build once, fix, test once, launch once.
5. Read the generated isolation choices aloud and say what you are checking before accepting them.

## Environment

- Verified today: Xcode 26.5 (17F42); both practice apps build and pass tests on the iPhone 17 Pro simulator.
- Fetch requires a whole-screen share with the AI tool visible. Close the vault, Mail, private repos, and the TCA26 beta checkout first.
- Fetch asks that no personal AI notetaker or recorder be added. Their Zoom Notetaker transcribes unless you opted out by reply.
- Fetch's setup guide lists the usual failures: tool not signed in, usage-limit messages, minutes lost to troubleshooting. Confirm the AI tool answers a prompt in the practice directory and have a browser tab as the fallback.
- Decide which app is the starting point if Brandon allows one, and check out the branch you mean to show. `infinitelist` is on `mentions` right now.

## Open before 2:30, in the order the sessions need them

1. One answer each for: a time AI hurt a team, coaching through an architectural tradeoff, a product or design tradeoff, future growth.
2. Say the two definitions aloud once, then the Sendable-protocol sentence.
3. Read the AI follow-up shape and the scenario list once.
4. Environment check above, and the Zoom test at zoom.us/test.
