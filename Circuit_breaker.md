# System Design Patterns — Step-by-Step Study Guide

**Repository:** `system-design-patterns`  
**Focus:** Resilient distributed systems, microservices, APIs and event-driven architecture  
**Technology examples:** C#, .NET, HTTP APIs, Kafka

---

## 1. Learning Roadmap

This guide covers seven important patterns used to build reliable distributed systems.

| Step | Pattern | Main problem it solves |
|---|---|---|
| 1 | Circuit Breaker | Repeated calls to an unhealthy dependency |
| 2 | Bulkhead | One dependency consuming all available resources |
| 3 | Backpressure | Incoming work exceeding processing capacity |
| 4 | Load Shedding | Too much traffic for the system to handle safely |
| 5 | Idempotency | Duplicate business operations caused by retries |
| 6 | Retries, Exponential Backoff and Jitter | Temporary failures and unreliable networks |
| 7 | Timeouts and Deadlines | Requests waiting too long or consuming resources indefinitely |

### How to study each pattern

For every pattern:

1. Understand the problem it solves.
2. Learn the real-world analogy.
3. Trace what happens when a request fails.
4. Study the example architecture.
5. Identify the trade-offs.
6. Answer the interview question.
7. Implement a small example and test failure scenarios.

---

# Step 1: Circuit Breaker Pattern

## 1.1 What problem does it solve?

A Circuit Breaker prevents an application from repeatedly calling a downstream service that is failing or responding too slowly.

Without it, repeated failed requests can consume threads, connections and other resources, potentially causing a cascading failure.

**Simple analogy:** An electrical circuit breaker disconnects a faulty circuit to prevent further damage.

## 1.2 Real-time example: Payment Service

Imagine an e-commerce application with an Order Service that calls a Payment Service.

The Payment Service is experiencing an outage.

Without a circuit breaker:

- The Order Service continues sending payment requests.
- Each request waits for a timeout.
- More requests accumulate.
- Application resources become exhausted.
- Other order operations may also become unavailable.

With a circuit breaker, the application temporarily stops making calls to the failing Payment Service.

## 1.3 Understand the three states

### State 1 — Closed

- Requests are allowed through.
- The application monitors failures.
- The breaker opens when its configured failure criteria are met.

### State 2 — Open

- Calls to the downstream service are blocked.
- The application fails fast instead of waiting for another likely failure.
- A fallback or appropriate error response is returned.

### State 3 — Half-open

- After a waiting period, the breaker allows a limited number of test calls.
- If recovery is successful, it closes.
- If failures continue, it opens again.

**State transition:**

`Closed → Open → Half-open → Closed`

The breaker may return to Open if the recovery test fails.

## 1.4 What should the application return?

For a mandatory payment operation, the application must not report that an order is paid when payment authorization has not been confirmed.

Depending on the business design, it can:

- Return a temporary service error.
- Keep the order in a pending state.
- Allow the customer to check the operation's status later.

A fallback is useful only if it preserves business correctness.

## 1.5 .NET implementation direction

For modern .NET applications, `Microsoft.Extensions.Http.Resilience` provides HTTP resilience pipelines that include circuit-breaking capabilities.

Example of registering a resilient HTTP client:

```csharp
builder.Services
    .AddHttpClient<PaymentClient>()
    .AddStandardResilienceHandler();
```

This registers a standard resilience pipeline, not just a circuit breaker. Review the package version and configure the pipeline according to your application's requirements.

## 1.6 When should you use it?

Use a circuit breaker when:

- Calling an external API.
- Calling another microservice.
- Depending on an unreliable third-party provider.
- Repeated failures could consume application resources.

It is not a replacement for timeouts. A circuit breaker needs failure signals, and a timeout helps ensure a slow request eventually produces one.

### Interview question

Your Payment Service has repeatedly timed out. The circuit breaker is now Open. What should happen to the next payment request?

**Expected answer:** Fail fast without making another downstream call, return an appropriate response, and preserve the ability to recover safely. The application must not assume that a previous payment failed merely because it timed out.

---

# Step 2: Bulkhead Pattern

## 2.1 What problem does it solve?

The Bulkhead Pattern isolates resources so a problem in one dependency cannot consume all the resources required by other dependencies.

**Simple analogy:** A ship has separate watertight compartments. Damage to one compartment should not sink the entire ship.

## 2.2 Real-time example: Shared application resources

Suppose an e-commerce application calls three services:

- Payment Service
- Inventory Service
- Notification Service

The Payment Service becomes very slow.

If all three dependencies share the same unrestricted pool of outbound requests, payment calls may consume the available connections or concurrency slots.

Inventory calls can then fail even though the Inventory Service itself is healthy.

With bulkheads, each dependency has a controlled resource allocation.

For example, an application might configure separate concurrency limits for payment and inventory calls. If payment reaches its limit, excess payment work is rejected or deferred rather than consuming all the capacity reserved for inventory.

These limits are illustrative; actual values must come from load testing and capacity planning.

## 2.3 How to design a bulkhead

1. Identify the dependencies that have different failure characteristics.
2. Decide which resources need isolation: concurrency, connection pools, worker pools or queues.
3. Set safe limits for each dependency.
4. Define what happens when a limit is reached.
5. Monitor rejections, queue depth and resource utilisation.
6. Test the system with one deliberately slow dependency.

## 2.4 Bulkhead vs Circuit Breaker

| Bulkhead | Circuit breaker |
|---|---|
| Limits resource consumption | Stops calls after specified failure conditions |
| Protects against resource exhaustion | Protects against repeated calls to an unhealthy dependency |
| Can reject excess work even if the dependency is healthy | Can block calls even when capacity is available |
| Focuses on isolation | Focuses on failure detection and temporary isolation |

These patterns work well together.

A bulkhead limits how much work can be in flight. A circuit breaker can stop additional calls when the dependency becomes unhealthy.

### Interview question

Your Payment Service is slow and consumes most outbound-call capacity. Inventory is healthy but cannot obtain a connection.

**Expected answer:** Isolate the resources allocated to each dependency using an appropriate bulkhead strategy. This prevents payment traffic from exhausting the resources needed by inventory.

---

# Step 3: Backpressure

## 3.1 What problem does it solve?

Backpressure addresses situations where incoming work arrives faster than the system can process it.

**Simple analogy:** A supermarket receives customers faster than its cashiers can serve them. If nothing changes, the queue continues growing.

## 3.2 Understand arrival rate and processing rate

Let:

- `λ` = incoming work per second.
- `μ` = completed work per second.

When the arrival rate exceeds the processing rate for a sustained period, backlog grows.

For example:

- Producers publish 10,000 events per second.
- Consumers process 6,000 events per second.
- The backlog grows by 4,000 events per second.

Calculation:

```text
Backlog growth = Arrival rate - Processing rate

               = 10,000 - 6,000

               = 4,000 events/second
```

Over five minutes, assuming constant rates:

```text
4,000 × 300 = 1,200,000 events
```

The backlog grows by approximately 1.2 million events.

## 3.3 Real-time example: Kafka consumers

Imagine a Kafka topic receiving order events faster than your .NET consumer can process them.

A sensible response is to:

1. Measure consumer lag and processing latency.
2. Determine whether the bottleneck is CPU, database capacity, external calls or something else.
3. Limit in-flight work and concurrency where necessary.
4. Increase consumer capacity if the workload and partitioning allow it.
5. Reduce incoming work or apply upstream flow control where supported.
6. Define what happens if the backlog exceeds safe limits.

Kafka can buffer events, but the buffer is not infinite. Storage, retention and the time required to catch up still matter.

## 3.4 Common backpressure techniques

- Limit the number of concurrent operations.
- Bound in-memory queues.
- Pause or reduce consumption where supported.
- Apply flow control between producers and consumers.
- Scale consumers when additional capacity is available.
- Reject or defer excess work if the system cannot safely accept it.

**Important:** Increasing concurrency is not always the solution. If a database is already overloaded, adding consumers may increase the pressure on that database.

### Interview question

Your consumer processes 6,000 events per second, while producers publish 10,000 events per second. The backlog continues to grow.

**Expected answer:** Control work intake, investigate the bottleneck and increase effective processing capacity where possible. Do not assume that unlimited buffering will solve the problem.

---

# Step 4: Load Shedding

## 4.1 What problem does it solve?

Load shedding deliberately rejects or drops selected work when demand exceeds the system's safe capacity.

The goal is to preserve critical functionality instead of allowing every operation to fail under overload.

**Simple analogy:** An emergency department prioritises urgent patients when demand exceeds available treatment capacity.

## 4.2 Real-time example: E-commerce traffic spike

Imagine your e-commerce application is experiencing a major traffic spike.

The application supports:

- Checkout and order placement.
- Product recommendations.
- Search suggestions.
- Analytics events.

During overload, recommendations and suggestions may be less important than the core purchasing journey.

A load-shedding strategy can reduce or reject selected nonessential work while preserving capacity for critical operations.

For example:

```text
System is overloaded
        |
        v
Are resources above safe limits?
        |
        v
Reduce or reject nonessential work
        |
        v
Protect capacity for critical operations
```

## 4.3 How to design load shedding

1. Identify critical and noncritical operations.
2. Define admission limits using signals such as queue depth, latency and available concurrency.
3. Reject or defer selected work when safe limits are exceeded.
4. Return an appropriate response, such as HTTP 503 for temporary unavailability.
5. Restore normal operation gradually when capacity recovers.

Load shedding should not randomly discard business-critical operations.

For example, a payment request that has an unknown outcome needs careful status reconciliation. Simply dropping the request can leave the customer unsure whether they were charged.

## 4.4 Load shedding vs Backpressure

| Backpressure | Load shedding |
|---|---|
| Controls the flow of work | Rejects or drops selected work |
| Tries to prevent overload from growing | Protects the system when demand exceeds safe capacity |
| Can slow upstream processing | Can refuse incoming requests |

They can be used together.

### Interview question

Your application is near CPU saturation. Product recommendations are expensive, but customers still need to place orders.

**Expected answer:** Temporarily disable or defer nonessential recommendation work, control excess traffic, and preserve resources for core order operations. Do not promise that every order will succeed regardless of available capacity.

---

# Step 5: Idempotency

## 5.1 What problem does it solve?

Idempotency ensures that repeating the same logical operation does not create repeated business effects.

**Simple analogy:** You submit a bank transfer, but the confirmation screen times out. You submit again because you are unsure whether the transfer succeeded.

Without protection, the transfer could happen twice.

With idempotency, the application recognises the repeated operation and prevents a duplicate effect.

## 5.2 Real-time example: Payment API

A client sends a payment request:

```http
POST /payments
Idempotency-Key: payment-abc123
Content-Type: application/json
```

The request body might contain:

```json
{
  "orderId": "ORD-101",
  "amount": 100
}
```

The payment service uses the idempotency key to identify the logical payment operation.

### First request

1. Validate the request.
2. Check whether the idempotency key already exists.
3. Atomically claim the key and record the operation.
4. Process the payment.
5. Persist the outcome.
6. Return the result.

### Duplicate request

If the client retries with the same key:

- If the operation has completed, return the stored result.
- If it is still processing, return its current status or a suitable response.
- If the operation failed, apply the defined recovery policy.

The service must not blindly create another payment.

## 5.3 Production implementation requirements

A robust design needs more than an in-memory dictionary.

- Store keys durably.
- Enforce uniqueness atomically to handle concurrent requests.
- Associate the key with the intended operation and request payload.
- Reject reuse of a key with a different payload.
- Define retention and expiry rules.
- Coordinate with the external payment provider when necessary.

A timeout does not prove that a payment failed. The provider might have processed the payment but lost the response.

## 5.4 HTTP methods and idempotency

| Method | Idempotent by HTTP semantics? | Example |
|---|---|---|
| GET | Yes | Read an order |
| PUT | Yes | Set a delivery address |
| DELETE | Yes | Delete a resource |
| POST | Not generally | Create an order or payment |

A POST endpoint can be made idempotent through application-level design.

### Interview question

A payment succeeds, but the HTTP response is lost. The client retries using the same idempotency key.

**Expected answer:** Recognise the existing operation and return its stored result or current status. Do not create a duplicate payment. Ensure that concurrent retries cannot bypass the uniqueness check.

---

# Step 6: Retries, Exponential Backoff and Jitter

## 6.1 What problem does it solve?

Retries can recover from temporary failures, but poorly designed retries can make an outage worse.

**Simple analogy:** If a telephone line is busy, calling again after a short wait is reasonable. Redialling continuously at the same time as thousands of other callers is not.

## 6.2 Exponential backoff

Exponential backoff increases the delay between successive retries.

Example using a one-second initial delay:

| Retry | Delay before retry |
|---|---:|
| 1 | 1 second |
| 2 | 2 seconds |
| 3 | 4 seconds |
| 4 | 8 seconds |

A common formula is:

```text
delay = min(maxDelay, initialDelay × 2^(retryNumber - 1))
```

The maximum delay prevents the waiting time from growing indefinitely.

## 6.3 Why jitter is necessary

Suppose 10,000 clients experience the same outage.

If every client waits exactly one second and then retries, all 10,000 requests may arrive together.

That creates another burst of traffic.

Jitter adds randomness to retry delays, spreading requests over time.

For example, with full jitter:

```text
delay = random(0, exponentialBackoffDelay)
```

A retry with a two-second maximum delay might occur at any random time between zero and two seconds.

## 6.4 When should a request be retried?

| Failure | Typical decision |
|---|---|
| Temporary network error | Retry may help |
| HTTP 429 | Respect Retry-After when provided |
| HTTP 503 | Retry may help within the deadline |
| HTTP 400 | Usually do not retry unchanged |
| HTTP 401 | Resolve authentication problems |
| Payment timeout with unknown outcome | Retry safely with the same idempotency key |

Always consider the operation's safety and the specific failure type.

## 6.5 Common retry mistakes

- Retrying indefinitely.
- Retrying nontransient errors.
- Retrying immediately without backoff.
- Generating a new idempotency key for every payment retry.
- Allowing multiple service layers to retry independently without considering the total attempt count.

If three layers each allow three attempts, one original request can cause up to 27 downstream attempts in the worst case.

## 6.6 Recommended retry workflow

```text
Request fails
     |
     v
Is the failure transient?
     |
     +-- No --> Return or handle the error
     |
     +-- Yes
           |
           v
Is the operation safe to retry?
           |
           +-- No --> Reconcile outcome or use idempotency
           |
           +-- Yes
                 |
                 v
Does the overall deadline permit another attempt?
                 |
                 +-- No --> Stop and handle the outcome
                 |
                 +-- Yes --> Wait with backoff and jitter
```

### Interview question

A payment request times out, and the outcome is unknown. You want to retry safely.

**Expected answer:** Use a bounded retry policy with exponential backoff and jitter, preserve the same idempotency key, and respect the overall deadline. Reconcile the payment status when necessary.

---

# Step 7: Timeouts and Deadlines

## 7.1 What problem do they solve?

Timeouts prevent individual operations from waiting indefinitely.

Deadlines define the latest time by which the complete operation should finish.

**Simple analogy:** A restaurant may take up to two minutes to answer a booking call. However, you might have only five minutes in total to complete the booking before your next meeting.

## 7.2 Timeout vs Deadline

| Timeout | Deadline |
|---|---|
| Limits the duration of a particular operation | Limits the total time available for the complete workflow |
| Applies to an HTTP request or database call | Applies across multiple dependent operations |
| Can be different for each call | Must account for time already spent |

## 7.3 Real-time example: Order processing

An API has a five-second deadline for completing an order request.

The request calls:

1. Inventory Service.
2. Payment Service.
3. Order database.

Suppose the first payment attempt consumes three seconds before timing out.

Only two seconds remain in the overall budget, before accounting for any other work.

A retry that starts with a fresh five-second timeout would violate the original deadline.

The service must consider the remaining budget before retrying.

## 7.4 Design rules

1. Set realistic timeouts for external calls.
2. Define an overall request deadline.
3. Propagate the remaining time budget through dependent calls.
4. Ensure retries fit within that budget.
5. Cancel work that is no longer needed when safe and supported.
6. Handle uncertain outcomes correctly.

**Important:** A client timing out does not necessarily cancel the downstream operation. The payment provider may still complete the transaction.

### Interview question

An order request has a five-second deadline. The first payment attempt consumes three seconds before timing out, and the retry policy proposes waiting two seconds.

**Expected answer:** The deadline has been consumed. Do not start a retry that cannot fit within the remaining budget. Safely resolve the uncertain payment status, potentially through an idempotent status check or reconciliation workflow.

---

# 8. How the Seven Patterns Work Together

Consider a single scenario: a large traffic spike causes the Payment Service to become slow.

A resilient application might respond as follows:

| Step | Pattern | Action |
|---|---|---|
| 1 | Timeout | Stop waiting for a payment call beyond its allowed duration |
| 2 | Retry with backoff and jitter | Retry eligible transient failures within the remaining deadline |
| 3 | Idempotency | Prevent duplicate payment effects |
| 4 | Circuit breaker | Temporarily stop calls when the dependency is unhealthy |
| 5 | Bulkhead | Prevent payment traffic from exhausting resources reserved for other dependencies |
| 6 | Backpressure | Control the rate of incoming or in-flight work |
| 7 | Load shedding | Reject or defer excess nonessential work when safe capacity is exceeded |

These patterns are complementary, not interchangeable.

For example, a circuit breaker can be Open while a bulkhead still protects resources allocated to other dependencies. Backpressure can control work intake before load shedding becomes necessary.

## 8.1 Simplified architecture

```text
Customer
   |
   v
API / Order Service
   |
   +--> Deadline and timeout
   |
   +--> Admission control / backpressure
   |
   +--> Payment resilience pipeline
             |
             +--> Bulkhead / concurrency limit
             |
             +--> Circuit breaker
             |
             +--> Bounded retry with backoff
             |
             v
       Payment Provider
             |
             v
       Idempotency store
       and payment status
```

This is a conceptual diagram, not a mandatory execution order. Actual ordering depends on the framework, configuration, and business workflow.

---

# 9. Suggested Bitbucket Repository Structure

Organise your notes and exercises so you can revise each pattern independently.

```text
system-design-patterns/
|
|-- README.md
|
|-- 01-circuit-breaker/
|   |-- README.md
|   |-- examples/
|   |-- exercises.md
|
|-- 02-bulkhead/
|   |-- README.md
|   |-- examples/
|   |-- exercises.md
|
|-- 03-backpressure/
|   |-- README.md
|   |-- examples/
|   |-- exercises.md
|
|-- 04-load-shedding/
|   |-- README.md
|   |-- examples/
|   |-- exercises.md
|
|-- 05-idempotency/
|   |-- README.md
|   |-- examples/
|   |-- exercises.md
|
|-- 06-retries-backoff-jitter/
|   |-- README.md
|   |-- examples/
|   |-- exercises.md
|
|-- 07-timeouts-deadlines/
|   |-- README.md
|   |-- examples/
|   |-- exercises.md
|
|-- common-exercises/
    |-- failure-scenarios.md
    |-- interview-questions.md
```

Start by committing this study guide as `README.md`. As you learn each concept, add a small runnable example, tests, and a short explanation of the trade-offs.

---

# 10. Revision Checklist

Use this checklist to assess whether you can explain the patterns without referring to your notes.

- [ ] Explain the three circuit breaker states.
- [ ] Explain how a bulkhead prevents resource exhaustion.
- [ ] Calculate backlog growth from arrival and processing rates.
- [ ] Distinguish backpressure from load shedding.
- [ ] Explain how idempotency prevents duplicate payment effects.
- [ ] Explain why exponential backoff and jitter are used together.
- [ ] Distinguish timeouts from overall deadlines.
- [ ] Explain how these patterns work together during a downstream outage.
- [ ] Describe one failure scenario and defend your chosen design.
- [ ] Implement and test at least one pattern in a .NET project.

## Final takeaway

The purpose of resilience patterns is not to guarantee that failures never happen. It is to limit their impact, preserve business correctness, and allow the system to recover safely.

A strong system designer understands not only how each pattern works, but also when to use it, what trade-offs it introduces, and how it interacts with the other patterns.
