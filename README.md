# FraudShield

## Inspiration

The gig economy runs on trust. Millions of delivery partners and on-demand workers depend on platforms that pay them fairly — and those platforms depend on honest reporting.

While building FraudShield, we uncovered a striking reality: **GPS coordinates can be faked in under 30 seconds using free apps**, and most payout systems have zero defense against it. A coordinated fraud ring — even a small one — can drain a platform of thousands of dollars per day by simulating fake stranded deliveries, all while appearing completely legitimate to a GPS-only verification system.

Platforms introduce strict anti-fraud rules such as precise GPS verification, geo-fencing, device binding, and movement checks. However, these automated rules can wrongly flag honest workers due to poor GPS signals or network issues — while determined fraudsters still find ways to bypass them.

We asked ourselves: what if you could build a system that was smarter than the attacker — one that doesn't just check coordinates, but reconstructs the full physical truth of whether a delivery actually happened?

That question became **FraudShield**.

---

## What it does

FraudShield is a **multi-layer fraud detection engine** that intercepts every delivery and payout event on a gig platform — catching both technical spoofing attacks *and* real-world delivery fraud where the driver reaches the location but never hands over the package.

---

### 🚨 The Two Fraud Types We Solve

#### Type 1 — GPS & Location Spoofing
Driver never goes to the location at all. Uses fake GPS apps, VPNs, or emulators to simulate being there.

#### Type 2 — Ghost Delivery Fraud *(the harder problem)*
Driver physically reaches the location, marks order as delivered, but never hands the package to the customer. All location signals look completely normal. Traditional systems miss this entirely.

FraudShield solves both.

---

### 🔍 Detecting Type 1 — Location Spoofing

**Multi-Signal Location Consensus**
We cross-verify GPS against IP geolocation, WiFi/cell tower triangulation, and IMU (accelerometer/gyroscope) data. All signals must agree. If GPS says one location but IP resolves elsewhere and the device shows zero physical motion — flagged instantly.

**Movement Physics Analysis**
Fake GPS apps produce geometrically perfect but physically impossible paths. We check for:
- Position jumps > 50 km in < 30 seconds
- Speeds exceeding 250 km/h in urban zones
- Zero IMU variance during claimed travel (real vehicles vibrate)
- GPS timestamps out of sync with system clock

The Movement Authenticity Score is computed as:

$$MAS = \alpha \cdot S_{physics} + \beta \cdot S_{sensor} + \gamma \cdot S_{entropy}$$

where \\( \alpha, \beta, \gamma \\) are learned weights penalizing specific spoofing signatures.

**AI Anomaly Detection**
Two unsupervised models run in parallel:
- **Isolation Forest** — flags behavioral outliers on an 8-dimensional feature vector (request frequency, GPS variance, IP volatility, sensor activity, etc.)
- **LSTM Autoencoder** — reconstruction error spikes when a session deviates from learned legitimate behavior

Both run at inference in **< 20ms**.

**Graph Neural Network Fraud Ring Detection**
The platform is modeled as a graph — accounts, devices, IPs, and locations as nodes; shared resources as edges. A **GraphSAGE GNN** flags accounts embedded in suspicious clusters *before* they strike, purely by graph proximity to known fraud rings.

**Device Integrity Checks**
Detects emulators, rooted devices, mock location providers, and debug-mode flags via Android SafetyNet and iOS DeviceCheck.

---

### 👻 Detecting Type 2 — Ghost Delivery Fraud

This is where location-only systems fail completely. The driver is physically present. Every signal looks clean. The fraud happens in the last 10 meters.

FraudShield solves this with **Delivery Confirmation Layers** — simple, practical methods that verify the *handoff*, not just the arrival.

**OTP Confirmation (Primary)**
When the driver marks an order as delivered, the customer receives a one-time PIN on their phone. The driver must enter it in the app to complete the delivery. No OTP entry = delivery not confirmed, payout blocked. This is the single most effective check — it requires real customer interaction and cannot be faked without the customer's phone.

**Photo Proof of Delivery**
Driver must upload a photo of the package at the door before the delivery is marked complete. The photo is:
- Timestamped and GPS-tagged at capture time
- Checked to ensure it wasn't uploaded from the camera roll (must be taken live)
- Flagged if the same photo hash is reused across multiple deliveries

**Customer App Confirmation**
The customer receives a push notification: *"Has your order arrived?"* A tap confirms delivery. If the customer marks it as not received within 10 minutes of the driver's completion, the case is automatically escalated. No response after the window = soft approval, reducing false holds.

**Dwell Time Check**
When a driver marks delivery complete, we check how long they were within 50m of the delivery address. A driver who arrived, spent 8 seconds, and immediately left is flagged — a genuine handoff takes longer.

| Dwell Time at Address | Risk Signal |
|---|---|
| > 90 seconds | ✅ Normal |
| 30 – 90 seconds | 🔄 Request photo proof |
| < 30 seconds | ⚠️ Flag for review |

**Customer Complaint Signal**
If a customer reports non-delivery, that event is logged against the driver's account. A single complaint is a soft flag. Three complaints within 30 days trigger automatic account review. This creates a feedback loop that catches repeat offenders who game individual deliveries.

---

### ⚖️ Unified Risk Scoring Engine

Every signal from every layer feeds into a single fraud risk score:

| Signal | Risk Delta |
|---|---|
| GPS / IP / IMU mismatch | +30 |
| Movement physics violation | +25 |
| AI anomaly flag | +20 |
| GNN fraud ring proximity | +35 |
| Suspicious device detected | +25 |
| OTP not entered | +40 |
| Dwell time < 30 seconds | +20 |
| No photo proof submitted | +15 |
| Customer non-delivery report | +25 |
| Strong historical reputation | −20 |

| Score | Decision |
|---|---|
| 0 – 30 | ✅ Approve automatically |
| 31 – 64 | 🔄 Hold — request OTP / photo proof |
| 65 – 84 | ⚠️ Escalate to senior review |
| 85+ | 🚫 Block payout, log to fraud graph |

---

### 🛡️ Protecting Genuine Workers

- Workers with strong track records get a −20 score buffer
- Every blocked payout generates an appeal case ID
- Medium-risk holds give workers a chance to submit proof — not an instant ban
- Dwell time flags account for urban buildings where parking forces drivers to walk further

---

## How we built it

- **FastAPI (Python)** — async API orchestration, < 50ms p99 latency
- **PyTorch + PyTorch Geometric** — LSTM Autoencoder and GraphSAGE GNN
- **scikit-learn** — Isolation Forest for behavioral anomaly detection
- **Neo4j** — native graph database for the live fraud relationship graph
- **Apache Kafka** — real-time event ingestion from device telemetry
- **Redis** — sub-millisecond risk score caching and session state
- **PostgreSQL** — transaction records and full audit logs
- **React + Grafana** — live monitoring dashboard with fraud heatmap
- **Docker Compose** — local orchestration wiring all services together

We split across four areas: ML pipeline, graph engine, backend API, and frontend dashboard. ML models are hot-loaded so inference never blocks the request path.

---

## Challenges we ran into

**Latency across seven layers.** GNN traversal over Neo4j can spike on large graphs. We designed a tiered evaluation order — fast rule-based checks first, ML layers only when needed — keeping overall p99 under 100ms.

**Training without labeled fraud data.** We trained the LSTM Autoencoder exclusively on synthetic legitimate sessions and calibrated anomaly thresholds to minimize false positives on genuine workers.

**Ghost delivery detection without customer friction.** OTP confirmation is effective but adds a step for customers. We made it optional for high-reputation drivers and only mandatory when the dwell time check or photo check raises a flag — balancing security with user experience.

**Dynamic fraud graph.** Fraud rings mutate constantly. We implemented incremental neighborhood sampling so the GNN updates node embeddings without full retraining.

---

## Accomplishments that we're proud of

- **Solved both fraud types** — technical GPS spoofing *and* ghost delivery fraud in one unified system
- **End-to-end working pipeline** — all layers run in a single API call, decision in < 100ms
- **GNN flags zero-history accounts** before they act, purely by graph proximity
- **False-positive rate below 3%** on our synthetic test set
- **OTP + dwell time + photo proof** working together as a practical last-mile verification stack
- Built a research-level, production-architected system in under 24 hours as a team of four

---

## What we learned

**Location verification and delivery verification are two completely different problems.** We started building a GPS anti-spoofing system and realized halfway through that the more common real-world fraud doesn't involve GPS at all — it involves a driver who shows up, does nothing, and leaves. Solving both required fundamentally different approaches.

**Graph databases make fraud rings obvious.** Patterns that looked clean in a flat table were glaring clusters the moment we visualized them in Neo4j.

**Unsupervised ML beats supervised ML at cold start.** No labeled fraud data exists when you launch. Anomaly detection on learned normal behavior is far more deployable from day one.

**Defense-in-depth is the only real strategy.** Every individual layer can be bypassed. All layers together make the cost of attack exceed the expected payout — which is the actual security goal.

---

## What's next for FraudShield

- **Federated learning across platforms** — shared fraud models trained without exposing raw user data, using differential privacy
- **Vision model for photo proof verification** — automatically validate delivery photos instead of manual review, flagging recycled or AI-generated images
- **Fully online GNN retraining** — real-time graph model that adapts to new fraud ring patterns without manual retraining cycles
- **Customer sentiment signal** — NLP on post-delivery ratings to surface soft non-delivery complaints that customers didn't formally report
- **Regulatory compliance layer** — GDPR and India DPDP-compliant audit trail so every risk decision is explainable and exportable for legal review
