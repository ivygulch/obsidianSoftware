# Prompt: build the sliding-window chat sample

Build a pure SwiftUI iOS sample app called InfiniteList in the current directory: a team chat screen over a large body of history, with a sliding window so the client never holds more than a bounded number of messages, bidirectional cursor pagination, live incoming messages, and deterministic tests. It is a starting point for adding features in an interview, so keep it small, readable, and conventional. Build it, run the tests, and launch it on an iPhone simulator before reporting done. Do not add features beyond this spec.

## Project setup

- Generate the Xcode project with xcodegen from a `project.yml`. iOS 18 deployment target, iPhone only. One app target and one unit-test target that depends on it, both in the app scheme.
- Swift 6 language mode with strict concurrency, Approachable Concurrency on, and default actor isolation set to MainActor on every target. This matches a new Xcode 26 project. Rely on the default rather than annotating types with `@MainActor`.
- Conventions: named constants live in a nested `enum Constants` inside the enclosing type, never as bare statics or globals. Dependencies are structs of closures, not protocols, so tests can supply partial behavior inline. No `Sendable` on any protocol. No third-party packages.

## Domain

- A message has a server-assigned, monotonically increasing integer id that doubles as its sort key, an author, text, a timestamp, and whether it is the current user's.
- Cursors are opaque tokens the server hands out; the client never constructs or interprets one.
- A fetched page is chronological and carries two cursors captured at fetch time: one for the page before it (nil at the beginning of history) and one for the page after it (nil when the page was the tail at fetch time).
- The client seam is a single struct with three closures: fetch a page by cursor and direction (nil cursor in the older direction means the latest page), subscribe to incoming messages as an async stream, and send a message that comes back with its server id.

## Mock server

- In-memory log of about 30 days of history at a few hundred messages per day, generated deterministically with a seeded generator so every launch is identical. Timestamps walk backward from a few minutes ago through each day's active hours and skip the nights, so today is a partial day and the newest page is this morning's.
- Pages of 30 with about half a second of simulated latency. One injected transient failure several days back so the error-and-retry path is reachable by scrolling.
- A background participant posts a new message every few seconds to all subscribers. Sending appends to the log and returns the stored message.
- A factory on the client struct wires the three closures to a server instance, with the chatter switchable off for tests.

## Window model

An `@Observable` model that owns the window and is the only place state mutates.

- Hold messages at page granularity, because the cursor that makes an evicted direction loadable again lives on the neighboring page. Expose a flattened chronological array for the view.
- Window limit of three pages. After loading in either direction, evict whole pages from the opposite end until the window fits. Never evict the last remaining page. Keep a seen-id set scoped to the window for dedupe across overlapping pages, and rebuild it on eviction.
- The window is at the tail when its last page has no newer cursor. Live messages append to the tail page only in that state and may trigger eviction from the front; otherwise they are counted as pending so the UI can offer a jump to latest.
- Loading older and loading newer are independent, each with its own in-flight task and its own phase (idle, loading, failed). Both are idempotent so the view can call them on every scroll frame. A failed direction does not retry on its own; scroll-driven calls are no-ops until an explicit retry, which asks for the same cursor.
- Initial load fetches the latest page and then starts listening to the incoming stream. Jump to latest resets the window to the latest page. Any reset bumps a generation counter and cancels in-flight loads, and results from an earlier generation are discarded rather than applied.
- A stop method cancels everything for when the screen goes away.
- Sending trims whitespace, ignores empty text, and treats the returned message like an incoming one. Send failures are out of scope.
- Debug aid: publish, for each message, whether it is among the first three or last three rows of its page, so the view can tint page boundaries.

## Screen

- Navigation stack with the title "Team chat" and the current window count in the trailing toolbar.
- A scroll view with a lazy vertical stack, not a List: the chat needs bottom anchoring, hold-on-prepend, and free bubble layout, and the window makes cell recycling unnecessary. Rows are identified by message id and the stack is a scroll target layout.
- Open at the newest message. Anchor the default scroll position to the bottom only for the initial offset and for alignment of short content. Do not use the unscoped bottom anchor: it also governs content-size changes, so every prepend would snap back to the bottom and history would appear never to load.
- Bind the scroll position to a message id with a bottom anchor so the row at the bottom of the viewport is held in place when an older page prepends above it.
- Pagination trigger: compute the distance from the viewport to the top and bottom of the content on every scroll geometry change, store it in a plain reference type (not State, not Observable, so per-frame updates never invalidate the view), and request older when near the top and newer when near the bottom, within a prefetch distance of a few hundred points. Re-run the same check whenever either load phase changes, so a viewport parked at an edge keeps paging after a page arrives. Do not reduce the geometry to a near-edge enum that only fires on transitions; that is how the viewport gets stuck.
- Edge rows at the top and bottom of the stack show a spinner while loading, a red error with a prominent Retry button on failure, and "Beginning of conversation" when history is exhausted. They also request their direction on appear as a fallback trigger. The bottom edge row exists only when there is a newer direction to load.
- Day dividers between messages on different calendar days, and above the first row in the window.
- Message bubbles: the current user's trailing-aligned in the accent color, others leading-aligned in a gray bubble with the author above and the time below.
- A pinned-to-bottom flag derived from scroll geometry; when a message appends while pinned and at tail, scroll to it.
- A "Jump to latest" pill over the bottom when the window is not at the tail or there are pending messages, showing the pending count; it resets the window and scrolls to the end.
- A composer in the bottom safe-area inset with a multiline text field and a send button disabled while the draft is blank.
- Debug tint on rows at page boundaries: three distinct cool colors for the first three rows of each page and three distinct warm colors for the last three.
- Stop the model on disappear.

## Tests

Swift Testing against the model, deterministic with no sleeps and no yields. Use a gated fake server that records every fetch, holds each one open on a continuation until the test releases or fails it, lets the test wait until the model has entered a fetch, and exposes the incoming stream so the test can push a message. Cover at least:

- Initial load is the latest page and lands at tail.
- Older prepends and dedupes an overlapping id.
- Scrolling up past the window limit evicts the tail page and makes newer loadable.
- Newer re-fetches the evicted page from the kept page's cursor and evicts the oldest.
- Live messages append only at tail; otherwise they are counted as pending.
- A live append at tail evicts from the front and keeps the older cursor.
- Jump to latest discards an older page that was in flight.
- A failure stops scroll-driven loads until retry, and retry asks for the same cursor.
- Older and newer can be in flight together, but neither twice.
- Send appends the returned message.
- One end-to-end test against the real mock server with zero latency: page back repeatedly, retry through the injected failure, and confirm the window crossed more than one day boundary while staying within its limit.

## Working method

Builds and test runs are the expensive steps, one to two minutes each, so batch work and validate in as few cycles as possible:

1. Write every source and test file first, without building in between.
2. Build once for an iPhone simulator. Fix every error and warning, then build again. Repeat only until clean.
3. Run the test target once. Fix failures and rerun only as needed.
4. Install and launch on the simulator exactly once, at the end, and take one screenshot to confirm the screen opens at the newest message with the window count and the warm tint on the newest page's last rows.

Do not build after each file, do not relaunch after each fix, and do not take screenshots to confirm things a passing build or test already proved. Scroll behavior (anchoring on prepend, continued paging at the top) cannot be verified by build, tests, or a screenshot; say so in the final report and leave that check to me.

## Report

When done, state: the number of build cycles and test cycles used, whether all tests passed, the generated scheme name and bundle id, and anything in the spec you could not satisfy.
