---
type: synthesis
series: Isolation
created: 2026-11-03
---

# Isolation Review Matrix

Review grid for Swift isolation across three facets: what kind of type (struct, class, actor), what kind of member access (stored property, function call, function argument), and which isolation setting is in play (module default MainActor or not, `nonisolated`, `sending`, `Sendable`). Each table isolates one slice; read across a row to see how a setting changes the answer.

## Assumptions

- Swift 6.2 / Xcode 26 language mode 6, strict concurrency.
- "Approachable concurrency" flags on unless a cell says otherwise: `NonisolatedNonsendingByDefault` (SE-0461) and `InferIsolatedConformances` (SE-0470). Cells marked **legacy** describe behavior with `NonisolatedNonsendingByDefault` off.
- "Default MainActor" means the per-target `-default-isolation MainActor` setting (SE-0466). Cells assume it is **off** unless the table says otherwise; Table 5 is the on/off comparison.
- "Crossing" means the caller's isolation domain differs from the callee's or the owning type's. Same domain never requires `Sendable`, `await`, or `sending`.
- Cells marked *(verify)* are ones I am not fully certain of; check against the proposal text or the compiler before relying on them.

Proposals referenced: SE-0306 actors, SE-0316 global actors, SE-0414 region isolation, SE-0420 `isolated` params / `#isolation`, SE-0430 `sending`, SE-0434 global-actor type usability, SE-0461 `nonisolated(nonsending)` / `@concurrent`, SE-0466 default isolation, SE-0470 isolated conformances.

## 1. Inferred isolation and Sendability by type kind

What the compiler assumes when you write nothing, and what each explicit annotation does.

| Type kind | No annotation (module default nonisolated) | `@MainActor` on the type | `nonisolated` on the type | Implicitly `Sendable`? |
|---|---|---|---|---|
| `struct` / `enum` | nonisolated; members run wherever the caller is | Every stored property and method isolated to MainActor; type becomes Sendable because global-actor types are Sendable | Opt out (only meaningful under default MainActor) | Yes if all stored properties are Sendable and the type is non-public; public needs explicit `Sendable` |
| `class` | nonisolated, **non-Sendable**; can only be used in one region at a time | Isolated to MainActor; type becomes Sendable (the reference may cross, access to state may not) | Opt out | Never inferred. Only valid to declare if `final` and all stored properties are immutable Sendable `let`s, or `@unchecked` |
| `actor` | Own isolation domain; every stored property and method isolated to the instance | **Error**: an actor cannot carry a global-actor attribute *(verify exact diagnostic)* | Per member only, not on the type | Always Sendable |
| `protocol` | nonisolated requirements | Conformers inherit MainActor isolation for the requirements | Opt out under default MainActor | n/a; `: Sendable` on a protocol forces all conformers Sendable (see Table 5 row "Protocol that inherits Sendable") |
| global `func` | nonisolated | Runs on MainActor; callers elsewhere must `await` | Opt out | n/a |
| closure literal | Inherits the enclosing context's isolation if not `@Sendable`; `Task { }` inherits via `@_inheritActorContext` | `@MainActor { }` pins it | `nonisolated { }` opts out | `@Sendable` closures must capture only Sendable values; **exception**: a global-actor-isolated closure may capture non-Sendable values and still be Sendable (SE-0434) |

## 2. Stored property access

Columns are the caller's position relative to the owning type. Rows are the owning type's isolation crossed with the property's declaration. "Sendable" and "non-Sendable" refer to the **property's type**.

| Owner | Property | Caller in same domain | Caller in a different domain |
|---|---|---|---|
| Plain `class` / `struct` (nonisolated, non-Sendable) | any `let` / `var` | Synchronous read and write | You should not hold the reference at all. The compiler blocks the *object* from crossing (region analysis), so there is no property-level rule. A non-Sendable value that did cross via `sending` is now in your region and behaves like the same-domain column |
| Plain `final class` declared `Sendable` | `let` Sendable | Sync read | Sync read from anywhere; that is the only reason it could be Sendable |
| `@MainActor` class or struct | `let` Sendable | Sync read | **Sync read, no `await`**: inferred `nonisolated` within the module (SE-0434). Across modules only if declared `nonisolated let` *(verify cross-module rule)* |
| `@MainActor` class or struct | `let` non-Sendable | Sync read | Not accessible. The value lives in the MainActor region and reading it would move a non-Sendable value out. `await obj.prop` is an error |
| `@MainActor` class or struct | `var` Sendable | Sync read and write | Read via `await obj.prop`. **Write is never allowed cross-actor**, even with `await`; call an isolated method instead. Struct only: may declare `nonisolated var` of Sendable type (SE-0434) *(verify)* |
| `@MainActor` class or struct | `var` non-Sendable | Sync read and write | Not accessible, same reason as `let` non-Sendable |
| `@MainActor` class or struct | `nonisolated(unsafe) var` | Sync | Sync, unchecked. Escape hatch; you own the data-race proof |
| `actor` | `let` Sendable | Sync | **Sync read, no `await`** within the module (SE-0306 same-module rule). Cross-module requires `nonisolated let` |
| `actor` | `let` non-Sendable | Sync | Not accessible; `nonisolated let` of non-Sendable type is itself an error |
| `actor` | `var` Sendable | Sync read and write | Read via `await actor.prop`. Write never allowed cross-actor; expose a method |
| `actor` | `var` non-Sendable | Sync read and write | Not accessible. Expose a method that returns a Sendable projection or a `sending` copy |
| `actor` | `nonisolated var` (computed only) | Sync | Sync; cannot touch isolated state in the body |
| `actor` | `nonisolated(unsafe) var` stored | Sync | Sync, unchecked |
| any isolated owner | Method with `isolated` parameter of the owner's type | Sync (you are the parameter) | `await` on the call site that supplies the actor |

## 3. Function calls

Rows are the callee's declaration. Columns are where the caller stands. "Hop" means the executor changes, which is where `Sendable` and `sending` are enforced.

| Callee | Caller in same domain | Caller in a different domain | Caller in nonisolated sync code | Executor the body runs on |
|---|---|---|---|---|
| nonisolated **sync** func or method | Sync call | Sync call, no hop | Sync call | Caller's. Args and result need nothing, but `self` must already be in the caller's region |
| nonisolated **async**, `nonisolated(nonsending)` (the default with the flag on) | `await`, no hop | `await`, no hop; inherits caller isolation | Not callable without an async context | Caller's. Args, result, and `self` need not be Sendable. Still a suspension point for reentrancy purposes |
| nonisolated **async**, `@concurrent` or **legacy** default | `await`, hops off | `await`, hops off | n/a | Global concurrent executor. Args and result must be Sendable or `sending`; `self` must be Sendable, so a method on a non-Sendable class gives "sending self risks data races" |
| `@MainActor` or actor-isolated **sync** method | Sync call | `await`, becomes implicitly async at the call site | `await` in a `Task`, or `MainActor.assumeIsolated` if you can prove you are already there | Callee's. Args Sendable or `sending`; result Sendable or `sending` |
| `@MainActor` or actor-isolated **async** method | `await`, no hop | `await`, hops | Same as above | Callee's. Same Sendable/`sending` rule, plus every internal `await` is a reentrancy window |
| Method with `isolated` parameter (`func f(isolated a: SomeActor)` or `isolated any Actor = #isolation`) | Sync when passing the actor you are on | `await` when passing a different actor | n/a | The passed actor's. Lets a free function join an isolation instead of forcing one |
| `nonisolated` method **on an actor** | Sync | Sync, no hop | Sync | Caller's. Body cannot touch isolated state except `let` Sendable |
| `Task { }` from an isolated context | Inherits the enclosing isolation | n/a | nonisolated | Enclosing actor. Closure is `sending`, so non-Sendable captures are allowed once and unusable afterward |
| `Task.detached { }` | Hops off | Hops off | Hops off | Global concurrent. Closure is `sending`; captures must be Sendable or disconnected |

## 4. Function arguments and return values

How a value gets from the caller's region to the callee's. Rows are the value; columns are the parameter or result declaration. Only applies when a boundary is crossed; same-domain calls pass anything.

| Value being passed | Plain parameter | `sending` parameter | `inout sending` | `sending` return | `isolated` parameter |
|---|---|---|---|---|---|
| Sendable value | Crosses freely | Allowed, redundant | Allowed | Allowed, redundant | n/a |
| Non-Sendable value in a **disconnected** region (freshly created, never aliased) | Allowed into an actor-isolated callee by region analysis (SE-0414): the value merges into the actor's region and the caller may not use it afterward. Into a legacy nonisolated async callee: error | Allowed; same post-condition, but the callee's signature now documents it and the callee may store or forward it | Allowed; callee gets exclusive access, value must be disconnected on exit (`Mutex.withLock` state) | Caller receives a fresh region it owns | n/a |
| Non-Sendable value in an **actor-isolated** region (stored on an actor or `@MainActor` type) | Error | Error: not disconnected | Error | Error unless the callee creates a new value rather than returning stored state | n/a |
| Non-Sendable value in a **task-isolated** region (a `sending` parameter you received, a captured `Task` value) | Error if crossing | Allowed if not used afterward | Allowed | Allowed | n/a |
| Same non-Sendable value used again **after** the send | Error: use after send | Error | Error | n/a | n/a |
| Non-Sendable value inside a **loop** | Cannot send; compiler cannot count iterations (ep 367) | Same | Same | Same | n/a |
| Two non-Sendable values in **one** region, sent to **different** owners | Error: regions merge on alias; split by constructing them independently (ep 368) | Same | Same | Same | n/a |
| An `actor` or global-actor-isolated object | Sendable, crosses freely; access to its state is still gated | Redundant | n/a | Redundant | Pass it as `isolated` to run the callee **inside** that actor with sync access |
| `@Sendable` closure | Captures must be Sendable, or the closure itself is global-actor-isolated (SE-0434) | n/a | n/a | n/a | n/a |
| Non-`@Sendable` closure, `sending` | n/a | May capture non-Sendable values if they are disconnected after; this is `Task.init` and `Mutex.withLock` | n/a | Closure returned as `sending` disconnects its captures from the callee | n/a |
| Non-`@Sendable` closure, plain parameter | Treated as a non-Sendable value: crosses only under region analysis; inherits caller isolation if non-escaping | n/a | n/a | n/a | n/a |

## 5. Default MainActor isolation: off vs on

Same declarations, two module settings. "Other module" means a dependency compiled without default MainActor; a reusable library should never be default-MainActor (ep 369).

| Declaration in this module | Default MainActor **off** | Default MainActor **on** |
|---|---|---|
| `struct` / `class` with no annotation | nonisolated, class is non-Sendable | `@MainActor`, implicitly Sendable; every method and stored property gated to main |
| `actor` | Own isolation | Unchanged; still its own isolation, and MainActor code must `await` into it |
| global `func` | nonisolated | `@MainActor` |
| `protocol` declared here | nonisolated requirements | `@MainActor` requirements; conformers from other modules must be MainActor or use isolated conformance |
| `nonisolated` on a type or member | No-op | The opt-out; required for anything called from a non-main context |
| Conformance to a **nonisolated protocol from another module** | Requirements must be `nonisolated` unless type is `@MainActor` and `InferIsolatedConformances` is on, which yields an isolated conformance usable only on main | Isolated conformance inferred; conformance is only usable from MainActor; passing the type where the protocol is used generically off-main is an error |
| Conformance to a protocol from another module that **inherits `Sendable`** | Fine for a nonisolated struct; a class needs real Sendability | **The ep 369 thorn**: a Sendable protocol cannot have an isolated conformance, so requirements must be `nonisolated`, which pushes them into a `nonisolated extension` with no access to MainActor state. Fix the protocol, not the conformer |
| Explicit `: Sendable` on a type here | Normal rules; non-Sendable classes cannot conform | Redundant; the type is already Sendable by virtue of `@MainActor`. Harmless |
| Async method with no annotation | `nonisolated(nonsending)`: runs on caller | `@MainActor async`: pulls callers onto main |
| `Task { }` at top level or in a type | Inherits enclosing isolation (none at top level) | Inherits MainActor |
| `Task.detached` / `@concurrent func` | Off main | Off main; the explicit way out |
| Test functions (Swift Testing) in a test target with the setting on | nonisolated, run concurrently | `@MainActor`; tests serialize onto main |
| `nonisolated(unsafe)` / `@unchecked Sendable` | Same escape hatches | Same; more tempting because more things are gated |

## 6. Escape hatches by facet

| Hatch | Facet | What it disables | Prefer instead |
|---|---|---|---|
| `nonisolated(unsafe) var` | property | All isolation checking on that property | `Mutex<State>` for shared mutable state; a justifying comment if kept (TCA26 style) |
| `@unchecked Sendable` | type | Sendable inference for the whole type | `sending` to move it once; actor ownership; `nonisolated(unsafe)` on the one offending property |
| `@preconcurrency import` | module boundary | Downgrades Sendable and isolation errors from that module to warnings | Pin to a fixed version of the dependency |
| `MainActor.assumeIsolated { }` / `actor.assumeIsolated` | function call | Runtime check instead of compile-time proof; traps if wrong | `isolated` parameter or `#isolation` to carry the proof |
| `@concurrent` | function call | Opts a nonisolated async function back to the hop-off behavior | Leave nonsending unless the work is CPU-bound and must leave the caller's executor |
| `Task.detached` | function call | Discards inherited isolation and priority | `Task { }` plus `@concurrent` callee, or an explicit executor |

## 7. The three `nonisolated` forms

They share a keyword but control different things. "Protection" means the actor's guarantee that nothing else touches the instance's isolated state while the member runs. "Executor" means where the body runs and therefore whether a caller must hop.

| | `nonisolated` | `nonisolated(nonsending)` | `nonisolated(unsafe)` |
|---|---|---|---|
| What it controls | Where code runs | Whether an **async** function hops off the caller | Whether the checker looks at a stored property |
| Applies to | Methods, computed properties, `let`s of Sendable type, whole types and functions under default MainActor | Async functions and methods only; meaningless on sync code | Stored properties, including global and `static var` |
| Protection | Given up; body cannot touch isolated `var`s, compiler enforces this | Not involved; the function was already nonisolated | Disabled for that storage; you provide the synchronization |
| Executor | The caller's, synchronously, no hop | The caller's; inherits caller isolation instead of hopping to the global concurrent executor | Unchanged |
| Sendable demands | None, nothing crosses | None: `self`, args, and result stay in the caller's region | None, nothing is checked |
| Compiler still verifies | Yes | Yes | No |
| Default with approachable concurrency | n/a | This is what plain `nonisolated async` already means; `@concurrent` is the explicit way back to hopping | n/a |
| One-line reading | This member does not need the actor, so run it on the caller | This async function should stay with the caller, so nothing has to be Sendable | Stop checking this property; I promise it is safe |
| Prefer instead | Nothing; this is the right tool when the body has no dependency on isolated state | Nothing; leave the flag on | `Mutex`, actor ownership, or immutability; keep a justifying comment if it stays (TCA26 style) |

### `nonisolated`, articulated

**One sentence.** `nonisolated` removes a declaration from the isolation domain it would otherwise inherit, so it runs on the caller's executor and gives up access to that domain's state.

**Three consequences, each following from the one above.**

1. The body may not touch isolated state. It can use its arguments, Sendable `let`s, other nonisolated members, and anything it creates itself. The compiler enforces this, so `nonisolated` is a checked promise, not an escape hatch.
2. Because the body depends on nothing the actor protects, there is no reason to run it on the actor. It runs on whatever executor the caller is already on.
3. Because no executor boundary is crossed, callers anywhere use it synchronously: no `await`, no hop, no Sendable requirement on arguments or results. If the function is also `async`, the `await` is still required for that reason alone, but there is still no hop.

**What it applies to.** Members of actors and `@MainActor` types; under default MainActor isolation, whole types and global functions as the opt-out; conformances, where `nonisolated` declares the requirements are satisfied without isolation.

**Three misreadings an explanation must rule out.**

- "It runs in the background." No. It runs wherever the caller is, including main if the caller is on main.
- "It is unsafe." No. Full checking remains; `nonisolated(unsafe)` is the one that disables checking, and it applies to storage.
- "A `nonisolated async` method is detached." No. With nonsending as the default it inherits the caller's isolation; `@concurrent` is the one that hops off.

**Test case.** A method whose body uses only a Sendable `let` of the actor. Marked `nonisolated`, it is a free synchronous call from anywhere. Left isolated, every outside caller must `await` and hop for nothing. Explaining why the first is correct and the second is wasteful is the proof of understanding.

### Why "does not need the actor" matters

Isolation bundles two things: the protection and the executor. A method needs the protection only if it reads or writes isolated `var`s. A method that uses only its arguments, Sendable `let`s, or other nonisolated members gains nothing from it, yet an isolated method still forces every cross-domain caller to `await`, hop, and make its arguments and result Sendable. `nonisolated` drops both halves: the body loses access to isolated state, and in exchange callers anywhere use it synchronously with no hop and no Sendable requirements.

```swift
actor Bank {
  let name: String
  var accounts: [Account] = []

  nonisolated var displayName: String { "Bank of \(name)" }   // only uses a Sendable let

  func deposit(_ amount: Int, to id: Account.ID) { ... }       // needs isolation
}

let bank = Bank(name: "First")
print(bank.displayName)              // sync from anywhere
await bank.deposit(10, to: id)       // must hop
```

The same reasoning applies to `@MainActor` types and default-MainActor modules: a formatting helper or a `Codable` conformance on a `@MainActor` model does not need the main thread, and `nonisolated` lets a background task call it without an `await`.

## 8. Where the settings live

`NonisolatedNonsendingByDefault`, `InferIsolatedConformances`, and default MainActor isolation are all per-target. The first two are upcoming-feature flags; default isolation is its own setting. All three become unconditional in Swift language mode 7 whenever that ships, so the flags are early adoption in mode 5 or 6.

| Build system | Nonisolated nonsending | Isolated conformances | Default MainActor |
|---|---|---|---|
| SwiftPM `Package.swift`, per target `swiftSettings` | `.enableUpcomingFeature("NonisolatedNonsendingByDefault")` | `.enableUpcomingFeature("InferIsolatedConformances")` | `.defaultIsolation(MainActor.self)`; needs `swift-tools-version: 6.2` |
| Xcode target, Build Settings, Swift Compiler Upcoming Features | "Approachable Concurrency" (`SWIFT_APPROACHABLE_CONCURRENCY = YES`) turns on both flags plus the other 6.2 approachable features. Single flag only: `SWIFT_UPCOMING_FEATURE_NONISOLATED_NONSENDING_BY_DEFAULT = YES` or `-enable-upcoming-feature NonisolatedNonsendingByDefault` in Other Swift Flags | Same bundle; single flag `SWIFT_UPCOMING_FEATURE_INFER_ISOLATED_CONFORMANCES = YES` | "Default Actor Isolation" (`SWIFT_DEFAULT_ACTOR_ISOLATION`), values `MainActor` or `nonisolated` |
| Command line | `swiftc -enable-upcoming-feature NonisolatedNonsendingByDefault`; through SwiftPM: `swift build -Xswiftc -enable-upcoming-feature -Xswiftc NonisolatedNonsendingByDefault` | `-enable-upcoming-feature InferIsolatedConformances` | `-default-isolation MainActor` |
| Xcode 26 new-project defaults | On for app targets via Approachable Concurrency | On, same bundle | `MainActor` for new app targets |

Point-Free's position on the three ([[365 - Approachability]], [[369 - MainActor Default]]): turn on both flags everywhere; leave default MainActor off, and never turn it on for a reusable library.

## 9. When `await` is required

`await` marks a potential suspension, not an `async` declaration. There are two sources: the callee is `async`, or the call crosses into an actor's isolation. Either one alone requires it. Every `await` is a reentrancy window whether or not an executor hop happens.

| Member on an actor or `@MainActor` type | From inside that isolation (self, `isolated` parameter, closure created inside, `assumeIsolated`) | From outside |
|---|---|---|
| Isolated **sync** method | Sync, no `await` | `await`: the call is implicitly async because getting onto the executor may suspend. Caller must be in an async context |
| Isolated stored `var` or computed property | Sync read and write | `await` to read; writes are an error regardless of `await`, go through a method |
| Isolated **async** method | `await`, no hop | `await`, hop |
| `nonisolated` **sync** method or property | Sync | Sync; runs on the caller. `await` here is an error: "no async operations occur within await expression" |
| `nonisolated` **async** method | `await`, no hop | `await`; no hop with nonsending, hop if `@concurrent` |
| Sendable `let` stored property | Sync | Sync within the module (implicitly nonisolated); other modules need an explicit `nonisolated let` |

```swift
actor Bank {
  var total = 0
  func deposit(_ amount: Int) { total += amount }   // sync, isolated
  nonisolated func describe() -> String { "Bank" }  // sync, nonisolated
}

let bank = Bank()

// From outside the actor
await bank.deposit(10)        // await required: crossing into the actor
let t = await bank.total      // same for an isolated property read
let d = bank.describe()       // no await: nonisolated, runs on caller

// From inside the actor
func audit(bank: isolated Bank) {
  bank.deposit(10)            // no await: already in the actor's isolation
  print(bank.total)
}
```

Compact rule: `await` whenever you cross into an isolation you are not already in, and whenever the callee is `async`. `nonisolated` removes the first reason, never the second.

## 10. Sendable `let`

A stored constant property whose declared type conforms to `Sendable`, on an actor or a global-actor-isolated type. It is the one kind of stored property readable from any isolation without an `await`, because both halves of a data race are ruled out by the declaration alone.

**Why it is safe without the actor.** A data race needs a write that can overlap another access. A `let` is written once, during initialization, before any other isolation can hold the instance. After that there are only reads. Sendable on the type closes the remaining hole: reading a Sendable value from any thread is never a mutation in disguise. The actor has nothing to protect, so the compiler treats the property as nonisolated.

**Why the type must be Sendable, not just the binding immutable.** `let` freezes the reference, not the thing it points to. `let cache: NSMutableDictionary` on an actor is a constant binding to a mutable object; handing it out synchronously would let two domains mutate the dictionary. A non-Sendable `let` stays isolated, and `nonisolated let` on it is an error. Same for a `let` of a struct that contains a non-Sendable class reference, since that struct is itself non-Sendable.

**Within the module it is automatic.** On an actor since SE-0306; on `@MainActor` and other global-actor-isolated classes and structs since SE-0434. `bank.name` reads synchronously anywhere in the same module with no annotation.

**Across modules you must write `nonisolated let`.** Inference stops at the module boundary because of library evolution: a library may replace a stored `let` with a computed property later without breaking source, and a computed property on an actor is isolated. Clients cannot assume a `let` in the interface stays synchronously readable. `nonisolated let` is the author's commitment that it will. Inside the module the two spellings are identical; the explicit form doubles as documentation.

| Declaration | Sync from any isolation? |
|---|---|
| `let name: String` on an actor, same module | Yes |
| `let name: String` on an actor, other module | Only if declared `nonisolated let` |
| `let id: UUID` on a `@MainActor` class, same module | Yes (SE-0434) |
| `let cache: NSMutableDictionary` on an actor | No; `nonisolated` on it is an error |
| `var count: Int` on an actor | No; `await` to read, method to write. `nonisolated var` is only for computed properties |
| `static let shared = Config()` with `Config: Sendable` | Yes; global and static `let`s of Sendable type are safe everywhere (SE-0412) |
| `static var` of any type | No; must be isolated to a global actor or marked `nonisolated(unsafe)` |

**Why Point-Free leans on it.** It is the cheapest way to let a differently isolated holder use part of an actor without a hop. Identity, names, configuration, and injected dependencies that are themselves Sendable belong in `let`s, which keeps the actor's isolated surface down to the state that actually changes.

## 11. Proving you are in an actor's isolation

Synchronous access to an actor's isolated state requires the compiler to see that the code location is inside that instance's isolation. "I know I am on the same thread" is not proof. For an instance actor there are exactly four accepted forms.

| Form | How you get there | What you get |
|---|---|---|
| The actor's own members | Defining the code on the actor or in an extension | Sync read and write of all isolated state |
| Closure created inside an isolated member | `Task { }` or a non-`@Sendable` closure written in an isolated method inherits the instance's isolation | Same, inside the closure |
| `isolated` parameter | `func audit(_ c: isolated Counter)`; the caller `await`s once to enter | Same, for the whole function body; lets a free function join the actor |
| `assumeIsolated` | `c.assumeIsolated { c in ... }` when you are already on the actor but the compiler cannot see it (legacy callbacks, custom executors) | Same inside the closure; runtime check, traps if wrong |

```swift
actor Counter {
  var count = 0
  func report() { print(count) }           // 1
  func later() { Task { print(count) } }   // 2, no await
}

func audit(_ c: isolated Counter) {        // 3
  print(c.count)
  c.count += 1
}

func legacyCallback(_ c: Counter) {        // 4
  c.assumeIsolated { c in print(c.count) }
}
```

Entering still costs one `await` at the boundary (`await c.report()`, `await audit(c)`). Inside, every access is synchronous and no other job on the actor can interleave until the code itself `await`s something.

What does not count: same thread, same dispatch queue, the actor being idle, or holding the reference from main or another actor. Those are different isolations and every access is `await`.

Contrast with global actors: for a `@MainActor` type, "same isolation" is any `@MainActor` context anywhere in the program, because there is one main actor. Instance actors are harder to use synchronously because the only way in is through the instance itself.

## 12. Why Apple recommends default MainActor

**The problem.** Most app code is not concurrent: it runs on main, touches UI state, and never meant to be anywhere else. Under Swift 6 with nonisolated as the default, that code sat in a different isolation from every `@MainActor` view and model it talked to, so it produced `await`s, Sendable errors, and "main actor-isolated property cannot be referenced from a nonisolated context" in code with no concurrency in it. The Swift vision document "Improving the approachability of data-race safety" names this directly: the compiler was demanding concurrency proofs from single-threaded code.

**What the setting does about it.** It puts the whole target into one isolation. Section 11 shows that synchronous access needs the compiler to see you are in the same isolation; for a global actor that proof is trivial because every `@MainActor` context is the same one. With the module on main, every struct, class, function, and closure is in that context by default, so property access is synchronous, non-Sendable classes pass freely, and `await` appears only where you deliberately leave main. Concurrency becomes opt-in through actors, `@concurrent`, and `Task.detached`. Apple's phrase: single-threaded by default, concurrency when you ask for it.

**Why it pairs with nonsending.** The other half of the pain was `nonisolated async` functions hopping to the global executor and dragging Sendable requirements along. `NonisolatedNonsendingByDefault` keeps async code on the caller. Apple ships both as "Approachable Concurrency" because each covers a case the other does not: nonsending handles async functions on non-Sendable types, default MainActor handles everything else in UI-heavy targets.

**Where Point-Free parts ways** ([[369 - MainActor Default]]). Once nonsending is on, most of what default MainActor bought is already there, and what remains has costs: libraries cannot use it, protocols that inherit Sendable break isolated-conformance inference, tests serialize onto main, and explicit `@MainActor` on the types that need it is little typing. This is a judgment about the trade, not about what the setting does. Apple bets the trade is clearly worth it for a typical app target; Point-Free bets explicit isolation is cheap enough to prefer. Both agree on nonsending and isolated conformances.

## 13. Isolation vs sendability

Two axes that are easy to blur in conversation because they meet at the boundary.

| | Isolation | Sendability |
|---|---|---|
| Property of | Code locations and declarations | Types and values |
| Question it answers | Who may run this, and what state may it touch | May a value of this type cross from one isolation to another |
| Vocabulary | `nonisolated`, `@MainActor`, `actor`, `isolated` parameter, `nonisolated(nonsending)`, `@concurrent` | `Sendable`, `sending`, regions, `@Sendable` closures |
| Decides whether you write `await` | Yes | Never |
| Decides whether a value may be an argument or result across a hop | No | Yes |
| Escape hatch | `nonisolated(unsafe)` on storage | `@unchecked Sendable` on a type |

A clean description of `nonisolated` uses the word Sendable in exactly one place: a `nonisolated let` must have a Sendable type. A clean description of Sendable never mentions `await`.

**Sentences that mix them.** Each is near something true.

- "`nonisolated` means it can run on any thread, so it has to be Sendable." A nonisolated sync function runs on the caller and crosses no boundary, so nothing must be Sendable. The confusion is the legacy `nonisolated async` hop, which did demand Sendable arguments; that was the hop's requirement, not `nonisolated`'s, and nonsending removed it.
- "`nonisolated` makes it thread-safe." It declares the member does not depend on isolated state. Safety still comes from the checker.
- "I mark it `nonisolated` so I can pass it across actors." Crossing is about the value's type being Sendable or sent. `nonisolated` on a method changes nothing about whether `self` can cross.
- "`Sendable` lets you call it without `await`." Sendable never removes an `await`. A Sendable actor still requires `await` for isolated members.
- "`nonisolated(unsafe)` and `@unchecked Sendable` are the same kind of thing." Both are escape hatches, on different axes.

**Where they legitimately meet.** Fair to say, as consequences rather than definitions.

- A `nonisolated let` needs a Sendable type because it is readable from any isolation.
- Crossing a boundary requires Sendable or `sending` values. `nonisolated` avoids creating a boundary, which is why it removes Sendable requirements rather than adding them.
- Actors and global-actor-isolated types are implicitly Sendable. Isolation implies sendability here, never the reverse.

**Clean answers.**

- `nonisolated`: removes a declaration from the isolation it would inherit. It runs on the caller, cannot touch isolated state, and the compiler checks that. Callers need no `await` unless it is also `async`.
- Sendable: a type whose values can be safely shared across isolation domains. It is what lets a value cross a boundary. It says nothing about where code runs.
