# Event-Driven Interview Labs: a booking system

**Why this exists:** in a system design interview about a booking system I got pushed on queue resilience, retries, DLQs, "why move from API calls to queues?" and "how do you make sure the DB write and the event both happen?" (the transactional outbox). I knew the words but had never *seen* these failures. Each lab below starts from one of those interview questions, makes the failure happen on my own laptop, fixes it, and finishes with a rehearsed answer.

**Domain:** **event ticketing with timed seat holds**, the same shape as the interview: `booking` · `payment` · `inventory (seats)` · `notification`. A hold expires after N minutes if unpaid.

**Rule for every lab:** ① build the naive version → ② **predict, then break it on purpose** → ③ fix it → ④ write the interview answer in your own words. All of it goes in **one `NOTES.md` per lab** (sections: Built · Experiments (prediction → observed → why) · Interview answer).

---

## Lab 0: Local setup (½ day, free)

One `docker-compose.yml` you reuse across all labs:

| Container | Purpose |
|---|---|
| Postgres | Service databases, outbox tables |
| **MiniStack** (`ministackorg/ministack`, port 4566) | Free MIT-licensed AWS emulator: SQS (FIFO, DLQ), SNS, EventBridge, Step Functions, DynamoDB Streams, Kinesis. No account needed |
| Redpanda or `apache/kafka` (KRaft) + Kafbat UI | Kafka labs |
| Toxiproxy | Inject latency, drop connections |
| `grafana/otel-lgtm` | Traces, metrics, logs in one container |
| Temporal dev server | Saga orchestration lab |

**Stack:** latest everything. **Java 25 + Spring Boot 4.x**, **Gradle Kotlin DSL** in one repo with several projects. Services: `booking` + `payment` in **Java 25**, `inventory` in **Kotlin + coroutines**, `notification` in **Go** (from Lab 1). First task: look up which Spring Cloud AWS 4.x version officially pairs with your Boot version.

**Basic tracing from day one:** add the `grafana/otel-lgtm` container + Boot's OpenTelemetry support in Lab 0, so you can *see* async flows while debugging. Lab 8 is the deep dive. Java talks to MiniStack via AWS SDK v2 with `endpointOverride("http://localhost:4566")`. Spring Cloud AWS (`io.awspring.cloud`) gives you `@SqsListener`.

> MiniStack caveat: its Lambda emulation is documented for Python. Run your Java consumers as normal Spring Boot apps polling SQS, or write the Lambda bits in Python.

**Repo layout:** one evolving system (not a code copy per lab), with a git tag per finished lab. Full layout and IDE setup (IntelliJ for Java/Kotlin, Zed for Go/Docker/scripts) in `docs/dev-setup.md` of the booking-labs repo.
```
booking-labs/
  services/{booking,payment}   Java 25 · services/inventory  Kotlin · services/notification  Go
  compose.yaml · infra/ · load/ (k6) · http/ (.http files) · Makefile
  labs/lab-NN/NOTES.md          built · experiments · interview answer
  git tag lab-NN-done           judge reviews the diff between tags
```

**Lab 0 is done when:**
- [ ] `[observed]` `docker compose up` from a fresh clone brings every container up **healthy**, with no manual steps.
- [ ] `[test]` A Spring Boot 4 "hello" service sends a message to MiniStack SQS and reads it back (Testcontainers smoke test).
- [ ] `[observed]` That round trip shows up as **one trace** in Grafana.
- [ ] `[test]` `./gradlew build` passes.

---

## Lab 0.5: Design the events before writing services (~1.5 h)

**Why:** in the interview, the hard questions only made sense once the events were clear. Decide the vocabulary before building anything.

**You produce `docs/events.md`:**
1. **The flow:** commands and events for the happy path and the failures, e.g. `HoldSeat` → `SeatHeld` → `PaymentRequested` → `PaymentSucceeded`/`PaymentFailed` → `BookingConfirmed` / `SeatReleased`, `HoldExpired`.
2. **Ownership:** which service owns which data and publishes which events (`booking` Java · `payment` Java · `inventory` Kotlin · `notification` Go).
3. **For each event:** name, producer, consumers, key (for ordering), payload fields, whether it's a notification or carries state.
4. **Sync vs async:** which interactions stay synchronous and why.
5. A sequence diagram (Mermaid or Excalidraw).

**Then:** run `/grill-me` on it. The mentor and judge check later labs against this document.

**Done when:** `[observed]` `docs/events.md` exists and has survived one `/grill-me` round.

---

## Lab 1: "Why can't we just call the API? Why move to queues?"

**Build (naive):** `booking` → synchronous REST → `payment` → synchronous REST → `notification`.

**Break it:**
- Use Toxiproxy to give `notification` 5 s of latency. Run k6 at 50 RPS against `booking`.
- Take `notification` down completely.

**What you should see:** booking latency climbs to the slowest dependency's latency. The Tomcat thread pool fills up, `booking` starts timing out, and **a notification outage takes down booking**. That's a cascading failure. Availability multiplies: three services at 99.9% each give about 99.7% end to end.

**Fix:** `booking` saves the booking and publishes `BookingConfirmed` to **SQS** (MiniStack). `notification` consumes at its own pace. Repeat both experiments: booking stays fast, and messages build up in the queue and drain once `notification` comes back.

**Also try:** a burst of 1000 requests. The queue absorbs it (load levelling) while the consumer processes at a steady rate.

**Interview answer skeleton:**
- *Why queues:* temporal decoupling (the consumer can be down), failure isolation, load levelling/absorbing spikes, independent scaling, fan-out to many consumers.
- *What it costs:* eventual consistency (the API returns `202 Accepted` + a status endpoint instead of a final answer), duplicates (at-least-once), ordering, harder debugging/observability, more infrastructure.
- *When NOT to:* the caller needs the answer right now (e.g. "is this seat available?" stays synchronous; "send confirmation email" goes async).
- *How to migrate:* strangler approach. Dual-publish, shadow consumers, then switch over feature by feature. You actually did this with Sonic MQ → SQS, so tell that story.

---

## Lab 2: "How do we make the queue resilient? Retries? DLQs?"

**Build:** `notification` consumes from SQS. Configure a **redrive policy** (`maxReceiveCount = 3`) to a DLQ.

**Break it, one experiment per error type:**

| Experiment | What you should see |
|---|---|
| **Poison message** (malformed JSON) | Fails 3×, then lands in the DLQ. The rest of the queue keeps flowing |
| **Transient error** (downstream 503 the first 2 times) | Succeeds on attempt 3, with nothing in the DLQ |
| **Consumer crashes** (`kill -9`) after processing but before delete | The message reappears after the visibility timeout → **processed twice**. Lab 3 deals with this |
| **Visibility timeout (5 s) shorter than processing (10 s)** | Another consumer picks up the same message while the first is still working → duplicates |
| **Downstream fully down for 10 minutes** | Without backoff, *every* message burns through its retries and the DLQ fills with perfectly good messages |
| **Redrive** the DLQ back to the main queue after fixing the bug | Messages get processed. Now you have a DLQ runbook |

**Fix / what an "error policy" means (this was the answer they wanted):**
1. **Classify errors.** *Transient* (timeouts, 503, throttling) → retry with **exponential backoff + jitter**. *Permanent* (validation, unknown booking ID) → don't retry, send straight to DLQ/parking lot with the reason attached.
2. **Bounded retries** (`maxReceiveCount`), and stretch the visibility timeout per attempt (`ChangeMessageVisibility`) to get backoff on SQS.
3. **DLQ + alarm on DLQ depth > 0**, plus message metadata (error, attempt count, trace ID).
4. **Redrive procedure:** fix the cause, replay from the DLQ, and make sure replay is safe (idempotency → Lab 3).
5. **Circuit breaker in the consumer:** if the downstream is completely down, *stop consuming* (pause the listener) instead of pushing every message into the DLQ.
6. Set the visibility timeout to at least 6× processing time (the AWS guideline for Lambda) and use long polling.

**Kafka version (½ day extra):** Kafka has no per-message redelivery. Compare **blocking retries** (preserve order, block the partition) with **non-blocking retry topics** (`@RetryableTopic` in Spring Kafka: `orders-retry-1`, `-retry-2`, `-dlt`), which give up ordering in exchange for throughput. Understanding that trade-off is what separates a senior answer.

---

## Lab 3: "What if the same message is processed twice?" (idempotency)

**Break it:** send the same `PaymentRequested` message 5 times, concurrently. The naive consumer charges the customer 5 times.

**Fix, two layers:**
- **Idempotent consumer:** a `processed_messages(message_id PK)` table, inserted **in the same DB transaction** as the business change. A duplicate hits the unique constraint → skip. Then test concurrency: two consumers, same message, same millisecond.
- **Idempotent API:** an `Idempotency-Key` header on `POST /bookings` (the Stripe pattern) that stores key → response, so a retried client gets the same booking back instead of a second one.
- Also: make operations *naturally* idempotent where possible (`SET status = PAID`, not `balance = balance - 100`).

**Interview answer:** "Queues are at-least-once, so exactly-once *processing* comes from idempotent consumers, not from the broker. Exactly-once *delivery* doesn't exist end to end."

---

## Lab 4: "How do we make sure the booking is saved AND the event is published?" (transactional outbox)

**Show the bug first (dual-write problem):**
```java
bookingRepo.save(booking);   // 1. DB commit
sqs.send(bookingCreated);    // 2. publish
```
- `kill -9` between 1 and 2 (or throw) → **a booking exists but no event was published**. Payment never happens.
- Swap the order, publish then save, with a failing DB → **a ghost event for a booking that doesn't exist**.
- Put `send` inside `@Transactional` → it still breaks, because the broker isn't part of your DB transaction. (Know why 2PC/XA isn't the answer: slow, poorly supported, and SQS/Kafka don't take part in it.)

**Fix A: polling publisher (build this first, ~1 day):**
1. In ONE transaction: insert `booking` + insert `outbox(id, aggregate_id, type, payload, created_at, published_at NULL)`.
2. A relay job: `SELECT ... WHERE published_at IS NULL ORDER BY id LIMIT 100 FOR UPDATE SKIP LOCKED` → publish → mark published.
3. Run 2 relay instances and check that `SKIP LOCKED` stops them from double-publishing the same rows.

**Fix B: CDC with Debezium (~1–2 days):** Debezium reads the Postgres WAL and its **Outbox Event Router** sends outbox rows to Kafka. No polling, low latency.

**Break it again:** kill the relay after publish but before marking published → **a duplicate event**. That's expected: the outbox gives you *at-least-once*, so consumers must be idempotent (Lab 3). The interviewer wants to hear that the two patterns depend on each other.

**Discuss:** ordering per aggregate (key by `bookingId`), outbox table cleanup, the **inbox pattern** on the consumer side, and "listen to yourself" as an alternative.

---

## Lab 5: "Payment failed after we reserved the seat. Now what?" (sagas)

**Flow:** reserve seat → charge payment → confirm booking → notify. A failure triggers compensations: payment fails → release the seat. The seat hold also needs a timeout (expire unpaid holds after 10 minutes).

1. **Choreography:** each service reacts to events (`SeatReserved` → payment, `PaymentFailed` → inventory releases). Then try to answer "what state is booking #123 in?" and notice how hard it is.
2. **Orchestration with Temporal (Java SDK):** one workflow with explicit compensations, retries and timers. Kill the worker mid-workflow and watch it resume exactly where it stopped.
3. *(Optional)* The same flow as an **AWS Step Functions** state machine on MiniStack.

**Interview answer:** no distributed transactions across services → sagas + compensations + idempotency. Choreography suits simple flows with few steps. Orchestration gives visibility and easier changes for business-critical flows like booking/payment. Also cover the semantic-lock pattern (`PENDING` status) to handle isolation anomalies.

---

## Lab 6: "Two people book the last seat at the same time" (concurrency and ordering)

**Break it:** k6 fires 100 concurrent requests for 1 remaining seat. The naive read-check-write oversells.

**Fixes to compare:** DB unique constraint on `(event_id, seat_id)` · optimistic locking (`@Version`) · `SELECT ... FOR UPDATE` · conditional update `UPDATE seats SET booked=true WHERE id=? AND booked=false` · serialising through **SQS FIFO with `MessageGroupId = seatId`** or a **Kafka key = seatId**.

**Learn:** SQS standard queues don't guarantee order. FIFO orders per message group, with throughput limits. Kafka orders per partition only. Being able to say *"ordering is per key, not global"* scores well in interviews.

---

## Lab 7: "SQS vs Kafka, when would you pick which?"

Rebuild Lab 2's consumer on Kafka and compare hands-on:
- Consumer groups, partitions, a **rebalance** when you kill an instance, **offset reset and replay** of last week's events (impossible with SQS, where deleted means gone).
- Consumer lag in the Kafbat UI. Use a slow consumer to watch the lag grow.

| | SQS | Kafka |
|---|---|---|
| Model | Queue, message deleted after ack | Log, retained, consumers track offsets |
| Replay | No (DLQ redrive only) | Yes |
| Ordering | FIFO per group | Per partition |
| Multiple consumers | Needs SNS fan-out | Consumer groups, natively |
| Ops | Zero | Brokers, partitions, retention (or MSK/Confluent) |
| Pick when | Task queues, work distribution, AWS-native | Event streams, multiple subscribers, replay, high throughput |

Add **SNS → SQS fan-out** and **EventBridge rules** on MiniStack. Also try the Azure equivalents (Service Bus emulator: queues, topics, sessions, built-in DLQ) to see that the concepts carry over across clouds.

---

## Lab 8: "How do you make a request traceable across services?" (distributed tracing)

**The interview question:** "A customer says booking #123 never got a confirmation email. How do you find out what happened?" The "annotation in Spring" they were hinting at is **Micrometer Observation / Tracing** (`@Observed`, `@NewSpan`). The full answer goes further than one annotation.

**Concepts to understand first:**
- **Trace ID** (the whole request journey) · **span** (one step: HTTP call, DB query, message publish/consume) · **parent/child** relationships.
- **W3C `traceparent` header**: the standard way the trace ID travels between services, in HTTP headers *and* message headers/attributes.
- **Correlation ID vs trace ID:** today the trace ID usually *is* the correlation ID. Business IDs (`bookingId`) go on spans as attributes or as **baggage**.
- Spring Boot 3+ uses **Micrometer Tracing** (Spring Cloud Sleuth is dead) with an OpenTelemetry bridge, exporting OTLP → an OTel Collector → Tempo/Jaeger. Spring Boot 4 adds a dedicated OpenTelemetry starter, so check the docs for your version.

**Build:**
1. `booking`, `payment`, `notification` with `actuator` + `micrometer-tracing-bridge-otel` + `opentelemetry-exporter-otlp` (or the Boot 4 OTel starter) → `grafana/otel-lgtm`.
2. Check what you get **for free**: incoming HTTP, `RestClient`/`WebClient` *built from Spring's builder bean*, Kafka (`spring.kafka.template.observation-enabled=true`, `spring.kafka.listener.observation-enabled=true`), Spring Cloud AWS SQS observation, JDBC (via `datasource-micrometer`).
3. Add custom spans: `@Observed(name = "booking.create")` on service methods (requires the `ObservedAspect` bean + AOP starter), or `@NewSpan` + `@SpanTag("bookingId")`. Add `bookingId` as a span attribute and as a low-cardinality metric tag (never as a high-cardinality one).
4. **Logs:** put `traceId`/`spanId` in every log line (MDC, `logging.pattern.correlation`), ship them to Loki, and jump from a log line to its trace in Grafana.
5. Return the `traceId` (or a booking reference) in error responses so support can search for it.

**Break it: each of these silently breaks the trace. Find and fix each one:**

| Experiment | What you'll see | Fix |
|---|---|---|
| `new RestTemplate()` instead of the builder bean | Trace stops at the HTTP call | Inject `RestClient.Builder` |
| `@Async` / `CompletableFuture.supplyAsync` / a virtual-thread executor | Child work shows up as a new, unrelated trace | Context propagation (`ContextPropagatingTaskDecorator`, `ContextSnapshot`) |
| Raw AWS SDK `SqsClient.sendMessage` | Consumer starts a brand-new trace | Inject `traceparent` into **message attributes** yourself, or use the instrumented Spring Cloud AWS template |
| **Outbox relay** (Lab 4) | Publish happens in the relay's own trace, disconnected from the original request | Store `traceparent` in the outbox row and restore it when publishing. This is a strong senior-level detail to bring up |
| Batch consumer (10 messages per poll) | One parent doesn't fit 10 producers | Use **span links** instead of parent/child |
| Message goes to the DLQ | Hard to explain why | Keep the `traceparent` + error in DLQ attributes, then find the original request from the DLQ message |

**Then the production questions:**
- **Sampling:** 100% in dev. In prod use head sampling (e.g. 10%) or **tail sampling** in the OTel Collector (keep 100% of errors and slow traces).
- **Metrics + alerts** (the "how would you *know*" part): RED metrics per endpoint, **queue depth / age of oldest message**, **DLQ depth > 0**, consumer lag, retry rate.
- **Backpressure:** slow downstream + rising queue age → scale consumers, rate-limit, or shed load.
- Baggage and PII: never put personal data in baggage, because it travels to every downstream service and vendor.

**Interview answer skeleton:** "Every request gets a trace ID via OpenTelemetry/Micrometer. It's propagated with the W3C `traceparent` header over HTTP *and* in message headers, including through the outbox. Logs carry the trace ID, so logs, traces and metrics link together. Business IDs are span attributes. We use tail sampling to keep all error traces, and alert on DLQ depth and queue age. Given booking #123, I'd search by bookingId → trace → see it reached the DLQ in `notification` with a template error."

## Optional: run Labs 1, 2 and 7 on real AWS (your free tier)

Do this *after* it works on MiniStack, using the same code with the endpoint override removed.
- Provision with **Terraform** (SQS + DLQ + redrive, SNS, EventBridge, maybe one Lambda) and `terraform destroy` when you're done.
- **First:** create an AWS Budget alert at $1. Check your account's current free-tier terms (AWS moved new accounts to a credit-based free plan in 2025).
- SQS, SNS and Lambda have monthly free allowances (around 1M requests each), so labs cost about nothing **as long as you use long polling** (`WaitTimeSeconds=20`). An idle consumer on short polling can burn millions of requests a month by itself.
- Never create: NAT Gateway, MSK, EKS, RDS left running, Elastic IPs. Run consumers on your laptop, not EC2.

---

## Study list (all free unless marked)

**Read these first, about one evening each:**
1. Amazon Builders' Library: *Timeouts, retries, and backoff with jitter* → Lab 2
2. Amazon Builders' Library: *Making retries safe with idempotent APIs* → Lab 3
3. Amazon Builders' Library: *Avoiding insurmountable queue backlogs* → Labs 2, 8
4. microservices.io: *Transactional outbox*, *Polling publisher*, *Transaction log tailing*, *Idempotent consumer*, *Saga* → Labs 3–5
5. Martin Fowler: *What do you mean by "Event-Driven"?* → overall framing
6. Uber Engineering: *Building Reliable Reprocessing and Dead Letter Queues with Apache Kafka* → Lab 2 (Kafka)
7. Stripe blog: *Designing robust and predictable APIs with idempotency* → Lab 3

**Docs you'll use while building:**
- AWS SQS Developer Guide: visibility timeout, DLQs/redrive, FIFO, long polling
- Spring Cloud AWS (SQS) docs · Spring Kafka "Non-blocking retries" docs
- Debezium Outbox Event Router docs
- Temporal Java SDK docs + free Temporal 101/102 courses
- Confluent Developer: free Kafka 101 and "Event Sourcing & Storage" courses
- enterpriseintegrationpatterns.com (Hohpe): the pattern vocabulary (competing consumers, DLQ, idempotent receiver)

**Books (library or buy one):** *Designing Data-Intensive Applications* (Kleppmann), especially chapters 7, 9 and 11 · *Building Event-Driven Microservices* (Bellemare) · *Microservices Patterns* (Richardson), especially the saga and messaging chapters.

**Interview practice:** after each lab, explain the topic aloud for 3 minutes with a whiteboard sketch. Then have Claude play the grilling interviewer on that topic.

---

## Effort estimates (self-paced, no deadlines)

Rough hours of focused work for someone with your background, including reading, building, breaking and writing notes. At ~1–1.5 h on weekdays plus the odd weekend block (≈6–10 h/week), Labs 0–8 take roughly **2–3 months**, but go at your own pace.

| Lab | Interview question you can then answer | Estimate | Split into sessions like |
|---|---|---|---|
| 0 Setup | — | 4–6 h | compose + health checks · Gradle multi-project skeleton · tracing visible in Grafana |
| 0.5 Event design | (makes every other answer clearer) | ~1.5 h | draft flow + ownership · /grill-me |
| 1 Sync vs async | Why queues instead of APIs? | 6–8 h | sync chain · Toxiproxy + k6 break · SQS fix · notes |
| 2 Retries / DLQ | How do you make the queue resilient? | 8–12 h | redrive + DLQ · one session per error type · backoff/circuit breaker · Kafka retry topics (optional +3 h) |
| 3 Idempotency | What about duplicates? | 5–7 h | consumer dedupe · concurrency test · Idempotency-Key API |
| 4 Outbox | How do you guarantee DB write + event? | 10–14 h | prove the dual-write bug · polling publisher + SKIP LOCKED · Debezium (biggest setup cost) |
| 5 Sagas | What if payment fails halfway? | 10–14 h | choreography · Temporal orchestration · hold expiry |
| 6 Concurrency | Two users, last seat? | 5–7 h | reproduce oversell · compare 4 fixes |
| 7 SQS vs Kafka | Which and why? | 8–10 h | Kafka consumer groups/replay · SNS/EventBridge fan-out · comparison notes |
| 8 Tracing (deep) | How do you trace a request across services? | 6–10 h | one session per propagation breakage |
| AWS real run | "Have you run this for real?" | 3–5 h | Terraform + destroy |
| **Total** | | **~65–95 h** | |

**Recommended order:** 0 → 0.5 → 1 → 2 → 3 → 4 cover every question from that interview (~35–45 h), then 8 → 5 → 6 → 7.
