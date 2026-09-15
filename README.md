# JivNetra

**Ambulance arrival failure early warning.**

JivNetra is a predictive safety layer for emergency medical services. It watches a live ambulance mission and answers four questions before the ambulance is actually late:

1. **Predict:** will this mission miss its time-to-care target?
2. **Explain:** what is adding the time?
3. **Counterfactual:** would another route or hospital prevent the miss?
4. **Act:** what should the dispatcher approve right now?

It does not replace navigation. Routing engines such as Google Maps or OSRM already handle live traffic. This layer sits above them and reasons about the whole mission: traffic that is still building, closures, rain, the destination ED's ambulance queue, the patient's clinical needs, and the time left in the budget.

Track: Healthcare / Mobility / AI

---

## Run it

The prototype is one self-contained file: `jivnetra.html`.

1. Download the file.
2. Open it in Chrome, Edge, Firefox or Safari.

There is nothing to install and no internet connection is needed.

---

## What you can do in the prototype

### Live mission tab

- **Replay AMB-204 scenario.** A scripted stroke mission from Miyapur to Hospital A. A slowdown builds on NH 65, the ED at Hospital A fills up, the system raises an alert and recommends a clinically eligible alternative. See [DEMO_SCRIPT.md](DEMO_SCRIPT.md).
- **New random mission.** Generates a new mission with random patient type, pickup, start time and events.
- **Click a road** to add a traffic jam or close it. Click a closed road to reopen it.
- **Start heavy rain, add an ED surge, or cut the destination's status feed** to see how prediction, explanation and confidence respond.
- **Mission settings** to change patient type, target time, tolerance and alert threshold mid-mission.
- **Pause for decisions** stops the clock when a critical alert comes with a recommendation, so you can approve or dismiss it.

The side panel shows:

- **Risk:** probability of missing the limit, with a Stable / At risk / Critical state.
- **Expected time to care:** with a likely range and a forecast confidence score.
- **Recommended action:** the suggested change, the minutes it saves, and a comparison of every option. Ineligible hospitals are listed with the reason.
- **Risk over the trip:** a chart with the alert threshold, the limit, event markers, and the minutes of warning before the limit.
- **What's adding time:** minutes contributed by traffic build-up, ED queue, rush-hour bottleneck, route choice, closure and rain.
- **Activity:** a timestamped log of world events, system alerts and dispatcher actions.

When a mission ends after an approved change, the prototype replays the same traffic without that change and reports the minutes saved.

### Replay evaluation tab

Runs 30, 60 or 100 simulated missions twice each with identical traffic and events: once with no action, and once approving the recommendation at the first alert. It reports detection rate, precision, warning time, failures prevented, a threshold trade-off table and a calibration chart. See [EVALUATION.md](EVALUATION.md).

### How it works tab

A short explanation of how JivNetra works, for judges and first-time viewers.

---

## What is real and what is simulated

| Part | Status |
|---|---|
| Failure probability (Monte Carlo over 200 futures) | Real computation |
| Counterfactual attribution of delay | Real computation |
| Route alternatives and hospital eligibility filtering | Real computation |
| Recommendation rules and dispatcher approval flow | Real computation |
| Replay evaluation method and metrics | Real computation |
| Road network | Schematic of real Hyderabad localities, approximate distances |
| Traffic, incidents, rain effects | Simulated |
| ED queues and hospital feeds | Simulated |
| Hospitals A to F | Fictional |
| Traffic forecaster | Rule-based diffusion forecaster; the slot where a trained GNN + LSTM plugs in |

The simulator hides incident sources from the predictor. The predictor only sees road speeds, their recent trend, event feeds and hospital feeds, so it has to infer that a slowdown is building.

---

## Documentation

| File | Contents |
|---|---|
| [ARCHITECTURE.md](ARCHITECTURE.md) | System design, engine components, formulas and parameters, how to swap in real components |
| [DEMO_SCRIPT.md](DEMO_SCRIPT.md) | A 3-minute live demo walkthrough and answers to likely judge questions |
| [EVALUATION.md](EVALUATION.md) | Replay method, metric definitions, default results and limitations |
| [PITCH_DECK.md](PITCH_DECK.md) | Content for the 8-slide pitch deck |

---

## Code layout

Everything lives in the single HTML file, in three parts:

1. **Styles** in the `<style>` block.
2. **Engine** in the first `<script>` block. This covers the road network, world simulator, predictor, attribution, recommender, mission generator and headless replay. It has no DOM dependencies and also runs in Node.js through `module.exports`.
3. **App** in the second `<script>` block. This covers map rendering, the side panel, controls, the animation loop and the replay tab.

---

## Path to a real system

1. Replace the schematic network with an OpenStreetMap extract served by OSRM, and use OSRM map matching on GPS traces.
2. Replace the simulator with historical 108 trips: dispatch, pickup, arrival and handover timestamps plus GPS traces.
3. Train the GNN + LSTM forecaster on those trips and plug it into the `forecastC` slot.
4. Connect hospital feeds through FHIR where available and custom connectors elsewhere, keeping data age and confidence on every value.
5. Re-run the replay evaluation on held-out real trips to get real detection and warning-time numbers.
