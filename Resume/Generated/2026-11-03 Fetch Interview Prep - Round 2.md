# Fetch — Interview Prep: Round 2 (three sessions)

- **Created**: 2026-11-03
- **Company**: Fetch (Fetch Rewards)
- **Job title**: Senior iOS Engineer I
- **Format**: three one-hour sessions. 1: hiring manager (Omar), receipt scanning and parsing revamp plus cross-functional stories. 2: iOS system design, UI, display, pagination. 3: coding round, an app from scratch with AI tools.
- **Round 1 feedback (2026-11-02)**: the only negatives were that `nonisolated` was not clearly articulated, and that descriptions of `nonisolated` and sendability were mixed together. Everything else passed.
- **Coding project as described**: minimal chat app; dynamically tag users in comments of a live chat session; handle modern concurrency, cancellation, performance, stale data, proper use of actors, deterministic testing. No inflight tracking, no building cross references. Assume the list of users is large.
- **Related**: [[2026-10-30 Fetch Interview Prep]] (round 1 plan, environment checklist, six concurrency exercises), [[Walmart - Checkout Platform 2021-2026]], [[CLEAR - Senior iOS Subcontract 2026-present]], [[Isolation Principles]], [[Isolation vs Other Architectures]]

## 1. Fix the round 1 feedback first

- Write the two definitions on one card and say them aloud until automatic. `nonisolated` is a declaration modifier: it removes a member or type from its enclosing actor's isolation, so it can be called synchronously from anywhere and in exchange cannot touch the actor's isolated mutable state. `Sendable` is a protocol on a type: values of that type can cross an isolation boundary without a data race. One is about where code runs and which state it may touch; the other is about which values may be carried across. They meet at a call: a nonisolated function can be called from any context, but its non-Sendable parameters still cannot come from another isolation domain unless marked `sending`.
- Learn the variants so they do not blur: `nonisolated(unsafe)` on globals and statics; `nonisolated(nonsending)` in Swift 6.2, an async function that runs on the caller's actor; `@concurrent`, which forces it onto the cooperative pool; `nonisolated` on a whole type under default MainActor isolation, which makes it non-isolated and therefore needs Sendable to be shared.
- Read the proposals once each, since the interviewer's wording will come from them: SE-0302 (Sendable), SE-0306 (actors), SE-0414 (region-based isolation), SE-0430 (`sending`), SE-0461 (`nonisolated(nonsending)`), SE-0466 (default actor isolation).
- Drill with ten small snippets: predict the Swift 6 compiler error before building. Cases: a nonisolated method reading a `var` on an actor; a non-Sendable class captured in `Task.detached`; a `sending` parameter used twice; a nonisolated class under default MainActor passed to an actor; a `let` of non-Sendable type read from a nonisolated method; a closure capturing `self` from a MainActor class inside an actor method.
- Separate the Point-Free stance ("avoid Sendable, isolation comes from the Store") from the language definitions. The stance is a design choice; the interview wants the definitions. That mix is the likely source of the round 1 confusion.
- Record a one-minute explanation of both terms, listen back, cut anything that makes one sound like a kind of the other.

## 2. Session 1: Omar, hiring manager

- Install Fetch and scan ten receipts: a long one that needs stitching, a crumpled one, a faded thermal one, the same receipt twice. Note where capture, stitching, and parsing each fail. This gives questions and a point of view on the revamp.
- Learn the Apple stack he will assume: VisionKit's document camera, Vision text recognition and the iOS 26 document-recognition request, image stitching basics (feature matching, homography, overlap detection), on-device versus server OCR, confidence scores, receipt structure (merchant, date, totals, line items), cost dimensions (latency, battery, upload size).
- Write six to eight STAR stories, two minutes each, one metric each, from the Walmart checkout platform and CLEAR:
  - stepped outside scope
  - a conflict with product or design and how it resolved
  - disagreed with a decision and either changed it or committed
  - a failure and what changed after
  - mentoring or raising a team's standard
  - shipping under ambiguity
  - a cross-team launch with a dependency that slipped
- Prepare a short point of view on the pipeline, capture to stitch to parse, with one tradeoff per stage. A stance he can push on, not a design.
- Five questions: what "revamp" means in scope; where the pipeline hurts today; on-device versus server split; how the ML team interfaces; how success is measured.

## 3. Session 2: iOS system design

- Fix a framework and rehearse it: clarify requirements, data model, API and pagination, layers, UI and state, caching and offline, performance, testing, observability. Pick the drawing tool (Zoom whiteboard, Excalidraw, FigJam) and test it on a screen share.
- Drill four prompts, 40 minutes each, one recorded: a receipts list with infinite scroll; an image-heavy feed; a chat screen; the receipt capture and upload flow.
- Pagination cold: cursor versus offset, page tokens, dedupe on refresh, stale-while-revalidate, placeholder and error rows, prefetch trigger distance, keeping scroll position across inserts at the top.
- Display layer cold: `List` versus `LazyVStack` versus compositional layout with a diffable data source; stable identity; image downsampling and decoding off the main actor; cell reuse cancellation; `@Observable` granularity; Dynamic Type; state restoration.

## 4. Session 3: coding round with AI tools

- Build the described app before the interview, twice: once slowly to settle the design, once timed at 45 minutes with the AI tool, talking aloud.
- Design settled in advance:
  - `@MainActor @Observable` chat model
  - `actor UserDirectory` holding the large list with precomputed lowercase names and a prefix search
  - a mention suggester fed by an `AsyncStream` of query changes, debounced on an injected `Clock`, latest-wins by cancelling the previous search task
  - an `AsyncStream` of incoming messages
  - mentions rendered as `AttributedString` runs
- Decide where `nonisolated` and `Sendable` appear in that design and narrate each: `User` as a Sendable struct crossing from the actor to the main actor; the search entry point as a nonisolated async function or `@concurrent` so filtering a large list never blocks the main actor; what stays MainActor-isolated and why.
- Treat "no inflight tracking, no cross references" as scope limits and say so at the start: no dedupe of concurrent requests, no user-to-message indexes.
- Deterministic tests with Swift Testing: a test clock drives the debounce; a test that a superseded query's result never surfaces; a `confirmation` that the previous task was cancelled; an injected user source with a fixed list. No sleeps.
- Ask at the start whether the project uses default MainActor isolation and the 6.2 approachable-concurrency flags; have both configurations ready in the sample package.
- Reuse the environment checklist from [[2026-10-30 Fetch Interview Prep]]: Swift 6.2 package, one AI tool signed in and tested, whole-screen share with the vault and private repos closed.
