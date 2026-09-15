# JivNetra Pitch Deck Content

Content for the 8-slide JivNetra deck.

This version differs from the `.pptx` in two places so that the slides match the working prototype:

- **Slide 6:** the walkthrough uses the prototype's AMB-204 numbers. The original example's recommended option still missed its 20-minute target.
- **Slide 8:** adds the replay results.

---

## Slide 1: Title

**JivNetra**

**Predict Ambulance Arrival Failure Before It Happens**

A predictive EMS safety layer that forecasts a missed arrival target, explains why, and recommends an intervention while there is still time to act.

- **Theme / track:** Healthcare / Mobility / AI
- **Team name:** *Your Team Name*
- **Team members:** Pramod Sai Tavva, *Member 2*, *Member 3*, *Member 4*

**Visual:** AMB-204 mission card showing risk rising from Stable 2% to Critical 97%.

---

## Slide 2: The Problem / Opportunity

**Delay builds silently across the mission**

An ambulance trip is a chain of phases, not a single route. Average phase times from Maharashtra 108 EMS data:

| To scene | At scene | To hospital | Handover |
|---|---|---|---|
| 23 min | 12 min | 39 min | 11 min |

- **Benchmarks:** NHM quality benchmarks aim to reach the patient within 20 min (urban) or 30 min (rural), then a health facility within the next 30 min.
- **Episode time:** the average emergency episode is **134.5 min**.
- **Scene to hospital:** travel averages **39 min** against a 30-minute benchmark.

`T_total = T_dispatch + T_scene + T_route + T_handover`

Navigation apps optimise only `T_route`. Nobody watches whether the whole mission is drifting toward a missed target.

**The opportunity:** flag the failure while it is still preventable, minutes before the ETA crosses the target.

*Source: analysis of Maharashtra 108 EMS data (PubMed Central); NHM benchmarks as reported in that study.*

---

## Slide 3: Target Users & Why It Matters

**Built for the people racing the clock**

| User | Need |
|---|---|
| EMS dispatchers and 108 command centres (primary) | Early warning on at-risk missions, with an explained, ranked action |
| Ambulance crews | Approved reroute or destination changes delivered to the vehicle |
| Emergency department teams | Pre-alerts and queue visibility so a bay is ready on arrival |
| Traffic and municipal authorities | Corridor alerts through existing, authorised traffic-control systems |
| EMS administrators and health departments | Response-time compliance and recurring-bottleneck analytics |

**Why it matters:**

- **Stroke:** in a study of **14,132** stroke thrombolysis patients, every extra minute of door-to-needle delay was linked to lower survival odds and worse function.
- **Handover:** **17.5%** of 141,000+ ambulance transports had handover delays of 30 minutes or more: 12.5% at 30 to 60 minutes and 5% at 60 or more.

Cutting avoidable transport and handover delay preserves the treatment window.

*Figures show association, not a fixed per-minute effect.*

---

## Slide 4: Existing Solutions / Current Approach

**Today's tools answer a different question**

| Tool | Asks | Does well | Stops short |
|---|---|---|---|
| Consumer maps (Google Maps, Apple Maps) | What's the fastest route right now? | Live, traffic-aware routing; crashes, closures, road work | No mission target, ED queue or clinical priority |
| Open-source routing (OpenStreetMap + OSRM) | What routes exist from A to B? | Routes, alternatives, travel-time matrices, GPS map matching | Static road speeds; a routing engine, not a predictor |
| Fleet tracking dashboards | Where is the ambulance now? | Live vehicle position for the control room | Lateness is visible only after the ETA slips |
| ETA prediction models | When will it arrive? | A travel-time estimate | No cause, no target check, no recommended action |

**The gap:** maps already use live traffic. What's missing is a system that asks whether this mission will miss its target, why, and what to do now.

---

## Slide 5: Proposed Solution

**A predictive safety layer above routing**

| Layer | Question | What it does |
|---|---|---|
| 1. Predict | Will this mission fail its target? | Failure probability and expected time to care from 200 simulated futures |
| 2. Explain | Why? | Minutes added by traffic, closure, ED congestion, route choice, rain, rush-hour bottleneck |
| 3. Counterfactual | What would prevent it? | Scores alternative routes and clinically eligible hospitals on the same futures |
| 4. Act | What should happen now? | Recommends the best option; a dispatcher approves before execution |

**Foundation:** OpenStreetMap + OSRM, or any maps API, supply candidate routes.

**When is a mission in trouble?** Two separate thresholds:

- **Failure:** a mission fails if `time to care > target × (1 + δ)`. The prototype uses δ = 15%.
- **Alert:** the system alerts when `P(failure | live state) ≥ θ`. The default θ is 80%.

| State | Risk | Action |
|---|---|---|
| Stable | below 50% | Continue |
| At risk | 50% to 80% | Monitor and prepare |
| Critical | 80% and above | Start intervention |

Targets are configurable: urban or rural, ALS or BLS, stroke, cardiac, trauma, and by state.

---

## Slide 6: How It Works / User Flow

**From live signal to approved action**

1. **Mission starts.** The dispatcher assigns an ambulance, and the target is set by patient type and area.
2. **Live state.** Map-matched GPS, traffic trends, weather and ED status stream in.
3. **Forecast risk.** The forecaster outputs failure probability and expected time to care.
4. **Explain.** Counterfactual attribution shows the main causes.
5. **Recommend.** What-if routes and eligible hospitals are ranked by time saved.
6. **Approve and act.** The dispatcher approves: reroute, pre-alert the ED, notify traffic control. The loop repeats.

**Walkthrough: AMB-204, stroke patient, target 35 min, limit 40.3 min**

| When | Risk | What happened |
|---|---|---|
| Pickup | 2%, Stable | Fastest traffic-aware route to Hospital A |
| +2.5 min | Rising to At risk | Slowdown building on NH 65 near Moosapet |
| +7 min | 97%, Critical | Hospital A ED reports 6 ambulances waiting; expected time to care 48 min |

**Recommendation:**

- **The switch:** go to stroke-capable Hospital B via Madhapur, about 37 min instead of 48, with risk falling from 97% to 5%.
- **Not considered:** Hospitals C and F, which have no stroke unit.
- **Result:** approved, and the patient was in care at about 36 min. The same traffic without the change gave about 53 min, so roughly **17 minutes were saved** with **33 minutes of warning**.

---

## Slide 7: What Makes Our Solution Different

**Not another navigation app**

- **Routing asks:** Where should I go?
- **ETA models ask:** When will I arrive?

**JivNetra asks:**

1. Will this mission miss its target?
2. Why?
3. How many minutes will it lose?
4. Can another route or hospital prevent it?
5. What should happen now?

| Capability | Consumer maps | OSRM | ETA models | JivNetra |
|---|---|---|---|---|
| Live traffic awareness | Yes | No | Partial | Yes |
| Predicts target failure in advance | No | No | Partial | Yes |
| ED queue and handover in the estimate | No | No | No | Yes |
| Explains the cause (multi-label) | No | No | No | Yes |
| What-if route alternatives | Partial | Partial | No | Yes |
| Clinically constrained hospital choice | No | No | No | Yes |

---

## Slide 8: Basic Implementation Approach

**Hackathon MVP: open data, proven tools**

| Stage | Components |
|---|---|
| Data | OpenStreetMap road graph, historical or simulated ambulance traces, weather API, synthetic ED congestion, event injection |
| Ingest | MQTT from vehicle to Kafka; GPS smoothing and OSRM map matching |
| Store | PostGIS for roads, hospitals and trips; TimescaleDB for telemetry; Redis for live state |
| Model | GNN + LSTM fusion in PyTorch Geometric, with XGBoost baseline |
| Decide and show | Route scoring, hospital eligibility, human approval; FastAPI + React/Leaflet dashboard |

**One model, three outputs:**

| Output | Loss |
|---|---|
| Failure probability | Focal loss |
| Minutes late (ΔETA) | Huber loss |
| 6-factor attribution | Multi-label binary cross-entropy |

**How we prove it works:**

- **Method:** historical replay, running past trips step by step as if live.
- **Baselines:** median ETA, XGBoost and LSTM, compared against GNN + LSTM.
- **Metrics:** PR-AUC, recall, F1, Brier score, ETA MAE and P90 error.
- **Headline metric:** lead time, the minutes of warning before the real failure.

**Prototype replay result (60 simulated missions, alert at 80%):**

- **Detection:** caught 12 of 17 failures, a median of 26.5 minutes before the limit.
- **Precision:** 86% of alerts were real.
- **Impact:** where a better option existed, the recommendation saved 17.1 minutes on average and prevented 3 failures.

*Simulated missions; the predictor shares assumptions with the simulator, so this validates the pipeline, not real-world accuracy.*
