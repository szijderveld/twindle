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

## Layer responsibilities (not a file tree)

The framework is organised by layer, not by a prescribed file tree. Steps below name modules by their *layer role* and the public API they must expose; the exact filenames and submodule split are an implementation detail. As a rule of thumb each layer fits under one top-level package under `src/twindle/`, but a layer may grow or shrink files as needed.

- **Domain layer.** The base type taxonomy (`Entity`, `Resource`, `Process`, `Event`, `Schedule`, `Assignment`, `Location`, plus the declarative value objects below) and field role annotations. This is what reference domains subclass.
- **Engine layer.** The framework guts: persistence (engine, sessions, scenario-tagged simulation tables), scenario application, simulation runtime + writer + trace, query tool generator, and the read-only `run_sql` fallback. These are domain-agnostic.
- **Agent layer.** Tool surface builder, prompt template, FastAPI app with the Pydantic AI AG-UI adapter, CLI entry point.
- **Transport layer.** WebSocket route for simulation playback. HTTP wiring beyond what the agent owns.
- **Testing layer.** Shared pytest fixtures (in-memory DB, fake clock, stub LLM).
- **Reference domain(s).** A reference domain is the single source of truth for its business: entities, processes, capacity windows, scenarios, seed data — co-located. v1 ships one such domain (catering). The reference domain only contains *bespoke* code that cannot be expressed declaratively against the domain layer's value objects.
- **Frontend.** Next.js package under `frontend/` (separate package; see D5, D18, D26).

Tests mirror the package they cover; layout under `tests/` is otherwise unconstrained.

The single source of truth contract: *for a domain that uses only the declarative facilities of the domain layer, the engine layer can build, run, and query a simulation with zero domain-specific code.* Bespoke per-domain code is an escape hatch for behaviour that does not fit the declarative spec, not the primary mechanism.

## Steps

### Step 0: Verify decisions are locked (verification-only)

- [ ] complete

**Goal.** Confirm SCOPE.md and DECISIONS.md are in their pre-build resolved state. No code is written.

**Prerequisites.** None.

**Read first.** All of SCOPE.md. All of DECISIONS.md.

**Deliverables.** None. This is a gate.

**Acceptance criteria.** SCOPE.md has no "Open items" section (or it is empty). DECISIONS.md contains entries D1 through D30, in order, with no gaps. The name in D1 matches the name used throughout SCOPE.md and the rest of DECISIONS.md. If any of these fail, fix the planning documents before proceeding to step 1; do not start coding.

**On success.** Mark this step complete in PLAN.md only. No commit (no code yet).

**On failure.** Edit SCOPE.md or DECISIONS.md to reach a stable state and re-run this step.

---

### Step 1: Bootstrap project

- [ ] complete

**Goal.** An empty but fully wired Python project that lints, type checks, and runs an empty test suite green in CI.

**Prerequisites.** Step 0.

**Read first.** SCOPE.md "Technology choices". DECISIONS.md D3, D4, D5, D6, D19, D20. PLAN.md "Global conventions".

**Deliverables.**

- `pyproject.toml` declaring the `twindle` package, Python 3.11+, dependencies pinned to current stable. Runtime: sqlmodel, pydantic, pydantic-ai (with the `ag-ui` extra), simpy, fastapi, uvicorn, websockets, aiosqlite, mcp-alchemy (the SQL fallback MCP server, per D29). Dev: pytest, pytest-asyncio, ruff, mypy. Use PEP 735 `[dependency-groups]` for the dev group.
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

**Goal.** The base taxonomy in the domain layer exists as SQLModel classes and Pydantic value objects rich enough that a domain can declare *everything the simulation needs* — process timing, resource requirements, resource capacity, and capacity changes over time — without writing bespoke simulation code.

**Prerequisites.** Step 1.

**Read first.** SCOPE.md "Architectural commitments" (base types paragraph, single source of truth). DECISIONS.md D7, D9, D10.

**Deliverables.**

- A `base` module in the domain layer defining the seven SQLModel base classes — `Entity`, `Resource`, `Process`, `Event`, `Schedule`, `Assignment`, `Location` — plus the declarative value objects below. Each SQLModel base carries a primary key `id`, a `name` where appropriate, and timestamps where appropriate. `Location` carries `latitude` and `longitude`. `Event` carries `start_time`, `end_time`, and a foreign key to `Location`. `Assignment` carries foreign keys to `Resource` and `Event`. `Schedule` carries a foreign key to `Process` and time bounds.
- **Declarative simulation fields**, in the same module:
  - `Duration` — Pydantic value object describing how long a process takes. Fields: `mean: float` (in simulation time units), `distribution: Literal["constant","triangular","normal","exponential","lognormal"] = "constant"`, optional `min: float`, `max: float`, `std: float`. The engine layer samples from it; the agent layer introspects it.
  - `ResourceRequirement` — Pydantic value object. Fields: `resource_type: type[Resource]` (or a string class name for serialisation), `quantity: int = 1`, `mode: Literal["occupy","consume"] = "occupy"`. `Process` instances carry a `requires: list[ResourceRequirement]`.
  - `Resource.capacity: int = 1` — base number of units / pool size for that resource class. Per D30, capacity is fixed for the duration of any simulation run.
  - `Process` gains `duration: Duration` and `requires: list[ResourceRequirement]` (defaulting to empty). These are declarative fields the engine reads; subclasses set them as class-level defaults or per-instance.
- A field-role annotations module in the domain layer defining `query_field`, `sim_field`, `viz_field` (as `Annotated` metadata or Pydantic `Field` extras). Passive markers the agent and frontend introspect later.
- Docstrings on every base type and value object stating its role.
- Tests under `tests/domain/` covering: a trivial subclass of each SQLModel base can be defined; `Duration` validates its distribution-specific fields (e.g. triangular requires `min` and `max`); `ResourceRequirement` round-trips through JSON; SQLModel builds a SQLite schema; an instance of each base type round-trips through a session.

**Acceptance criteria.** All tests pass. mypy strict passes. Importing the domain `base` module produces no warnings. The seven SQLModel base classes are named exactly as in D10. A trivial domain can express a process's full v1 simulation behaviour — duration and resource needs (at fixed capacity) — using only fields declared here, with no SimPy import. Time-varying capacity is explicitly deferred to v1.1 per D30 and is not part of this step.

**On success.** Commit `step 2: implement base type taxonomy`.

**On failure.** Re-read D10 and the SCOPE.md base-types paragraph; the names and roles of the seven SQLModel types are fixed. The value objects (`Duration`, `ResourceRequirement`) may be refined in shape but their *responsibilities* — timing and requirements — must be expressible declaratively before this step is considered done.

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
- `src/twindle/query/sql.py` with a small `run_sql(query: str, session)` helper that allows read-only `SELECT` and raises on anything else. This is a thin-slice helper for proving the query path end to end before the agent layer exists; per D29 it is superseded at step 8 by mcp-alchemy mounted as an MCP server, and is dropped from the agent's tool surface at that point.
- `tests/e2e/test_thin_slice.py` defining a one-entity domain inline (e.g. `Truck(Entity)` with a `capacity` field), creating a schema, seeding three rows, calling each generated tool, and the fallback `run_sql`.

**Acceptance criteria.** All tests pass. The generated tool names follow the `<verb>_<entity>` convention. `run_sql` rejects an `INSERT`.

**On success.** Commit `step 5: thin end-to-end query slice`.

---

### Step 6: Simulation runtime

- [ ] complete

**Goal.** A synchronous simulation runner that *interprets* the declarative fields on the domain (process duration, resource requirements, fixed base capacity) to build the SimPy graph, runs it to completion, and writes results to `SimulationEvent` and `SimulationState` tagged with `scenario_id`. Domain-specific Python is an escape hatch, not the default path. Per D30, capacity is fixed for the whole run — there is no mid-run resize mechanism in v1.

**Prerequisites.** Step 5.

**Read first.** SCOPE.md "Architectural commitments" (single source of truth, sim output is queryable state, spatial split). DECISIONS.md D9, D15, D16, D23, D24, D25, D30.

**Deliverables.**

- A `runtime` module in the engine layer exposing `run_simulation(domain, scenario, session, until) -> ScenarioRunResult`. This is a **synchronous** function. It creates a SimPy `Environment`, materialises entities and `Process` instances from the database using a sync `Session`, applies scenario primitives to the in-memory state, builds the SimPy resource and process graph from the *declared* domain (see below), runs `env.run(until=until)`, and returns when the event loop is empty. Per D23, this function is intended to be called directly from async request handlers and will block them; per D24, all events are written to the database before the function returns.
- **Declarative interpretation.** For each `Resource` subclass the runtime instantiates a SimPy `Resource` (or `PriorityResource` / `Container` if the `mode` on a requirement implies it) with capacity equal to the base `capacity` field. Capacity does not change during the run (D30). For each `Process` instance the runtime generates a SimPy process function automatically: it requests the resources listed in `requires` with the specified `quantity`, samples a hold time from `duration`, yields `env.timeout(...)`, releases occupied resources, and emits one `SimulationEvent` row at each transition. The runtime needs no domain-specific code to do any of this.
- **Bespoke escape hatch.** A registration mechanism (e.g. a `@bespoke_process(ProcessClass)` decorator exported from the engine layer, or a `bespoke_processes` attribute on a reference domain module) lets a domain override the auto-generated process for a specific `Process` subclass when its behaviour cannot be expressed declaratively (e.g. stochastic event injectors, processes that branch on simulation state). When such an override is registered, the runtime uses it instead of the declarative interpreter for that class only.
- A `SimWriter` in the engine layer that the runtime passes to processes (auto-generated and bespoke alike); processes call `writer.event(...)` and `writer.state(...)` and the writer batches inserts to the DB on a configurable threshold and on `flush()` at end of run. The writer does **not** publish to any in-process broker (the WebSocket reads from the DB after the run completes; see D24).
- A `trace` module in the engine layer exposing `get_trace(scenario_id, entity_type, entity_id, session)` (ordered events for one entity in one scenario) and `get_event_trace(scenario_id, event_id, session)` (all simulation events relating to a single domain `Event` across all participating entities — supports "why did event 47 finish late?").
- Tests under `tests/simulation/`:
  - `test_runtime_declarative.py`: a domain with one `Resource` subclass (capacity 2), one `Process` subclass with `Duration(mean=5, distribution="constant")` and `requires=[ResourceRequirement(That Resource, 1)]`, three `Process` instances. Asserts that the runtime executes all three without any domain-specific Python, that two run concurrently and the third queues, and that events land with the right `scenario_id`.
  - `test_runtime_bespoke.py`: registers a bespoke override for one `Process` subclass and asserts the override fires instead of the auto-generated path. The override is used in step 13 to demonstrate that "occupy the resource for a sampled duration" is enough to model temporary unavailability without needing capacity changes.
  - `test_trace.py`: asserts `get_trace` returns events in order for one entity, and `get_event_trace` returns events for all entities participating in a given domain event.

**Acceptance criteria.** Tests pass. A second `run_simulation` call with a different `scenario_id` does not overwrite or mix with the first run's rows. `run_simulation` is annotated and behaves as a synchronous function (not `async def`). **A reference domain that uses only the declarative facilities runs end to end with zero bespoke process code** (verified by `test_runtime_declarative.py` and the matching catering check in step 13).

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

**Goal.** A function that, given a domain module, returns a list of Pydantic AI tools spanning: typed query tools, the ad-hoc SQL fallback (mcp-alchemy via MCP), scenario construction tools (one per primitive), `run_scenario`, `get_trace`, `get_event_trace`, and introspection tools for the declarative simulation fields on the domain.

**Prerequisites.** Step 7.

**Read first.** SCOPE.md "Architectural commitments" (single source of truth, generated agent tool surface). DECISIONS.md D13, D14, D29, D30.

**Deliverables.**

- A `builder` module in the agent layer exposing `build_tool_surface(domain_module) -> list[Tool]`. Tools wrap step 5's typed query tools, step 6's `run_simulation`, `get_trace`, and `get_event_trace`, and one tool per scenario primitive that returns a primitive instance (the agent composes a `Scenario` by calling these). The `run_scenario` tool wraps `run_simulation`; since `run_simulation` is synchronous (D23), the tool is implemented as a Pydantic AI tool that simply calls it and returns when it returns.
- **SQL fallback via mcp-alchemy.** Per D29, the framework launches mcp-alchemy as a stdio MCP subprocess bound to the persistence engine and registers it on the Pydantic AI agent (e.g. via `pydantic_ai.mcp.MCPServerStdio`). mcp-alchemy's `list_tables`, `describe_table`, and `execute_query` tools become available to the agent alongside the typed query tools. The step-5 `run_sql` helper is no longer exposed in the agent's tool surface from this step onwards. Subprocess lifecycle (start at app boot, terminate at shutdown) is wired in step 9.
- **Declarative-field introspection tools** (all auto-generated from the domain):
  - `describe_process(process_type: str)` — returns the process's declared `Duration` (mean, distribution, bounds) and `requires` list.
  - `get_resource_capacity(resource_type: str)` — returns the base capacity declared on the resource class. Per D30, capacity is fixed for the run, so this tool takes no time parameter.
- A `prompt` module in the agent layer containing a template that consumes domain class docstrings, field descriptions, and the declared `Duration` / `requires` on each `Process` subclass to produce a system prompt naming the entities, the typed tools, the scenario vocabulary, and a one-line summary of each process's timing and resource needs.
- Tests under `tests/agent/test_builder.py` asserting: for the one-entity test domain extended with one `Process` subclass, the expected tool names exist (including `describe_process`, `get_resource_capacity`); tool schemas validate; the mcp-alchemy MCP server is registered and exposes `execute_query` to the agent; the rendered prompt contains the entity name, the six primitive names, and the declared duration of the process. Tests may launch mcp-alchemy against a temporary SQLite file.

**Acceptance criteria.** Tests pass. Tool descriptions are non-empty and derived from the domain (not hard-coded). The agent can answer "what is the capacity of resource X" purely from the declarative-introspection tools without falling back to `execute_query`. mcp-alchemy starts cleanly and rejects non-SELECT statements end to end.

**On success.** Commit `step 8: agent tool generation`.

---

### Step 9: Agent server with AG-UI adapter

- [ ] complete

**Goal.** A Pydantic AI agent running behind FastAPI with the AG-UI adapter for streaming, exposed over HTTP, that can call the generated tools end to end against a real database.

**Prerequisites.** Step 8.

**Read first.** DECISIONS.md D13, D17.

**Deliverables.**

- `src/twindle/agent/server.py` exposing `make_app(domain_module, engine)` that returns a FastAPI app with one `/chat` route handled by the Pydantic AI agent and the AG-UI adapter. The agent's tools come from `build_tool_surface`. The agent's system prompt comes from `prompt.render(domain_module)`. The app's lifespan handler starts the mcp-alchemy subprocess (per D29) at boot, points it at the engine's SQLite file, and terminates it on shutdown.
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

### Step 11: Declare the catering domain (the SSOT)

- [ ] complete

**Goal.** The catering domain — entities, relationships, processes with their declared timing and resource requirements, and base capacities — exists as a single declarative module that is the *only* place catering simulation behaviour is defined. Bespoke Python is deferred to step 13 and limited to what cannot be declared.

**Prerequisites.** Step 10.

**Read first.** SCOPE.md "v1 capability bar", "In scope for v1" (reference catering domain), "Success criteria" (the catering definition reads as the single source of truth). DECISIONS.md D10, D21.

**Deliverables.**

- A `domain` module in the catering reference (under the reference layer) defining catering entities as subclasses of the framework base types. At minimum: `Truck(Resource)` with a base `capacity` (typically 1 — one job at a time per truck) and any domain-specific fields (e.g. `payload_kg`); `Employee(Resource)` with base `capacity` 1; `Venue(Location)`; `CateringEvent(Event)`; `Shift(Schedule)`; `EventAssignment(Assignment)`; `Booking(Entity)` (or equivalent linking a customer to a catering event). Each class has a docstring describing its role and field-level descriptions on every column. Relationships are declared with SQLModel `Relationship` where needed.
- **Declared processes** as `Process` subclasses with their `duration: Duration` and `requires: list[ResourceRequirement]` set as class-level defaults:
  - `TruckTravel(Process)` — duration sampled from a distribution (e.g. `Duration(mean=..., distribution="triangular", min=..., max=...)`) parameterised by the leg distance via a small helper; requires one `Truck`.
  - `EventSetup(Process)` — duration distribution; requires one `Truck` (occupied) plus N `Employee`s.
  - `EventService(Process)` — duration tied to the `CateringEvent` size; requires N `Employee`s.
  - `EventTeardown(Process)` — duration distribution; requires N `Employee`s and (optionally) one `Truck`.
- **Shifts are data, not sim constraints in v1.** Per D30, capacity is fixed for the simulation run. The catering domain stores `Shift` rows so the agent can answer queries about shift history and assignments, but the runtime treats every seeded `Employee` as available across the entire simulation window. No `CapacityWindow` is declared, generated, or interpreted. Shift-driven availability returns in v1.1 when time-varying capacity is reintroduced.
- **No bespoke process file in this step.** `truck_breakdown` is deferred to step 13 because it is a stochastic event injector and does not fit the declarative spec. No other catering process file is created here.
- Tests under `tests/reference/catering/`:
  - `test_domain.py`: every catering entity inherits from the correct framework base; the schema builds without error; the generated tool surface (step 8) includes a list/get/filter tool for every catering entity and `describe_process` returns non-empty timing/requirements for each declared process.
  - `test_declarative_completeness.py`: assert that all four declared processes (`TruckTravel`, `EventSetup`, `EventService`, `EventTeardown`) carry a non-empty `duration` and `requires`, and that running the engine's declarative interpreter on a tiny seeded catering DB produces simulation events with no bespoke override registered.

**Acceptance criteria.** Tests pass. The catering domain module is under 500 lines, including the declared processes and their timing/requirements (this is the SSOT — the line count covers it). Every public class and field has a description. The declarative interpreter alone runs catering processes against a seeded DB.

**On success.** Commit `step 11: declare catering domain`.

**On failure.** If the module exceeds 500 lines, simplify before moving on; the success criterion is non-negotiable. If a process cannot be expressed declaratively, first try to extend the value objects in step 2 rather than introducing bespoke code here.

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

### Step 13: Catering bespoke logic and demo scenarios

- [ ] complete

**Goal.** Implement the *only* parts of the catering simulation that cannot be expressed declaratively in step 11 (currently: stochastic truck breakdowns), and the demo what-if scenarios that exercise the agent layer.

**Prerequisites.** Step 12.

**Read first.** SCOPE.md "v1 capability bar" and the catering paragraph of "In scope for v1". DECISIONS.md D9, D16, D23, D30. Step 6 (bespoke escape hatch) and step 11 (declared processes).

**Deliverables.**

- A small bespoke module in the catering reference implementing `truck_breakdown` as a registered override (per the bespoke escape hatch in step 6): a stochastic event injector that periodically samples a breakdown event and starts a long-running "repair" SimPy process that occupies the affected truck (holds a unit of the `Truck` resource) for a sampled repair duration. While the repair process holds the unit, no other catering process can acquire that truck. When the repair completes, the unit is released back to the pool. Per D30, this occupy-and-release pattern is the v1 way to model temporary unavailability — no capacity resize is involved.
- The bespoke module is short: target under 100 lines including imports and docstrings. If it grows beyond that, audit whether parts can be lifted into the declarative spec.
- A demo scenarios module in the catering reference exporting at minimum: "add two more trucks to the fleet" (two `AddEntity` primitives for `Truck`), "increase event duration by 20%" (`ScaleParameter` on `Duration.mean`), "remove one truck" (`RemoveEntity` for a `Truck`). All three apply to the full sim window; per D30 there is no sub-window framing in v1.
- Tests under `tests/reference/catering/test_simulation.py` asserting: a baseline simulation over one week of seeded data completes; on-time arrival rates can be derived from the trace; the "add two trucks" scenario produces a measurably different on-time rate from baseline; the breakdown injector actually occupies and releases trucks (verified by inspecting trace events for repair-process start/end on the affected truck).

**Acceptance criteria.** Tests pass. Baseline simulation over a week completes in under 30 seconds. The "add two trucks" scenario changes on-time rate by at least one percentage point. Bespoke catering code is under 100 lines.

**On success.** Commit `step 13: catering bespoke logic and demo scenarios`.

---

### Step 14: Frontend bootstrap

- [ ] complete

**Goal.** A Next.js app under `frontend/` that builds, type checks, and renders a placeholder layout matching the v1 capability bar: map area, chat panel, timeline, comparison view.

**Prerequisites.** Step 13.

**Read first.** SCOPE.md "v1 capability bar". DECISIONS.md D5, D18, D26.

**Deliverables.**

- `frontend/package.json` declaring Next.js (App Router), TypeScript, React, MapLibre GL, deck.gl (`deck.gl` and `@deck.gl/geo-layers` for `TripsLayer`, per D28), CopilotKit (`@copilotkit/react-core`, `@copilotkit/react-ui`, `@copilotkit/runtime`), and `@ag-ui/client`. pnpm is the package manager; `pnpm-lock.yaml` is committed.
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

**Goal.** The map renders venues as markers and trucks as moving paths driven by the WebSocket playback stream. Position interpolation is handled by deck.gl's `TripsLayer` per D28; Twindle does not implement its own interpolation.

**Prerequisites.** Step 14.

**Read first.** SCOPE.md "Architectural commitments" (spatial split). DECISIONS.md D16, D18, D24, D28.

**Deliverables.**

- `frontend/src/components/Map.tsx` rendering a MapLibre GL base map centred on the catering region, with venue markers and a deck.gl overlay. Truck animation is rendered by `TripsLayer` (from `@deck.gl/geo-layers`), fed per-truck `path` (array of `[lng, lat]`) and `timestamps` (parallel array of sim times) and driven by a `currentTime` prop. There is no hand-rolled interpolation utility; `TripsLayer` interpolates internally.
- `frontend/src/api/ws.ts` connecting to `/ws/sim/{scenario_id}`, parsing waypoint messages, and accumulating per-truck `path` and `timestamps` arrays in the shape `TripsLayer` consumes. The hook also sends pause/resume/seek control messages back over the socket.
- `currentTime` is held in a small React state owned by the page; the WebSocket advances it as messages arrive, and the timeline scrubber (step 17) overrides it. Seeking and pausing become local state changes on `currentTime`, not WebSocket round-trips, except where the user wants the server to skip ahead in its emission rate.
- Two map modes (baseline and what-if) selectable from a toggle in the UI; both share the same map component, parameterised by `scenario_id`.
- A Playwright or Vitest-component test (whichever the team set up in step 14) asserting the map mounts and that, given a fixed waypoint sequence in a stubbed WebSocket and a fixed `currentTime`, the rendered truck position matches the expected interpolated point on the path.

**Acceptance criteria.** Frontend tests pass. Manual check: with the agent server running and the baseline simulation already recorded, opening the frontend and connecting to the baseline `scenario_id` shows trucks visibly moving between venues at playback speed 1.0, with trails rendered by `TripsLayer`.

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
- `frontend/src/components/Timeline.tsx` showing the simulation's time range with a scrubber. The scrubber's value writes directly to the `currentTime` state that the map's `TripsLayer` consumes (per D28), so seeking moves trucks to their interpolated position at that time without re-fetching from the WebSocket.
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
