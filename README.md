# soup-kitchen

Soup Kitchen Experiment (Standalone)

A compact simulation of an eatery where persistent staff learn policies over repeated episodes while ephemeral patrons traverse a service pipeline—rendered in Unity, simulated in Python.

1) Goals

- Python-first simulation core: deterministic, testable, fast iteration.
- Unity GUI client: visualization + spawning controls + metrics dashboard.
- Learning agents: start with contextual bandits per role (fast training, explainable).
- Portfolio quality: clear loop, measurable outcomes, visible learning curves.

2) Non-negotiable requirements

- Every character has, at minimum: race/species, culture, ideology, job role (fully exposed initially).
- Staff persist and learn from experience (across episodes).
- Patrons are non-persistent (spawn at entry, despawn at exit).
- GUI supports spawning custom-designed or random staff/patrons.

Portfolio-safe enforcement guardrail

To keep this portfolio-ready and ethically clean: enforcement decisions should depend on observable behavior and situational cues (agitation, detected contraband, incident history, queue pressure), not identity labels. Identity traits can still shape preferences, talkativeness, misunderstandings, gossip affinity, etc.

3) Facility layout (stations)

- Street entry
- Security checkpoint (queue + contraband lockers)
- Reception lobby (purchase service tokens + booth assignment)
- Serving counter
- General eating area (benches)
- Private booths (wait service)
- Kitchen
- Restrooms (optional later)
- Storage room (food stocks)
- Exit security checkpoint (retrieve contraband + theft check)
- Street exit

4) Patron flow (canonical)

1. Patron spawns at entry.
2. Queues for security check; contraband is secured in lockers.
3. Proceeds to host; pays currency → receives service token:
   - General: soup in general area
   - Booth: meal in private booth with wait service
4. General: queues at serving, exchanges token for soup, sits and eats.
5. Booth: goes to assigned booth; waiter collects token, fetches meal, delivers meal.
6. After eating: leaves bowl/tray and proceeds to exit checkpoint.
7. Exit: retrieves contraband; may attempt to conceal pilfered property.
8. Despawns.

5) Staff roles (canonical)

- Security
- Host
- Waiter
- Busser
- Server
- Cook

Each role is implemented as discrete actions with:

- Preconditions
- Duration
- State deltas (effects)
- Optional event emissions (incident log)

6) Simulation model (Python)
6.1 Timebase

Use a discrete-time loop (e.g., dt = 0.25s or 0.5s) plus action durations:
- Agents select an action when idle.
- Actions complete after duration; effects apply atomically.

6.2 Entities & state

Patron (ephemeral)
- Traits: species (Human|Gowachin), culture, ideology, role
- Observables: agitation (0..1), talkative (0..1), group_id, visible_items
- Hidden: theft_propensity (0..1), violence_propensity (0..1), intent (general|booth)
- State: location, queue_id, has_token, has_food, is_ejected

Staff (persistent)
- Role: security/host/waiter/busser/server/cook
- Policy: scripted | bandit (v0.1) | RL (later)
- Internal: fatigue, experience_level, learned params snapshot
- State: current_action, action_remaining

World
- Queues per station
- Booth occupancy map
- Lockers (contraband custody mapping)
- Inventory: raw/prepped/ready food; cutlery/dish counts (optional early)
- Incident log: time/type/participants/location/severity
- Metrics: revenue, throughput, wait times, incidents, satisfaction distribution

7) North-star metrics (in-game dashboard)

Business
- Revenue/min
- Cost/min (food usage + breakage)
- Throughput (patrons served/hr)
- Wait time per station (security/host/serving)

Safety
- Violence incidents/hr
- Contraband found rate
- False positive rate (intrusive actions that find nothing)
- Theft attempts + recovery rate

Experience
- Satisfaction (distribution + mean)
- Mood change over visit

These metrics also drive reward shaping.

8) Discrete action sets (MVP-friendly)

Security actions
- WAVE_THROUGH
- STANDARD_SEARCH
- INTENSIVE_SEARCH
- HOLD_FOR_QUESTIONING
- EJECT_PATRON

Host actions
- SELL_GENERAL
- SELL_PRIVATE (requires booth available)
- ASSIGN_BOOTH(strategy=nearest|quiet|separate_groups)
- SMALL_TALK
- CALL_SECURITY

Server actions
- SERVE_NEXT_GENERAL
- SERVE_NEXT_WAITER
- CHECK_STOCKS
- ASSIST_COOK

Cook actions
- COOK_SOUP_BATCH
- PREP_INGREDIENTS
- MAINTAIN_EQUIPMENT

Add Waiter + Busser in v0.2 (once the single-lane loop is rock solid).

9) Learning (v0.1)
9.1 Method: Contextual bandit (per role)

Start with bandits because they:
- train fast,
- are explainable,
- and fit “station-level repeated decisions.”

First learning target: Security search decision.

Context features (examples)
- queue_len_security
- recent_incidents_60s
- patron_agitation
- time_to_close
- throughput_last_60s

Actions
- wave / standard / intensive / hold

9.2 Reward shaping (simple + defensible)

Example per-decision reward:

reward =
+ (contraband_found * 1.0)
+ (patron_processed * 0.2)
- (wait_time_seconds * 0.01)
- (intrusiveness_penalty[action])
- (violence_incident * 2.0)
- (theft_success * 1.0)

Keep weights in a config file so you can tune and demonstrate tradeoffs.

9.3 Persistence

Persist learned parameters per staff member or per role:
- Save to JSON in sim/state/policies/
- Load on sim start

10) Python <-> Unity integration

Architecture
- Python = authoritative simulation
- Unity = renderer + input + UI

Transport (recommended)
- WebSocket on localhost with JSON messages.

Unity -> Python commands
- cmd.spawn_patron { traits | random_seed }
- cmd.spawn_staff { role, policy_type }
- cmd.pause { bool }
- cmd.set_speed { multiplier }
- cmd.reset_episode { seed }
- cmd.set_config { spawn_rates, reward_weights }

Python -> Unity events
- evt.snapshot { time, entities, queues, booths, inventory, metrics } (fixed rate, e.g., 10 Hz)
- evt.incident { time, type, participants, location } (immediate)
- evt.training { episode, avg_reward, action_counts }

11) MVP milestones (ship in slices)

v0.1 “Single lane” (recommended first shippable)
- Stations: entry -> security -> host -> serving -> eat -> exit
- Service types: General soup only
- Staff: Security, Host, Server, Cook
- Incidents: contraband found + theft attempt at exit
- Learning: Security bandit policy
- UI: spawn patron/staff, pause/speed, metrics dashboard, live queues

Acceptance tests
- 100 patrons can complete loop without deadlocks.
- Metrics update correctly; incident log captures events.
- Bandit improves at least one target metric over episodes (e.g., contraband found or reduced incidents).

v0.2
- Add booths + waiter
- Add talkative/taciturn and satisfaction impacts
- Add busser + missing cutlery reports

v0.3
- Add violence/altercations + instigator uncertainty
- Add learning for host upsell + server prioritization

12) Suggested repo layout
soup-kitchen-experiment/
  README.md
  LICENSE
  .gitignore

  sim/                      # Python authoritative simulation
    pyproject.toml
    src/
      skx/
        __init__.py
        main.py             # entrypoint: run sim server
        config/
          default.yaml
        core/
          world.py          # world state + tick
          entities.py       # Patron/Staff definitions
          stations.py       # checkpoint/lobby/serving/eating/exit
          actions.py        # action definitions + effects
          metrics.py
          incidents.py
        policies/
          scripted.py
          bandit.py
        net/
          ws_server.py      # WebSocket server
          protocol.py       # message schemas
        utils/
          rng.py
          logging.py
    tests/
      test_flow_general.py
      test_no_deadlocks.py
      test_reward_shapes.py

  unity/                    # Unity client project
    SoupKitchenUnity/
      Assets/
        Scripts/
          Net/
            WsClient.cs
            Protocol.cs
          UI/
            SpawnPanel.cs
            MetricsPanel.cs
            IncidentPanel.cs
          View/
            EntityRenderer.cs
            QueueRenderer.cs
            BoothRenderer.cs
      ProjectSettings/

  docs/
    design_notes.md
    reward_tuning.md
    demo_script.md

13) Demo script (portfolio-ready)

A 60–90 second recorded demo that shows:
1. Spawn random patrons + one security staff (bandit).
2. Run 3–5 episodes at high speed.
3. Show a simple chart/overlay: incidents decrease OR throughput rises.
4. Pause and inspect one security decision with context features visible.
5. Conclude with “policy saved” and “policy loaded” persistence.
