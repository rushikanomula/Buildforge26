# JivNetra Demo Script

A 3-minute live demo of JivNetra, then answers to likely judge questions.

Numbers below come from a run at 30x speed. Prediction timing depends slightly on playback speed, so small differences of a few tenths of a minute are normal.

---

## Before you start

- Open `jivnetra.html` in a full-screen browser window, at 1440 px wide or more if possible.
- Keep **Speed** at 30x and **Pause for decisions** ticked.
- Press **Replay AMB-204 scenario** to reset, then press **Pause** until you are ready to begin.

---

## Part 1: The setup (20 seconds)

**Say:**

> "This is JivNetra. On screen is AMB-204, a suspected stroke patient picked up in Miyapur, heading to Hospital A in Ameerpet. The target is 35 minutes to care, and with a 15% tolerance, the limit is 40.3 minutes. The route on the map is what a traffic-aware navigation app would pick. Right now JivNetra gives it about a 2% chance of missing the limit."

**Point at:**
- The dark route line from Miyapur to Ameerpet.
- The big risk number and "Stable."
- "Usual rush-hour bottleneck" already adding about 5 minutes. It is 17:52, the start of evening peak.

Press **Play**.

---

## Part 2: The slowdown (20 seconds)

At about 2.5 minutes, a slowdown starts on NH 65 between Moosapet and Erragadda.

**Say:**

> "A slowdown just started ahead. Our system never sees the incident itself. It only sees speeds dropping and projects how the jam will grow and spread. Watch the risk climb even though the ambulance hasn't reached that road."

**Point at:**
- Segments turning orange and red near Moosapet.
- Risk rising into "At risk."
- "Traffic building up" growing in the explanation.

---

## Part 3: The alert (30 seconds)

At about 7 minutes, Hospital A's ED reports 6 ambulances waiting. Risk jumps to about 97%, the state turns Critical, and the simulation pauses.

**Say:**

> "Now the destination ED has filled up. A navigation app would still say the route is fine. Our system says 97% chance of missing the limit, at minute 7, with 33 minutes still on the clock before the limit. And it tells you why: about 9 minutes from the ED queue, 5 from rush hour, and 3 to 4 from the slowdown."

**Point at:**
- The chart: the risk line crossing the 80% alert line, and the purple bracket showing about 33 minutes of warning.
- "What's adding time," led by the ED queue.

---

## Part 4: The recommendation (40 seconds)

**Say:**

> "It then checks every option on the same 200 simulated futures: other routes to Hospital A, and every hospital that can treat a stroke. Hospitals C and F are excluded because they have no stroke unit, even though C is closest. The recommendation is Hospital B via Madhapur. It has a stroke unit and no queue, about 37 minutes to care instead of 48, and risk drops from 97% to 5%."

**Point at:**
- The purple dotted suggested route on the map.
- The recommendation card.
- Open **Compare all options** to show the full table and the "Not considered" line.

**Say:**

> "The system doesn't act on its own. The dispatcher decides."

Press **Approve and send to crew**, then press **Play**. Switch speed to 60x or 120x to finish faster.

---

## Part 5: The outcome (30 seconds)

When the patient is handed over at Hospital B, a green box appears.

**Say:**

> "Patient in care at about 36 minutes, inside the limit. The system then replays the exact same traffic without our change: about 53 minutes, a missed target. That's roughly 17 minutes of treatment window saved, with a warning 33 minutes before the limit would have passed."

---

## Part 6: The evidence (40 seconds)

Switch to **Replay evaluation** and press **Run replay**. It takes a few seconds.

**Say:**

> "One scenario is a story. This is the evidence. We ran 60 random missions, each twice with identical traffic: once with no action, once approving our recommendation at the alert. We caught 12 of 17 failures before they happened, a median of 26.5 minutes early, and 86% of our alerts were real. Where a better option existed, approving it saved 17 minutes on average and prevented 3 failures. In the other 8 caught failures, no route or eligible hospital was better. The system says so rather than suggesting a bad move."

**Point at:**
- The threshold table: lower thresholds catch more failures but raise more false alerts.
- The calibration chart: dots near the diagonal mean the percentages are honest.

**Close with the caveat, before a judge raises it:**

> "These are simulated missions and the predictor shares assumptions with the simulator, so this proves the pipeline works end to end, not real-world accuracy. The next step is running the same replay on recorded 108 trips."

---

## Optional extras if time allows

- **Road closure:** switch "Clicking a road" to "closes it," then click a road ahead of the ambulance. A closure marker appears, "Road closure" shows up in the explanation, and options avoid it.
- **Stale hospital data:** press **Cut destination's status feed.** Confidence drops and the confidence note says "hospital feed stale." The system treats the hospital as unknown, not normal.
- **Changing clinical priority:** open Mission settings and change the patient type. Eligible hospitals and targets update immediately.
- **Alert fatigue:** press **Dismiss for 3 min.** The recommendation stays quiet unless risk rises another 10 points.

---

## Likely judge questions

**Google Maps already uses live traffic. What's new?**
Yes, and we don't compete with it. Maps answers "what's the fastest route now." We answer "will this mission miss its clinical target, why, and what should we do." That needs the ED queue, the patient's required unit, the time budget and uncertainty. None of those are in a navigation app.

**Where is the GNN?**
The forecaster in the prototype is a rule-based diffusion model that projects congestion spread. It sits in one function, `forecastC`, which is exactly where the trained GNN + LSTM plugs in. We built the pipeline around it first so the model can be evaluated against baselines once trained on real trips.

**Isn't your evaluation circular?**
Partly, and we say so. The predictor and simulator share traffic assumptions. The predictor does not see hidden incidents, only speeds, trends and feeds, so it still has to infer what is happening. The evaluation method is what transfers: replaying real trips as if live and measuring warning time.

**How do you get cause labels for training?**
Counterfactual re-runs, not hand-labelling. For any trip we re-compute the expected time with each factor removed. The minutes that disappear become the label, the same method the "What's adding time" panel uses.

**Won't dispatchers ignore constant alerts?**
Four safeguards:
- **Minimum benefit:** the system only recommends a change that saves at least 3 minutes and lowers risk.
- **Severity weighting:** the alert level is scaled by severity, so a routine transfer at 80% risk stays on the dashboard while a stroke at 80% escalates.
- **Snooze:** dismissed recommendations stay quiet for 3 minutes.
- **Tunable threshold:** the replay's threshold table lets an EMS operator pick the trade-off.

**Who is liable if a reroute goes wrong?**
The system never acts on its own. It recommends with the expected benefit and uncertainty, and a human approves. Hospital options are filtered by clinical capability first, and in production medical-control rules would also apply.

**Do Indian hospitals expose live bed or queue data?**
Mostly not today. The design treats hospital data as optional. It uses FHIR where available and custom connectors elsewhere, and a missing or stale feed is treated as unknown with wider uncertainty, which the demo shows. Even without hospital data, the traffic side still works.

**Why is prevention only 3 of 12?**
Because often there genuinely is no better option: the ambulance is too far along, or every eligible hospital is further away. The system reports "no better option" instead of pushing a change. The 12 early warnings are still valuable because the destination ED can prepare before the patient arrives.

**What would you build next?**
1. Load a real Hyderabad OpenStreetMap extract into OSRM.
2. Run the replay on historical 108 trips.
3. Train and compare the GNN + LSTM against median-ETA, XGBoost and LSTM baselines.
4. Pilot as a read-only dashboard in one control room before enabling recommendations.
