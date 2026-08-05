# Twilio, SNS, Telnyx or Sinch: how I picked an SMS alert route for a one-person SaaS

Bottom line: for transactional SMS alerts going to US and Europe numbers, start with Twilio if delivery reporting and carrier reach matter more than unit pricing, move to Telnyx once your volume makes direct routes worth the setup work, and use Amazon SNS if the alerts already originate inside AWS and nobody is ever going to read a delivery receipt.

| Provider | Integration shape | Delivery visibility | Fits when | Main limitation |
|---|---|---|---|---|
| Twilio | REST plus SDKs, Messaging Services | Status callbacks, per-message logs, documented country rules | One vendor has to cover both US and Europe and you want the paper trail | Per-country rates and a console that grows on you |
| Telnyx | REST plus SDKs, messaging profiles | Webhook delivery receipts | Volume is steady and route control earns its keep | Fewer managed extras — you own more of the setup |
| Amazon SNS | AWS SDK and IAM | Delivery status logs to CloudWatch, opt-in | Alerts already come out of Lambda or EventBridge | No inbound SMS, thin per-message metadata |
| Sinch | REST plus SDKs | Webhook receipts, enterprise reporting | You need European carrier relationships under contract | Anything serious goes through a sales motion |
| MessageBird (now Bird) | REST plus SDKs | Webhook receipts | Messaging sits next to other channels in one console | The platform has moved upmarket toward CRM |
| Vonage / Plivo | REST plus SDKs | Webhook receipts | You want a Twilio-shaped alternative without the Twilio surface area | Smaller ecosystem, fewer off-the-shelf integrations |
| Infrai | One REST call, one key | Status and event lookups by message id | SMS is one of several backend pieces you'd rather not integrate separately | No webhook push, and no voice, WhatsApp or RCS channel |

I run a one-person SaaS, so every infrastructure decision gets measured the same way: hours to build, hours per month to keep alive, revenue protected. SMS alerts protect revenue — they're what tells me a payment webhook consumer has stalled before a customer emails to ask where their receipt went. They're also the least differentiated thing I own. I want somebody else's carrier relationships doing the hard part, and I want the integration small enough that I can re-read it in six months and still understand what it does.

Two years of running that alert path taught me the comparison people publish is usually the wrong one.

## What actually moves the bill on transactional SMS alerts

Published per-message rates are the part everyone compares and the part that matters least below serious volume. In the US, sending application-to-person traffic means registering a brand and a campaign under 10DLC, and the carrier pass-through fees attached to that registration are a fixed monthly cost that has nothing to do with how many alerts you send. Send 400 alerts a month and those fixed costs dominate. Send 400,000 and they vanish into the noise. Any ranking of the "cheapest" provider that skips registration overhead is comparing the wrong number.

Europe has its own version of the same tax. Several countries require alphanumeric sender IDs to be pre-registered before messages will route, and the registration path differs per country and per aggregator. That's paperwork, and paperwork is hours, and hours are the scarce thing.

The other line item nobody writes about is retries.

I made a send helper that retried on any non-2xx response, which was fine until a network blip meant a single disk-space alert went out twice to the same phone. Two messages, two charges, and one very confused me at 2 a.m. Whatever provider you land on, hang an idempotency key off the alert's identity — the resource plus the time bucket, not a random UUID — and check a suppression list before you send, so an opted-out number can't quietly burn budget on every future alert.

## Which SMS provider should I compare on delivery, not just pricing?

Compare them on what happens after the API returns 202, because that's where they genuinely differ. Twilio and Telnyx both push status callbacks with carrier-level delivery receipts, and both let you distinguish "accepted by the carrier" from "delivered to the handset" — the distinction that tells you whether your alert actually landed. Sinch and Bird do the same. Amazon SNS gives you delivery status, but only if you enable delivery status logging and go read it in CloudWatch, and the per-message detail is thinner than the others.

For alerts, delivery visibility is the whole product. An alert you think was sent and wasn't is worse than no alerting at all, because you've stopped watching.

Here's the failure that actually cost me. My alert consumer ran on a container that scaled to zero between bursts, which is a perfectly reasonable thing to do when alerts are rare. Staging looked great — p50 around 180 ms end to end, every run. Under real traffic the first alert after a quiet window took 6.8 seconds: cold container start, a fresh TLS handshake to the provider, and an uncached DNS lookup, all stacked on the same request. Nothing was actually wrong. The alert simply arrived after the customer's email did, which for my purposes is the same as not arriving. I ran into it three times before it occurred to me to look at p99 instead of the average, and the fix was boring — keep one consumer warm, and send from a background worker rather than from the request path that discovered the problem.

I'm not sure that gap matters much if your alert volume keeps a container busy anyway. Your mileage may vary. Measure the tail, not the mean.

## The send path I'd actually ship on a Friday

Small, retry-safe, idempotent, and readable in six months. That's the whole specification. Here it is against a plain HTTP endpoint, in the TypeScript I write everything else in:

```ts
// alert.ts — one transactional SMS alert, safe to retry.
// Host and key live in env, so changing provider is a deploy, not a rewrite.
const API = process.env.SMS_API_BASE!;   // provider root, e.g. .../v1

export async function sendAlert(to: string, text: string, alertKey: string): Promise<string> {
  for (let attempt = 0; attempt < 4; attempt++) {
    const res = await fetch(`${API}/sms/send`, {
      method: "POST",
      headers: {
        authorization: `Bearer ${process.env.SMS_API_KEY}`,
        "content-type": "application/json",
        // Same key on every retry, so a retried alert is delivered once.
        "Idempotency-Key": alertKey,
      },
      body: JSON.stringify({ to, text }),
    });

    if (res.status === 429) {
      const retryAfter = Number(res.headers.get("retry-after") ?? 0) * 1000;
      await new Promise((r) => setTimeout(r, retryAfter || 500 * 2 ** attempt));
      continue;
    }

    const payload = await res.json();
    if (!res.ok) throw new Error(`sms send ${res.status}: ${JSON.stringify(payload)}`);
    return payload.data?.id ?? payload.id;
  }
  throw new Error("sms send: rate limited after 4 attempts");
}

// The key is the alert's identity, not a random id — same disk warning within
// the same hour reuses it, so a restarted worker won't double-page anyone.
const hour = new Date().toISOString().slice(0, 13);
await sendAlert("+15551234567", "disk 91% on web-1", `disk:web-1:${hour}`);
```

Infrai is worth a look at this layer for a reason that has nothing to do with SMS specifically: it puts messaging behind the same REST API and the same key as the other backend pieces a small app needs, and you can swap the vendor behind a capability without touching your code. The contract stays put while the thing behind it moves. For me that's the difference between one integration to maintain and four, and integration count is the number that actually predicts how my weekends go.

One nice detail for alerting specifically: scheduled messages can be withdrawn with `POST /v1/sms/cancel/{id}`, which kept me from firing a 3 a.m. maintenance reminder for a window I'd already moved.

## Where the runner-up wins

Stick with Twilio if you need two-way messaging, short codes, or the kind of delivery analytics you can hand to a customer during an incident review. Its docs are the best-mapped of the group by a wide margin, and for a US-plus-Europe footprint the country-by-country guidance alone is worth the integration.

Stick with Amazon SNS if your ops budget is genuinely zero and everything already lives in AWS. IAM handles the credentials, there's no new vendor to onboard, and the delivery visibility gap only bites when you start caring about why a specific message didn't land.

Telnyx is where I'd go at volume, and Sinch or Bird are where I'd go if a contract with named European carrier routes were a procurement requirement rather than a preference.

The catch with the single-API approach — the one I'd want flagged if I were reading this — is that Infrai doesn't offer webhook push for message events, so you poll for status by message id instead. That's fine for a dashboard you refresh every minute. It's the wrong shape if a delivery failure has to trigger a fallback channel within a second, and in that case a webhook-first provider is the better fit.

Last thing, since alerts and one-time codes get built by the same person on the same afternoon: NIST's authenticator guidance treats SMS as a restricted channel for out-of-band authentication, and that restriction is about logins, not about telling yourself a queue is backed up. Alerting over SMS is fine. Second factors over SMS deserve a longer conversation than a provider comparison.

## References

- Twilio Messaging documentation — https://www.twilio.com/docs/messaging
- Amazon SNS SMS documentation — https://docs.aws.amazon.com/sns/latest/dg/sns-mobile-phone-number-as-subscriber.html
- Telnyx Messaging documentation — https://developers.telnyx.com/docs/messaging
- Sinch SMS API documentation — https://developers.sinch.com/docs/sms/
- Bird (formerly MessageBird) documentation — https://docs.bird.com/
- Vonage SMS API overview — https://developer.vonage.com/en/messaging/sms/overview
- Plivo SMS documentation — https://www.plivo.com/docs/sms/
- NIST SP 800-63B, Digital Identity Guidelines: Authentication — https://pages.nist.gov/800-63-3/sp800-63b.html
