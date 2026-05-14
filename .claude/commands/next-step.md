---
description: Read PLAN.md and complete the next uncompleted step
---

Read `PLAN.md` (fall back to `plan.md` if not found) in the project root and identify the next uncompleted step. Steps are typically marked with checkboxes (`- [ ]` uncompleted, `- [x]` completed) or numbered headings.

Then:

1. Find the first step that is not yet completed.
2. Briefly state which step you are about to do (one sentence).
3. **Research before coding.** Identify every non-trivial library, tool, or protocol the step touches (SimPy, Pydantic AI, the AG-UI adapter, mcp-alchemy, SQLModel async sessions, deck.gl `TripsLayer`, CopilotKit `HttpAgent`, AG-UI client, FastAPI lifespan handlers, MapLibre GL, etc.). For each one, **do not assume you know the current API.** Verify before writing code:
   - Use `WebSearch` and `WebFetch` to read the library's current docs, GitHub README, or canonical example.
   - For Python packages, also run `uv pip show <pkg>` and inspect `.venv/lib/.../<pkg>` source where useful (the installed version is ground truth, not training data).
   - For TypeScript packages, inspect `frontend/node_modules/<pkg>` types and the package's GitHub.
   - Confirm: current version, import path, function/class signatures, idiomatic usage pattern, and any v1-vs-v2 breaking changes. A 30-second check beats a 30-minute debug.
   - Skip research only for things you genuinely know cold (stdlib, pytest basics, plain SQL, basic Python/TypeScript syntax). When in doubt, look it up — silent guessing on small tools is the failure mode this step exists to prevent.
   - State in one line per tool what you confirmed (e.g. "SimPy 4.1.1: `env.process(gen)` returns a `Process`; use `simpy.Resource(env, capacity=n)`, no public resize API"). This evidence trail makes the implementation reviewable.
4. Complete that step — implement code, edit files, run commands, etc. as the step requires. If a research finding contradicts what PLAN.md or DECISIONS.md assumed, stop and surface the conflict rather than silently working around it.
5. After the step is finished, update `PLAN.md` to mark the step as completed (e.g., flip `- [ ]` to `- [x]`, or add a ✅ / "Done" marker matching the file's existing style). If you discovered non-obvious API facts during research that future steps will need (e.g., "mcp-alchemy expects `DB_URL` env var, not a CLI arg"), append them as a new entry in `DECISIONS.md` so the next session inherits them.
6. End with a one-sentence summary of what changed and what the next uncompleted step is.

If every step is already complete, say so and do nothing else.
