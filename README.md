# FraudShield

## Inspiration

The gig economy runs on trust. Millions of delivery partners and on-demand workers depend on platforms that pay them fairly — and those platforms depend on honest reporting.

While building FraudShield, we uncovered a striking reality: **GPS coordinates can be faked in under 30 seconds using free apps**, and most payout systems have zero defense against it. A coordinated fraud ring — even a small one — can drain a platform of thousands of dollars per day by simulating fake stranded deliveries, all while appearing completely legitimate to a GPS-only verification system.

That hit us hard. The workers who are genuinely stranded get caught in blanket crackdowns. The actual fraudsters keep exploiting the system. Nobody wins except the attackers.

We asked ourselves: what if you could build a system that was *smarter* than the attacker — one that doesn't just check coordinates, but reconstructs the full physical truth of whether someone was really where they claimed to be?

That question became **FraudShield**.

---

## What it does

FraudShield is a **7-layer adversarial defense engine** that intercepts every payout request on a gig platform and runs it through cross-signal verification, physics-based movement analysis, AI anomaly detection, and graph-based fraud ring identification — all in real time.

**Layer 1 — Multi-Signal Location Consensus**
We don't trust GPS alone. Every request is cross-verified against IP geolocation, WiFi/cell tower triangulation, and inertial sensor (IMU) data. All signals must agree. If GPS says one city but the IP resolves elsewhere and WiFi triangulates to a third location — it's flagged instantly.

**Layer 2 — Movement Physics Analysis**
Fake GPS apps produce geometrically perfect but physically impossible trajectories. We analyze every movement sequence for speed violations, path entropy, and IMU continuity. The Movement Authenticity Score is computed as:

$$MAS = \alpha \cdot S_{physics} + \beta \cdot S_{sensor} + \gamma \cdot S_{entropy}$$

where ( \alpha, \beta, \gamma \\) are learned weights and each sub-score penalizes specific spoofing signatures. A real moving vehicle vibrates. A fake one doesn't.

**Layer 3 — AI Anomaly Detection**
Two unsupervised models run in parallel on an 8-dimensional behavioral feature vector:
- **Isolation Forest** flags statistical outliers with no labeled fraud data required
- **LSTM Autoencoder** trained on legitimate sessions — reconstruction error spikes when a session deviates from learned normal behavior

Both models run at inference in **< 20ms**.

**Layer 4 — Graph Neural Network Fraud Ring Detection**
The entire platform is modeled as a heterogeneous graph — accounts, devices, IPs, and locations as nodes; shared resources as edges. A **GraphSAGE-based GNN** classifies nodes by neighborhood context. An account sitting two hops from a known fraud cluster gets flagged *before* it strikes — purely based on graph proximity.

**Layer 5 — Device Integrity Checks**
We detect the tools attackers use: emulators, rooted/jailbroken devices, mock location providers, and debug-mode flags via Android SafetyNet and iOS DeviceCheck.

**Layer 6 — Unified Risk Scoring Engine**
Every signal feeds into a single weighted fraud score:

| Score | Decision |
|---|---|
| 0 – 30 | Approve automatically |
| 31 – 64 | Hold — request photo/video proof |
| 65 – 84 | Escalate to senior review |
| 85+ | Block payout, log to fraud graph |

**Layer 7 — Genuine Worker Protection**
Workers with strong reputation histories get a score buffer. Every blocked payout generates an appeal case ID. No instant bans — only evidence-based escalation.

---

## How we built it

We chose a stack optimized for **real-time inference at scale** while keeping every decision auditable and explainable:

- **FastAPI (Python)** — async API orchestration with < 50ms p99 latency
- **PyTorch + PyTorch Geometric** — LSTM Autoencoder and GraphSAGE GNN
- **scikit-learn** — Isolation Forest for behavioral anomaly detection
- **Neo4j** — native graph database storing the live fraud relationship graph
- **Apache Kafka** — real-time event ingestion from device telemetry streams
- **Redis** — sub-millisecond risk score caching and session state
- **PostgreSQL** — transaction records and full audit logs
- **React + Grafana** — live monitoring dashboard with fraud heatmap
- **Docker Compose** — local orchestration wiring all services together

We split across four areas: ML pipeline, graph engine, backend API, and frontend dashboard. ML models are hot-loaded so inference never blocks the request path. The GNN uses incremental neighborhood sampling so fraud graph updates don't require full retraining.

---

## Challenges we ran into

**Getting all seven layers to agree in real time** was the hardest engineering problem. Each layer has different latency characteristics — GNN traversal over Neo4j can spike on large graphs. We designed a tiered evaluation order: fast rule-based checks first, ML layers only when needed, keeping overall p99 under 100ms.

**Training the LSTM Autoencoder without labeled fraud data.** We had no real fraud dataset, so we trained exclusively on synthetic legitimate sessions and relied on reconstruction error at inference time. Calibrating the anomaly threshold to minimize false positives on genuine workers consumed most of our iteration cycles.

**GraphSAGE on a dynamic, mutating graph.** Fraud rings evolve — accounts get banned, new ones appear. Full retraining on every graph change wasn't feasible. We implemented incremental neighborhood sampling so node embeddings update without retraining from scratch.

**Cross-platform device integrity.** Android SafetyNet and iOS DeviceCheck have completely different APIs, response formats, and failure modes. Abstracting them into a single unified `DeviceIntegrityScore` that the risk engine could consume took more engineering than expected.

---

## Accomplishments that we're proud of

- **End-to-end working pipeline** — from a raw payout request to a risk decision, all seven layers run in sequence in a single API call
- **GNN fraud ring detection that flags zero-history accounts** — by propagating suspicion through graph neighborhoods, new fraud accounts are caught before they act
- **False-positive rate below 3%** on our synthetic test set — genuine workers with normal behavior almost never get flagged
- **The reputation buffer system** — a design choice that protects long-tenured honest workers from being penalized by signal noise
- **Real-time inference under 100ms end-to-end**, including GNN traversal and two ML model inferences
- Built a research-level, production-architected system in under 24 hours as a team of four

---

## What we learned

**Graph databases are underused in fraud detection.** The moment we put account/device/IP relationships into Neo4j and visualized them, fraud rings became *obvious* — patterns that looked clean in a flat table were glaring clusters in the graph.

**Unsupervised ML is more practical than supervised ML for fraud in cold-start scenarios.** You rarely have clean labeled fraud data when launching. Anomaly detection on learned normal behavior is far more deployable from day one.

**Defense-in-depth is the only real strategy.** Every individual layer we built can be bypassed by a sophisticated attacker. All seven together create a system where the *cost of attack exceeds the expected payout* — which is the actual security goal.

**Protecting honest workers is a technical constraint, not an afterthought.** False positives in fraud systems have real human consequences. Designing the reputation buffer and appeal pathway early forced us to build more careful score calibration throughout every layer.

---

## What's next for FraudShield

- **Federated learning across platforms** — train shared fraud models without sharing raw user data between competing gig platforms, using differential privacy guarantees so no platform exposes its users
- **Fully online GNN retraining** — move from incremental neighborhood updates to a real-time graph learning model that adapts to new fraud ring patterns as they emerge, with no manual retraining cycles
- **Multimodal proof verification** — use vision models to automatically verify photo/video proof submissions for medium-risk holds, removing the manual review bottleneck entirely
- **Regulatory compliance layer** — GDPR and India DPDP-compliant audit trail generation so every risk decision is explainable and exportable for legal review
- **Adversarial red-teaming dashboard** — a simulation mode where operators can replay known attack patterns against the current model to measure resilience before deploying updates
