# JivNetra Architecture

This document describes how the JivNetra prototype works and how each part maps to the production design.

---

## 1. Layers

```
                 Live state
   road speeds + trends, closures, rain, hospital queues, mission
                      │
                      ▼
   ┌──────────────────────────────────────────┐
   │ 1. PREDICT   Monte Carlo time-to-care    │ → P(fail), expected time, range, confidence
   └──────────────────────────────────────────┘
                      │
                      ▼
   ┌──────────────────────────────────────────┐
   │ 2. EXPLAIN   one-factor-removed re-runs  │ → minutes added per factor
   └──────────────────────────────────────────┘
                      │
                      ▼
   ┌──────────────────────────────────────────┐
   │ 3. COUNTERFACTUAL  routes + eligible     │ → options scored on the same futures
   │    hospitals                             │
   └──────────────────────────────────────────┘
                      │
                      ▼
   ┌──────────────────────────────────────────┐
   │ 4. ACT       recommendation rules        │ → dispatcher approves or dismisses
   └──────────────────────────────────────────┘
                      │
                      ▼
               new state → loop
```

In the live view, the loop re-runs every 10 simulated seconds.

---

## 2. Key definitions

**Time to care.** Minutes from pickup until the patient is handed over to the ED:

```
time_to_care = drive_time + ED_handover_wait
```

**Limit.** The target plus a tolerance:

```
limit = target × (1 + δ)        default δ = 15%
```

**Failure.** The mission fails if `time_to_care > limit`.

**Alert.** An alert is raised when `P(time_to_care > limit | current state) ≥ θ`, with a default θ of 80%.

The two thresholds do different jobs: δ defines what counts as a failure, and θ defines how sure the system must be before it alerts.

**Risk states:**

| State | Risk |
|---|---|
| Stable | below 50% |
| At risk | 50% up to θ |
| Critical | θ and above |

---

## 3. Engine components

The engine is plain JavaScript with no DOM dependencies.

| Component | Main functions | Role |
|---|---|---|
| Road network | `NODES`, `EDGES`, `dijkstra`, `kPaths` | 21 localities and 31 road segments, shortest and alternative paths |
| World simulator | `createWorld`, `stepWorld`, `applyEvent` | Ground truth: congestion, incidents, closures, rain, ED queues |
| Ambulance | `createAmbulance`, `moveAmbulance` | Drives the route, detours around closures it encounters, waits for handover |
| Observation | `observe` | What the predictor is allowed to see |
| Predictor | `forecastC`, `makeSamples`, `predict` | Congestion forecast and Monte Carlo time to care |
| Attribution | `attribute` | Counterfactual minutes per factor |
| Recommender | `recommend`, `eligibility`, `applyOption` | Options, clinical filter, scoring, recommendation |
| Missions | `scenarioAMB204`, `generateMission` | Scripted and random missions with events |
| Replay | `runHeadless` | Fast simulation for evaluation |

---

## 4. World simulator (ground truth)

### Road network

Each segment has a free-flow speed, an evening peak multiplier, a rain sensitivity and a length. Length is the straight-line distance times 1.3 to account for winding roads.

### Usual rush-hour congestion

The evening peak is centred at 18:45 with a spread of about 55 minutes:

```
m(t) = 1 + (peak − 1) × exp(−½ × ((t − 18:45) / 3300 s)²)
historical_congestion = clamp((1 − 1/m) / 0.8, 0, 0.9)
```

Congestion `c` runs from 0 (free flow) to 0.95 (jammed). The mapping is chosen so that congestion of `(1 − 1/m)/0.8` produces travel-time multiplier `m`.

### Speeds

```
weather_factor = 1 − min(0.2, rain/25 × 0.15)
traffic_speed  = free_flow × max(0.10, 1 − 0.8c) × weather_factor
ambulance_speed = free_flow × max(0.12, 1 − 0.8c) × weather_factor × 1.15
```

The 1.15 multiplier approximates the siren advantage.

### Congestion dynamics

On every step, each segment moves toward a target:

```
target = historical + rain_add + incident + spill
rain_add = min(0.35, rain/25 × 0.25 × rain_sensitivity)
spill    = 0.55 × max(0, highest neighbour congestion − 0.45)
c ← c + (target − c) × (1 − exp(−dt/τ)) + small noise
```

τ is 140 s when congestion is rising and 420 s when it is clearing. Spill is how a jam propagates to connected roads.

### Incidents

Incidents ramp up over 150 s, hold at their peak for their duration, then clear. **The predictor never sees incidents directly.**

### ED queues

- **Handover wait:** 4 min plus 3.5 min per waiting ambulance, with log-normal noise.
- **Queue drain:** a surge drains by one ambulance every 10 minutes.
- **Feed outage:** if a hospital's feed is cut, observers only see the last reported queue and its age.

### Determinism

The world uses its own seeded random stream, one draw per segment per step. Two runs with the same seed and events therefore have identical traffic even if the ambulance takes different routes. The replay evaluation relies on this.

---

## 5. Predictor

### What it observes

- Current congestion on every segment.
- Congestion trend over the last ~60 seconds.
- Known closures and current rainfall, from event feeds.
- Each hospital's queue, or the last known queue plus data age if the feed is down.

### Congestion forecast at horizon τ

This is `forecastC`, the GNN + LSTM slot.

```
anomaly  = c_now − historical_now − rain_add          (unexplained slowdown)
growth   = trend × min(τ, 300) × 0.5                  (only if rising)
add_spill = projected rise in neighbour spill × (1 − exp(−τ/200))
decay    = exp(−τ/2400) if the slowdown is not still rising, else 1

forecast = historical(t + τ) + rain_add + anomaly × decay + growth + add_spill
```

For each sampled future the forecaster adds:

- **Citywide shift:** ~ N(0, 0.03).
- **Per-segment noise:** with σ = 0.035 + 0.07 × min(1, τ/1500), so uncertainty grows with horizon.
- **Unforeseen incident:** each segment has a 0.006 × length_km chance of one, adding 0.2 to 0.5 congestion that ramps in over 10 minutes.

### Time to care for one sampled future

The remaining route is walked segment by segment. The congestion used for each segment is the forecast at the moment the ambulance is expected to reach it. Then the ED wait is added:

- **Feed current:** expected queue at arrival = max(baseline, current − drive_minutes/10), noise ±0.8 ambulances, log-normal σ 0.25.
- **Feed stale:** the last known queue decays toward baseline, noise ±2 ambulances, log-normal σ 0.6.

### Outputs

The predictor runs 200 sampled futures in live mode and 120 in replay.

- **P(fail):** the share of futures where time to care exceeds the limit.
- **Expected time and range:** the mean, 10th percentile and 90th percentile.
- **Confidence:** `clamp(1 − (P90 − P10)/limit − 0.15 if hospital feed is stale, 0.3, 0.97)`.

### Closures

If a known closure lies on the route, the plan detours at the node just before the closure, using forecast travel times.

---

## 6. Attribution

The expected trip is computed once with all noise removed. It is then recomputed with one factor removed at a time. The minutes that disappear are that factor's contribution.

| Factor | What is removed |
|---|---|
| Traffic building up | The unexplained slowdown, its growth and projected spill |
| ED queue at destination | Queue reset to the hospital's normal level |
| Usual rush-hour bottleneck | Time-of-day peak profile |
| Route choice | Replaced by the best alternative route to the same hospital |
| Road closure | Closures removed, original route restored |
| Rain | Rain set to zero |

Contributions are multi-label and can overlap, so they are not expected to sum to the total delay.

In the UI, bars are coloured by size: 4 min or more is high, 1.5 to 4 min is medium, and less is low.

---

## 7. Recommender

### Where rerouting can start

If the ambulance is exactly at a junction, rerouting starts there. Otherwise it starts at the end of the current segment, since a U-turn mid-segment is not allowed.

### Options

1. **Keep the current plan.**
2. **Up to 3 alternative routes to the same hospital.** These come from repeated shortest-path searches that penalise already-used segments by ×1.8.
3. **The fastest route to every other eligible hospital.**

### Clinical eligibility

A hospital is only considered if it has the unit the patient needs: a stroke unit, a cardiac cath lab, or a trauma centre. Routine transfers can go anywhere. Ineligible hospitals are shown with the reason.

### Scoring

Every option is scored on the **same** sampled futures, so the comparison is fair:

```
score = expected_time + 6 × P(fail) + 0.15 × (P90 − P10)
```

### Recommendation rule

The best-scoring option is recommended only if both of these hold:

- it saves at least 3 minutes of expected time, and
- it lowers P(fail) by at least 5 points.

### When the recommendation is shown

- The mission is Critical, or
- The mission is At risk and the option cuts risk by more than 30 points.

### Alert fatigue controls

- **Dismissal:** a dismissed recommendation stays hidden for 3 minutes, unless risk rises by 10 or more points.
- **Alert level:**

```
alert_score = P(fail) × severity × preventability
severity:       stroke 1.0, cardiac 1.0, trauma 0.9, routine 0.4
preventability: 1.0 if a recommendation exists, else 0.5
```

A score of 0.45 or more is escalated to the dispatcher. Anything lower is shown on the dashboard only.

### Human in the loop

Nothing is executed without approval. Approving a hospital switch also logs a pre-alert to the new ED. The signal-priority button only logs a request and has no effect on traffic.

---

## 8. Live loop vs replay

| | Live view | Replay evaluation |
|---|---|---|
| World step | 1 s | 5 s |
| Prediction interval | every 10 simulated s | every 30 simulated s |
| Futures per prediction | 200 | 120 |
| Actions | Dispatcher approves in the UI | Policy approves at first alert, max 2, 3 min apart |

---

## 9. Mapping to the production architecture

| Prototype | Production |
|---|---|
| Schematic network, `dijkstra`, `kPaths` | OpenStreetMap + OSRM routes, alternatives and matrices |
| Simulated congestion | Traffic probe data, map-matched ambulance GPS |
| `observe()` trend | Stream processing (Kafka / Flink) feature windows |
| `forecastC` | Spatio-temporal GNN + LSTM (or Temporal Fusion Transformer) |
| Monte Carlo sampling | Ensemble, MC dropout or quantile heads, calibrated |
| Simulated ED queues | FHIR Location or custom hospital connectors with data age |
| Event list | Municipal closures, weather API, incident feeds with confidence |
| In-memory state | Redis for live state, PostGIS + TimescaleDB for history |
| Activity log | Durable audit log |

---

## 10. Swapping in a trained model

Replace `forecastC(obs, edge, tau, sample, counterfactual)` with a function that returns predicted congestion, or speed converted to congestion, for a segment at a horizon.

Keep three behaviours so that attribution and recommendation still work:

1. **Counterfactual flags.** Honour `noAnomaly`, `noRain` and `noBottleneck` by zeroing the matching inputs.
2. **Deterministic mode.** When `sample` is null, return the point forecast.
3. **Sampled mode.** When a sample is given, return one draw from the predictive distribution.
