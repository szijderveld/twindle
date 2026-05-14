# Twindle — Decisions

Numbered, append-only log of locked design decisions. Every entry has a title, a one-line decision, and a short rationale. Future build steps append new entries here when they make non-obvious decisions of their own. Existing entries are not edited or renumbered. If a decision is superseded, a new entry records the supersession and references the old number.

---

## D1. Project name: Twindle

**Decision.** The project is named Twindle. The Python package is `twindle`. The repository is `twindle/`.

**Rationale.** Short, distinctive, easy to type. The "twin" root names the central idea (digital twin of a small business) without overpromising. The "-dle" tail keeps it light rather than industrial. Candidates considered: Loomscape (too poetic), Ploma (no anchor), Verity (name clash, too abstract), Sandtable (two-syllable friction, military connotation). Twindle wins on memorability and honesty.

---

## D2. Licence: MIT

**Decision.** The project ships under the MIT licence.

**Rationale.** Permissive, well-understood, frictionless for contributors and adopters. Aligns with the framework's positioning as a tool people pick up and try.

---

## D3. Python version: 3.11+

**Decision.** Minimum supported Python is 3.11.

**Rationale.** Gives access to modern typing (`Self`, `TypedDict` improvements, `ExceptionGroup`) and Pydantic v2's best behaviour, without pushing the floor so high that adopters with stable installs are excluded. 3.11 is widely available on package indexes and OS distributions by 2026.

---

## D4. Package manager: uv

**Decision.** Dependencies and virtual environments are managed by uv. The project does not commit to compatibility with pip-tools, Poetry, or Hatch workflows.

**Rationale.** uv is fast, has a coherent lockfile, handles Python version installation, and gives a single setup command. Single tool keeps the "ten minute clone-to-demo" success criterion realistic.

---

## D5. Project layout: src/ pattern, monorepo

**Decision.** The Python package lives under `src/twindle/`. The React frontend lives under `frontend/` in the same repository as a separate package. Both are versioned and released together.

**Rationale.** The src/ pattern prevents accidental imports from the working directory during tests. The monorepo keeps the Python types and the frontend's typed contract co-located so changes to the domain are caught before they ship.

---

## D6. Validation library: Pydantic v2

**Decision.** All data validation, domain modelling, and tool schemas use Pydantic v2.

**Rationale.** SQLModel sits on Pydantic, Pydantic AI uses Pydantic for tool schemas, and FastAPI (likely transport) speaks Pydantic. Single validation library across all layers.

---

## D7. ORM: SQLModel

**Decision.** Entities are declared as SQLModel classes. SQLModel sits on Pydantic and SQLAlchemy and provides both ORM mapping and Pydantic validation from a single class definition.

**Rationale.** The single-source-of-truth commitment requires that one class definition feeds the database, the simulation, the agent's tool surface, and the frontend's contract. SQLModel is the only library that natively combines Pydantic and SQLAlchemy. Plain SQLAlchemy plus separate Pydantic models would duplicate every entity.

---

## D8. Databases supported: SQLite for local, PostgreSQL-compatible for scaling

**Decision.** v1 targets SQLite for the local demo and writes SQL that is PostgreSQL-compatible. No database-specific extensions are used.

**Rationale.** SQLite removes setup friction for the demo. PostgreSQL compatibility leaves a clear upgrade path without committing to a server runtime in v1.

---

## D9. Simulation engine: SimPy

**Decision.** The simulation runtime is built on SimPy. The framework does not implement its own discrete event scheduler.

**Rationale.** SimPy is mature, pure Python, and its process-and-resource primitives map cleanly onto the framework's base types. Building a scheduler would be a project of its own.

---

## D10. Base type taxonomy

**Decision.** Every domain inherits from exactly these base types, defined in `twindle.domain.base`:

- `Entity`, anything with identity that persists.
- `Resource`, an entity that can be consumed or occupied during a process.
- `Process`, a unit of work that takes time and may need resources.
- `Event`, a business event with schedule, location, and required resources. Distinct from SimPy's internal event plumbing, which is wrapped, not exposed.
- `Schedule`, binds processes to entities over time.
- `Assignment`, links resources to events.
- `Location`, spatial position for map rendering.

**Rationale.** Closed taxonomy is what allows the agent surface, schema, and visualisation to be generated from the domain. Each base type maps onto SimPy primitives in domain language so implementers reason in business terms while the runtime gets standard semantics.

---

## D11. Scenario primitives

**Decision.** The framework defines exactly these scenario primitives in `twindle.scenarios.primitives`:

- `AddEntity`
- `RemoveEntity`
- `ModifyAttribute`
- `ScaleParameter`
- `ChangeSchedule`
- `InjectEvent`

The agent is restricted to composing `Scenario` objects from these. Domains may register additional primitives through a documented extension point. The agent cannot invent primitives at the model layer.

**Rationale.** Closed vocabulary at the agent layer makes scenarios verifiable and reproducible. Open extension at the domain layer keeps the framework general. When a user request falls outside the vocabulary the agent surfaces the limit rather than improvising.

---

## D12. Extensibility stance: fixed primitives, open at domain layer

**Decision.** The agent's tool surface, base type taxonomy, and scenario primitive set are closed. The points of extensibility are: declaring new entities as base type subclasses, registering domain-specific scenario primitives, and adding processes. The framework does not accept user-supplied modifications to the agent's reasoning loop or to the tool generation strategy in v1.

**Rationale.** Closed core gives predictability and verifiability. Open domain layer makes the framework useful for new businesses without rewriting it.

---

## D13. Agent framework: Pydantic AI with AG-UI adapter

**Decision.** The agent is built on Pydantic AI. Streaming uses the built-in `AGUIAdapter` and the `handle_ag_ui_request` Starlette/FastAPI helper from `pydantic_ai.ui.ag_ui`.

**Rationale.** Pydantic AI uses Pydantic for tool schemas which matches the rest of the stack. The AG-UI adapter gives a working stream contract with the AG-UI client on the React side without writing protocol glue.

---

## D14. Agent tool surface is generated from the domain

**Decision.** The framework introspects the domain module and generates a standard set of typed query tools (one set per entity, covering list, filter, and get-by-id) plus the scenario construction tools, plus a fallback `run_sql` tool. The agent's prompt is templated from descriptions on the domain classes.

**Rationale.** Adding a new entity to a domain should extend what the agent can reason about without any prompt engineering. Tool generation is the mechanism that delivers this. Aggregate queries are not auto-generated because what counts as a meaningful aggregate is entity-specific; the `run_sql` fallback covers ad-hoc aggregation in v1.

---

## D15. Simulation output is queryable state

**Decision.** Every simulation run writes results to `simulation_events` and `simulation_state` tables, tagged by `scenario_id`. The baseline is a scenario like any other. Scenario comparison is a query, not a separate code path.

**Rationale.** One query surface for the agent across baseline and what-if. Halves the tool surface and the testing burden.

---

## D16. Spatial behaviour split: sim teleports, frontend interpolates

**Decision.** The simulation moves entities between known locations as discrete state transitions at discrete time points. The frontend receives waypoints (location, time, entity) and interpolates positions smoothly between them.

**Rationale.** Continuous-space simulation is expensive and largely visual. Splitting the responsibility keeps simulation steps fast and gives the user smooth visual motion at no simulation cost.

---

## D17. Transport: WebSocket for sim updates, HTTP for chat

**Decision.** Live simulation updates stream over WebSocket. Agent chat uses HTTP via the AG-UI adapter.

**Rationale.** Simulation is high-frequency and naturally push-based. Chat is request-response with streaming inside the response. Using each protocol where it fits keeps the transport layer simple.

---

## D18. Frontend stack: Next.js with CopilotKit, @ag-ui/client, MapLibre GL

**Decision.** The frontend is Next.js (App Router) with TypeScript. Chat uses CopilotKit's React components, wired to the Pydantic AI backend through `@ag-ui/client`'s `HttpAgent` and CopilotKit's runtime, mounted as a Next.js API route. The map is MapLibre GL.

**Rationale.** CopilotKit's official Pydantic AI integration runs on Next.js. The `HttpAgent` from `@ag-ui/client` is the canonical bridge between CopilotKit's runtime and a Pydantic AI AG-UI endpoint. Plain React with Vite is possible but requires self-hosting CopilotKit's Node runtime as a sidecar service, which adds a third runtime to the monorepo for no real gain. The Vercel AI SDK was considered but is a parallel stack to CopilotKit, not a layer below it, and is not part of the path Pydantic AI documents. MapLibre GL is open source, mature, and renders the waypoint stream without imposing a tile-server cost.

---

## D19. Testing: pytest with pytest-asyncio in auto mode

**Decision.** Tests are written in pytest. Async tests use pytest-asyncio with `asyncio_mode = "auto"` set in pyproject.toml.

**Rationale.** Auto mode lets async tests be declared with `async def` without per-test decorators. Reduces noise. pytest is the de facto standard.

---

## D20. Lint, format, type check: ruff + mypy strict

**Decision.** Lint and format are handled by ruff with line length 100. Type checking is mypy in strict mode. Both run in CI and block merging on failure.

**Rationale.** ruff combines flake8, isort, and black-equivalent formatting in one fast tool. mypy strict catches real bugs and enforces that the typed contract the framework promises is actually typed.

---

## D21. Reference domain ships in the framework repository

**Decision.** The catering reference implementation lives in `src/twindle/reference/catering/` and is shipped as part of the framework package. It is not a separate repository or a separate package.

**Rationale.** The reference implementation is how new users learn the framework. Keeping it in-tree makes it impossible for the framework to drift from a working example. The catering domain is also what the v1 capability bar is tested against, so framework and example are tested together.

---

## D22. Conversation history is not persisted in v1

**Decision.** Chat conversation history lives for the duration of the user's session and is dropped afterwards.

**Rationale.** Persistence is a separate concern with separate UX (history search, sharing, deletion). v1 demonstrates the framework's claims without it. Adding it later does not change any architectural decision. Note: AG-UI and CopilotKit both transmit the full message history on each request, so the user does see continuity within a session even without server-side persistence.

---

## D23. SimPy runs synchronously; the agent endpoint blocks until the run completes

**Decision.** The simulation runtime calls SimPy's `env.run()` synchronously from within the async request handler that triggered it. There is no thread pool, no `asyncio.to_thread`, no background task. The HTTP request that requests a what-if waits for the simulation to finish before returning. The same applies to the baseline run at startup.

**Rationale.** SimPy is generator-based, not async-native. Its scheduler is fundamentally synchronous and its real-time mode is explicitly documented as not asynchronous. Building an `asyncio` bridge around SimPy would add complexity for v1 with no user-visible benefit, because (a) catering simulations over a week of seeded data are expected to complete in under 30 seconds (acceptance criterion on step 13), and (b) v1 is single-user, so blocking one HTTP request blocks only one user. The cost is that no other agent request can be served while a simulation is running on the same worker; this is acceptable for the v1 demo. If post-v1 use uncovers a need for concurrent simulations, the runtime can be wrapped with `run_in_executor` without changing the public API.

**Consequence for the "no `asyncio.run` inside library code" convention.** That convention is unchanged. SimPy `env.run()` is a blocking sync call, not an `asyncio.run` call. The async endpoint simply invokes a blocking function, which is permitted.

---

## D24. WebSocket transport streams a recorded run, not a live one

**Decision.** The simulation completes in full and writes all `SimulationEvent` rows to the database before the WebSocket starts sending anything. The WebSocket then streams the recorded events to the frontend at playback speed (configurable, default real-time). The timeline scrubber on the frontend lets the user pause, rewind, or fast-forward through the recorded events for a `scenario_id`.

**Rationale.** D23 commits the simulation to running synchronously to completion, so there are no events to stream "live" in the original sense. Streaming a recorded run gives the same visual outcome (trucks visibly moving on the map) and naturally supports the scrubber, replay, and side-by-side comparison features that the v1 capability bar requires. The single source of truth for any scenario is the rows in `simulation_events` and `simulation_state`; the WebSocket is a view onto those rows, not a separate pipeline.

**Implication for the `SimWriter`.** The writer still batches inserts to the DB as in D15. It does not publish to an in-process broker during the run. The WebSocket route reads from the DB and streams.

---

## D25. Async DB sessions use SQLModel's `AsyncSession`; simulation reads with a sync `Session`

**Decision.** Async code paths (FastAPI request handlers, the agent's query tools, the WebSocket route) use `sqlmodel.ext.asyncio.session.AsyncSession`. The simulation runtime, which executes SimPy's synchronous scheduler, uses SQLModel's synchronous `Session` for its DB reads and writes during the run. Both session types are bound to engines created by `twindle.persistence.engine`.

**Rationale.** SimPy processes cannot `await`, so an async session is no use to them during a run. Exposing both session types from the persistence module keeps the two worlds cleanly separated: async layers see `AsyncSession`, the simulation runtime sees `Session`, and neither leaks into the other. SQLModel's `AsyncSession` is preferred over SQLAlchemy's directly because its `exec()` integrates cleanly with SQLModel's `select()` ergonomics.

---

## D26. Frontend package manager: pnpm

**Decision.** The frontend uses pnpm. The lockfile is `pnpm-lock.yaml`.

**Rationale.** pnpm has the strictest dependency resolution of the three mainstream Node package managers, the smallest disk footprint via its content-addressable store, and is the default in the CopilotKit Pydantic AI starter. Picking it here locks in compatibility with the example code we will inevitably crib from.

---

## D27. Domain declares; engine interprets

**Decision.** Everything the discrete event simulation needs about a domain — process timing, resource requirements, base resource capacity, and capacity changes over time — is declared on the domain itself, using a fixed set of value objects in the domain layer:

- `Duration` on every `Process` subclass (mean + distribution + optional bounds/std).
- `ResourceRequirement` entries in `Process.requires` (resource type, quantity, occupy-vs-consume mode).
- `Resource.capacity` for the base pool size of each resource class.
- `CapacityWindow` rows that override a resource's effective capacity within a time range.

The simulation runtime is a generic interpreter over these declarations. Given a domain that uses only the declarative facilities, the runtime builds the SimPy resource pool, schedules capacity-window transitions, and generates a SimPy process function per `Process` instance with zero domain-specific code. Bespoke per-domain Python is an explicit escape hatch — a `@bespoke_process(ProcessClass)` registration — used only for behaviour that genuinely cannot be expressed declaratively (e.g. stochastic event injectors, processes that branch on simulation state).

**Rationale.** The single-source-of-truth claim in SCOPE.md is only true if the domain module is also the source of truth for the *simulation*, not just the schema and the query surface. Pushing timing, requirements, and capacity into per-domain SimPy code would make the reference catering domain readable but a second domain — and the agent's ability to introspect timing and capacity — would require rewriting simulation code each time. Declarative-first keeps the catering reference under 500 lines including its full simulation behaviour, lets the agent answer "what's the effective capacity of trucks next weekend?" through generated introspection tools rather than `run_sql`, and gives a clear, narrow contract a second reference domain (v1.1) must satisfy.

**Consequence for the bespoke escape hatch.** Bespoke modules in a reference domain stay short by design (catering targets <100 lines for `truck_breakdown`). If a bespoke module grows large, the right fix is to lift the missing concept into the domain layer's value objects rather than accreting per-domain SimPy code.

---

## D28. Frontend waypoint playback: deck.gl TripsLayer

**Decision.** Smooth interpolation of vehicle positions between recorded waypoints is delegated to deck.gl's `TripsLayer`, rendered as a deck.gl overlay on top of the MapLibre GL base map. The frontend feeds it per-truck `path` and `timestamps` arrays from the WebSocket stream and drives `currentTime` from the timeline scrubber. Twindle does not implement its own interpolation utility.

**Rationale.** `TripsLayer` is purpose-built for "vehicle paths from waypoint + timestamp data with a single `currentTime` knob" — the exact shape Twindle produces under D16 and D24. It removes a hand-rolled interpolation utility, gives trail rendering for free, and integrates cleanly with MapLibre as the base map. Adding deck.gl to the frontend dependency tree is cheaper than maintaining bespoke animation code that re-implements a small slice of what `TripsLayer` already does well.

**Consequence for the timeline scrubber.** The scrubber's value is wired directly to `TripsLayer`'s `currentTime` prop; seeking and pause/resume become local state changes on a number, not WebSocket round-trips, except where the user wants the server to skip ahead in playback emission.

---

## D29. `run_sql` fallback: mcp-alchemy as an MCP server

**Decision.** The read-only SQL fallback exposed to the agent is not implemented bespoke. The framework runs `mcp-alchemy` as a stdio MCP subprocess bound to the simulation database and registers it with the Pydantic AI agent (via `pydantic_ai.mcp.MCPServerStdio` or equivalent). mcp-alchemy's `list_tables`, `describe_table`, and `execute_query` (SELECT-only) tools become available to the agent alongside Twindle's typed query tools. The typed per-entity query tools (one set per `Entity` subclass: list / filter / get-by-id) are still generated by the framework — mcp-alchemy is strictly the ad-hoc SELECT fallback, not the primary query surface.

**Rationale.** mcp-alchemy already handles SELECT-only validation, schema introspection, and result rendering correctly. Re-implementing that against SQLModel saves nothing. The typed generator stays because the agent prompt benefits from explicitly listing entity names and their fields; generic SQL access does not give the agent that prior. Closed primary surface, open fallback — same shape as the scenario primitives.

**Consequence for runtime.** The agent server starts mcp-alchemy as a child process at app boot and tears it down at shutdown. One subprocess, no separate deployment. The thin-slice `run_sql` wrapper introduced in step 5 is a transitional helper for testing the query slice without the agent layer; it is superseded by mcp-alchemy as soon as the agent layer (step 8) lands.

---

## D30. Time-varying capacity deferred; v1 capacity is fixed at sim start

**Decision.** Resource capacity is set at SimPy resource creation and does not change for the duration of a simulation run. The `CapacityWindow` value object originally proposed in step 2 and D27 is removed from v1 — it is neither stored, declared, nor interpreted. Shift coverage, seasonal fleet changes, and any other time-varying capacity concern are modelled at sim *initialisation* (the resource pool is sized once for the run's sim window) rather than as scheduled transitions during the run. Truck breakdowns are modelled as a long-running SimPy process that occupies the affected truck, not as capacity changes.

**Supersedes.** The `CapacityWindow` clause of D27. The remainder of D27 — duration, resource requirements, base capacity, bespoke escape hatch — stands unchanged.

**Rationale.** Mid-run capacity resize is not native to SimPy: `simpy.Resource` does not expose a public way to change its capacity after creation, and emulating it requires a custom token/Container pattern that adds non-trivial code and edge cases to the engine layer. That work is exactly the kind of sprawl that would balloon step 6. The four v1 capability-bar interactions can all be expressed with fixed-capacity runs ("add two trucks" becomes a sim with two more trucks for the entire window, "remove one truck" becomes the same with one fewer truck). Deferring this cleanly preserves the declarative-interpreter claim for the v1 surface and leaves a tidy re-add path for v1.1: re-introduce `CapacityWindow`, pick a SimPy resize pattern (Container, token-store, or pre-built capacity-stepping process), and wire it through the interpreter.

**Implications for the v1 demo.**
- Capability-bar question 2 reworded from "what if we added two more trucks during summer weekends?" to "what if we added two more trucks?" The sim window can be set to a summer weekend if the seasonal framing matters for the demo video.
- Catering shifts in step 11 are stored as `Shift` rows for the agent to query about, but the simulation treats every seeded employee as available across the sim window. Shift-driven availability returns with time-varying capacity in v1.1.
- The step 13 `truck_breakdown` bespoke override is rewired: it spawns a stochastic "repair" process that occupies the affected truck for a sampled duration, then releases it. Same observable behaviour (truck temporarily unavailable), no capacity resize.
