---
title: 'Idempotency in Distributed APIs: The Practical Path to “Exactly-Once” Outcomes'
description: 'Retries are inevitable. Idempotency is how you prevent duplicate side effects and get reliable outcomes in at-least-once systems.'
pubDate: 2026-03-15
---

Retries are not a bug — they’re a fact of life in distributed systems.

Networks drop packets. Clients time out. Proxies reset connections. Deployments roll. If you run production systems long enough, your “one request” becomes “maybe many attempts.”

Without idempotency, those attempts can create **duplicate side effects**:
- double charges
- duplicate jobs
- repeated emails/notifications
- corrupted counters

Teams often talk about “exactly-once,” but most real systems don’t provide exactly-once delivery end-to-end. What you can build reliably is:

**at-least-once delivery + idempotent effect handling** → *exactly-once outcomes* (in practice).

This post explains how.

## First principles: what idempotency actually means

An operation is **idempotent** if doing it multiple times has the same effect as doing it once.

In HTTP terms, methods like `GET`, `PUT`, and `DELETE` are *defined* as idempotent in the semantics (though your implementation can still break this!). `POST` is generally not idempotent.

But for business APIs, the more useful definition is:

> **The side effect is idempotent** when repeated attempts don’t duplicate real-world outcomes.

Example: “Create payment” is *not* naturally idempotent. “Create payment with idempotency key X” can be.

## Why retries happen (and why you should assume them)

Retry triggers are everywhere:
- client timeouts (the server may have processed it, but the client didn’t see the response)
- load balancer/proxy resets
- transient 5xx errors
- flaky mobile networks
- background job retries

So if your design assumes “requests run once,” it will fail under real traffic.

## The core failure mode: duplicate side effects

The dangerous scenario is:

1) Client sends request
2) Server processes it and commits the side effect
3) Response is lost / client times out
4) Client retries
5) Server processes again → **duplicate side effect**

This is why “just retry” is not safe unless the server can detect duplicates.

## The practical pattern: Idempotency keys

The most common production approach:

- Client generates an **idempotency key** for a logical operation (e.g. `payment:create`)
- Client sends the key with the request
- Server stores the result of the first successful execution and returns the same result for subsequent attempts with the same key

Stripe popularized this pattern for payments, because “double charge” is a career-ending bug.

### Minimal server-side requirements

To implement idempotency keys correctly, you need:

- a storage record keyed by `(idempotency_key, user_or_tenant, endpoint)`
- status: `in_progress | completed | failed`
- stored response (or at least the created resource ID)
- TTL/retention policy

### Concurrency rule (important)

If two identical requests arrive at the same time:
- one wins and performs the side effect
- the others must wait/return the cached result

This usually means a **unique constraint** and/or **transactional lock** on the idempotency record.

## “Exactly-once delivery” vs “exactly-once outcomes”

Exactly-once delivery is hard because it requires coordination across clients, servers, brokers, and storage. Many systems give you at-least-once, because it’s simpler and more robust.

So your job is to make the *effect* idempotent.

If you use Kafka or any message queue, you should treat “duplicate messages” as normal. Your consumer should be written to handle duplicates safely.

## Common implementation strategies

### Strategy A — Store the first response (request replay)
Store the full response payload for the first successful request and replay it for retries.

Pros:
- simplest client behavior (same response)

Cons:
- response storage can be large
- careful with PII / compliance

### Strategy B — Store the created resource ID (resource replay)
Store the canonical resource identifier (e.g. `payment_id`) and reconstruct response.

Pros:
- less storage

Cons:
- response may change over time unless you snapshot fields

### Strategy C — Make the write itself naturally idempotent
Example: `PUT /users/{id}` with a full desired state, rather than `POST /users`.

Pros:
- aligns with HTTP semantics

Cons:
- not always possible (some operations are inherently “create new event”)

## Where idempotency fails (the stuff people forget)

1) **Key scope mistakes**
If your idempotency key isn’t scoped by tenant/user + operation, you can get collisions.

2) **Non-deterministic operations**
If the operation depends on time/randomness, retries may create different outcomes unless you persist the decision.

3) **Partial failures**
What if you wrote to DB but failed to publish an event? Or published event but failed to respond?

This is where outbox patterns and careful transaction boundaries matter.

4) **TTL too short**
If your idempotency record expires while clients still retry, duplicates come back.

## A simple checklist for production APIs

- [ ] Identify operations with dangerous side effects (payments, emails, provisioning, job creation)
- [ ] Add an idempotency key contract to those APIs
- [ ] Enforce a unique constraint on `(tenant, operation, key)`
- [ ] Store at least the resulting resource ID
- [ ] Handle concurrent duplicates safely
- [ ] Choose a retention period aligned with client retry behavior
- [ ] Add metrics: duplicate rate, key collisions, in-progress timeouts

## Final take

Retries are inevitable. **Idempotency is the mechanism that turns unreliable delivery into reliable outcomes.**

Don’t chase mythical exactly-once delivery everywhere.
Design for at-least-once, and make the effects idempotent.

---

## References

- RFC 9110 — HTTP Semantics (idempotent methods): <https://www.rfc-editor.org/rfc/rfc9110>
- AWS Builders’ Library — Making retries safe with idempotent APIs: <https://aws.amazon.com/builders-library/making-retries-safe-with-idempotent-APIs/>
- Stripe Docs — Idempotent requests: <https://docs.stripe.com/api/idempotent_requests>
- Confluent — Exactly-once semantics in Kafka: <https://www.confluent.io/blog/simplified-robust-exactly-one-semantics-in-kafka-2-5/>
