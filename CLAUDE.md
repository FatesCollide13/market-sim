# market-sim

A food marketplace simulation with utility-AI NPCs (customers, stall owners) that
runs unattended and grows day over day. Full design context lives in `docs/SPEC.md` —
read it before implementing anything in `sim/`.

## Architecture rules (non-negotiable)

- `sim/` imports nothing from `brain/` or `server/`. Ever. It calls out through an
  interface it defines itself. This is what keeps the simulation free, deterministic,
  and testable without a network connection.
- Rules own outcomes. The model only ever generates flavor text or modifies a
  mechanical roll that the rules already computed. The model never decides whether
  something happens.
- Every action (Buy, Cook, Pitch, Clean, etc.) is a self-contained file in
  `sim/actions/`. Adding a new action or a new role must not require editing
  `agents/agent.py` or the scoring loop itself.
- Decisions use top-N random selection, not argmax. See `sim/agents/utility.py`.
  Never change this to "always pick the best scoring action" — that's what makes
  agents read as state machines instead of characters.

## Invariants — these must never break

- Total money in the simulation is conserved every tick. `tests/test_conservation.py`
  asserts this. If a change makes that test fail, the change is wrong, not the test.
- Same seed produces a byte-identical run. `tests/test_determinism.py` checks this.
- No permanent starvation spiral, no permanent bankruptcy spiral over a 30-day run.
  `tests/test_no_starvation.py` is the soak test.

Run the test suite before considering any `sim/` change done:
```
uv run pytest
```

## Where the model is allowed to be called (not yet active — this is v0.3)

Only three sites, ever: dialogue between two agents in proximity, an owner's Pitch
action, and periodic reflection when an agent's accumulated memory importance
crosses a threshold. All three are gated by cooldowns and a hard daily budget guard
in `brain/budget.py`. Do not add a model call anywhere else without flagging it
first — it's very easy to accidentally reintroduce a call-every-tick pattern that
turns a $0.25 simulated day into a $28 one.

## Build order

Do not start on `server/` or `client/` until `sim/` v0.1 passes all seven criteria
listed in `docs/SPEC.md` under "Definition of done." Renderers make a broken
simulation look like a working one — get the invariants passing headless, first.

## Style

- Plain, readable prose in comments, docstrings, and commit messages. Active voice,
  natural sentence rhythm.
- No em dashes.
- Type hints on everything. Pydantic models for anything that crosses a boundary
  (server responses, scenario config loading).
- Prefer small, single-purpose files over large ones. One action per file in
  `sim/actions/`, one role per file in `sim/roles/`.

## Don't

- Don't add API calls inside `sim/`.
- Don't reach for the Phaser renderer or FastAPI server before `sim/` v0.1 is green.
- Don't add new dependencies without checking they're actually needed — this project
  intentionally avoids Docker, avoids a database, and avoids anything beyond Mesa +
  Pydantic + FastAPI + the Anthropic SDK.
- Don't guess at tuning numbers (hunger decay rate, price adjustment %, etc.) —
  these live in `scenarios/market_v1.yaml` and get found empirically, not hardcoded.
