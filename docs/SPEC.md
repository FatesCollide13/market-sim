# Marketplace Sim — Baseline Spec v0.1

## The goal

A food market that runs unattended for weeks, never needs intervention, never prints money, and looks measurably different on day 30 than it did on day 1.

Not a demo. A system with a pulse.

---

## Definition of done

The baseline is finished when all seven are true:

1. Runs 30 simulated days headless, with zero API calls and zero intervention.
2. Total money is conserved. Asserted every tick.
3. No permanent starvation. No permanent bankruptcy spiral.
4. Prices settle into an equilibrium band and stay there.
5. Day 30 differs measurably from day 1 in prices, reputation, or roster.
6. Same seed produces a byte-identical run.
7. Kill the process at any tick, restart, and it resumes from the last checkpoint.

If all seven hold, the renderer and the LLM are decoration you can add safely. If any fail, adding them just hides the bug.

---

## The economy

The single most important design decision. Money moves in a cycle, not a line.

```
customers ──pay for dishes──> stall owners
    ^                              │
    │                              │
    │                         buy ingredients
   wages                           │
    │                              v
    └──────────────────────── supplier
```

Three flows, one loop, money conserved:

- Customers pay owners for dishes.
- Owners pay the supplier for ingredients.
- The supplier pays customers a daily wage.

The supplier is not a character. It is the town's clearing house, a float that closes the loop. It never runs dry because everything that leaves it comes back.

**The invariant:**

```python
assert abs(total_money() - INITIAL_TOTAL) < 0.01
```

Run this every tick in debug mode. It is the cheapest bug detector in the project. If money is leaking, you know within one tick instead of forty days.

### Why this matters

If customers only spend, they are broke by day three and the market is a ghost town. This failure is invisible until you run it, and it kills more market sims than any other single cause.

---

## Agents

### Needs

Scalars 0–100. They drift each tick. Personality weights multiply them, so two agents with identical needs behave differently.

| Need | Who | Behavior |
|---|---|---|
| Hunger | Customers | Rises steadily. Drives buying. |
| Wallet | All | Finite. Refilled by wages (customers) or sales (owners). |
| Energy | All | Drains with activity. Resets overnight. |
| Patience | Customers | Drains while queuing. Low patience means they leave. |
| Social | All | Rises when alone. |
| Stock | Owners | Ingredients on hand. Constraint, not a need. |
| Reputation | Owners | Slow-moving consequence of past sales. |

### Personality weights

A vector per agent, sampled at creation, stored in YAML for the starting roster.

- `thrift` — how much price suppresses utility
- `novelty` — pull toward untried stalls
- `loyalty` — pull toward remembered-good stalls
- `ambition` — probability of opening a stall when rich (customers only)
- `patience` — queue tolerance

These are the dials that make agents distinguishable. They are also what reflection nudges over time.

### Roles (baseline)

**Customer.** Actions: `Browse`, `Queue`, `Buy`, `Eat`, `Chat`, `Idle`, `OpenStall`.

**Stall owner.** Actions: `Cook`, `Restock`, `Pitch`, `Serve`, `SetPrice`, `Idle`, `CloseStall`.

That is it for v0.1. Employees, janitors, and trash are v0.2 and must require zero changes to agent code. That constraint is the real test of the architecture.

---

## The decision system

Smart objects with advertisements. Straight from The Sims.

Agents do not know what exists in the world. Entities advertise what they offer. Agents score nearby advertisements against their own needs and personality, then act.

```python
@dataclass
class Action:
    name: str
    provider: EntityID
    requirements: list[Requirement]   # role, proximity, stock, money
    effects: dict[str, float]         # need deltas
    duration: int                     # ticks
    def utility(self, agent) -> float: ...
```

### The scoring loop

```python
def choose(agent, world):
    ads = world.advertisements_near(agent.pos, radius=AD_RADIUS)
    viable = [a for a in ads if a.requirements_met(agent)]
    scored = [(a.utility(agent), a) for a in viable]
    scored.sort(reverse=True)
    top = scored[:TOP_N]                  # TOP_N = 3
    return weighted_random(top)           # NOT max()
```

**`weighted_random(top)`, not `max()`.** This is the single most important line in the file.

The Sims does not pick the best action. It picks randomly from among the top scorers. Argmax agents read as state machines. Top-N random reads as alive, and it means agents never perfectly optimize, which is what leaves room for a story to happen.

---

## The day boundary

Night is not idle time. It is where every slow dynamic lives. This sequence is the heartbeat.

1. Market closes. Agents go home.
2. Owners settle books: `profit = revenue - ingredient_cost`.
3. **Price adjustment.** Sold out early, raise price 5%. Sat unsold, lower 5%. Clamp to a floor and ceiling.
4. **Bankruptcy check.** Owner cannot afford tomorrow's minimum restock, stall closes, **owner converts to a customer** and keeps their wallet. Money conserved. Population conserved.
5. **Entry check.** Customer with `savings > STALL_COST` and a high `ambition` roll opens a stall, converts to owner, pays `STALL_COST` into the supplier float.
6. Supplier pays each customer their daily wage.
7. Reflection fires for any agent over the significance threshold. (v0.3. Stub it now.)
8. Needs partially reset. Energy full, hunger reduced but not zeroed.
9. **Checkpoint written to disk.**
10. Daily metrics appended to the log.
11. `day += 1`

### Where growth comes from

Not from agents getting smarter. From four dials:

- **Price discovery** finds an equilibrium.
- **Reputation** compounds. Good stalls get more traffic.
- **Entry and exit** reshape the roster. Class mobility both directions.
- **Memory** shifts individual preference. (v0.3.)

A failed owner wandering the market as a customer, and a customer who saved enough to open a stall next to the one that beat them, are both free consequences of step 4 and step 5. That is emergent narrative for about twenty lines of code.

---

## Where the model fires (v0.3, not now)

Three sites. Nowhere else.

| Site | Model | Frequency | Effect |
|---|---|---|---|
| Dialogue | Haiku 4.5 | ~25 / sim day | Flavor only. No mechanical effect. |
| Pitch | Haiku 4.5 | ~30 / sim day | Modifies a persuasion roll. |
| Reflection | Sonnet | ~33 / sim day | Nudges personality weights. |

**~90 calls per simulated day, roughly $0.25.** Naive (every agent, every tick) is ~31,680 calls and ~$28/day.

Gates: proximity plus action selection, per-agent cooldowns, significance threshold for reflection, and a hard budget guard in `brain/` that returns templated strings once the daily ceiling is hit.

**The principle:** rules own outcomes, the model owns texture. If the model decides whether a sale happens, you lose determinism, testability, replay, and fast-forward all at once.

---

## Layout

```
sim/                    # pure python. no llm. no network.
  core/                 entity, world, clock, spatial index
  components/           position, needs, inventory, role, memory
  actions/              base.py + one file per action
  agents/               agent.py, utility.py, personality.py
  roles/                customer.py, stall_owner.py
  economy/              supplier.py, pricing.py, ledger.py
brain/                  # the ONLY anthropic imports
  client.py, dialogue.py, reflection.py, budget.py
server/                 # fastapi + websocket (v0.2)
  app.py, schema.py
client/                 # typescript + phaser (v0.2)
scenarios/
  market_v1.yaml        # the world as data
tests/
  test_conservation.py  # the money invariant
  test_determinism.py   # same seed, same run
  test_no_starvation.py # 30-day soak
```

**The rule:** `sim/` imports nothing from `brain/` or `server/`. It calls out through an interface it defines itself. Tests pass a `FakeBrain` returning canned strings.

If you cannot run 30 simulated days in CI with no network and no API key, the architecture is wrong.

---

## What lives in YAML

Everything you will want to tune at 1am without touching Python:

```yaml
world:
  tick_seconds: 10
  day_start_hour: 8
  day_end_hour: 20
  seed: 42

economy:
  initial_total_money: 10000
  daily_wage: 20
  stall_cost: 200
  ingredient_cost: 3
  price_floor: 2
  price_ceiling: 30

needs:
  hunger_per_tick: 0.35
  energy_per_tick: 0.12
  patience_per_queue_tick: 1.5

decision:
  top_n: 3
  ad_radius: 8

agents:
  customers: 8
  stall_owners: 3
```

Hunger rate versus day length is the tuning that matters most. Too fast and they starve by noon. Too slow and they never buy. This gets found empirically, not guessed, which is exactly why it is data.

---

## Build order

| Version | Scope | Gate to pass |
|---|---|---|
| **v0.1** | Headless sim. Needs, ads, scoring, economy, day loop, checkpoint. Text log only. | All seven criteria above |
| v0.2 | FastAPI + websocket + Phaser client + NPC inspector + speed controls | Watch a day without touching the sim |
| v0.3 | `brain/` wired in. Dialogue, pitch, reflection. Budget guard. | 30 days under $10 |
| v0.4 | Employees, janitors, trash. **Zero changes to agent code.** | The modularity test |
| v0.5 | Bartering. Two-party actions. | — |

Do not start v0.2 until v0.1 passes all seven. The temptation to add sprites early is the strongest one in this project and it is the one that produces a pretty simulation that dies on day three for reasons you cannot find.

---

## Prior art

| Source | Take | Skip |
|---|---|---|
| The Sims (utility AI) | Advertisements, top-N random pick, motive multipliers | — |
| Mesa (Apache-2.0) | Grid, scheduler, DataCollector, `batch_run` | SolaraViz — you have Phaser |
| `joonspk-research/generative_agents` (Apache-2.0) | Memory stream, importance scoring, reflection trigger, fork/save/resume | Django frontend, 2023 scaffolding |
| DeepMind Concordia | Component architecture, economic GM design | The LLM Game Master. Too expensive for a persistent world. |

Read `reverie.py` end to end before writing `agents/`. Read Mesa's economy tutorial before writing `economy/`.
