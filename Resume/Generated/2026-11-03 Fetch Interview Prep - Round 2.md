# Fetch — Interview Prep: Round 2 (three sessions)

- **Created**: 2026-11-03
- **Company**: Fetch (Fetch Rewards)
- **Job title**: Senior iOS Engineer I
- **Schedule (confirmed 2026-11-06)**: Monday 2026-11-09, Eastern time. 2:30 to 3:30 Hiring Manager Interview, Omar Al Masri (Sr Mgr, Engineering). 3:30 to 4:30 iOS Systems Design + AI Integration, Ricky Witherspoon (Mgr, Engineering). 4:30 to 5:00 break. 5:00 to 6:00 Coding Interview, Swift Concurrency, Brandon Erbschloe (Staff iOS Eng).
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

## 2. Session 1: Omar Al Masri, hiring manager (2:30 ET)

- Install Fetch and scan ten receipts: a long one that needs stitching, a crumpled one, a faded thermal one, the same receipt twice. Note where capture, stitching, and parsing each fail. This gives questions and a point of view on the revamp.
- Learn the Apple stack he will assume: VisionKit's document camera, Vision text recognition and the iOS 26 document-recognition request, image stitching basics (feature matching, homography, overlap detection), on-device versus server OCR, confidence scores, receipt structure (merchant, date, totals, line items), cost dimensions (latency, battery, upload size).
- Write six to eight STAR stories, two minutes each, one metric each, from the Walmart checkout platform and CLEAR. Drafts: [[2026-11-05 Fetch STAR Stories]].
  - stepped outside scope
  - a conflict with product or design and how it resolved
  - disagreed with a decision and either changed it or committed
  - a failure and what changed after
  - mentoring or raising a team's standard
  - shipping under ambiguity
  - a cross-team launch with a dependency that slipped
- Prepare a short point of view on the pipeline, capture to stitch to parse, with one tradeoff per stage. A stance he can push on, not a design.
- Five questions: what "revamp" means in scope; where the pipeline hurts today; on-device versus server split; how the ML team interfaces; how success is measured.

## 3. Session 2: Ricky Witherspoon, iOS systems design and AI integration (3:30 ET)

- Fix a framework and rehearse it: clarify requirements, data model, API and pagination, layers, UI and state, caching and offline, performance, testing, observability. Pick the drawing tool (Zoom whiteboard, Excalidraw, FigJam) and test it on a screen share.
- Drill four prompts, 40 minutes each, one recorded: a receipts list with infinite scroll; an image-heavy feed; a chat screen; the receipt capture and upload flow.
- Pagination cold: cursor versus offset, page tokens, dedupe on refresh, stale-while-revalidate, placeholder and error rows, prefetch trigger distance, keeping scroll position across inserts at the top.
- Display layer cold: `List` versus `LazyVStack` versus compositional layout with a diffable data source; stable identity; image downsampling and decoding off the main actor; cell reuse cancellation; `@Observable` granularity; Dynamic Type; state restoration.

### SwiftUI vs UIKit for complex lists (state as of iOS 26, mid-2026)

Summary: SwiftUI is fully adequate for the common infinite-list case, UIKit still wins on the hard edges, and the hybrid is the production norm at scale.

**Where SwiftUI stands**

- `List` is backed by `UICollectionView` since iOS 16, so it recycles cells and handles very large collections. Row identity and cheap `body`s matter more than row count.
- `LazyVStack` in `ScrollView` is lazy on creation but does not recycle, so memory grows with scrolled distance. Fine for hundreds of rows, a problem for thousands with images. Use it for layout freedom `List` will not give; otherwise use `List`.
- Scroll control caught up over iOS 17 and 18: `scrollPosition(id:)`, `scrollTargetLayout`, `defaultScrollAnchor(.bottom)` for chat, `onScrollGeometryChange` for distance-from-bottom math, `onScrollVisibilityChange(threshold:)` for row visibility, `onScrollPhaseChange` for drag and decel state. Geometry-based pagination triggers replace the old `onAppear` on the last row, which fired unreliably and had no distance threshold.
- iOS 26: measurable `List` and lazy stack performance gains, a SwiftUI Instruments template that attributes long updates to the view body that caused them, and UIKit observing `@Observable` models directly, which makes mixed stacks less awkward.
- Images: `AsyncImage` is not production grade for feeds. Write or adopt a loader that downsamples with `CGImageSource` off the main actor and caches decoded images; UIKit needs the same work.

**Where UIKit still wins**

- Prefetching: `UICollectionViewDataSourcePrefetching` gives cells ahead of visibility with cancellation. SwiftUI has no equivalent; approximate with geometry callbacks.
- Scroll offset precision: keeping position when inserting above the viewport, pull-to-load-older in chat, anchoring to a row after a diff. Diffable data source plus `contentOffset` adjustment is deterministic. SwiftUI's `scrollPosition` with anchors handles most of it but has edge cases with variable-height rows and animated updates.
- Compositional layouts: orthogonally scrolling sections, carousels inside a vertical feed, sticky headers with custom behavior, per-section layouts. `UICollectionViewCompositionalLayout` does this directly; SwiftUI needs nested lazy stacks and hand-built pinning.
- Fine-grained update control: batch updates, reconfigure without reload, measured sizing. SwiftUI diffing is implicit; a bad identity or an equatable miss re-renders more than intended.

**The hybrid most large feed apps run**

`UICollectionView` owns the scroll container, diffable data source owns identity and diffs, prefetching data source feeds the image pipeline, and `UIHostingConfiguration` (iOS 16) puts SwiftUI inside each cell. The reverse, a `UIViewControllerRepresentable` collection view inside a SwiftUI screen, is the route when the screen is otherwise SwiftUI.

**Pagination specifics that are the same in both**

Cursor over offset; dedupe by stable ID when a refresh overlaps a page; stale-while-revalidate from a cache; loading and error rows as real items in the data source, not overlays; prefetch trigger measured in rows or points from the end; cancel in-flight page requests when the screen goes away.

**Interview position**: default to SwiftUI `List` with geometry-based triggers; name the three UIKit cases that move you to a hosted collection view (prefetching, offset control, compositional layout); the content inside cells is SwiftUI either way.

### List vs LazyVStack for the sample chat screen

The reference app uses `ScrollView` plus `LazyVStack`. The same screen builds on `List` with `.listStyle(.plain)`, hidden separators, cleared row backgrounds and insets, the two edge rows as ordinary rows, and the same composer below in the `VStack`. `onScrollGeometryChange` and `onScrollPhaseChange` apply to `List` (Apple's scroll-views overview: lists take the same modifiers as explicit scroll views, and `ScrollPhase` names both). `scrollPosition(id:)` does not: it is documented against `scrollTargetLayout`, which `List` rows do not take, and as of May 2026 `ScrollViewReader` is still the only native programmatic scrolling for `List` (Panferova). So the geometry and phase logic carries over; the id-tracking hold and the id-based follow do not.

**What `List` gives**

- Cell reuse. `List` is a `UICollectionView`; offscreen rows are recycled. `LazyVStack` creates rows lazily and does not recycle them. The sliding window caps rows at a few pages, so memory is bounded either way here; without the window, `List` is the only one that stays flat over a long session.
- Faster convergence of content size. Self-sizing cells are measured as they appear and cached. The blank band under the last row that cost two fixes came from `LazyVStack` trusting its estimated content height. Unmeasured rows above the viewport still use estimates in `List`.
- Keyboard handling. The collection view adjusts its content inset for the keyboard and keeps bottom content in place. On `LazyVStack` that is the app's job and is the one case still unverified in the sample app.
- Swipe actions, selection, context menus, and insert and delete animations for free. The insert-by-id path in `receive` would animate the new bubble into place.

**What `List` costs**

- Prepend hold. The deciding constraint. Inserting older rows above the viewport shifts a collection view's content offset unless compensated, and SwiftUI exposes no hook for the compensation. The anchorless `ScrollPosition(idType:)` on `ScrollView` is what holds the view still when a page prepends. On `List`, `scrollPosition(id:)` tracks the id but the hold is not reliable across a diffed insert of thirty rows. The workaround is the flipped list, a 180-degree rotation of the list and every row so prepend becomes append, which then needs explaining.
- Styling goes through row modifiers. Bubbles, the day divider, and the page tints are reachable with the plain style, but each visual change is a negotiation with defaults rather than layout.
- Less visibility into layout. The row container is not yours; the measured geometry conventions (top inset in `distanceToTop`, the resting-bottom formula, id-scroll anchor behaviour) would need re-measuring.
- Readability in the session. The current code shows the mechanics: lazy layout, scroll target layout, position tracking, follow logic. `List` hides them, which is better for shipping and worse for demonstrating that you understand them.

**What `LazyVStack` gives and costs**

- Gives: full layout control, the prepend hold that works today, and scroll modifiers whose behaviour has been diagnosed end to end on this app.
- Costs: no recycling, so memory scales with rows created rather than rows visible; estimated content size, the source of both scroll defects so far; keyboard handling by hand.

**The prepend hold, unpacked**

- Prepend here means `loadOlder` inserting about thirty rows at index zero of the window. The content gets taller by their height, all of it above the viewport.
- A scroll view knows only its content offset, a y distance from the top of the content. If rows are inserted above and the offset number stays the same, the viewport now shows the new rows and the user's rows have moved down by their combined height. Holding still means growing the offset by exactly that height in the same layout pass, and that height is unknown until the rows are measured.
- UIKit does nothing automatic. The app computes the delta from measured cells or from the content-size change and adds it to the offset inside the batch update or through a layout invalidation context. Deterministic because the app owns the layout. Estimated heights make a naive delta drift; production apps measure first or correct after layout.
- The reference `ScrollView` tracks a row by identity: `scrollTargetLayout` marks the rows, and an id-typed `ScrollPosition` with no modifier-level anchor keeps the tracked row where it was when content changes. Measured on this app: a prepended page holds the viewport. A modifier-level anchor broke it by re-aligning the tracked row on every content change, appends included.
- This is the documented behaviour, not an accident: the `scrollPosition(id:anchor:)` reference says SwiftUI attempts to keep the identified view visible when the backing data is re-ordered or the scroll view's size changes, and that with no anchor it scrolls the minimal amount. The April 2025 forum poster who saw a jump had `defaultScrollAnchor(.bottom)` with no role, which also applies to size changes and re-anchors to the bottom on every content change; the reference app limits it to the `.initialOffset` and `.alignment` roles. Plausible explanation of the difference, not verified against their code.
- Why `List` loses here: it has no id-tracking position at all, only `ScrollViewReader.scrollTo`, so there is nothing to hold the viewport across a prepend. Any hold would be a manual offset correction against estimated cell heights, which is the UIKit problem without UIKit's hooks. The `List` version has not been built.

**The flipped list, and where its problem comes back**

- The workaround: rotate the list 180 degrees and rotate every row back, so the newest message is index zero at the top of the content. History loads become appends at the end, and appends never move existing content because offsets are measured from the top.
- Pinned at the bottom: the user is at content offset zero. Inserting a live message at index zero leaves the offset at zero, so the new bubble appears at the bottom and older ones slide up. Follow comes for free. Holding still while typing therefore needs extra work, because following is the default.
- Scrolled up into history: the offset is positive, the live insert at index zero makes the content above the viewport taller, and the rows being read jump by the bubble's height. Same compensation problem as prepending history in the unflipped list, estimated heights included.
- How chat apps close that case: never insert while scrolled up. Buffer live messages, show a "3 new messages" pill, insert when the user returns to offset zero where the insert is free. With that rule nothing is ever inserted above the viewport in content terms and the offset math disappears. The flip plus buffering is the standard pairing; the flip alone only moves the problem.
- The reference app already buffers when the tail page has been evicted (`receive` counts and the pill shows it). When the tail page is still in the window and the user has scrolled up within it, the message is inserted at the end of the tail page, below the viewport in the unflipped list, so nothing moves. A flipped version would widen the buffering rule to "whenever not pinned", which is the `isPinnedToBottom` flag the view already keeps.
- Costs of the flip beyond that: scroll indicator and gestures run backwards unless compensated, VoiceOver reads in reverse, pull-to-refresh is at the wrong end, keyboard and safe-area behaviour needs re-checking, and the list is upside down when someone reads the code.
- References: [Building the inverted scroll of a messaging app](https://swiftwithvincent.squarespace.com/blog/building-the-inverted-scroll-of-a-messaging-app) (Vincent Pradeilles) walks through the flip in SwiftUI, a π rotation plus horizontal mirror on the `ScrollView` and again on each row, with older messages appended from an `onAppear` on the last row; it does not cover incoming messages while scrolled up. [How do I maintain a stable scroll position when inserting items above in a ScrollView?](https://developer.apple.com/forums/thread/781282) (Apple Developer Forums, April 2025, unanswered) reports the prepend jump with `defaultScrollAnchor(.bottom)` plus `scrollPosition`, and that the `scaleEffect(y: -1)` flip shifted the scroll view horizontally and broke `.contextMenu`.

**Is the flip still current?** (checked 2026-11-05)

Nothing recent recommends it for a full chat screen. The 2025 and 2026 material either uses the iOS 17 and 18 scroll APIs on a normal list or drops to a collection view that preserves the offset.

- [Modern SwiftUI APIs for programmatic scrolling](https://nilcoalescing.com/blog/ModernSwiftUIAPIsForProgrammaticScrolling/) (Natalia Panferova, 19 May 2026): chat example on `ScrollPosition` with `scrollPosition(_:anchor:)`, `defaultScrollAnchor(.bottom)` for the opening position, the `.alignment` role for short content. No flip. Does not cover prepending history.
- [FluidGroup/swiftui-messaging-ui](https://github.com/FluidGroup/swiftui-messaging-ui) (requires Xcode 26, so late 2025 or later): `UICollectionView`-backed with SwiftUI cells and a virtual layout anchored at the centre of a huge content space, so prepending extends upward without touching the offset. Its README names SwiftUI `List` and `ScrollView` as what jumps and answers with layout, not a transform.
- [unionst/swift-chat](https://github.com/unionst/swift-chat) (iOS 18, Xcode 26.1): collection view underneath; `chatLoadsOlderMessages` holds the reader's position when a page lands, mechanism not described. No flip.
- [Apple forums 781282](https://developer.apple.com/forums/thread/781282) (April 2025, unanswered): poster tried the flip, reports a horizontal shift and a broken `.contextMenu`.
- [Apple forums 769695](https://developer.apple.com/forums/thread/769695) (November 2024, watchOS): flipping inverted the Digital Crown; the DTS reply (July 2025) suggests reversing the data and filing feedback, and does not endorse the flip.
- [TanStack Virtual chat guide](https://tanstack.com/virtual/v3/docs/chat.md) (web): a normal top-down list with end anchoring and stable keys instead of inverted transforms.
- The flip still appears in Q&A answers from 2020 to 2024 and in the Vincent article (8 July 2024, pre-iOS 18). Each recent sighting is someone trying it and reporting a side effect. A search snippet mentioned an iOS 26 navigation-bar fade problem with inverted lists; not verified.
- Trajectory: the flip existed because SwiftUI before iOS 17 could not open at the bottom or track a row through a content change. `defaultScrollAnchor` took the first job in iOS 17, id-typed `ScrollPosition` the second in iOS 18, which is what holds the reference app on prepend. What native SwiftUI still lacks is a measured offset adjustment on a collection view, which is why the two current chat packages reach for UIKit for that part rather than a transform.
- Framing for the session: a known pre-iOS 17 workaround; know why people used it and what it costs; the current app does not need it.

**The statement for the session**: the window gives `LazyVStack` the property that normally sends people to `List`, bounded memory. The prepend hold is the property `List` does not reliably give back. `List` wins on an unbounded list and loses on the prepend hold; the window is what makes the other choice sound.

## 4. Session 3: Brandon Erbschloe, coding round, Swift concurrency (5:00 ET)

### Two design choices in the sample app, and why

**No `Sendable` on protocols.** Under default MainActor isolation, a protocol declared in the app module is `@MainActor`, and a conforming type gets an isolated conformance, which is what lets its requirements touch main-actor state. A protocol that inherits `Sendable` cannot have an isolated conformance. The compiler then forces the requirements to be `nonisolated`, which pushes them into a `nonisolated extension` with no access to the type's main-actor state. That is the [[369 - MainActor Default]] thorn and section 5 of [[Isolation Review Matrix]]. Since the app's types all live on main, a `Sendable` protocol would fight the module setting on the first conformance. The series' broader point: a Sendable requirement on a protocol commits every conformer forever, which is the wrong direction for isolation.

**Structs of closures, not protocols.** The client has three operations. As a struct, each test supplies only the closures it cares about, inline, and the gated test server is a plain class that hands out a `client` value rather than conforming to anything. No protocol to carry isolation, no existential boxing, no isolated-conformance question, no mock class per scenario. Under default MainActor the closure types are `nonisolated(nonsending)` async functions, so they run on the caller (main) and nothing in the model has to be Sendable. Same style as Point-Free Dependencies.

**What they do not mean.** Not a claim that protocols are wrong. If the interviewer's codebase uses protocols and mock classes, the swap is mechanical: a `ChatClient` protocol with three requirements, the mock server conforming to it, the gated server conforming to it in tests. Keep `Sendable` off the protocol either way.

**Interview risk.** A reviewer who expects a protocol may read the struct as unfamiliar. Two sentences to have ready: the struct avoids isolated-conformance questions, and it lets tests supply partial behavior inline. Offer the protocol version if they prefer it.

**Where to see it in the sample app** (`/t/fp/infinitelist`):
- `InfiniteList/ChatClient.swift` lines 5–12: the `ChatClient` struct with `fetch`, `incoming`, `send` closures.
- `InfiniteList/ChatClient.swift` lines 189–196: `ChatClient.mock(server:chatter:)` wiring the closures to `MockChatServer`.
- `InfiniteListTests/ChatModelTests.swift` lines 21–37: `GatedServer.client`, a test double built as a value with no conformance; `send` is a one-line stub.
- `InfiniteList/ChatModel.swift` line 44: the model stores the struct and calls `client.fetch(...)` directly.
- There is deliberately no example of the Sendable-protocol failure in the app. To see it, add `: Sendable` to any protocol in a default-MainActor module and conform a class to it; the transcript walkthrough is in [[369 - MainActor Default]], and TCA26's `@Feature` macro error on `Sendable` is at `/d/ivgp/tca/TCA26/ComposableArchitecture2Macros/FeatureMacro.swift`.

### Performance implications of the sample app's design

"Window" below means the number of messages currently held, at most `maxMessages` (90). Costs are bounded by it, never by how far the user has scrolled or by conversation length. Page size (30) bounds per-page work; the mock's 12,000-message log is the server's cost, not the client's.

- **Every page load rebuilds the whole window.** `apply` flattens all pages into a new `messages` array and recomputes the debug position map: O(window). The array assignment notifies observers, so the `ForEach` re-diffs 90 identities and the `enumerated()` wrapper allocates a 90-tuple array per body evaluation. Nothing at 90; not nothing at 2,000. First thing to remove if the window grows: `enumerated()`, replaced by a precomputed set of day-start ids.
- **Live messages pay the same full rebuild.** `receive` appends one message then flattens everything. Invisible at one message per three seconds; dozens per second would flatten 90 rows dozens of times a second on main. Fix is incremental append when at tail, rebuild only on eviction. Current code trades that for simpler invariants.
- **Per scroll frame.** Geometry closure builds a two-field struct; action does two comparisons and two guarded calls that read `pages.first` / `pages.last` (O(1)) and return early. No `@State` written, so no body re-evaluation. The pinned flag is written only when a scroll phase ends, and the content-size modifier fires only when the content size changes.
- **Memory.** Bounded by the window: 90 structs, 90 lazy-stack render trees, a `seen` set of 90, a position map of at most 18. Text rows cost a few KB each. With images, row cost dominates and the window size is set by image memory, not row count.
- **Per page load, outside the model.** Network, JSON decoding, date parsing all live in the client closure. Under default MainActor it is `nonisolated(nonsending)`, so it runs on main. Fine for 30 messages; a large payload or image work wants `@concurrent` on the fetch, returning a Sendable `Page`.
- **Dedupe.** `seen` is a set: O(page size) per page, O(window) rebuild on eviction.
- **Eviction thrash.** Reversing across a window boundary evicts in one direction and re-fetches in the other, 500 ms each. Prefetch distance 600 pt plus a three-page window puts the boundary about two screens away, so reading does not thrash; a fast fling does. First real-app addition: a small cache of recently evicted pages keyed by cursor, making reversal free.
- **SwiftUI-specific.** Prepending into `LazyVStack` builds render trees only for rows near the viewport. `scrollPosition` tracking and hold-on-prepend are light. `Calendar.isDate(_:inSameDayAs:)` runs per row per body evaluation, 90 calendar calls per window change; precompute day starts per page instead. `Text(date, style: .time)` formats once per row's first appearance, cached until eviction.
- **Where it shows first.** A busy live channel (full rebuild per message), then image rows (window too large for memory), then eviction thrash on fast scrolling. None visible with text rows at the mock's rate.
- **Two sentences to have ready.** "The rebuild is O(window) and should become incremental for a high-rate channel." "An evicted-page cache keyed by cursor removes the re-fetch on reversal."

### Follow-live: why the view did not scroll for live messages, and the fix

Found 2026-11-04 by Doug on main; fixed in commit `83c561f` of `/t/fp/infinitelist`. Diagnosed by logging `ScrollGeometry` transitions with timestamps into an on-screen overlay and reading them off screenshots at 2 and 11 seconds. Two independent defects, both present in every generated run's approach as well.

**Defect 1: the "at the bottom" test was wrong by the top inset.** The pinned check computed `contentSize.height - (contentOffset.y + containerSize.height)`. In `ScrollGeometry`, `containerSize` is the safe-area viewport (670 pt on an iPhone 17 Pro) and `visibleRect` spans the whole container including the strips under the navigation bar and the composer, so at the resting bottom that expression equals `contentInsets.top`, 116 pt, which never passes a 40-pt tolerance. The user was never pinned, so nothing ever followed. The right distance is `contentSize.height - visibleRect.maxY + contentInsets.bottom`, which reads 0 at rest. The same expression now drives the prefetch trigger's bottom distance. Lesson: derive scroll distances from `visibleRect`, not from offset plus container, whenever there are safe-area insets.

**Defect 2: the id-anchored `scrollPosition` binding fought the follow.** `scrollPosition(id: $position, anchor: .bottom)` does two things: SwiftUI writes the id of the row at the anchor as the user scrolls, and on any content change it re-aligns that row to the anchor. The anchor is measured against the full visible rect, so on each live append it moved the old last row's bottom to the screen bottom, under the composer, shifting the view up by one row. That re-alignment also overrides `defaultScrollAnchor(.bottom, for: .sizeChanges)`, so the role-scoped size-change anchor cannot be used to follow while an id binding exists. Writing `position = newLastID` in `onChange` did nothing because the new row was not laid out yet in that update.

**Fix, three parts, all in `ChatView.swift`.**

1. The binding is a `ScrollPosition(idType: Message.ID.self)` value with `.scrollPosition($position)` and no anchor. It still tracks the visible row and holds it when an older page prepends, without the bottom re-alignment.
2. Pinned state is decided only when a scroll phase ends: `onScrollPhaseChange` on `.idle` sets `isPinnedToBottom` from the corrected distance. Deciding it from geometry alone flipped it on the transient layouts a content change passes through. The jump pill sets it true explicitly.
3. One `onScrollGeometryChange(for: CGFloat.self) { $0.contentSize.height }` handler: while pinned and at the tail, request `position.scrollTo(edge: .bottom)`. An edge scroll respects the insets; an id scroll does not. The same handler settles the launch position after the lazy stack measures its rows (the initial `.initialOffset` anchor is applied against an estimated content size that later changes), covers sends, and covers live messages. On the mentions branch it defers while the composer has focus and catches up once when focus ends.

**Verified** by timeline: launch rests on the newest message with distance 0 at 1.2 s; after each of three live appends the distance returned to 0 and the newest row sat above the composer. Not exercised: hand scroll, prepend hold, jump pill, keyboard.

**Two sentences to have ready.** "The pinned check has to come from `visibleRect` because the container excludes the safe area." "An id-anchored scroll position re-aligns on every content change, so follow-live uses an edge scroll and the binding stays anchorless."

### Network-like fakes: what the mocks do beyond a sleep

On the `mentions` branch of `/t/fp/infinitelist` from commit `253b80c` (2026-11-04). The point: a fake that only sleeps on the main actor exercises none of the concurrency the brief grades. These fakes make the model cross an isolation boundary, cope with out-of-order replies, and handle failures, with tests that stay instant.

**Four properties, each on purpose.**

- **Off the main actor.** `MockChatServer` and `MemberDirectory` are actors. Every `fetch`, `send`, and `search` runs on the cooperative pool; the main-actor `ChatModel` awaits across the boundary. This is the one place in the app where "proper use of actors" is visible: the server owns mutable state (the log, the listeners, a failure flag, a seeded generator) that must be serialized, and the client never touches it directly.
- **Values that cross are nonisolated Sendable structs.** `Message`, `Page`, `Cursor`, `Direction`, `Member` are declared `nonisolated` so that under default MainActor isolation the actor can construct and return them without a hop, and `Sendable` so they may leave the actor. This is the clean articulation of the round 1 feedback: `nonisolated` says no actor owns the type; `Sendable` says a value may be carried across. The model, the view, and `ChatClient` stay main-actor and are neither.
- **Jittered latency.** `NetworkConditions` (`FakeNetwork.swift`) draws each request's wait from 400 ms ± 75 percent using a seeded generator held by the actor, so replies land out of order between requests. The duration is drawn synchronously and the sleep follows, because an actor's stored generator cannot be borrowed `inout` across an `await`. Tests pass `.none`. Two consequences the jitter forced into the model: a send's reply can arrive after a live message that beat it to the server, so `receive` inserts into the tail page by id rather than appending; and a search for `a` can land after the search for `an`, so a result is kept only if the draft's token still has the same prefix.
- **Injectable failure.** The chat server fails once per cursor at a known page; the directory has `failNextRequest()` and throws `DirectoryError.unavailable`. A failed search leaves the previous candidates in place; a failed page shows the error row with Retry.

**Opening the live stream** stays synchronous for the caller: the actor's `incoming()` is `nonisolated`, creates the `AsyncStream`, and registers the continuation inside the actor from a task, the way a connection completes after the call returns. A message emitted before registration completes is missed, which is also true of a real socket.

**The directory never holds the directory.** `FakeUserService` is a nonisolated stateless struct that generates a deterministic user list of any size (200,000 by default) in pages of 10,000, with unique handles at any count. A prefix query scans pages one at a time and stops at the limit, so at most one page exists at any moment. `MemberDirectory` holds no users, only the request conditions. Results come back in id order, the fake's stand-in for server ranking. The client holds one page of twenty candidates and nothing else. That is the "arbitrarily large list" answer: the client asks for a page per keystroke and keeps a result only if it still matches the draft.

**What the fakes cost.** The scan is linear in the directory per keystroke, about a hundred milliseconds for 200,000 off the main actor, which is the server's cost standing in for an index query. The chat log is 12,000 structs on the server actor. Neither cost is on the client.

**Two sentences to have ready.** "The fakes are actors with jittered latency so the model has to handle a boundary crossing and out-of-order replies, not just a delay." "What crosses the boundary is nonisolated and Sendable; what owns state is an actor; the model and view stay on main."

### Working method for AI-assisted builds (timed rounds)

Builds and test runs are the expensive steps, one to two minutes each; the simulator launch is about thirty seconds. Wall time is the number of build-fix cycles, so batch work and validate in as few cycles as possible.

1. Write every source and test file first, without building in between.
2. Build once for the simulator. Fix every error and warning, build again, repeat only until clean. The build is the only thing that validates strict concurrency; nothing substitutes for the compiler on isolation errors.
3. Run the test target once. Fix failures and rerun only as needed. Model tests are seconds once built.
4. Launch exactly once at the end with one screenshot, and only to confirm the first render.
5. Scroll by hand once before calling it done, and wait for a live message. All three view bugs in the sample (anchor snapping on prepend, trigger stuck at the top, follow-live never firing) were invisible to the build, the tests, and a single screenshot.

Do not build after each file, relaunch after each fix, or screenshot what a green build already proved. Put this in the prompt (it is in [[2026-11-04 Fetch Sample Chat App Prompt]] under "Working method") and ask the session to report build and test cycle counts so runs can be compared. If a run's cycle count is high, the fix is in the spec's concurrency rules, not the method.

Reference timings from the 2026-11-04 session, incremental in a warm context: the first feed version took about five minutes from empty directory to green tests and a screenshot, two of them writing; the chat rewrite about eight. A cold start against the full spec on Fable 5.1 ran 25+ minutes, most of it in xcodebuild cycles.

### Minimal prompt for the sample app (two paragraphs)

The one-paragraph version run on Opus 5.5 (2026-11-04, `/t/fp/infinitelist3`) produced a competent model layer but four real bugs: id-derived cursors with flat trimming, swallowed errors with no retry, `.task(id:)` on the spinner row cancelling fetches mid-flight with no prefetch, and the unscoped bottom anchor snapping back on every prepend. Each is prevented by one sentence in the second paragraph. Screen features (composer, bubbles, pill, dividers, tints) are deliberately absent; add one sentence listing them if wanted.

```
Build a pure SwiftUI iOS sample app called InfiniteList2 in the current directory: a team chat screen over a large body of history, with a sliding window so the client never holds more than a bounded number of messages, bidirectional cursor pagination, live incoming messages, and deterministic tests. It is a starting point for adding features in an interview, so keep it small, readable, and conventional. Build it, run the tests, and launch it on an iPhone simulator before reporting done. Do not add features beyond this spec.

Cursors are opaque tokens issued by the server with each page, never derived from message ids, so the window is kept at page granularity and evicts whole pages. Each direction has its own idle, loading, and failed state; a failure shows an error row with an explicit Retry and is never retried on its own. The model owns every load as a stored task started from a scroll-distance trigger, so scrolling a row off screen can neither start nor cancel a fetch, and there is a prefetch distance of a few hundred points. Anchor the initial scroll position to the bottom using the role-scoped form of defaultScrollAnchor, not the unscoped form, and hold the visible row in place when an older page prepends. The fake server's history ends a few minutes ago so live messages continue the timeline, and it can inject a failure for tests. Dependencies are structs of closures; do not put Sendable on any protocol. Scroll behavior cannot be verified by build, tests, or a screenshot; say so in your report and leave that check to me.
```

| Sentence | Prevents |
|---|---|
| Opaque cursors, page granularity | Id-derived cursors and flat trimming that lock the model to integer ids |
| Per-direction state with explicit Retry | Swallowed errors; a spinner that never retries |
| Model-owned stored task, distance trigger, prefetch | `.task(id:)` on the spinner row cancelling fetches; no prefetch |
| Role-scoped anchor, hold the row on prepend | Snap-to-bottom on every older page |
| History ends now, injectable failure | Timeline jump on the first live message; no failure tests |
| Structs of closures, no Sendable protocols | House style; removes `nonisolated` on every value type |
| Scroll cannot be verified, report it | A "done" that was never scrolled |

Two ideas from the Opus session worth keeping: enable the simulator tool's touch automation so the session can verify scrolling itself, and add a long scripted test that pages through all history and back, checking after every step for window size, gaps, and duplicates. The full design-level prompt remains [[2026-11-04 Fetch Sample Chat App Prompt]].

- Sample app to start from: sliding-window chat built 2026-11-04 at `/t/fp/infinitelist` (pure SwiftUI, 30 days of mock history, bidirectional cursors, live messages, 11 deterministic tests on `main`). The `mentions` branch holds the worked answer to the tagging task with actor-based network-like fakes and 23 tests; start the live session from `main`. Recreate-from-scratch prompt: [[2026-11-04 Fetch Sample Chat App Prompt]] (also saved as `PROMPT.md` in the project).

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
