# JivNetra Evaluation

How the JivNetra prototype measures whether it catches failures early and whether its recommendations help.

---

## 1. Method: paired historical replay

Each mission is simulated **twice** with the same seed, so traffic, incidents, rain and ED surges are identical in both runs.

1. **No-action run.** The ambulance follows its original plan. Every 30 simulated seconds the predictor runs as if live and records its failure probability. This run gives the true outcome and the time of the first alert.
2. **Action run.** Only for missions that raised an alert. Identical, except that when risk reaches the alert threshold and a recommendation exists, it is approved automatically. At most 2 approvals are made, at least 3 minutes apart.

Because the world's random stream does not depend on the ambulance's route, any difference between the two runs is caused only by the approved action.

### Mission generator

Each random mission draws the following:

| Item | How it is drawn |
|---|---|
| Patient type | Stroke 35%, cardiac 30%, trauma 20%, routine transfer 15% |
| Pickup | Random locality |
| Start time | Uniform between 17:00 and 20:00 |
| Normal ED queue | 0 to 2 ambulances per hospital |
| Destination | Fastest eligible hospital by current traffic, as a navigation-only dispatcher would choose |
| Target | Higher of the patient-type default and the undisturbed expected time × 1.05 to 1.20, rounded up to 5 min |
| Tolerance | 15% |

Patient-type defaults are stroke 30, cardiac 30, trauma 35 and routine 60 minutes. Events are drawn independently:

| Event | Probability | Details |
|---|---|---|
| Traffic incident on the route | 55% | Peak 0.60 to 0.85 congestion, lasts 15 to 40 min |
| ED surge at destination | 35% | +3 to +6 ambulances |
| Rain | 30% | 6 to 30 mm/h |
| Road closure on the route | 12% | 30 min |
| Destination status feed outage | 10% | Until mission end |

---

## 2. Metrics

| Metric | Definition |
|---|---|
| **Failure** | Time to care exceeds the limit (target × 1.15) |
| **Caught (true positive)** | A failure whose first alert came **before** the limit passed |
| **Missed (false negative)** | A failure with no alert, or an alert only after the limit passed |
| **False alert** | An alert on a mission that did not fail |
| **Recall** | Caught ÷ failures |
| **Precision** | Caught ÷ (caught + false alerts) |
| **Warning time (lead time)** | Limit − time of first alert, for caught failures |
| **Prevented** | A caught failure that met the limit in the action run |
| **Minutes saved** | No-action time to care − action time to care, where an action was approved |
| **Brier score** | Mean of (predicted probability − actual outcome)² over every prediction |
| **Calibration** | For predictions grouped in 10% bands, average predicted risk vs observed failure rate |

---

## 3. Results: default set

Settings: mission set 1000, 60 missions, alert threshold 80%.

The set contains 24 stroke, 15 cardiac, 11 trauma and 10 routine missions. 17 of them failed without action.

### Headline

**Caught 12 of 17 failures before they happened, a median of 26.5 minutes early.**

| Metric | Value |
|---|---|
| Recall | 71% (12 of 17) |
| Precision | 86% (12 caught vs 2 false alerts) |
| False alerts | 2 in 60 missions |
| Median warning time | 26.5 min |
| Caught failures where a better option existed | 4 of 12 |
| Failures prevented by approving the recommendation | 3 of 12 |
| Average minutes saved when an action was approved | 17.1 min |
| Actions approved on false alerts | 1, which did not cause a failure |
| Brier score | 0.149 |
| Brier score of always predicting the base rate | 0.203 |

### Choosing the alert threshold

| Alert at | Failures caught | Alerts that were real | False alerts | Median warning |
|---|---|---|---|---|
| 50% | 16 of 17 | 76% | 5 | 28.0 min |
| 60% | 15 of 17 | 75% | 5 | 27.5 min |
| 70% | 14 of 17 | 78% | 4 | 27.5 min |
| **80%** | **12 of 17** | **86%** | **2** | **26.5 min** |
| 90% | 10 of 17 | 91% | 1 | 26.5 min |

Lowering the threshold catches more failures and warns slightly earlier, at the cost of more false alerts. The 80% default favours fewer false alerts, which matters for dispatcher trust.

### Missions where an action was approved on a caught failure

| Mission | No action | With action | Result |
|---|---|---|---|
| AMB-718 | 38.8 min | 31.4 min | Prevented |
| AMB-723 | 61.6 min | 21.2 min | Prevented |
| AMB-744 | 46.5 min | 34.6 min | Prevented |
| AMB-745 | 51.2 min | 42.7 min | Still missed, 8.5 min better |

In the other 8 caught failures, no alternative route or eligible hospital saved 3 or more minutes while lowering risk, so nothing was recommended.

### Scripted scenario AMB-204

| Item | Value |
|---|---|
| Target / limit | 35 min / 40.3 min |
| First alert | ~7.2 min after pickup, risk ~97% |
| Warning before limit | ~33 min |
| Recommendation | Hospital B via Madhapur, ~37 min expected vs ~48 min |
| Time to care with approval | ~36 min, limit met |
| Time to care without approval | ~53 min, limit missed |

---

## 4. Limitations

- **Shared assumptions.** The predictor's forecaster uses the same congestion model family as the simulator, which flatters accuracy. The predictor does not see incident sources, durations or future events, but real traffic will be less predictable. Treat these numbers as proof that the pipeline works end to end, not as expected real-world performance.
- **Small sample.** 60 missions and 17 failures give wide uncertainty. Use 100 missions and several mission sets before quoting rates.
- **Schematic network.** 21 localities and 31 segments limit how many alternative routes exist. That partly explains the low prevention rate.
- **Simplified ED model.** Queue size, a fixed per-ambulance offload time and linear draining. Real handover delays depend on staffing, bed availability and triage.
- **Late failures.** Some failures only become predictable near the limit, for example a mission that ends 0.3 minutes over. Those count as missed.
- **Automatic approval.** The action run approves every recommendation instantly. Real dispatchers take time and sometimes decline.

---

## 5. Reproducing

**In the browser:**
1. Open the prototype and go to **Replay evaluation**.
2. Choose 60 missions and an 80% threshold.
3. Press **Run replay**. Mission set 1000 is the default and fully reproducible.

**In Node.js:** save the engine `<script>` block as `engine.js`, then run:

```js
const E = require("./engine.js");
const theta = 0.8;
const results = [];
for (let i = 0; i < 60; i++) {
  const spec = E.generateMission(1000 + i);
  const noAction = E.runHeadless(spec, theta, "none");
  let action = null;
  if (noAction.firstAlert !== null) action = E.runHeadless(spec, theta, "act");
  results.push({ spec, noAction, action });
}
```

Each result contains `ttc` (time to care), `limit`, `failed`, `firstAlert` (minutes, or null), `hist` (a list of [minute, probability] pairs) and `actions`.

---

## 6. Evaluation on real data (next step)

1. Replace the simulator with recorded trips: GPS traces, dispatch, pickup, arrival and handover timestamps.
2. Replay each trip minute by minute, giving the model only data available at that moment.
3. Report the same metrics, plus ETA error (MAE and 90th-percentile error), against baselines: historical median ETA, XGBoost, LSTM, and GNN + LSTM.
4. Estimate the benefit of each intervention with counterfactual routing on the recorded traffic, and validate it in a read-only pilot before enabling recommendations.
