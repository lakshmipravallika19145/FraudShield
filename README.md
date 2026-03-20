# FraudShield

## Inspiration

We started thinking about a simple question: **how do gig platforms know a delivery actually happened?**

Most platforms trust GPS. If the app shows the driver reached the address, the delivery is marked complete and the payout goes through. That sounds reasonable — until you realize GPS can be faked by anyone using a free app downloaded in minutes.

But the problem goes even deeper. What about a driver who genuinely travels to the address, stands outside for 20 seconds, marks the order as delivered, and leaves — without ever knocking on the door or handing over the package? The GPS looks perfect. The route looks real. The system has no idea.

We kept finding more cracks like this. Fake accounts sharing the same phone. Drivers marking deliveries complete from a parking lot two streets away. Coordinated groups filing dozens of fake "stranded" requests at the same time.

And even when we added OTP verification, we realized it has a social engineering hole: a fraudulent driver can simply call the customer, pretend to be support, say *"tell me the OTP and we'll be there in 5 minutes"* — and the customer, trusting them, reads it out. OTP confirmed. Delivery marked complete. Package never handed over.

So we rethought the entire delivery confirmation system from scratch.

That became **FraudShield**.

---

## What it does

FraudShield sits between a delivery event and a payout. Every time a driver marks an order complete, FraudShield verifies it across multiple layers before any money moves.

There are two completely different types of fraud we solve:

---

### Fraud Type 1 — Fake Location (Driver never went there)

The driver uses a fake GPS app to pretend they're somewhere they're not. They never left home.

**How we catch it:**

**Cross-checking multiple signals**
GPS is just one signal. We also look at the IP address location, nearby WiFi networks, and the phone's motion sensors. If GPS says the driver is in one city but the IP address points to another city and the phone hasn't moved at all — something is wrong. All signals need to match.

**Checking if the movement makes physical sense**
Real people move in messy, imperfect ways. Fake GPS apps move in perfectly straight lines at constant speeds. We check: Did the phone vibrate like a vehicle in motion? Did the route have normal turns and stops? Did the speed stay humanly possible? A driver who "teleported" 40 km in 10 seconds fails this check immediately.

**Spotting unusual patterns with AI**
We track each account's normal behavior over time — how often they request payouts, how active their sensors are, how their GPS signal varies. When something suddenly looks different from their usual pattern, the system flags it automatically.

**Finding fraud groups**
If ten different driver accounts all share the same phone or log in from the same internet connection, they're linked. When one gets flagged, the whole group gets a closer look — even the ones that haven't done anything suspicious yet.

**Checking if the device is tampered**
We detect if the phone is running fake GPS software, is a virtual emulator pretending to be a real phone, or has been modified to bypass security checks.

---

### Fraud Type 2 — Ghost Delivery (Driver went there but didn't deliver)

The driver physically travels to the address. GPS looks perfect. Route looks real. But they never handed over the package.

This is the harder problem — and the one most systems miss completely.

We also discovered that the most common fix — OTP verification — has a critical flaw:

> A fraudulent driver can call the customer, pretend to be a support agent, say *"please share the OTP and your order will be delivered in 5 minutes"* — and the customer believes them and reads it out. The OTP gets entered. The delivery is marked complete. The package is never handed over.

**OTP can be socially engineered. So we don't use it.**

Instead, FraudShield uses three tamper-proof methods:

---

**✅ Fix 1 — QR Code Scanner (replaces OTP)**

Instead of sending a code the customer can read out loud, we generate a unique QR code that lives only inside the customer's app.

- When the driver arrives, they open their scanner in the delivery app
- They physically scan the QR code shown on the customer's phone screen
- The scan only works if both phones are within 1 meter of each other (proximity check)
- The QR code expires in 60 seconds and cannot be reused

This requires **real, in-person interaction**. There is no code to read out over the phone. The driver must be physically next to the customer and scan their screen. This cannot be faked remotely.

---

**✅ Fix 2 — Customer Uploads a Photo to Confirm Receipt**

After the driver hands over the package, the **customer** — not the driver — must open the platform's website or app and upload a photo of the package they just received.

- The photo must be taken live (camera roll uploads are blocked)
- It is automatically tagged with the time and GPS location at the moment of capture
- The customer's location at photo time must match the delivery address
- The delivery is only marked complete after this photo is submitted

This flips the proof of delivery from the driver's side to the customer's side. A fraudulent driver cannot fake a photo that the customer has to take on their own device, at their own location, in real time.

---

**✅ Fix 3 — Dwell Time Check (how long the driver stayed)**

Even with the scanner and photo, we check how long the driver was physically near the delivery address. A real handoff takes time.

| Time spent within 50m of address | What happens |
|---|---|
| More than 90 seconds | Looks normal, approved |
| 30 to 90 seconds | Customer photo proof required |
| Less than 30 seconds | Flagged for review |

A driver who arrived and left in 20 seconds did not have a real interaction, regardless of what other checks say.

---

**Supporting checks:**

**Customer complaint history**
If a customer reports a non-delivery, it is logged against that driver. One complaint is a soft flag. Three complaints in 30 days trigger an automatic account review. This catches drivers who game deliveries one at a time over a longer period.

**Customer confirmation notification**
After a delivery is marked complete, the customer receives a push notification asking if their order arrived. If they report no within 10 minutes, the case is escalated automatically.

---

### The scoring system

Every check above adds or subtracts points from a fraud risk score:

| What we found | Points added |
|---|---|
| GPS and IP location don't match | +30 |
| Phone shows no real movement | +25 |
| Behavior looks unusual for this account | +20 |
| Account is connected to a known fraud group | +35 |
| Tampered or fake device detected | +25 |
| QR scan was not completed | +40 |
| Driver left address in under 30 seconds | +20 |
| Customer did not upload receipt photo | +30 |
| Customer reported non-delivery | +25 |
| Driver has a strong, clean history | −20 |

| Final score | Decision |
|---|---|
| 0 – 30 | Payout approved automatically |
| 31 – 64 | Payout held — driver asked to resolve the missing check |
| 65 – 84 | Sent to a human reviewer |
| 85 and above | Payout blocked, account flagged |

---

### Why this combination is hard to fake

| Attack | Why it fails |
|---|---|
| Driver calls customer and asks for OTP | There is no OTP. QR scan requires physical presence |
| Driver takes their own photo and submits it | Photo must come from the customer's device, at customer's location |
| Driver uploads a fake or old photo | Camera roll blocked, photo is live-tagged with time and GPS |
| Driver stands nearby but doesn't knock | Dwell time check catches < 30 second stays |
| Driver colludes with customer | Customer complaint history and repeat pattern detection catches this over time |

---

### Protecting honest workers

- Drivers with a long, clean history get a 20-point buffer — minor signal issues don't penalize them
- A blocked payout is never an instant ban — it generates a case ID the driver can appeal
- Medium-risk deliveries give the driver a chance to resolve the missing check before anything is blocked
- Human reviewers handle every borderline case

---

## How we built it

- **FastAPI (Python)** — handles every incoming delivery event and runs the checks in order
- **Scikit-learn** — runs behavioral anomaly detection (Isolation Forest)
- **PyTorch** — pattern-matching model trained on legitimate delivery sessions
- **PyTorch Geometric** — fraud group detection across the account-device-IP graph
- **Neo4j** — stores connections between accounts, phones, and IP addresses as a graph
- **Apache Kafka** — streams live device events (location, sensor data) in real time
- **Redis** — caches risk scores so checks on the same account are instant
- **PostgreSQL** — stores all delivery records and audit logs
- **React + Grafana** — live dashboard showing flagged deliveries and fraud hotspots on a map

---

## Challenges we ran into

**OTP is the industry standard — but it's broken.** The social engineering attack (calling the customer and asking for the code) is simple, requires no technical skill, and works on trusting customers every time. Replacing OTP with a QR scanner meant redesigning how both the driver app and customer app interact at the moment of delivery.

**Making the customer photo upload frictionless.** Asking the customer to take a photo and upload it adds a step most people will skip if it's annoying. We made it a single tap from the delivery notification — the camera opens automatically, the photo uploads in one step, and the customer sees their order confirmed instantly.

**Making everything fast enough.** Each check takes a different amount of time. We run fast checks first and only trigger the heavier ones when early checks raise a flag — so clean deliveries get approved in milliseconds.

**No real fraud data to train on.** We trained the AI models on normal, legitimate behavior and flag anything that looks significantly different. Getting the threshold right without penalizing too many genuine drivers took a lot of testing.

---

## Accomplishments that we're proud of

- **Identified and closed the OTP social engineering hole** — replaced it with a QR scanner that requires physical presence, which cannot be faked over a phone call
- **Flipped photo proof to the customer side** — the customer uploads the receipt photo, not the driver, making it impossible for the driver to fake
- **Solved both fraud types** — fake location attacks and ghost deliveries — in one connected system
- **Fraud groups get caught** even when individual accounts look clean, by tracing shared devices and IP addresses
- **Honest workers are protected** through reputation buffers and a human review step for every borderline case
- Built and connected the full pipeline in under 24 hours as a team of four

---

## What we learned

**The most obvious fix is often the most exploitable one.** OTP verification sounds secure. In practice, a 30-second phone call defeats it. The better question is always: what does this check require that cannot be done remotely or over a phone?

**Flipping who provides the proof changes everything.** When the driver submits proof, the driver can fake it. When the customer submits proof — on their own device, at their own location, in real time — there is nothing for the driver to fake.

**Connecting accounts, devices, and IPs as a graph changes everything.** In a regular database, ten fraud accounts look like ten separate normal accounts. In a graph, they immediately cluster together and expose themselves.

**Protecting real workers is not a nice-to-have — it shapes every technical decision.** Every time we made a check stricter, we asked: what happens to a real driver with a bad GPS signal, a customer who didn't open the app, or a slow internet connection? That question changed how we calibrated every threshold.

---

## What's next for FraudShield

- **Automatic customer photo review** — use a vision model to check receipt photos for fakes (recycled images, wrong items, AI-generated) instead of manual review
- **User-side fraud detection** — catching customers who falsely report non-delivery to get a refund while keeping the product
- **Shared fraud intelligence** — let platforms share anonymized fraud patterns so a fraud ring caught on one platform can't just move to another
- **Full audit trail for compliance** — exportable records of every fraud decision to meet data protection requirements in India (DPDP) and Europe (GDPR)
- **Simulation mode for operators** — let platform teams replay past fraud attacks against the current system to test whether new changes would have caught them
