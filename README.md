# FraudShield

## Inspiration

We started with a simple question: **how does a delivery platform actually know a delivery happened?**

Most platforms just trust GPS. If the app shows the driver reached the address, the payout goes through. Simple enough — until you realize GPS can be faked in under a minute using a free app.

But then we found a bigger problem. What about a driver who actually travels to the address, scans the QR code to confirm arrival, tells the customer "we'll be back in 10 minutes" — and then just leaves? GPS is real. Route is real. Scan is done. Package never delivered.

And then we found another hole. What if we sent an OTP to the customer instead? A fraudulent driver could just call the customer, pretend to be support staff, say "please share the OTP and your order will arrive shortly" — and the trusting customer reads it out. OTP confirmed. Delivery marked complete. Package never handed over.

Every time we thought we had it solved, a new real scenario broke it.

So we kept going until every hole was closed. That became **FraudShield**.

---

## What it does

FraudShield checks every delivery before a payout is approved. It catches two types of fraud:

- **Type 1** — Driver fakes their location and never goes to the address at all
- **Type 2** — Driver goes to the address but never hands over the package

---

### Catching Type 1 — Driver fakes their location

When a driver claims to be at a delivery address, we run three quick checks:

**Do all location signals agree?**
We don't just look at GPS. We also check the IP address and nearby WiFi networks. If GPS says the driver is at the delivery address but their internet connection places them in a completely different area — that's a red flag. A real driver at a real location will have all three signals roughly matching.

**Does the movement look real?**
Fake GPS apps move in perfectly straight lines at perfectly constant speeds. Real people slow down at traffic lights, take imperfect routes, and their phone vibrates while in a moving vehicle. If the movement looks too perfect, or the phone shows no vibration at all during claimed travel, we flag it.

**Is the device legitimate?**
We check if the phone is running a fake GPS app, is a virtual device pretending to be a real phone, or has been tampered with to bypass security. These are the tools fraudsters need — detecting them early raises a warning before anything else happens.

That's it. Three checks. Simple, fast, and effective for catching drivers who never left home.

---

### Catching Type 2 — Driver goes there but doesn't deliver

This is the harder problem. The driver is genuinely at the location. All location signals are clean. The fraud happens in the last few meters.

We solved this with a two-step confirmation that **both the driver and the customer must complete** — and neither step alone is enough.

---

**Step 1 — Driver scans the customer's QR code**

Every delivery has a unique QR code that lives inside the customer's app. When the driver arrives:

- The driver opens the scanner in their app
- They scan the QR code on the customer's phone screen
- This only works if both phones are physically close to each other
- The code expires in 60 seconds and cannot be reused

This proves the driver was physically present. There is no number to read out over a phone — the driver must be standing next to the customer to scan their screen. This is why we replaced OTP with a scanner. OTP can be tricked over a phone call. A QR scan cannot.

The scan alone does **not** complete the delivery. It starts a 10-minute window for Step 2.

---

**Step 2 — Customer uploads a photo of the received package**

After receiving the package, the customer opens the app and takes a live photo of it.

- The photo must be taken in the moment — uploading old photos from the camera roll is blocked
- It is automatically tagged with the time and the customer's location
- The customer's location when taking the photo must match the delivery address
- Only after this photo is submitted does the delivery get marked complete

The driver has zero control over this step. If they never handed over the package, the customer has nothing to photograph.

---

**Closing the scan-and-run hole**

A driver could try: arrive → scan the QR code → tell the customer "we'll be back in 10 minutes" → leave.

This fails because the scan only opens a 10-minute window. If the customer doesn't upload a photo within that window, the delivery stays incomplete and the payout is held. The driver cannot close the delivery themselves — only the customer's photo does that.

```
Driver scans QR code
        ↓
10-minute window opens
        ↓
Customer uploads receipt photo
        ↓
Delivery confirmed → Payout approved

No photo within 10 minutes → Delivery not confirmed → Payout held
```

---

**What if the customer genuinely forgets to upload?**

This happens. A real delivery occurs, the customer receives their package, and they close the app and forget. We handle this with a simple fallback:

- **2 minutes after scan** — customer gets a reminder notification
- **5 minutes after scan** — second reminder with a one-tap "Yes, I received it" button — no photo needed
- **10 minutes after scan, no response, driver has clean history** — delivery is auto-approved
- **10 minutes after scan, no response, driver is new or flagged** — held for manual review

| Situation | What happens |
|---|---|
| Customer uploads receipt photo | Confirmed immediately |
| Customer taps "Yes, I received it" | Confirmed |
| No response + driver has clean history | Auto-approved after 10 minutes |
| No response + driver is new or flagged | Held for manual review |
| Customer reports not received | Escalated immediately |

---

### Why even a perfect GPS spoof doesn't work

Even if a driver perfectly fakes their GPS, IP address, and movement — they still cannot complete the delivery without:

1. **Physically scanning the QR code** on the customer's phone — requires being next to the customer in person
2. **The customer uploading a receipt photo** — the driver has no control over this at all

| What the fraudster tries | Can they fake it? |
|---|---|
| Spoof GPS to look like they're at the address | Possible |
| Fake IP address and WiFi signals |  Very difficult |
| Fake phone movement |  Very difficult |
| Complete the QR scan without being there | Impossible |
| Get the customer to upload a receipt photo without receiving anything | Impossible |

Location fraud gets them past the first set of checks. It gets them nowhere near completing the delivery.

---

### The risk score

Every check adds or subtracts points from a fraud risk score for that delivery:

A single signal firing never hard-blocks anyone. Multiple signals firing together escalate the decision step by step. Only when most signals fire at once — meaning deliberate, coordinated fraud — does the payout get blocked outright.

---

### Protecting honest workers

- Drivers with a long clean history get a 20-point buffer — small issues don't penalize them
- A blocked payout is never an instant ban — every block gets a case ID the driver can appeal
- The fallback ladder means genuine deliveries never get permanently stuck because a customer forgot to respond
- Human reviewers handle every borderline case — the system never makes a final call alone when things are unclear

---

## How we built it

- **FastAPI** — receives every delivery event and runs all checks in order
- **Scikit-learn** — spots unusual behavior patterns per account over time
- **PyTorch** — learns what normal delivery sessions look like and flags sessions that deviate
- **PyTorch Geometric** — finds connected fraud groups across accounts, devices, and IP addresses
- **Neo4j** — stores the connections between accounts, phones, and IPs as a graph so fraud rings are easy to spot
- **Apache Kafka** — streams live location and sensor data from devices in real time
- **Redis** — saves risk scores so repeated checks on the same account are instant
- **PostgreSQL** — stores all delivery records and audit logs
- **React + Grafana** — live dashboard for platform operators showing flagged deliveries and suspicious areas on a map

---

## Challenges we ran into

**OTP is the industry standard — and it is broken.** The social engineering attack requires no technical skill at all: call the customer, ask for the code, done. Replacing it with a QR scanner that requires physical proximity meant rethinking how the driver and customer apps interact at the exact moment of handoff.

**The scan-and-run problem.** Once we added the QR scanner, we immediately realized a driver could scan and leave. Making the scan open a window that only the customer's photo can close — and linking the two steps explicitly — is what makes the combination work.

**Customers forgetting to confirm.** A strict "photo required or payout blocked" rule would hold back thousands of legitimate payouts every day. Building the fallback ladder — reminders, one-tap button, and auto-approval for trusted drivers — took the most iteration to get right.

**Speed.** Checking fraud groups across a large graph of connected accounts takes time. We run the fast checks first and only trigger the heavier checks when something early looks suspicious — so clean deliveries get approved in milliseconds.

---

## Accomplishments that we're proud of

- Found and closed the OTP social engineering hole — replaced with a QR scanner that requires physical presence
- Found and closed the scan-and-run hole — the scan alone never completes a delivery, only the customer's photo does
- Built a fallback system so genuine deliveries are never permanently blocked because a customer forgot to tap confirm
- GPS spoofing is stopped not by detecting the spoof itself but by requiring real customer interaction to complete the delivery
- Fraud groups are caught by connecting the dots between accounts, phones, and IP addresses — even before any individual account does something wrong
- Built the entire system end to end in under 24 hours as a team of four

---

## What we learned

**Every fix creates a new attack surface.** OTP fixed ghost delivery fraud until we saw it could be read out over a phone. The QR scanner fixed that until we saw a driver could scan and leave. Thinking through the next attack after every fix is what separates a real fraud system from a checklist.

**The two confirmation steps only work because they are linked.** If the scan and the photo were independent, either one could be gamed on its own. Tying them together — scan opens the window, photo closes it — is what makes the combination hard to beat.

**Flipping who provides proof changes everything.** When the driver submits proof, the driver controls it. When the customer submits proof from their own phone at their own location, the driver has nothing to fake.

**Protecting honest workers shapes every decision.** Every time we tightened a check, we asked: what happens to a real driver whose customer forgot to open the app? That question drove the entire fallback design.

---

## What's next for FraudShield

- **Automatic photo checking** — use a vision model to catch fake receipt photos such as recycled images or photos taken at the wrong location, instead of relying on manual review
- **Catching user-side fraud** — customers who falsely report non-delivery to get a refund while keeping the product, detected by cross-checking their complaint history against driver scan and photo data
- **Shared fraud signals across platforms** — a fraud ring caught on one platform should not be able to simply move to another
- **Compliance audit trail** — a full exportable record of every fraud decision to meet data protection laws in India and Europe
- **Operator replay mode** — let platform teams test whether their current system would have caught past fraud attacks, before pushing any changes live
