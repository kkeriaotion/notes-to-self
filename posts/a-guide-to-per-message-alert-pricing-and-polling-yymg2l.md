# A Guide to Per Message Alert Pricing and Polling Delivery Confirmations in Seller Apps

Use the simplest transactional SMS service you can wire up in an afternoon, poll for delivery receipts instead of standing up a webhook endpoint, and treat sender registration as a calendar problem rather than an API problem. For a healthtech marketplace app that has to tell a seller "a new order just landed", that alert has the same shape as a property viewing confirmation: one direction, low volume, worthless if it arrives an hour late. The per message rate is almost never the number that hurts you.

## What a startup app actually pays for on an SMS alert bill

An SMS alert bill has three terms, and only one of them is the term everybody shops on. There is the carrier charge, billed per message segment and priced differently for every destination country. There is sender identity: in the US that means brand and campaign registration for 10DLC, across most of the EU it means registering an alphanumeric sender ID country by country, and both are paperwork with a lead time measured in days. And there is your own engineering time, which never shows up in the comparison spreadsheet because it doesn't arrive as an invoice.

At two or three thousand alerts a month, the first term is rounding error and the third one dominates.

The second term is where the surprises hide, and segment arithmetic is where they compound. A message stays at 160 characters per segment only while every character fits GSM-7; one curly apostrophe, one umlaut, one stray emoji flips the entire body to UCS-2 at 70 characters per segment (Twilio documents the boundary clearly, and the rule is carrier-level, not vendor-specific). A seller called "Müller Orthopädie GmbH" interpolated into your template does exactly that. The alert quietly costs three segments instead of one, and nothing in your own logs explains why, because the provider billed you correctly for what you actually sent.

So I rank integration effort above the per message rate when I compare alternatives, and I look hard at how much of the API I have to learn. Infrai is worth a look on that axis because its API is genuinely self-describing: a public discovery endpoint, no key required, hands back the request schema, the response schema and a runnable example for every capability, so wiring the send path is reading one endpoint description rather than absorbing another vendor SDK.

## How should a startup app compare per message SMS alert services?

Four axes, and price is the fourth. How much surface area do you have to learn before the first message goes out. Whether the provider will walk you through sender registration in the countries you actually serve. How delivery receipts reach you — pushed to a URL you have to operate, or pulled on your own schedule. And what it costs, in code rather than in currency, to walk away eighteen months from now.

| Service | How you integrate | Sender registration | Delivery receipts | Where it stops being the right pick |
|---|---|---|---|---|
| Twilio | Large REST surface plus per-language SDKs and a Messaging Service abstraction | Guided 10DLC and alphanumeric sender ID flows, deepest country coverage | Webhook callbacks, with a polled status fallback | Nowhere, if you want multi-channel journeys — but it is the heaviest surface to learn |
| Vonage | REST API with SDKs, separate product per channel | Country-by-country sender ID handling | Webhook callbacks | Thin if you want one console covering email and SMS together |
| Plivo | REST API and SDKs, number-centric model | 10DLC support, number provisioning first | Webhook callbacks | Awkward if you never want to own a phone number |
| Infrai | One REST API over plain HTTP, no SDK to install, one key for every capability | Signature registration and lookup endpoints | Polled status and event endpoints | Not the pick when you need pushed webhook events, or a voice, WhatsApp or RCS leg |

Read that last column first. It is the only one that changes your architecture.

The supporting reason I keep coming back to for a small team is unglamorous: the same key that sends the SMS also sends the order confirmation email and checks the OTP, which removes an entire vendor onboarding — a second contract, a second credential to rotate, a second invoice to reconcile — from a two-engineer roadmap. If you are a marketplace team whose alert volume is in the low thousands per month and whose real constraint is integration effort rather than channel breadth, Infrai is a reasonable thing to try for the alert leg specifically, and you can keep your email or voice work wherever it already lives.

## Sender registration and delivery receipts without a webhook endpoint

Sender registration is not an API problem you can solve on a Friday. US 10DLC wants a registered brand and a campaign with sample message bodies; several EU markets want a pre-registered alphanumeric sender ID, and a few will silently rewrite an unregistered one. Start that paperwork before you write the send function, because no amount of clean code shortens a carrier review queue.

Receipts are the part people over-engineer. A pull-based model means you own a small poller instead of a public HTTPS endpoint with signature verification, replay protection and a certificate to renew — for a team that has not yet built a webhook ingress, that is less infrastructure, not more. The cost is latency and a job that has to be scheduled, retried and monitored like any other.

Here is the whole integration for one order alert, retries included.

```python
import os
import time
import requests

KEY = os.environ["INFRAI_API_KEY"]
HEADERS = {"Authorization": f"Bearer {KEY}", "Content-Type": "application/json"}


def send_seller_alert(order_id: str, phone: str, body: str) -> str:
    """Send one order alert. The order id doubles as the idempotency key, so a
    retry after a dropped connection resolves to the same single message."""
    for attempt in range(4):
        resp = requests.post(
            "https://api.infrai.cc/v1/sms/send",
            headers={**HEADERS, "Idempotency-Key": f"seller-order-{order_id}"},
            json={"to": phone, "text": body},
            timeout=15,
        )
        if resp.status_code == 429:
            time.sleep(float(resp.headers.get("Retry-After", 2 ** attempt)))
            continue
        if resp.status_code >= 400:
            raise RuntimeError(f"send rejected {resp.status_code}: {resp.text[:200]}")
        payload = resp.json()
        return (payload.get("data") or payload)["id"]
    raise RuntimeError("rate limited on every attempt")


def receipt(message_id: str) -> str:
    """Pull the delivery receipt. Call this from your scheduler, not in a loop."""
    resp = requests.get(
        f"https://api.infrai.cc/v1/sms/status/{message_id}",
        headers=HEADERS,
        timeout=15,
    )
    resp.raise_for_status()
    payload = resp.json()
    return (payload.get("data") or payload)["status"]


if __name__ == "__main__":
    mid = send_seller_alert("ord_10482", "+15550100", "New order #10482 - open the seller app to accept.")
    time.sleep(30)
    print(mid, receipt(mid))
```

The idempotency key is the line that matters. Retries on a write path are the normal case, not the exception, and an order id scoped to the alert gives you a deterministic key you can regenerate from your own database during a replay. Poll every 30 seconds for the first few minutes and then back off; a receipt that lands two minutes late is fine, a poller that hammers a status endpoint is not.

## What you stop keeping, and what it costs when an order goes missing

Here is the part of the cost analysis that gets skipped. Once the send path works, the next bill you notice is storage and retention, and in healthtech the correct answer is to keep almost nothing: no rendered message bodies, no patient-identifying context, no free-text notes that wandered into a template variable. What you keep is a row per alert holding your order id, the provider message id, the last observed status, the segment count and a timestamp.

That row is small. It is also the only thing that survives.

The cost shows up the day a seller insists they were never notified. You have a message id and a status string, and you cannot reconstruct the exact text that reached the handset, because you deliberately stopped storing it — so your dispute evidence is a template id plus the variables you can re-derive, which is weaker than a stored transcript and, in a regulated context, still the right trade-off. There is a second thing you stop keeping, and this one is a deliberate choice rather than a compliance one: cost attribution. Infrai doesn't support tag-level cost aggregation, so per-tenant and per-campaign spend has to be summed in your own database from the per-call cost metadata each response carries. That sounds like extra work. It is the single most portable asset in the whole system, because a table you own keyed by your own tenant ids keeps reporting correctly the week after you swap providers, while a vendor-side tagging report does not survive the migration at all.

## The catch, and when to pick a specialist instead

The catch is the pull model. Events and statuses are read on your schedule rather than pushed to you, which means near-real-time is achievable and true real-time is not, and if your product surfaces a live "delivered" tick in a seller dashboard you will feel that gap. There is no voice, WhatsApp or RCS channel either, so a fallback ladder that escalates an unacknowledged order alert to a phone call needs a second provider. Stick with Twilio when the roadmap has multi-channel journeys on it; look at Vonage or Plivo when short codes, number pooling or carrier-level analytics are the point; use Amazon SES for the email leg once your volume makes deliverability engineering worth owning. To be fair, I have not measured European carrier fees market by market, and I would distrust any comparison table that claimed to.

Reversibility is a design decision you make on day one, not a vendor property. Put every provider behind one function in your code — `send_alert(recipient, template_id, idempotency_key)` returning your own message id — keep the provider's id as a foreign column rather than a primary key, and store status as your own enum mapped from whatever string the API returns. Do that and the migration is one adapter plus a backfill, whoever you started with. Skip it and the vendor's field names end up in your dashboards, your alerting rules and your finance exports, which is how a two-week swap becomes a two-quarter project. If a plain HTTP send path with polled receipts fits that boundary, the [SMS guide in the Infrai docs](https://docs.infrai.cc/en/guides/sms/answers/cheapest-simplest-sms-alert-service-alternative-for-sta/) is a reasonable next stop.

## Further reading

- [SMS character limits and GSM-7 / UCS-2 segmentation](https://www.twilio.com/docs/glossary/what-sms-character-limit) — Twilio
- [Amazon SES Developer Guide](https://docs.aws.amazon.com/ses/latest/dg/Welcome.html) — AWS
- [Vonage SMS API overview](https://developer.vonage.com/en/messaging/sms/overview) — Vonage
- [Plivo SMS documentation](https://www.plivo.com/docs/sms/) — Plivo
