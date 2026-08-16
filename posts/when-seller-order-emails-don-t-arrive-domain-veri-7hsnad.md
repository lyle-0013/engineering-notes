# When Seller Order Emails Don't Arrive: Domain Verification, DKIM, and Bounce Suppression

Use one dedicated sending subdomain, get SPF, DKIM and DMARC aligned on it before the first message leaves the app, and treat bounce suppression and delivery-event polling as one loop rather than two backlog tickets. For a Node.js backend that sends a single transactional email per event — a property-management marketplace where a manager books a repair and the vendor on the other end has to learn that an order is waiting — that is the least complex setup that clears a real deliverability bar.

| Sending path | What it gives you | What it costs a one-person team |
|---|---|---|
| SMTP through an ordinary mailbox account | Shared reputation, daily caps, no per-message events | Nothing to build, and nothing to debug with |
| Provider HTTP API on a shared IP pool | Warmed pool, DKIM signing at the edge, per-message events | One integration plus your own suppression store |
| Provider HTTP API on a dedicated IP | Isolated reputation once volume is steady | A warm-up schedule and constant volume to keep it warm |
| Self-hosted MTA | Full control of queues, retries and headers | Reputation, TLS, feedback handling, on-call |

Row two is the answer for almost every one-person product. A dedicated IP is a downgrade until sending is both steady and large, because an address that goes quiet looks unfamiliar to receivers when it wakes up again. Self-hosting an MTA is interesting engineering and a poor revenue-per-hour trade when mail is one feature of a product that earns its money somewhere else — the undifferentiated half of this problem is exactly the half worth handing to someone whose full-time job it is.

## How much does SPF, DKIM and DMARC setup actually buy you for transactional email?

They buy the right to be judged on content and behaviour instead of being dropped at the door. SPF (RFC 7208) declares which hosts may put your domain in the envelope sender. DKIM (RFC 6376) attaches a signature to the header that survives most forwarding. DMARC (RFC 7489) ties both back to the address a human actually reads by requiring alignment — the receiver checks that the visible From domain matches the domain that passed SPF or DKIM, and applies your published policy when neither lines up.

Domain verification on the provider side is a smaller thing than the name suggests: the provider reads your DNS, confirms the records say what they should, and only then accepts mail for that domain. Make it a deploy-time check rather than a launch-day scramble.

Propagation isn't the hard part. Alignment is.

Send from `mail.example.com` or `orders.example.com` and leave the root domain to the humans who use it for contracts and support threads. Start the DMARC record at `p=none` with an `rua` address, read the aggregate reports until every legitimate stream is accounted for, then tighten to quarantine and finally reject. That sequence matters more for a marketplace than for a newsletter, because the order-alert stream, the billing system and whatever ticketing tool the operations side signed up for last quarter may all claim the same From domain; a policy tightened before that inventory is complete will quietly kill the one message a vendor needed. I'm not sure there's a universal timeline for the ramp. The honest version is that you tighten when the reports stop surprising you.

## Bounce suppression belongs in your own database

A hard bounce is a permanent verdict about an address — `5.1.1`, no such user, in the enhanced status codes of RFC 3463 — and a complaint is a human pressing the spam button. Both have to block every future send to that address until a person changes something. Soft bounces, the full mailbox and the temporary rejection, deserve a bounded number of retries and then the same treatment.

Keep the row yourself: address, event type, status code, the provider's event id, first seen and last seen. Check it before you enqueue the message, not after the send call comes back. Providers maintain their own suppression lists, and those lists are perfectly good until the day you add a second sending stream or move the account, at which point your policy walks with you only if you wrote it down. There's a support benefit too, and for a solo founder it's the bigger one: when a property manager asks why their vendor never got the work order, the answer is a row with a timestamp and a reason code instead of an afternoon in someone else's log viewer. That same table is what lets the product degrade well — a suppressed address turns the order alert into an in-app task rather than a message that silently evaporates.

Opens are not delivery. Apple's Mail Privacy Protection prefetches remote content through a proxy, so an open can be recorded for a message no human ever looked at, which makes open rate useless as a proxy for whether the vendor was reached. Accepted, delivered, bounced, complained, plus real product actions — a vendor tapping "accept job" is a delivery receipt with revenue attached.

## A worked example: send path, suppression check, and polling job

```ts
type Order = { id: string; sellerEmail: string; propertyRef: string };

const API = process.env.MAIL_API_URL;
const KEY = process.env.MAIL_API_KEY;
if (!API || !KEY) throw new Error("MAIL_API_URL and MAIL_API_KEY are required");

async function notifySeller(order: Order): Promise<void> {
  if (await db.isSuppressed(order.sellerEmail)) {
    await db.queueInAppTask(order.id);            // still reachable, just not by mail
    return;
  }

  const res = await fetch(`${API}/messages`, {
    method: "POST",
    headers: { authorization: `Bearer ${KEY}`, "content-type": "application/json" },
    body: JSON.stringify({
      from: "orders@mail.example.com",
      to: order.sellerEmail,
      subject: `New work order ${order.id} - ${order.propertyRef}`,
      text: `A property manager booked you for ${order.propertyRef}.`,
      idempotency_key: `order-${order.id}`,       // a retried job must not mail twice
    }),
  });

  if (!res.ok) throw new Error(`send rejected with ${res.status}`);
  const { message_id } = (await res.json()) as { message_id: string };
  await db.recordSend(order.id, message_id);      // acceptance, not delivery
}
```

A 2xx here means the provider took custody of the message. Nothing more. The mailbox on the far side has not spoken yet, and the gap between those two facts is where every "but I sent it" conversation lives.

```ts
type DeliveryEvent = {
  id: string;
  type: "delivered" | "bounced" | "complained";
  address: string;
  code?: string;                                  // enhanced status, e.g. "5.1.1"
  occurredAt: string;
};

async function pollDeliveryEvents(): Promise<void> {
  const cursor = await db.getCursor("mail-events");
  const res = await fetch(`${API}/events?after=${encodeURIComponent(cursor)}&limit=200`, {
    headers: { authorization: `Bearer ${KEY}` },
  });

  if (res.status === 429) return;                 // the next tick resumes from the same cursor
  const { events } = (await res.json()) as { events: DeliveryEvent[] };

  for (const ev of events) {
    if (await db.seenEvent(ev.id)) continue;      // event streams are at-least-once
    if (ev.type === "bounced" && ev.code?.startsWith("5.")) await db.suppress(ev.address, ev.code, ev.id);
    if (ev.type === "complained") await db.suppress(ev.address, "complaint", ev.id);
    await db.markSeen(ev.id, ev.occurredAt);
  }

  if (events.length) await db.setCursor("mail-events", events.at(-1)!.occurredAt);
}
```

Two properties carry this job: a cursor that only advances after the batch is written, and a dedupe on the provider's event id, because any at-least-once stream will hand you the same bounce twice sooner or later. A one-minute interval is fine for work orders. If the product needs a reaction inside a second, polling doesn't get you there and a push callback does — that's a genuine trade-off, not a detail to paper over with a shorter interval and a bigger bill.

## When should you skip email for the order alert entirely?

The catch is that email is best-effort by design. Every control above raises the odds and none of them guarantee that a vendor reads anything, so if someone has to be on site for a burst pipe within ten minutes, email is not suitable as the primary channel; an SMS or a push notification is, with the email left as the durable record. When mail itself is the product — campaign editors, segmentation, a template someone non-technical reviews on a Friday — a general backend doesn't support that work and a specialist stack earns its keep. Stick with an SMTP relay when a legacy component can only speak SMTP and rewriting it buys nothing. And if contractual or residency constraints force the mail to stay inside your own network, the self-hosted row comes back with all of its costs intact.

The decision worth writing down isn't which vendor to sign up with. It's which of these failures you're prepared to own: reputation, suppression policy, and the delay between a message being accepted and you finding out what happened to it.

## Further reading

- [RFC 7489: Domain-based Message Authentication, Reporting, and Conformance (DMARC)](https://datatracker.ietf.org/doc/html/rfc7489)
- [RFC 7208: Sender Policy Framework (SPF)](https://datatracker.ietf.org/doc/html/rfc7208)
- [RFC 6376: DomainKeys Identified Mail (DKIM) Signatures](https://datatracker.ietf.org/doc/html/rfc6376)
- [RFC 3463: Enhanced Mail System Status Codes](https://datatracker.ietf.org/doc/html/rfc3463)
- [Apple: use Mail Privacy Protection](https://support.apple.com/guide/iphone/use-mail-privacy-protection-iphf084865c7/ios)
