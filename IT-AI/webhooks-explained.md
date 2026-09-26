# Webhooks — Explained Simply

## What is a webhook?

A **webhook** is a way for one service (Server A) to notify another
service (Server B) the instant something happens — by sending it an HTTP
request, automatically, without B ever asking.

It's the reverse of how you normally use an API. Normally, *you* call the
API to ask "did anything happen?" A webhook flips that: instead of asking
repeatedly, you give the service a URL, and *it* calls *you* when there's
something to report.

Compare it to the other tools already explained:
- **cURL/HTTP request** = you call someone to ask a question, get one answer.
- **WebSocket** = a phone call that stays open, both sides talk anytime.
- **Webhook** = giving someone your phone number and saying "call me when
  the package arrives" — you don't call them, they call you, once, when the
  event happens.

---

## How it works

1. **You register a URL** with the service you want notifications from
   (e.g. in Stripe's dashboard, GitHub's repo settings, etc.):
   ```
   https://myapp.com/webhooks/stripe
   ```
   This is an endpoint on **your own server** that you build to receive
   the notification.

2. **Something happens** on their side — a payment succeeds, someone pushes
   code, an order ships.

3. **Their server sends an HTTP POST request** to your registered URL,
   with details about the event as JSON in the body:
   ```
   POST /webhooks/stripe HTTP/1.1
   Host: myapp.com
   Content-Type: application/json

   {
     "type": "payment_intent.succeeded",
     "data": { "amount": 2000, "currency": "usd" }
   }
   ```

4. **Your server receives it, processes it, and responds** (usually just
   `200 OK` to say "got it").

5. If your server doesn't respond in time or errors out, most webhook
   providers **retry** a few times (with growing delays) before giving up.

### Key difference from a WebSocket
A webhook is a **single, one-way HTTP request per event** — not a
persistent connection. Your server needs to be reachable at a public URL,
but it doesn't need to keep a connection open waiting.

---

## Real use-case examples

- **Payments (Stripe, PayPal):** "payment succeeded" → your server updates
  the order status and emails a receipt.
- **GitHub:** "someone pushed code" → triggers your CI/CD pipeline to run
  tests and deploy.
- **Shopify/e-commerce:** "order placed" → your warehouse system gets
  notified to start fulfillment.
- **Twilio (SMS):** "message received" → your app auto-replies.
- **Calendly/scheduling:** "meeting booked" → adds it to your CRM.
- **Slack:** "message posted in channel" → triggers a bot to respond.

---

## Security: verifying a webhook is real

Because your endpoint is a public URL, anyone could send a fake POST
request pretending to be Stripe/GitHub/etc. Providers guard against this
with a **signature**: they hash the payload with a shared secret and send
the hash in a header; you recompute it and compare.

```
X-Hub-Signature-256: sha256=abc123...
```

**Always verify this signature before trusting the payload** — never
process a webhook body blindly.

---

## Building a webhook receiver in Python

### Minimal receiver with Flask
Install: `pip install flask`

```python
from flask import Flask, request, jsonify

app = Flask(__name__)

@app.route("/webhooks/stripe", methods=["POST"])
def stripe_webhook():
    event = request.json
    print("received event:", event["type"])

    if event["type"] == "payment_intent.succeeded":
        amount = event["data"]["amount"]
        print(f"payment succeeded: {amount}")

    return jsonify({"received": True}), 200

app.run(port=5000)
```

### Verifying the signature (Stripe example)
```python
import stripe  # pip install stripe

endpoint_secret = "whsec_..."  # from your Stripe dashboard

@app.route("/webhooks/stripe", methods=["POST"])
def stripe_webhook():
    payload = request.data
    sig_header = request.headers["Stripe-Signature"]

    try:
        event = stripe.Webhook.construct_event(
            payload, sig_header, endpoint_secret
        )
    except stripe.error.SignatureVerificationError:
        return "invalid signature", 400

    # event is now verified and safe to trust
    print(event["type"])
    return jsonify({"received": True}), 200
```

---

## Building a webhook receiver in JavaScript (Node.js)

### Minimal receiver with Express
Install: `npm install express`

```javascript
const express = require("express");
const app = express();
app.use(express.json());

app.post("/webhooks/stripe", (req, res) => {
  const event = req.body;
  console.log("received event:", event.type);

  if (event.type === "payment_intent.succeeded") {
    console.log("payment succeeded:", event.data.amount);
  }

  res.status(200).json({ received: true });
});

app.listen(5000);
```

### Verifying the signature (Stripe example)
```javascript
const express = require("express");
const stripe = require("stripe")("sk_...");
const app = express();

const endpointSecret = "whsec_...";

// raw body needed for signature check — don't use express.json() here
app.post(
  "/webhooks/stripe",
  express.raw({ type: "application/json" }),
  (req, res) => {
    const sig = req.headers["stripe-signature"];
    let event;

    try {
      event = stripe.webhooks.constructEvent(req.body, sig, endpointSecret);
    } catch (err) {
      return res.status(400).send(`invalid signature: ${err.message}`);
    }

    console.log(event.type);
    res.status(200).json({ received: true });
  }
);

app.listen(5000);
```

---

## Testing webhooks locally

Your machine usually doesn't have a public URL, so services can't reach
`localhost`. Common fix: use a tunnel tool to expose your local server
temporarily.

```bash
# ngrok is the most common tool for this
ngrok http 5000
# gives you a public URL like https://abc123.ngrok.io
# → register THIS URL with the webhook provider while testing
```

---

## Webhook vs. WebSocket vs. Polling — quick comparison

| Approach | Who initiates | Connection | Best for |
|---|---|---|---|
| **Polling** | You ask repeatedly | New request each time | Simple, no server changes needed on your side |
| **WebSocket** | Either side, anytime | One persistent open connection | Real-time, two-way (chat, games) |
| **Webhook** | The *other* server, once per event | One-off HTTP request per event | Event notifications between two servers (payments, CI/CD, order updates) |

---

## Quick equivalence cheat sheet

| Concept | Python (Flask) | JavaScript (Express) |
|---|---|---|
| Receive endpoint | `@app.route("/hook", methods=["POST"])` | `app.post("/hook", handler)` |
| Read JSON body | `request.json` | `req.body` (with `express.json()`) |
| Respond OK | `return jsonify(...), 200` | `res.status(200).json(...)` |
| Verify signature | `stripe.Webhook.construct_event(...)` | `stripe.webhooks.constructEvent(...)` |

---

## Why this matters

Webhooks are how independent systems stay in sync in real time without
constant polling — they're the backbone of integrations like "when a
Stripe payment succeeds, update my database" or "when code is pushed,
deploy automatically." Understanding them means: (1) build a public
endpoint to receive them, (2) always verify the signature, (3) respond
fast with `200 OK` and do slow work asynchronously afterward.
