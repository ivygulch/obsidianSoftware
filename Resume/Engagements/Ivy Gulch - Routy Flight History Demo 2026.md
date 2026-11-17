---
type: engagement
company: ""
brand: "Routy"
role: "Architect and Technical Lead, agent-directed build (personal demo)"
employment: independent
start: 2026-05
end: 2026-06
concurrent_with: ["[[CLEAR - Senior iOS Subcontract 2026-present]]", "[[Jason Poremba - FirefighterSpot 2026-present]]"]
industry: [Aviation, Travel, Data Visualization, Mobile Applications]
team_size: "solo, directing Claude Code agents"
team_context: "architect, reviewer, and acceptance bar for an agent-built full-stack demo"
scale: "portfolio demo; 1,584 real flight tracks rendered on the globe"
tech: [Swift, SwiftUI, The Composable Architecture, TCA, Swift Testing, Swift Package Manager, Mapbox Maps iOS, GeoJSON, Kotlin, Ktor, kotlinx-coroutines, PostgreSQL, HikariCP, OkHttp, Caffeine, Fly.io, Docker, AeroAPI, FlightAware, pg_cron, Claude Code, Linear]
practices: [Agent-Directed Development, iOS Architecture, Map-Based Interfaces, Data Visualization, Backend Development, API Design, Pagination, Caching, Code Review, Concurrency, Automated Testing, Secrets Handling, SPM Modularization]
themes: [agentic-tooling, mapbox, maps, visualization, tca, full-stack, kotlin, postgres, pagination, concurrency, demo, aviation]
deliverables: [routy-ios-client, routy-backend-service, routy-history-globe]
status: past
created: 2026-11-03
updated: 2026-11-03
---

# Ivy Gulch - Routy Flight History Demo 2026

## Context

Routy is a personal, agent-built portfolio demo: a flight-history visualizer that lazy-fetches a flight's real ADS-B trajectory from FlightAware's AeroAPI, caches it in PostgreSQL, and renders every flight the user has taken as a smooth ribbon on a Mapbox globe. Built May to June 2026 to mirror a target employer's stack end to end (SwiftUI and TCA on iOS; Kotlin, Ktor, and Postgres on Fly.io). Doug set the architecture, the milestones, and the acceptance bar, directed Claude Code agents through the build, and reviewed the iOS client and the globe; the agents wrote the Swift and Kotlin. The backend (a lazy-fetch write-through cache over AeroAPI with a license-compliant schema, sealed-class errors, structured logs, and an agent-written adversarial review whose fixes the agents also made) was built with little attention from Doug and is not claimed as his work. It is a demo with synthetic flight dates over real track geometry, not a production or shipped app.

## Role

Architect and technical lead for an agent-directed build. Chose the stack and module boundaries, specified the hero visualization and its encodings, and held the line on secrets handling and client test coverage. Trusted the agents on the backend. No hand-written application code.

## Achievements

- Directed an agent-built full-stack flight-history demo end to end: a SwiftUI and TCA iOS client, a Kotlin/Ktor and PostgreSQL backend deployed on Fly.io, and FlightAware's AeroAPI as the trajectory source. #agentic-tooling #agent-directed #full-stack #architecture #kotlin #tca
- Specified and reviewed the Mapbox Maps iOS globe that is the app's hero view: one GeoJSON source feeding glow, main, and selected LineLayers, with data-driven expressions encoding recency as opacity, corridor frequency as width, and a selectable color-by dimension as hue. #mapbox #maps #visualization #geojson
- Designed filter and legend-focus behavior so changes cross-fade instead of hard-cutting, by folding them into the layer's opacity expression rather than a layer filter; added tap-to-select, a cold-open fly-in to the user's home hub, and satellite and dark base styles. #mapbox #interaction #motion #ux
- Verified the globe on the simulator with 1,584 real flight tracks rendering as GPU-batched threads from a single source and layer. #performance #mapbox #scale
- Isolated the Mapbox SDK to a single view target in the Swift package so feature logic and its tests link neither the SDK nor an access token, keeping the package testable on a macOS host. #spm #modularization #architecture #testing
- Replaced a monolithic visualization bundle with paginated `/stats` and `/flights` endpoints and TCA paged loading, so threads render progressively as pages arrive. #pagination #api-design #tca #performance
- Held a TCA TestStore suite in Swift Testing across the feature, service, and app targets as the acceptance bar for the agents' client code. #swift-testing #tca #testing
- Kept Mapbox and vendor tokens out of source and the shared Xcode scheme: a gitignored env file and a private scheme supply them at launch. #security #secrets

## Tech Stack

Swift, SwiftUI, The Composable Architecture 1.25, swift-dependencies, Swift Testing, Swift Package Manager (five targets), Mapbox Maps iOS 11 with GeoJSON sources and data-driven LineLayer expressions; Kotlin 2.4, Ktor 3.5 on Netty, kotlinx-coroutines, HikariCP, PostgreSQL with pg_cron, OkHttp, Caffeine; Fly.io with Docker; AeroAPI (FlightAware); Claude Code with Linear for tracking.

## Private Notes

_Agent-directed, not hand-written. Doug managed the agents and never worked in the code himself. **Backend (2026-11-03): Doug paid little attention to it and trusted the agents; he removed the backend-review bullet from the Raya docx himself. Never claim the backend design, its review, or its fixes as his work; the backend may be named only as part of the stack the agents built.** Never present this as production experience, a shipped app, or hands-on Swift or Kotlin work. Honest framing: what agents produced end to end under direction, and what review found and fixed. On 2026-10-07 Doug rejected a Routy entry in the American Eagle Outfitters generation and said personal Claude Code projects were not resume material; on 2026-11-03 he asked for this engagement to be created so the Mapbox and TCA work is available when a posting calls for it (Raya Places, 2026-11-03). Use per target; keep the agent-directed wording in any bullet that goes out._

_Facts verified in the repo 2026-11-03: 36 commits 2026-05-27 to 2026-06-17; 52 `@Test` functions across three Swift test targets; six Kotlin test files; Mapbox Maps iOS 11 in the package manifest, imported only by VisualizationView. The backend's AppIO dispatcher, striped-mutex coalescing, pagination, and LazyFetchServiceTest are present in code (agent work; see above). The `prep/*.md` docs in the repo are stale and still describe those fixes as open. Flight dates in the demo corpus are synthetic (redated across 12 years); track geometry, times of day, and durations are real. The 1,584-track figure is from the 2026-06 simulator verification screenshots in `docs/spikes/`. The "mirror a target employer's stack" was Flighty; keep the employer unnamed in output._

## Related

- Concurrent engagements: [[CLEAR - Senior iOS Subcontract 2026-present]], [[Jason Poremba - FirefighterSpot 2026-present]]
- Repo: `~/ivygulch/flighty-routes` (prep docs under `prep/`, spike notes and screenshots under `docs/spikes/`)
- Skills: _(to be linked once Skills/ pages exist)_
