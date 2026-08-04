# React Native Verification Login: API Design for Autofill, Retries, and Abuse Controls

If you just want the recommendation: keep OTP challenge state in your backend, put SMS delivery behind a replaceable adapter, and let the React Native app do only three things: request, autofill, and verify a code. For a one-person SaaS, this gives me the best balance of weekly shipping speed and control over resend abuse.

| Choice | I would use it when | Main trade-off |
|---|---|---|
| Backend-owned challenge plus SMS adapter | Login policy is product-specific and I need one auditable state machine | I own expiry, attempt limits, and operational alerts |
| Managed verification service | Telecom operations are consuming feature time | Policy and portability are constrained by the service boundary |
| App-generated or app-validated codes | Never for authentication | The client cannot be trusted with the secret or the decision |

My default is the first row. The runner-up is sensible when delivery operations cost more revenue-per-hour than the extra control returns. I ship weekly, so I outsource undifferentiated message transport, but I keep the login decision on my side of the API.

## How should a mobile app backend API handle autofill and abuse prevention?

The app should never decide that an OTP is valid. It asks the backend to create a challenge, receives an opaque challenge ID plus display-safe timing data, and later submits that ID with the code. The backend owns the phone normalization rule, code digest, expiration, resend eligibility, attempt count, and final consume operation. One successful verification consumes the challenge. A replay gets the same generic rejection as any other invalid state.

That boundary matters more than the SMS provider choice. A mobile client can be modified, requests can be replayed, and local timers can be reset. The countdown on screen is UX, not enforcement. I treat every start, resend, and verify request as hostile input — including requests from my own released app.

For autofill, I use a single code field with the platform's one-time-code hints and accept a pasted full value. I don't split six digits into six clever boxes. Those boxes add focus bugs, accessibility friction, and paste handling for no revenue upside. The app submits only after it has the complete expected length; the server still performs the real check.

The response vocabulary stays small: accepted, pending, verified, expired, or rejected. I avoid telling an attacker whether a phone number has an account, whether the code was close, or which limit fired. The UI can still say, “We couldn't verify that code. Request a new one if it expired.” It's plain, and support can understand it. There is one subtle contract: a request being accepted is not proof that a text arrived. “Accepted” means the backend recorded the operation and handed delivery to its adapter. Verification is the only proof that the user received the code. That distinction shapes both retries and monitoring.

The server decides.

## Two criteria beat a long vendor checklist

My first criterion is state ownership. I want one server-side record to answer: which challenge is current, when may it be resent, how many verification attempts remain, and has it already been consumed? If those answers live partly in the app, partly in a queue, and partly in provider callbacks, debugging login becomes archaeology. A compact state machine gives me one place to enforce transitions and one event stream to inspect.

My second criterion is failure visibility. I once treated an HTTP 200 as proof that an SMS side effect had happened. It hadn't. I found out 3 hours later, after a customer emailed me and I traced an accepted request with no corresponding delivery event. I'm not sure why I trusted the transport response that much; it was a category error, not useful optimism. Now I correlate a locally generated operation ID across challenge creation, the delivery adapter, and verification.

This changes what I measure. I care about the ratio from accepted challenge to successful verification, broken down by coarse region and app version; resend volume per challenge; time from first request to verification; and how often limits reject an action. I don't put raw phone numbers or OTP values in logs. A keyed, rotating pseudonymous identifier is enough to group events during an incident without turning logs into an authentication-secret archive.

Keep it boring.

Cost belongs here, but not as a price-per-message beauty contest. A cheap send that creates support work is expensive for a solo founder. I compare engineering hours, delivery visibility, portability, and the blast radius of a provider change. Your mileage may vary if you already have an operations team; I don't, so every hour spent reconciling delivery records competes directly with shipping a paid feature.

## A small TypeScript contract keeps the edges honest

The public contract has two operations and one retry mode rather than provider concepts. `start` creates a challenge. Calling it again with the same idempotency key returns the same logical operation. `verify` atomically checks and consumes. Another delivery attempt is a start request tied to the existing challenge, and the backend decides whether it is eligible.

Retries are writes.

```ts
type StartOtpInput = {
  phone: string;
  idempotencyKey: string;
  priorChallengeId?: string;
};

type StartOtpResult = {
  challengeId: string;
  status: "accepted";
  retryAfterSeconds: number;
};

type VerifyOtpInput = {
  challengeId: string;
  code: string;
};

type VerifyOtpResult =
  | { status: "verified"; sessionToken: string }
  | { status: "rejected" | "expired" };

interface OtpBackend {
  start(input: StartOtpInput): Promise<StartOtpResult>;
  verify(input: VerifyOtpInput): Promise<VerifyOtpResult>;
}

async function submitAutofilledCode(
  backend: OtpBackend,
  challengeId: string,
  autofilledValue: string,
): Promise<VerifyOtpResult> {
  const code = autofilledValue.replace(/\D/g, "");
  if (code.length !== 6) return { status: "rejected" };
  return backend.verify({ challengeId, code });
}
```

In one current project, my policy is a 10-minute expiry, a 60-second resend floor, five verification attempts per challenge, and additional rolling limits per phone pseudonym, account, IP range, and device signal. Those are implementation settings, not universal constants. I keep them server-configured so an app release isn't required to tune them. I also cap the number of live challenges for one destination and invalidate older ones when policy creates a replacement.

The critical database operation is conditional: verify only if the challenge is live, unused, below its attempt ceiling, and its stored digest matches; then mark it consumed in the same transaction. The code sample leaves storage and transport behind interfaces on purpose. They are the parts I may replace. The state transition is the part I need to reason about.

For tests, I use a fake clock and fake delivery adapter. I cover duplicate starts with the same idempotency key, concurrent verify calls, resend before and after eligibility, expiry at the exact boundary, and a delivery acceptance with no later verification. Then I run one thin integration test through the real adapter in staging.

## When is the runner-up the better OTP login backend?

A managed verification service is the better choice when I can't justify owning delivery routing, regional sender requirements, fraud tuning, and on-call diagnosis. The catch is that its challenge lifecycle and abuse controls may not match a product with unusual account recovery rules or multiple authentication factors. I would stick with the backend-owned design when those rules are differentiating behavior, when audit events must fit an existing security model, or when switching message transport without changing login semantics is important.

The reverse is also true. My preferred design is not suitable when I lack the time to operate the state machine carefully. Owning it means migrations, concurrency tests, secret handling, monitoring, retention rules, and incident drills. A clean interface doesn't erase that work — it only contains it. If message delivery spans many countries and I have no telecom expertise, the managed boundary can be the more honest decision.

I also wouldn't force SMS into every authentication job. An authenticator app or a passkey may fit users who can enroll ahead of time, while email can be a recovery channel if its sender authentication and reputation are operated carefully. Channel choice changes threat and delivery assumptions, so I don't hide it behind one generic “send code” button in the domain model.

Before deployment, the test suite has a pass-or-fail job: duplicate requests are idempotent, expired and consumed codes cannot establish authenticated login state, concurrent verification has one winner, retry limits survive an app restart, logs contain no code or raw destination, and an accepted delivery operation can be traced without treating it as receipt. Once those tests pass and the alerting watches the accepted-to-verified funnel, I can ship the login flow and get back to the feature customers actually pay for.

No heroics. Just a narrow contract, explicit state, and evidence when the side effect stops short of the user.

## References

- https://www.twilio.com/docs/sms
- https://support.google.com/a/answer/81126
