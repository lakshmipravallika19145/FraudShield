# FraudShield

## Inspiration

We started thinking about a simple question: **how do gig platforms know a delivery actually happened?**

Most platforms trust GPS. If the app shows the driver reached the address, the delivery is marked complete and the payout goes through. That sounds reasonable — until you realize GPS can be faked by anyone using a free app downloaded in minutes.

But the problem goes even deeper. What about a driver who genuinely travels to the address, stands outside for 20 seconds, marks the order as delivered, and leaves — without ever knocking on the door or handing over the package? The GPS looks perfect. The route looks real. The system has no idea.

We kept finding more cracks like this. Fake accounts sharing the same phone. Drivers marking deliveries complete from a parking lot two streets away. Coordinated groups filing dozens of fake "stranded" requests from the same neighborhood at the same time.

Platforms try to fight this with GPS checks and account bans — but honest workers get caught in those nets too, while the actual fraudsters adapt and keep going.

We wanted to build something smarter: a system that doesn't just check if a GPS coordinate looks right, but actually figures out whether a real delivery happened.

That became **FraudShield**.

---

## What it does

FraudShield sits between a delivery event and a payout. Every time a driver marks an order complete or requests a payout, FraudShield checks it across multiple layers before any money moves.

There are two completely different types of fraud we solve:

---

### Fraud Type 1 — Fake Location (Driver never went there)

The driver uses a fake GPS app to pretend they're somewhere they're not. They never left home.

**How we catch it:**

**Cross-checking multiple signals**
GPS is just one signal. We also look at the IP address location, nearby WiFi networks, and the phone's motion sensors. If GPS says the driver is in Hyderabad but the IP address points to Mumbai and the phone hasn't moved at all — something is wrong. All signals need to match for a delivery to pass.

**Checking if the movement makes physical sense**
Real people move in messy, imperfect ways. Fake GPS apps move in straight lines at perfectly constant speeds. We check things like: Did the phone vibrate like a vehicle in motion? Did the route have normal turns and stops? Did the speed stay humanly possible? A driver who "teleported" 40 km in 10 seconds fails this check immediately.

**Spotting unusual patterns with AI**
We track each account's normal behavior over time — how often they request payouts, how their GPS signal varies, how active their sensors are. When something suddenly changes — like five payout requests in one hour from an account that usually does one a day — the system flags it automatically without needing anyone to write a specific rule for it.

**Finding fraud groups**
Fraud rarely happens alone. We connect the dots between accounts, phones, and IP addresses. If ten different accounts all share the same phone or all log in from the same internet connection, they're linked. When one gets flagged, the whole group gets a closer look — even the ones that haven't done anything suspicious yet.

**Checking if the device is tampered**
We detect if the phone is running fake GPS software, is an emulator pretending to be a real phone, or has been modified to bypass security checks.

---

### Fraud Type 2 — Ghost Delivery (Driver went there but didn't deliver)

The driver physically travels to the address. GPS looks perfect. Route looks real. But they never knocked on the door or handed over the package.

This is the harder problem — and the one most systems completely miss.

**How we catch it:**

**OTP Confirmation**
When the driver taps "Delivered," the customer instantly gets a 4-digit code on their phone. The driver must type that code into the app to complete the delivery. No code = no payout. The only way to get that code is for the customer to actually receive it and share it — which means there was real interaction.

**Time spent at the address**
We check how long the driver stayed within 50 meters of the delivery address. A real delivery takes time — parking, walking to the door, waiting for someone to answer, handing over the package. A driver who arrived and left in under 30 seconds didn't deliver anything.

| Time spent at address | What happens |
|---|---|
| More than 90 seconds | Looks normal, approved |
| 30 to 90 seconds | Asked to submit a delivery photo |
| Less than 30 seconds | Flagged for review |

**Photo of the delivery**
The driver must take a photo of the package at the door before completing the delivery. The photo must be taken live in the moment — not uploaded from the camera roll. It gets tagged with the time and GPS location automatically. If the same photo gets reused across multiple deliveries, the system catches it.

**Customer confirmation**
After the driver marks delivery complete, the customer gets a notification asking if their order arrived. If they say no within 10 minutes, the case is automatically escalated. If they don't respond, it's treated as a soft approval after the window closes.

**Complaint history**
If a customer reports a non-delivery, it's logged against that driver. One complaint is a soft flag. Three complaints in 30 days trigger an automatic account review. This catches drivers who consistently game the system one delivery at a time.

---

### The scoring system

Every check above adds or subtracts points from a fraud risk score for that delivery:

| What we found | Points added |
|---|---|
| GPS and IP location don't match | +30 |
| Phone shows no real movement | +25 |
| Behavior looks unusual for this account | +20 |
| Account is connected to a known fraud group | +35 |
| Tampered or fake device detected | +25 |
| OTP was never entered | +40 |
| Driver left address in under 30 seconds | +20 |
| No delivery photo submitted | +15 |
| Customer reported non-delivery | +25 |
| Driver has a strong, clean history | −20 |

| Final score | Decision |
|---|---|
| 0 – 30 | Payout approved automatically |
| 31 – 64 | Payout held — driver asked for OTP or photo |
| 65 – 84 | Sent to a human reviewer |
| 85 and above | Payout blocked, account flagged |

---

### Protecting honest workers

A fraud system that wrongly punishes real workers is a failed system. FraudShield is built around this:

- Drivers with a long, clean history get a 20-point buffer — small signal issues don't hurt them
- A blocked payout is never an instant ban — it generates a case ID the driver can appeal
- Medium-risk deliveries give the driver a chance to submit proof before anything is blocked
- Human reviewers handle every borderline case — the system never makes the final call alone on uncertain situations

---

## How we built it

We kept the architecture straightforward — each layer does one job and passes its result to the scoring engine.

- **FastAPI** handles every incoming delivery event and runs the checks in order
- **Scikit-learn** runs the behavioral anomaly detection (Isolation Forest)
- **PyTorch** powers the pattern-matching model that learns what normal sessions look like
- **PyTorch Geometric** runs the fraud group detection across the account-device-IP graph
- **Neo4j** stores the connections between accounts, phones, and IP addresses as a graph database — this makes fraud rings easy to spot visually
- **Apache Kafka** streams live device events (location, sensor data) into the system in real time
- **Redis** caches risk scores so repeated checks on the same account are instant
- **PostgreSQL** stores all delivery records and audit logs
- **React + Grafana** gives platform operators a live dashboard showing flagged deliveries, fraud hotspots on a map, and unusual payout spikes

We divided the work into four areas across our team: the AI models, the fraud graph engine, the backend API, and the frontend dashboard.

---

## Challenges we ran into

**Making everything fast enough.**
Each check takes a different amount of time. The fraud group detection in particular can slow down when the graph gets large. We solved this by running the fast checks first and only triggering the heavier ones when the fast checks raise a flag — so most clean deliveries get approved in milliseconds.

**No real fraud data to train on.**
We couldn't train our AI models on actual fraud examples because we didn't have any. Instead, we trained them on normal, legitimate behavior — and the models flag anything that looks significantly different from that. Getting this threshold right without flagging too many real deliveries took a lot of testing.

**Ghost deliveries are hard to detect without annoying real users.**
OTP confirmation is the most reliable check, but it adds friction for customers. We made it conditional — only required when other signals already look suspicious. For drivers with strong histories, photo proof alone is enough.

**Android and iPhone handle security checks differently.**
The tools we use to check if a device has been tampered with work completely differently on Android versus iPhone. We had to build a unified scoring system that translates both into a single number the rest of the system can use.

---

## Accomplishments that we're proud of

- We solved two completely different fraud types — fake location attacks and ghost deliveries — in one connected system
- The OTP + dwell time + photo proof combination gives a simple, practical answer to ghost delivery fraud that platforms can actually implement
- Fraud groups get caught even when individual accounts look clean, by tracing shared devices and IP addresses
- Honest workers with good history are protected from false flags through the reputation buffer
- The whole pipeline runs in under 100 milliseconds per delivery event
- We built and connected all of this in under 24 hours as a team of four

---

## What we learned

**The hardest fraud to catch is the most physically realistic one.** GPS spoofing is technically impressive but leaves signals everywhere. A driver who just doesn't knock on a door leaves almost no signal at all — catching that requires rethinking what "proof of delivery" actually means.

**Connecting accounts, devices, and IPs as a graph changes everything.** In a regular database, ten fraud accounts look like ten separate normal accounts. In a graph, they immediately cluster together and expose themselves. This was the single biggest insight from building FraudShield.

**Protecting real workers isn't a nice-to-have — it shapes every technical decision.** Every time we made a check stricter, we asked: what happens to a real driver with a bad GPS signal or slow internet? That question changed how we calibrated every threshold in the system.

**Simple checks beat complex ones when they're well-placed.** OTP confirmation is not a sophisticated technique. But placed at exactly the right moment — when the driver taps "Delivered" — it makes ghost delivery fraud nearly impossible. The right check at the right moment matters more than a complicated algorithm.

---

## What's next for FraudShield

- **Automatic photo review** — use a vision model to check delivery photos for obvious fakes (recycled images, wrong locations, generated images) instead of relying on manual review
- **Customer feedback as a signal** — analyze patterns in low delivery ratings to catch ghost deliveries that customers didn't formally report
- **Shared fraud intelligence** — let platforms share anonymized fraud patterns with each other so a fraud ring that gets caught on one platform can't just move to another
- **Full audit trail for compliance** — generate explainable, exportable records of every fraud decision to meet data protection requirements in India (DPDP) and Europe (GDPR)
- **Simulation mode for operators** — let platform teams replay past fraud attacks against the current system to test whether new changes would have caught them
