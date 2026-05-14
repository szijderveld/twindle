# Twindle — Build Plan

This plan takes the project from an empty directory to v1 against [SCOPE.md](SCOPE.md) and [DECISIONS.md](DECISIONS.md). Each step is self-contained: it states what to read, what to build, and how to verify success. A fresh Claude session should be able to execute any single step cold given this file plus SCOPE.md plus DECISIONS.md.

## How to use this plan

- Run `/next-step` to execute the first uncompleted step.
- The command reads SCOPE.md, DECISIONS.md, and the step's prompt; executes the deliverables; runs acceptance criteria; marks the step done; appends any non-obvious decisions to DECISIONS.md.
- Do not skip steps. If you must, edit the checkbox manually.
- Each step ends with a commit. Commits use the message `step N: <title>`.
- If a step fails its acceptance criteria, fix it and re-run rather than moving on.

## Global conventions

These apply to every step. Steps do not restate them.

- **Python version:** 3.11+. Set `requires-python = ">=3.11"` in pyproject.toml.
- **Layout:** `src/twindle/` for the Python package. `frontend/` for the React package. Both at repository root.
- **Package manager:** uv. All install, run, lock, and venv operations go through uv. `uv sync` is the single setup command.
- **Async style:** native `async def` in the agent, transport, HTTP routes, and WebSocket. The simulation runtime is synchronous because SimPy is generator-based; it is invoked as a blocking call from within async request handlers (see D23). No `asyncio.run` inside library code. No background threads or executors for the simulation in v1.
- **Validation:** Pydantic v2. Domain classes are SQLModel (which is Pydantic-backed). Tool schemas, scenario primitives, and config are plain Pydantic.
- **Tests:** pytest, with `[tool.pytest.ini_options]` setting `asyncio_mode = "auto"`. Tests live in `tests/` mirroring the `src/twindle/` tree.
- **Lint and format:** ruff with `line-length = 100`. Run `ruff check` and `ruff format` in CI.
- **Type check:** mypy in strict mode (`strict = true`). All public functions are fully typed. No `# type: ignore` without a comment explaining why.
- **Commit policy:** one commit per step, message `step N: <title>`. No squashing on `main` for the duration of v1.
- **Style:** terse, no marketing language in docstrings or comments. Docstrings on public classes and functions only. Single-line docstrings unless behaviour is non-obvious.
- **No premature abstraction.** A second implementation of anything is the trigger for an abstraction, not the first.

## Module layout (target)

```
twindle/
  pyproject.toml
  uv.lock
  README.md
  SCOPE.md
  DECISIONS.md
  PLAN.md
  LICENSE
  .github/
    workflows/
      ci.yml
  src/
    twindle/
      __init__.py
      domain/
        __init__.py
        base.py            # Entity, Resource, Process, Event, Schedule, Assignment, Location
        annotations.py     # field role markers (query, sim, viz)
      persistence/
        __init__.py
        engine.py          # SQLModel/SQLAlchemy engine + session factory
        snapshot.py        # baseline DB snapshot, scenario-tagged tables
      scenarios/
        __init__.py
        model.py           # Scenario base
        primitives.py      # AddEntity, RemoveEntity, ModifyAttribute, ScaleParameter, ChangeSchedule, InjectEvent
        registry.py        # domain-supplied primitive registration
      simulation/
        __init__.py
        runtime.py         # SimPy environment, process orchestration
        writer.py          # writes simulation_events / simulation_state
        trace.py           # per-event trace inspection
      query/
        __init__.py
        tools.py           # generated typed query tools
        sql.py             # run_sql fallback
      agent/
        __init__.py
        builder.py         # introspects domain, builds tool surface + prompt
        prompt.py          # prompt template
        server.py          # Pydantic AI app + AG-UI adapter
      transport/
        __init__.py
        ws.py              # WebSocket sim stream
        http.py            # HTTP routes (FastAPI)
      testing/
        __init__.py
        fixtures.py        # reusable pytest fixtures (in-memory DB, fake clock)
      reference/
        __init__.py
        catering/
          __init__.py
          domain.py        # all catering entities, single source of truth
          processes.py     # SimPy processes for catering
          seed.py          # realistic seed data generator
          scenarios.py     # demo what-if scenarios
  frontend/
    package.json
    pnpm-lock.yaml
    next.config.ts
    tsconfig.json
    src/
      app/
        layout.tsx
        page.tsx
        api/
          copilotkit/
            route.ts        # mounts CopilotKit runtime against AG-UI HttpAgent
      api/
        ws.ts               # client hook for /ws/sim/{scenario_id}
        chat.ts             # any glue for the AG-UI message format
      components/
        Map.tsx
        Chat.tsx
        Timeline.tsx
        Comparison.tsx
      types/
        domain.ts           # generated from twindle domain
  tests/
    domain/
    persistence/
    scenarios/
    simulation/
    query/
    agent/
    reference/
      catering/
    e2e/
```

## Steps

### Step 0: Verify decisions are locked (verification-only)

- [ ] complete

**Goal.** Confirm SCOPE.md and DECISIONS.md are in their pre-build resolved state. No code is written.

**Prerequisites.** None.

**Read first.** All of SCOPE.md. All of DECISIONS.md.

**Deliverables.** None. This is a gate.

**Acceptance criteria.** SCOPE.md has no "Open items" section (or it is empty). DECISIONS.md contains entries D1 through D26, in order, with no gaps. The name in D1 matches the name used throughout SCOPE.md and the rest of DECISIONS.md. If any of these fail, fix the planning documents before proceeding to step 1; do not start coding.

**On success.** Mark this step complete in PLAN.md only. No commit (no code yet).

**On failure.** Edit SCOPE.md or DECISIONS.md to reach a stable state and re-run this step.

---

### Step 1: Bootstrap project

- [ ] complete

**Goal.** An empty but fully wired Python project that lints, type checks, and runs an empty test suite green in CI.

**Prerequisites.** Step 0.

**Read first.** SCOPE.md "Technology choices". DECISIONS.md D3, D4, D5, D6, D19, D20. PLAN.md "Global conventions".

**Deliverables.**

- `pyproject.toml` declaring the `twindle` package, Python 3.11+, dependencies pinned to current stable. Runtime: sqlmodel, pydantic, pydantic-ai (with the `ag-ui` extra), simpy, fastapi, uvicorn, websockets, aiosqlite. Dev: pytest, pytest-asyncio, ruff, mypy. Use PEP 735 `[dependency-groups]` for the dev group.
- `uv.lock` committed.
- `src/twindle/__init__.py` with version string only.
- `tests/test_smoke.py` with a single `def test_imports(): import twindle` test.
- `ruff` config in pyproject.toml: line length 100, sensible rule set (E, F, I, B, UP, RUF).
- `mypy` config in pyproject.toml: `strict = true`, targeting `src/twindle`.
- `.gitignore` covering `.venv`, `__pycache__`, `*.pyc`, `.pytest_cache`, `.mypy_cache`, `.ruff_cache`, `dist/`, `*.db`.
- `LICENSE` (MIT, copyright Twindle contributors).
- `README.md` stub with project name, one-paragraph description, "status: in development", links to SCOPE.md / DECISIONS.md / PLAN.md.
- `.github/workflows/ci.yml` running on push and PR: install uv, `uv sync`, `uv run ruff check`, `uv run ruff format --check`, `uv run mypy src`, `uv run pytest`.

**Acceptance criteria.** Locally: `uv sync` succeeds. `uv run ruff check`, `uv run ruff format --check`, `uv run mypy src`, and `uv run pytest` all exit 0. CI passes on the commit.

**On success.** Commit `step 1: bootstrap project`. Tick the checkbox.

**On failure.** Fix the failing tool's configuration and re-run.

---

### Step 2: Implement base type taxonomy

- [ ] complete

**Goal.** The seven base types in `twindle.domain.base` exist as SQLModel classes that can be subclassed in a domain module.

**Prerequisites.** Step 1.

**Read first.** SCOPE.md "Architectural commitments" (base types paragraph). DECISIONS.md D7, D10.

**Deliverables.**

- `src/twindle/domain/base.py` defining `Entity`, `Resource`, `Process`, `Event`, `Schedule`, `Assignment`, `Location` as SQLModel base classes. Each carries a primary key `id`, a `name` field where appropriate, and timestamps where appropriate. `Location` carries `latitude` and `longitude`. `Event` carries `start_time`, `end_time`, a foreign key to `Location`. `Assignment` carries foreign keys to `Resource` and `Event`. `Schedule` carries a foreign key to `Process` and time bounds.
- `src/twindle/domain/annotations.py` defining field role markers: `query_field`, `sim_field`, `viz_field` (as `Annotated` metadata or Pydantic `Field` extras). These are passive markers that later steps will introspect.
- Docstrings on every base type stating its role.
- `tests/domain/test_base.py` exercising: a trivial subclass of each base type can be defined; SQLModel can create a SQLite schema from them; an instance round-trips through a session.

**Acceptance criteria.** All tests pass. mypy strict passes. Importing `twindle.domain.base` produces no warnings.

**On success.** Commit `step 2: implement base type taxonomy`.

**On failure.** Re-read D10 and the SCOPE.md base-types paragraph; the names and roles are fixed and must match exactly.

---

### Step 3: Persistence layer and scenario-tagged tables

- [ ] complete

**Goal.** A persistence module that creates a SQLite database, manages both async and sync sessions, and supports tagging simulation output by `scenario_id`.

**Prerequisites.** Step 2.

**Read first.** DECISIONS.md D7, D8, D15, D25.

**Deliverables.**

- `src/twindle/persistence/engine.py` exposing `make_engine(url: str)` for the synchronous engine, `make_async_engine(url: str)` for the async engine, and matching session factories. The async session is `sqlmodel.ext.asyncio.session.AsyncSession`; the sync session is SQLModel's `Session`. Both are bound to the same SQLite file by default (`sqlite:///./twindle.db` for sync, `sqlite+aiosqlite:///./twindle.db` for async).
- `src/twindle/persistence/snapshot.py` defining two SQLModel tables: `SimulationEvent` (id, scenario_id, sim_time, entity_type, entity_id, event_type, payload JSON) and `SimulationState` (id, scenario_id, sim_time, entity_type, entity_id, state JSON). Both tables are indexed on `(scenario_id, sim_time)`.
- A `bootstrap_schema(engine)` function that creates all tables registered on the SQLModel metadata, including any user-defined domain classes that have been imported before it is called. It accepts either a sync or async engine.
- `tests/persistence/test_engine.py` and `tests/persistence/test_snapshot.py` covering: schema creation, round-trip on simulation tables via both sync and async sessions, scenario_id-scoped query returns only rows for that scenario.

**Acceptance criteria.** Tests pass. mypy strict passes. A second engine pointing at a different SQLite file does not share state with the first.

**On success.** Commit `step 3: persistence layer`.

---

### Step 4: Scenario model and primitives

- [ ] complete

**Goal.** A `Scenario` Pydantic model that holds an ordered list of typed primitives, plus the six primitive types and a registry for domain-supplied primitives.

**Prerequisites.** Step 3.

**Read first.** SCOPE.md "Architectural commitments" (fixed scenario vocabulary). DECISIONS.md D11, D12.

**Deliverables.**

- `src/twindle/scenarios/primitives.py` defining `AddEntity`, `RemoveEntity`, `ModifyAttribute`, `ScaleParameter`, `ChangeSchedule`, `InjectEvent` as Pydantic models. Each carries the minimum fields needed to apply it to a domain (entity type, entity id or selector, attribute name, value or factor, time bounds where relevant).
- `src/twindle/scenarios/model.py` defining `Scenario` with a `scenario_id` (uuid), a `name`, a description, and a `modifications: list[ScenarioPrimitive]` discriminated union.
- `src/twindle/scenarios/registry.py` exposing `register_primitive(cls)` that allows a domain to add to the discriminated union at import time. Registered primitives must be Pydantic models with a literal `kind` field.
- `tests/scenarios/test_primitives.py` covering: serialising a Scenario with all six built-in primitives to JSON and back; rejecting a Scenario that contains an unknown primitive when no extension is registered; accepting one after `register_primitive` is called.

**Acceptance criteria.** Tests pass. The fixed primitive set in `primitives.py` matches D11 exactly.

**On success.** Commit `step 4: scenario model and primitives`.

---

### Step 5: Thin end-to-end slice — one entity, queryable

- [ ] complete

**Goal.** A trivial test domain (a single entity, no simulation yet) that runs end to end through schema creation, seed data, and a generated typed query tool. Proves the "single source of truth" claim before the simulation and agent are wired in.

**Prerequisites.** Step 4.

**Read first.** SCOPE.md "Architectural commitments" (single source of truth, generated tool surface). DECISIONS.md D14.

**Deliverables.**

- `src/twindle/query/tools.py` with `build_query_tools(domain_module)` that introspects a module for SQLModel classes inheriting from `Entity` and generates, for each: `list_<entity>()`, `get_<entity>_by_id(id)`, `filter_<entity>(**kwargs)`. Tools are plain async functions taking a session, returning Pydantic-validated rows.
- `src/twindle/query/sql.py` with a `run_sql(query: str, session)` fallback that allows read-only `SELECT` and raises on anything else.
- `tests/e2e/test_thin_slice.py` defining a one-entity domain inline (e.g. `Truck(Entity)` with a `capacity` field), creating a schema, seeding three rows, calling each generated tool, and the fallback `run_sql`.

**Acceptance criteria.** All tests pass. The generated tool names follow the `<verb>_<entity>` convention. `run_sql` rejects an `INSERT`.

**On success.** Commit `step 5: thin end-to-end query slice`.

---

### Step 6: Simulation runtime

- [ ] complete

**Goal.** A synchronous simulation runner that, given a domain module, a `Scenario`, and a database snapshot, runs SimPy processes and writes results to `SimulationEvent` and `SimulationState` tagged with `scenario_id`.

**Prerequisites.** Step 5.

**Read first.** SCOPE.md "Architectural commitments" (sim output is queryable state, spatial split). DECISIONS.md D9, D15, D16, D23, D24, D25.

**Deliverables.**

- `src/twindle/simulation/runtime.py` exposing `run_simulation(domain, scenario, session, until) -> ScenarioRunResult`. This is a **synchronous** function. It creates a SimPy `Environment`, materialises entities from the database using a sync `Session`, applies scenario primitives to the in-memory state, runs registered processes by calling `env.run(until=until)`, and returns when SimPy's event loop is empty. Per D23, this function is intended to be called directly from async request handlers and will block them; per D24, all events are written to the database before the function returns.
- `src/twindle/simulation/writer.py` exposing a `SimWriter` that the runtime passes to processes; processes call `writer.event(...)` and `writer.state(...)` and the writer batches inserts to the DB on a configurable threshold and on `flush()` at end of run. The writer does **not** publish to any in-process broker (the WebSocket reads from the DB after the run completes; see D24).
- `src/twindle/simulation/trace.py` exposing `get_trace(scenario_id, entity_type, entity_id, session)` that returns the ordered events for one entity in one scenario. Also exposes `get_event_trace(scenario_id, event_id, session)` that returns all simulation events relating to a single domain `Event` (e.g. catering event 47) across all participating entities, to support the "why did event 47 finish late?" capability.
- A process registration mechanism in the domain (e.g. a `@process` decorator in `twindle.simulation.runtime` or a `processes` attribute on the domain module) that the runtime discovers.
- `tests/simulation/test_runtime.py` using the one-entity domain from step 5 extended with a trivial process (a Truck that "drives" for a fixed duration and emits one event), asserting events are written with the correct `scenario_id`.
- `tests/simulation/test_trace.py` asserting `get_trace` returns events in order for one entity, and `get_event_trace` returns events for all entities participating in a given domain event.

**Acceptance criteria.** Tests pass. A second `run_simulation` call with a different `scenario_id` does not overwrite or mix with the first run's rows. `run_simulation` is annotated and behaves as a synchronous function (not `async def`).

**On success.** Commit `step 6: simulation runtime`.

---

### Step 7: Scenario application

- [ ] complete

**Goal.** Each of the six built-in scenario primitives actually mutates the simulation's in-memory state when applied by the runtime.

**Prerequisites.** Step 6.

**Read first.** DECISIONS.md D11, D12, D15.

**Deliverables.**

- An `apply_scenario(scenario, sim_state)` function in `twindle.scenarios.model` (or a sibling module) that dispatches each primitive to its handler.
- Handlers for the six built-in primitives. `AddEntity` inserts a new row; `RemoveEntity` flags out; `ModifyAttribute` updates a field; `ScaleParameter` multiplies a numeric attribute; `ChangeSchedule` swaps a `Schedule` row's bounds; `InjectEvent` inserts a new `Event` row with required attributes.
- An extension hook so a domain can register a handler for a custom primitive registered via `register_primitive`.
- `tests/scenarios/test_apply.py` exercising each primitive against a small in-memory state and asserting the expected mutation. One test registers a custom primitive and confirms the handler runs.

**Acceptance criteria.** All six handlers pass tests. A scenario with one primitive of each type applies cleanly in order.

**On success.** Commit `step 7: scenario application`.

---

### Step 8: Agent tool generation

- [ ] complete

**Goal.** A function that, given a domain module, returns a list of Pydantic AI tools spanning: typed query tools, the `run_sql` fallback, scenario construction tools (one per primitive), `run_scenario`, `get_trace`, and `get_event_trace`.

**Prerequisites.** Step 7.

**Read first.** SCOPE.md "Architectural commitments" (generated agent tool surface). DECISIONS.md D13, D14.

**Deliverables.**

- `src/twindle/agent/builder.py` exposing `build_tool_surface(domain_module) -> list[Tool]`. Tools wrap step 5's query tools, step 6's `run_simulation`, `get_trace`, and `get_event_trace`, and one tool per scenario primitive that returns a primitive instance (the agent composes a `Scenario` by calling these). The `run_scenario` tool wraps `run_simulation`; since `run_simulation` is synchronous (D23), the tool is implemented as a Pydantic AI tool that simply calls it and returns when it returns.
- `src/twindle/agent/prompt.py` containing a template that consumes domain class docstrings and field descriptions to produce a system prompt naming the entities, the typed tools, and the scenario vocabulary.
- `tests/agent/test_builder.py` asserting: for the one-entity test domain, the expected tool names exist; tool schemas validate; the rendered prompt contains the entity name and the six primitive names.

**Acceptance criteria.** Tests pass. Tool descriptions are non-empty and derived from the domain (not hard-coded).

**On success.** Commit `step 8: agent tool generation`.

---

### Step 9: Agent server with AG-UI adapter

- [ ] complete

**Goal.** A Pydantic AI agent running behind FastAPI with the AG-UI adapter for streaming, exposed over HTTP, that can call the generated tools end to end against a real database.

**Prerequisites.** Step 8.

**Read first.** DECISIONS.md D13, D17.

**Deliverables.**

- `src/twindle/agent/server.py` exposing `make_app(domain_module, engine)` that returns a FastAPI app with one `/chat` route handled by the Pydantic AI agent and the AG-UI adapter. The agent's tools come from `build_tool_surface`. The agent's system prompt comes from `prompt.render(domain_module)`.
- A small CLI `twindle.cli` (e.g. `python -m twindle.cli serve --domain twindle.reference.catering.domain`) that boots the server with a chosen domain.
- `tests/agent/test_server.py` using FastAPI's TestClient and a stub LLM (Pydantic AI's test model) to assert: a `/chat` POST that triggers a tool call returns a streamed AG-UI event sequence including a tool call and a final assistant message. Tests do not call a real LLM.

**Acceptance criteria.** Tests pass against the stub LLM. The CLI boots without error against the step 5 test domain.

**On success.** Commit `step 9: agent server with AG-UI`.

---

### Step 10: WebSocket transport for simulation playback

- [ ] complete

**Goal.** A WebSocket endpoint that, given a `scenario_id` for a completed simulation, streams the recorded simulation events to the frontend at a configurable playback speed.

**Prerequisites.** Step 9.

**Read first.** DECISIONS.md D16, D17, D24.

**Deliverables.**

- `src/twindle/transport/ws.py` exposing a `/ws/sim/{scenario_id}` route on the FastAPI app. The client opens a connection naming a scenario_id, optionally a playback speed multiplier (default 1.0 = real-time), and an optional start time. The server reads `SimulationEvent` rows for that scenario in `sim_time` order from the database (using an async session) and emits them over the WebSocket, sleeping between emissions so the gaps in client wall-clock time match the gaps in sim time divided by the playback multiplier. The client may send a control message to pause, resume, seek, or change speed; the server adjusts emission accordingly.
- A waypoint contract: each emitted message contains `(entity_type, entity_id, from_location, to_location, depart_sim_time, arrive_sim_time)` for movement events, and `(entity_type, entity_id, state, sim_time)` for state changes. The simulation never produces per-tick positions; the frontend interpolates between waypoints (this step delivers the contract, the interpolation lives in the frontend step).
- `tests/transport/test_ws.py` using FastAPI's WebSocket test client to assert: connecting to a scenario_id that has recorded events produces ordered messages; connecting to an unknown scenario_id closes cleanly with an explanatory message; a pause control message stops emission; a seek control message resumes from the new sim time.

**Acceptance criteria.** Tests pass. Messages match the documented schema. At playback speed 1.0, the gap between two emitted messages matches the gap between their `sim_time` values to within 100ms tolerance.

**On success.** Commit `step 10: websocket playback transport`.

---

### Step 11: Design the catering domain (in code)

- [ ] complete

**Goal.** The catering domain's entities, relationships, and process names exist as SQLModel classes with rich descriptions, but processes are stubs. This step is the design step for the reference implementation; the design lives in the code, not in a separate document.

**Prerequisites.** Step 10.

**Read first.** SCOPE.md "v1 capability bar" and "In scope for v1" (reference catering domain). DECISIONS.md D10, D21.

**Deliverables.**

- `src/twindle/reference/catering/domain.py` defining catering entities as subclasses of the framework base types. At minimum: `Truck(Resource)`, `Employee(Resource)`, `Venue(Location)`, `CateringEvent(Event)`, `Shift(Schedule)`, `EventAssignment(Assignment)`, plus a `Booking(Entity)` or equivalent linking a customer to a catering event. Each class has a docstring describing its role and field-level descriptions on every column.
- Relationships are declared with SQLModel `Relationship` where needed.
- `src/twindle/reference/catering/processes.py` declaring (as stubs that raise `NotImplementedError`) the process names: `truck_travel`, `event_setup`, `event_service`, `event_teardown`, `truck_breakdown`. Each carries a docstring explaining its role.
- `tests/reference/catering/test_domain.py` asserting: every catering entity inherits from the correct framework base; the schema builds without error; the generated tool surface (via step 8) includes a list/get/filter tool for every catering entity.
- The whole `domain.py` file is under 500 lines (success criterion).

**Acceptance criteria.** Tests pass. `domain.py` line count is under 500. Every public class and field has a description.

**On success.** Commit `step 11: design catering domain`.

**On failure.** If `domain.py` exceeds 500 lines, simplify before moving on; the success criterion is non-negotiable.

---

### Step 12: Catering seed data

- [ ] complete

**Goal.** A deterministic seed generator that populates a database with realistic catering data covering at least one month of history.

**Prerequisites.** Step 11.

**Read first.** SCOPE.md "v1 capability bar" and the catering paragraph of "In scope for v1".

**Deliverables.**

- `src/twindle/reference/catering/seed.py` exposing `seed(session, *, seed_value=42)` that inserts: 6–10 trucks, 20–30 employees, 15–25 venues spread across a recognisable geographic region, at least 60 catering events spanning the previous month with associated assignments and shifts.
- Realism: events vary by size and duration; venues have plausible coordinates; some events overlap in time and tax the resources.
- Determinism: the same `seed_value` produces the same database state.
- `tests/reference/catering/test_seed.py` asserting: row counts are in the expected range; coordinates fall within plausible bounds; a second call with the same seed produces the same content (hash compare).

**Acceptance criteria.** Tests pass. `seed()` completes in under five seconds on a developer laptop.

**On success.** Commit `step 12: catering seed data`.

---

### Step 13: Catering simulation processes

- [ ] complete

**Goal.** The catering process stubs from step 11 are implemented as SimPy processes against the framework runtime, producing a non-trivial simulation.

**Prerequisites.** Step 12.

**Read first.** SCOPE.md "v1 capability bar" and the catering paragraph of "In scope for v1". DECISIONS.md D9, D16, D23.

**Deliverables.**

- `src/twindle/reference/catering/processes.py` implementing: `truck_travel` (depart-arrive with travel time as a function of distance and a stochastic delay), `event_setup`, `event_service`, `event_teardown` (each consuming time and requiring assigned employees), `truck_breakdown` (a stochastic event that takes a truck out of service for a duration).
- Process registration with the framework runtime so `run_simulation(catering_domain, ...)` picks them up automatically.
- A small set of demo what-if scenarios in `src/twindle/reference/catering/scenarios.py`: at minimum "add two more trucks during summer weekends", "increase event duration by 20%", "remove one truck for a week".
- `tests/reference/catering/test_simulation.py` asserting: a baseline simulation over one week of seeded data completes; events get on-time arrival rates derived from the trace; running a "add two trucks" scenario produces a measurably different on-time rate from baseline.

**Acceptance criteria.** Tests pass. Baseline simulation over a week completes in under 30 seconds. The "add two trucks" scenario changes on-time rate by at least one percentage point.

**On success.** Commit `step 13: catering simulation processes`.

---

### Step 14: Frontend bootstrap

- [ ] complete

**Goal.** A Next.js app under `frontend/` that builds, type checks, and renders a placeholder layout matching the v1 capability bar: map area, chat panel, timeline, comparison view.

**Prerequisites.** Step 13.

**Read first.** SCOPE.md "v1 capability bar". DECISIONS.md D5, D18, D26.

**Deliverables.**

- `frontend/package.json` declaring Next.js (App Router), TypeScript, React, MapLibre GL, CopilotKit (`@copilotkit/react-core`, `@copilotkit/react-ui`, `@copilotkit/runtime`), and `@ag-ui/client`. pnpm is the package manager; `pnpm-lock.yaml` is committed.
- `frontend/src/app/layout.tsx` and `frontend/src/app/page.tsx` rendering a four-pane layout (map, chat, timeline, comparison) with placeholder content in each. The `<CopilotKit>` provider wraps the page tree and points at `/api/copilotkit` as its `runtimeUrl`.
- `frontend/src/app/api/copilotkit/route.ts` mounting CopilotKit's runtime against the Pydantic AI backend using `HttpAgent` from `@ag-ui/client`, following the pattern in CopilotKit's `pydantic-ai-todos` showcase. The agent name and the backend URL are read from environment variables (`AGENT_URL`, default `http://localhost:8000/`).
- `frontend/tsconfig.json` with `strict: true`.
- A frontend CI job in `.github/workflows/ci.yml` running `pnpm install`, `pnpm run typecheck`, and `pnpm run build`.
- `frontend/README.md` with one-paragraph instructions to run the dev server (`pnpm dev`).

**Acceptance criteria.** `pnpm run build` succeeds. TypeScript strict typecheck passes. CI passes. With the backend mock-stubbed, `pnpm dev` renders the four-pane layout at `http://localhost:3000`.

**On success.** Commit `step 14: frontend bootstrap`.

---

### Step 15: Map view with waypoint interpolation

- [ ] complete

**Goal.** The map renders venues as markers and trucks as moving markers driven by the WebSocket playback stream. Truck positions interpolate smoothly between waypoints.

**Prerequisites.** Step 14.

**Read first.** SCOPE.md "Architectural commitments" (spatial split). DECISIONS.md D16, D18, D24.

**Deliverables.**

- `frontend/src/components/Map.tsx` rendering a MapLibre GL map centred on the catering region, showing venue markers and truck markers.
- `frontend/src/api/ws.ts` connecting to `/ws/sim/{scenario_id}`, parsing waypoint messages, and feeding them to the map. The hook also sends pause/resume/seek control messages back over the socket.
- A small interpolation utility that, given waypoints and the current playback time, returns the truck's current lat/lng.
- Two map modes (baseline and what-if) selectable from a toggle in the UI; both share the same map component, parameterised by `scenario_id`.
- A Playwright or Vitest-component test (whichever the team set up in step 14) asserting the map mounts and that, given a fixed waypoint sequence in a stubbed WebSocket, the truck marker moves between two coordinates over time.

**Acceptance criteria.** Frontend tests pass. Manual check: with the agent server running and the baseline simulation already recorded, opening the frontend and connecting to the baseline `scenario_id` shows trucks visibly moving between venues at playback speed 1.0.

**On success.** Commit `step 15: map view with interpolation`.

---

### Step 16: Chat panel via CopilotKit

- [ ] complete

**Goal.** The chat panel uses CopilotKit's React components, which route through the local `/api/copilotkit` runtime to the Pydantic AI AG-UI endpoint, and renders streamed assistant messages, tool calls, and tool results.

**Prerequisites.** Step 15.

**Read first.** DECISIONS.md D13, D17, D18.

**Deliverables.**

- `frontend/src/components/Chat.tsx` using CopilotKit's `CopilotChat` (or `CopilotSidebar`) component, configured against the `<CopilotKit runtimeUrl="/api/copilotkit">` provider set up in step 14.
- Any glue in `frontend/src/api/chat.ts` if needed between the chat component and the AG-UI message format.
- Visible rendering of tool call boxes (the agent's action) and tool result boxes (what came back) inline in the chat, not hidden in a debug panel.
- A test (component or e2e) asserting that, given a stubbed AG-UI stream, the chat renders the messages in order.

**Acceptance criteria.** Tests pass. Manual check: typing "list trucks" produces a tool call and a list of trucks visible in the chat.

**On success.** Commit `step 16: chat panel via CopilotKit`.

---

### Step 17: Scenario comparison and timeline scrubber

- [ ] complete

**Goal.** The comparison view shows baseline and what-if metrics side by side. The timeline scrubber lets the user replay simulation time, and the map state follows.

**Prerequisites.** Step 16.

**Read first.** SCOPE.md "In scope for v1" (frontend paragraph). DECISIONS.md D15, D16.

**Deliverables.**

- `frontend/src/components/Comparison.tsx` showing, for a baseline `scenario_id` and a what-if `scenario_id`, four metrics: on-time rate, truck utilisation, total cost, customer-facing outcome score. Bar charts or simple side-by-side numerics, not custom visualisations.
- `frontend/src/components/Timeline.tsx` showing the simulation's time range with a scrubber. The scrubber's value drives the map's playback time.
- Backend support for these views: an HTTP endpoint that, given a `scenario_id`, returns aggregated metrics computed from `SimulationEvent` / `SimulationState`.
- Tests on the metrics endpoint asserting baseline and what-if return different numbers for the demo "add two trucks" scenario.

**Acceptance criteria.** Tests pass. Manual check: running the demo, the comparison view shows the baseline beside the what-if with visibly different bars. Scrubbing the timeline moves trucks on the map.

**On success.** Commit `step 17: comparison view and timeline scrubber`.

---

### Step 18: End-to-end demo wiring

- [ ] complete

**Goal.** A single command boots the entire system: create the database, seed catering data, run the baseline simulation to completion, start the agent server, and serve the frontend. The four capability-bar interactions all work.

**Prerequisites.** Step 17.

**Read first.** SCOPE.md "v1 capability bar" and "Success criteria". DECISIONS.md D23, D24.

**Deliverables.**

- A `uv run twindle demo` CLI command (under `src/twindle/cli.py`) that: creates a fresh SQLite DB, seeds catering data, runs the baseline simulation synchronously (committing its `SimulationEvent` and `SimulationState` rows), launches the agent server, and prints instructions for starting the frontend (`cd frontend && pnpm dev`).
- A short `docs/demo.md` walking through the four capability-bar interactions with expected outcomes, including the note that running a new what-if takes up to ~30 seconds during which the chat will pause.
- A small `tests/e2e/test_capability_bar.py` that, against a real running stack but a stub LLM, exercises each of the four interactions (data question, what-if, side-by-side comparison, trace-based explanation) and asserts the expected tool calls and metric differences.

**Acceptance criteria.** `uv run twindle demo` boots without manual intervention. The e2e test passes. Manual run of all four capability-bar interactions succeeds against a real LLM.

**On success.** Commit `step 18: end-to-end demo wiring`.

---

### Step 19: Documentation pass

- [ ] complete

**Goal.** Documentation is enough for a new developer to clone, run the demo in under ten minutes, and sketch their own domain.

**Prerequisites.** Step 18.

**Read first.** SCOPE.md "Success criteria".

**Deliverables.**

- Updated `README.md` with: project pitch, the four capability-bar examples, a "ten minute quickstart" (clone, install uv, `uv sync`, `uv run twindle demo`, then `cd frontend && pnpm install && pnpm dev`), a link to a recorded demo video, and a "build your own domain" link.
- `docs/build-your-own-domain.md` walking through subclassing each base type, writing a process, registering scenario primitives, and seeding data. Uses the catering implementation as the worked example.
- `docs/architecture.md` summarising the layer split (domain → persistence/sim/query → agent → transport → frontend) with a single diagram.
- A `CONTRIBUTING.md` with conventions, lint and test commands, and the "one commit per step" policy now relaxed for post-v1 contributions.

**Acceptance criteria.** A team member who has not seen the project clones it, follows the README's quickstart, and reaches a working demo in under ten minutes (timed). If not, simplify the setup or the docs and re-run.

**On success.** Commit `step 19: documentation pass`.

---

### Step 20: Success criteria sweep

- [ ] complete

**Goal.** Explicitly verify every success criterion in SCOPE.md and record results.

**Prerequisites.** Step 19.

**Read first.** SCOPE.md "Success criteria" in full.

**Deliverables.**

- `docs/success-criteria.md` with one section per criterion, the test or measurement applied, and the result.
- Hand-craft 10 benchmark data questions and 5 what-if prompts. Run each against the agent with a real LLM. Record the agent's response and whether it passed.
- Record the catering `domain.py` line count.
- Time a fresh clone-to-demo run and record the duration.
- Record screen capture for a 60-second demo video; embed the link in README.
- A short retrospective at the bottom listing anything that did not meet the bar and the smallest fix.

**Acceptance criteria.** Every success criterion is verified. Data benchmark passes at 8/10 or better. Scenario benchmark passes at 4/5 or better. Domain file under 500 lines. Quickstart under ten minutes. If any criterion fails, fix it and re-run this step; do not declare v1 shipped until all green.

**On success.** Commit `step 20: success criteria sweep`. Tag the commit `v1.0.0`.

---

## After v1

v1.1 adds a second reference domain to validate the generalisation claim. That work belongs to a separate plan and reuses this framework's contract unchanged. Decisions made for v1.1 are appended to DECISIONS.md starting at D27.
