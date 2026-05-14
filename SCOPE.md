# Twindle — Scope

## What this is

Twindle is a unified Python framework for building operational digital twins of small businesses. From one domain definition it produces three coupled capabilities driven by a single source of truth:

1. A queryable database of business state.
2. A discrete event simulation of operations.
3. A conversational AI agent that answers questions about current state and runs what-if scenarios against the simulation.

The framework is opinionated about three things. Domains are defined in Pydantic-first Python. State flows through one schema consumed by many layers. The agent interacts with data and the simulation through typed tools rather than prompt engineering.

The v1 deliverable is the framework plus a fully working catering business reference implementation. The reference implementation lives in the same repository as the framework. It exists to prove the framework's claims and to teach new users how to build their own domains.

## What this is not

Twindle is not a no-code platform. Defining a domain requires writing Python.

It is not a general purpose simulation engine. It does not compete with AnyLogic or Simulink.

It is not a BI tool. It does not replace Tableau, Metabase, or any dashboard.

It is not a chatbot framework. The agent is one component, tightly coupled to the domain and simulation.

It is not multi-tenant or production-deployment focused. v1 is single user, runs locally.

It is not yet domain-agnostic in practice. The architecture is built to generalise but only catering is built and tested in v1. A second domain is committed for v1.1.

## Architectural commitments

**Single source of truth.** A domain module declares entities as SQLModel classes carrying rich descriptions and semantic annotations. That module is the only place these things are defined. The database schema, the LLM tool surface, the simulation entities, and the frontend's typed contract all derive from it.

**Generated agent tool surface.** The framework introspects the domain module and produces a standard set of tools. Adding a new entity to the domain automatically extends what the agent can reason about. The agent's prompt is templated from the domain's descriptions.

**Strong domain prescription via base types.** Every domain inherits from a small set of base concepts that map onto SimPy primitives but are named in domain terms:

- `Entity`, base for anything with identity that persists
- `Resource`, an entity that can be consumed or occupied during a process
- `Process`, a unit of work that takes time and may need resources
- `Event`, a business event with a schedule, location, and required resources, distinct from SimPy's internal event plumbing
- `Schedule`, binds processes to entities over time
- `Assignment`, links resources to events
- `Location`, spatial position for map rendering

A domain implementer declares entities as the appropriate base types, adds domain-specific fields, and writes the processes. The agent surface, database schema, and basic visualisation come for free.

**Fixed scenario vocabulary.** When the agent runs a what-if it composes a typed `Scenario` object from a closed set of primitives:

- `AddEntity`
- `RemoveEntity`
- `ModifyAttribute`
- `ScaleParameter`
- `ChangeSchedule`
- `InjectEvent`

The agent cannot invent new modification types. When a user request falls outside the vocabulary the agent explains the limitation. Domains can register additional primitives so the framework is closed at the agent layer but open at the domain layer.

**Spatial behaviour split between layers.** The simulation teleports entities between known locations at discrete time points and writes those transitions to the database. The frontend reads the recorded waypoint stream over WebSocket and interpolates positions smoothly between them. Visual fidelity without simulation complexity.

**Sim output is queryable state.** Every simulation run writes results to scenario-tagged tables. The same query tools work on baseline and what-if data, so scenario comparison is a structured query rather than a separate code path. The WebSocket transport is a view onto these tables, not a parallel pipeline.

**Streaming via AG-UI.** Pydantic AI's built-in AG-UI adapter handles agent streaming on the server side. On the client side, CopilotKit's React components consume the AG-UI event stream via `@ag-ui/client`'s `HttpAgent`. The map and timeline subscribe to a separate WebSocket that replays recorded simulation events.

## Technology choices

Domain and ORM: SQLModel on top of Pydantic and SQLAlchemy. Database: SQLite for local development, PostgreSQL-compatible for any scaling. Simulation: SimPy, run synchronously to completion from within the async request handler that triggered it. Agent: Pydantic AI with the AG-UI adapter (`AGUIAdapter` and `handle_ag_ui_request` from `pydantic_ai.ui.ag_ui`). Frontend: Next.js (App Router) with TypeScript, CopilotKit for the chat UI, and `@ag-ui/client`'s `HttpAgent` bridging CopilotKit's runtime to the Pydantic AI endpoint. Map: MapLibre GL. Transport: WebSocket for streaming recorded simulation events to the frontend at playback speed, HTTP for chat. Python 3.11+, managed by uv, src/ layout. Frontend package manager: pnpm. Validation: Pydantic v2. Tests: pytest with pytest-asyncio in `asyncio_mode = "auto"`. Lint and format: ruff with line length 100. Type check: mypy in strict mode. Licence: MIT. Monorepo with the Python framework and the Next.js frontend as separate packages.

## v1 capability bar

The demonstration v1 must pass end to end:

A user opens the frontend. The map shows the operating region with trucks moving between event locations, an event schedule, employees with their current assignments, and an agent panel front and centre. The simulation runs on a baseline scenario seeded from realistic catering data.

The user asks "which events last month had the worst on-time arrival rate?" The agent queries historical data, returns a ranked list, and highlights those events on the map.

The user asks "what if we added two more trucks during summer weekends?" The agent constructs a `Scenario` object and runs the simulation. Once the run completes (in under 30 seconds), the map plays back the what-if state alongside the baseline. The agent summarises the difference, the user sees side-by-side metrics, and they can drill in.

The user asks "why did event 47 finish late in the baseline?" The agent inspects the simulation trace for that event and explains the bottleneck in plain language.

Everything in v1 serves this demonstration. Everything not needed for it is cut.

## In scope for v1

**Framework core, as a Python package.** A domain definition pattern where entities are SQLModel classes inheriting from the framework base types, annotated for their role in queries, simulation, and visualisation. A query layer that exposes typed query tools to the agent, executing against SQLAlchemy, with a small fallback `run_sql` tool for ad-hoc questions the typed tools cannot answer. A simulation runtime that takes the domain definition, a scenario object, and a database snapshot, then runs SimPy processes over the entities, writing results to a `simulation_events` table and a `simulation_state` table, tagged by `scenario_id`. A scenario system with a base `Scenario` Pydantic model and the fixed primitives listed above. An agent module built on Pydantic AI with the generated tool surface, the AG-UI adapter, and a domain-aware prompt template.

**Reference catering domain.** Realistic seeded data including trucks, employees, venues, events, assignments, and shifts. A non-trivial simulation covering trucks travelling between venues, loading and unloading time, employee shifts, event durations, and breakdowns modelled as a stochastic event. A set of demo what-if scenarios that produce meaningful outcome differences.

**React frontend, as a separate package in the monorepo.** A map view using MapLibre GL showing trucks, venues, and events with live position updates over WebSocket, with two map modes (baseline and what-if) that are switchable or side-by-side. A chat panel using CopilotKit connected to the Pydantic AI agent. A scenario comparison view showing baseline versus what-if metrics in a small fixed set of visualisations covering on-time rate, utilisation, cost, and customer-facing outcomes. A simulation timeline scrubber for replaying time.

**Documentation.** Sufficient for a developer to clone the repo, run the catering demo locally, and start sketching their own domain. Not exhaustive.

## Out of scope for v1

User authentication, multi-user support, deployment infrastructure, role-based access. A second domain (committed to v1.1). Stochastic distribution fitting from historical data. Streaming data ingestion. An evaluation harness for agent accuracy. Optimisation: the framework runs what-ifs that the user or agent specifies, it does not search the scenario space for an optimum. Custom visualisations beyond the fixed map, timeline, and comparison view. Multi-agent orchestration. Persistence of conversation history beyond the current session. Anything commercial: authentication, billing, hosted version.

## Success criteria

The project ships under the MIT licence. A developer who has never seen the project can clone it, run a single setup command, and have the catering demo running in under ten minutes. The demo runs the capability bar scenario end to end without manual intervention. The catering domain definition is under 500 lines of Python and reads as the single source of truth for the entire system. The agent answers data questions correctly on at least 8 of 10 hand-crafted benchmark questions, and successfully constructs and runs scenarios for at least 4 of 5 hand-crafted what-if prompts. The frontend is visually good enough that a demo video is shareable on social media without embarrassment. A second developer can take the framework and sketch the entity definitions for a different small business in an afternoon.
