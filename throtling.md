# System Design Study Notes: Rate Limiting, Throttling, Bandwidth Limiting & Token Bucket

> **Goal:** Understand what each concept does, when to use it, and how
> to explain it in a system-design interview. These notes are written to
> be saved in a Bitbucket repository and revised as you study.

## Learning path

1.  Understand the three problems: request frequency, overload, and data
    transfer.
2.  Learn how a Token Bucket works.
3.  Walk through two realistic examples.
4.  Know the trade-offs and common mistakes.
5.  Test yourself with the review questions.

------------------------------------------------------------------------

## 1. The three concepts

### 1.1 Rate limiting --- "How many requests may this client make?"

A rate limiter restricts how many requests a client can make over a
period of time.

**Example rule:** A client may make up to 100 API requests per minute.

When the client exceeds the limit, the system might reject the request,
commonly with HTTP `429 Too Many Requests`. Some systems include a
`Retry-After` header to tell the client when to try again.

**Use it for:** - Protecting APIs from excessive use. - Applying
per-user, per-API-key, or per-tenant quotas. - Helping prevent
accidental request storms and some forms of abuse. - Sharing service
capacity fairly between clients.

**Remember:** A rate limiter does not necessarily queue requests. Many
implementations reject excess requests immediately.

### 1.2 Throttling --- "How should the system control incoming work?"

Throttling is a broader term for controlling the rate or flow of work.
Depending on the system, it may slow traffic, queue work, reduce
processing, or reject requests.

**Example:** A service normally handles 200 requests per second. During
a spike, it limits incoming work to a sustainable rate and rejects
excess requests rather than allowing the database to become overwhelmed.

**Use it for:** - Protecting a service or dependency from overload. -
Controlling work sent to a database or downstream API. - Managing
traffic spikes. - Keeping a service within operational limits.

**Remember:** Terminology varies. Some platforms use "throttling" and
"rate limiting" almost interchangeably. In an interview, explain the
specific behaviour rather than relying only on the label.

### 1.3 Bandwidth limiting --- "How much data can be transferred per second?"

Bandwidth limiting controls the amount of data transferred over time,
rather than the number of requests.

**Example rule:** Limit each file download to 5 MB/s.

One request may transfer a few kilobytes; another may transfer several
gigabytes. A request-count limit treats both as one request, while a
bandwidth limit focuses on the amount of data moving.

**Use it for:** - Controlling file-upload or download speeds. - Sharing
network capacity between users. - Preventing one client from consuming
too much transfer capacity. - Applying traffic-shaping policies.

### Quick comparison

  -----------------------------------------------------------------------
  Concept                 Main question           Example
  ----------------------- ----------------------- -----------------------
  Rate limiting           How many requests?      100 requests per minute
                                                  per API key

  Throttling              How do we control the   Queue or reject excess
                          flow of work?           work during overload

  Bandwidth limiting      How much data per       5 MB/s per download
                          second?                 
  -----------------------------------------------------------------------

**Interview takeaway:** Rate limiting controls request frequency;
throttling controls the flow of work; bandwidth limiting controls data
transfer volume over time. Rate limiting can be one form of throttling.

------------------------------------------------------------------------

## 2. Real-world analogy: a supermarket

Imagine a supermarket with a limited number of checkout counters.

-   **Rate limiting:** A customer can use a particular self-service
    facility only a certain number of times in a given period.
-   **Throttling:** When too many customers arrive, staff manage the
    flow, ask customers to wait, or temporarily stop admitting more
    people.
-   **Bandwidth limiting:** Each checkout has a limit on how quickly it
    can process items.

The analogy is not exact, but it helps separate the questions each
mechanism answers.

------------------------------------------------------------------------

## 3. Token Bucket --- a common rate-limiting algorithm

Token Bucket is a way to implement a rate limit while allowing short
bursts of traffic.

### Step 1: Imagine a bucket of tokens

The bucket has: - **Capacity:** The maximum number of tokens it can
hold. - **Refill rate:** How quickly tokens become available again. -
**Request cost:** How many tokens a request consumes. For a simple
limiter, each request costs one token.

### Step 2: Process an incoming request

1.  Check whether the bucket has enough tokens.
2.  If it does, remove the required token(s) and allow the request.
3.  If it does not, apply the configured policy: reject the request or
    delay it if the implementation supports waiting.
4.  Refill tokens over time, but never let the bucket exceed its
    capacity.

### Step 3: Work through the numbers

Assume: - Bucket capacity = 5 tokens. - Refill rate = 1 token per
second. - Each request costs 1 token. - The bucket starts full.

  ------------------------------------------------------------------------
  Moment           Event               Tokens after event Result
  ---------------- ---------------- --------------------- ----------------
  Start            Bucket is full                       5 Ready

  Immediately      Request 1                            4 Allowed

  Immediately      Request 2                            3 Allowed

  Immediately      Request 3                            2 Allowed

  Immediately      Request 4                            1 Allowed

  Immediately      Request 5                            0 Allowed

  Immediately      Request 6                            0 Rejected or
                                                          delayed by
                                                          policy

  After 1 second   One token                            1 One more request
                   refills                                can be allowed
  ------------------------------------------------------------------------

The bucket permits a burst of five requests at the beginning. After
those tokens are used, new requests depend on tokens becoming available
again.

**Important:** Token Bucket limits long-term average use while
permitting bursts up to its capacity. It does not guarantee that
requests arrive evenly spaced.

------------------------------------------------------------------------

## 4. Real-time example 1: a banking API

Imagine a banking API serving account balances.

**Policy:** Each API key has a bucket with a capacity of 5 requests and
a refill rate of 1 token per second. Each request costs one token.

### What happens?

1.  A mobile app opens and sends five requests quickly.
2.  All five requests are allowed because the bucket has five tokens.
3.  The app immediately sends a sixth request.
4.  There are no tokens, so the API gateway rejects it with
    `429 Too Many Requests` (assuming that is the configured policy).
5.  After approximately one second, a token is available again, allowing
    another request.

### Why is this useful?

-   It allows a short burst when an app loads.
-   It limits sustained request volume.
-   It can protect downstream services from clients sending too many
    requests.

### Design details to consider

-   **Scope:** Is the limit per user, API key, tenant, IP address, or
    globally?
-   **Response:** Should excess requests be rejected or queued?
-   **Retries:** Clients should avoid immediate retry loops. Use a
    sensible delay and exponential backoff where appropriate.
-   **Distributed deployment:** If multiple API gateway instances handle
    the same client, decide how token state is shared or coordinated.
    Independent in-memory buckets on each instance may allow a client to
    exceed the intended global limit.
-   **Different request costs:** A cheap read and an expensive
    report-generation request may need different token costs or separate
    policies.

These are design choices, not automatic features of the Token Bucket
algorithm.

------------------------------------------------------------------------

## 5. Real-time example 2: large file downloads

Imagine a service that lets users download large reports.

**Policy:** Each client is limited to 5 MB/s of transfer.

A user downloads a 500 MB file. Another user downloads a 2 MB file.

-   A request-count limiter sees one download request from each user.
-   A bandwidth limiter controls how quickly the file data is
    transferred.
-   A service may use both controls: a limit on how many downloads can
    be started and a separate limit on transfer speed.

**Why use bandwidth limiting?**

It helps avoid one download consuming a disproportionate share of
network capacity. The actual implementation may live in a proxy, web
server, application, or network traffic-shaping layer.

**Unit reminder:** MB/s means megabytes per second; Mbps means megabits
per second. One byte is eight bits, so do not compare these units
without converting them.

------------------------------------------------------------------------

## 6. Token Bucket versus other common algorithms

You will encounter these when designing API gateways and distributed
services.

  -----------------------------------------------------------------------
  Algorithm         How it works      Useful when       Main limitation
  ----------------- ----------------- ----------------- -----------------
  Token Bucket      Tokens refill     Bursts are        Does not smooth
                    over time;        acceptable but    every request
                    requests spend    sustained use     into an even flow
                    tokens            must be limited   

  Leaky Bucket /    Work is processed You want a        Queues can add
  traffic shaping   or emitted at a   smoother outgoing latency and can
                    controlled rate,  flow              fill up
                    often using a                       
                    queue                               

  Fixed Window      Count requests    Simple limits are Requests can
                    inside fixed time sufficient        bunch up around a
                    windows                             window boundary

  Sliding Window    Estimate or count More consistent   Often needs more
                    requests across a time-based limits state or
                    moving time       are needed        computation than
                    interval                            a simple fixed
                                                        counter
  -----------------------------------------------------------------------

**Terminology note:** "Leaky Bucket" is used for related but not
identical implementations. Some implementations focus on queueing and
smoothing traffic; others describe a rate-control counter. State the
actual behaviour when discussing a design.

### A fixed-window boundary problem

Suppose the limit is 100 requests per minute. A client sends 100
requests just before the minute ends and another 100 just after the next
minute starts. A fixed-window counter may allow 200 requests within a
very short period across the boundary.

A sliding-window approach can reduce this boundary effect.

------------------------------------------------------------------------

## 7. How the concepts fit into a service architecture

A typical request path might look like this:

1.  **Client:** Sends an API request.
2.  **API gateway or rate limiter:** Checks the client's request
    allowance.
3.  **Overload controls:** Queue, slow, or reject work where
    appropriate.
4.  **Application service:** Performs business logic.
5.  **Database or downstream service:** Handles the work it receives.
6.  **Transfer controls, if needed:** Limit the rate of large data
    transfers.

These are logical responsibilities, not mandatory separate components. A
system may combine several controls in one gateway or distribute them
across different layers.

**Example:** A gateway applies a per-user request quota, while a
report-generation service limits concurrent jobs and a file server
limits download speed.

------------------------------------------------------------------------

## 8. Common mistakes to avoid

-   **Mistake:** Saying rate limiting always queues excess requests.\
    **Better:** It may reject them, commonly with HTTP 429; queueing is
    a separate policy or mechanism.

-   **Mistake:** Saying bandwidth limiting controls API request counts.\
    **Better:** It controls data transfer volume over time.

-   **Mistake:** Assuming Token Bucket allows only evenly spaced
    requests.\
    **Better:** It allows bursts up to bucket capacity.

-   **Mistake:** Applying limits independently on every server and
    assuming they enforce a global quota.\
    **Better:** Consider how counters or tokens are coordinated across
    instances.

-   **Mistake:** Retrying every rejected request immediately.\
    **Better:** Use sensible retry delays, respect `Retry-After` when
    present, and apply exponential backoff with jitter when appropriate.

-   **Mistake:** Treating a rate limit as a complete overload-protection
    strategy.\
    **Better:** Also consider concurrency limits, bounded queues,
    timeouts, backpressure, load shedding, and downstream capacity.

------------------------------------------------------------------------

## 9. Step-by-step study checklist

Work through one section at a time.

-   [ ] Explain rate limiting in one sentence.
-   [ ] Explain throttling in one sentence.
-   [ ] Explain bandwidth limiting in one sentence.
-   [ ] Calculate the result of a Token Bucket request sequence.
-   [ ] Explain why Token Bucket allows bursts.
-   [ ] Explain what happens when the bucket is empty.
-   [ ] Compare Token Bucket with Fixed Window and Sliding Window.
-   [ ] Explain why distributed services need to consider shared limiter
    state.
-   [ ] Explain how rate limiting differs from backpressure and load
    shedding.

### Quick self-test

**Q1.** An API allows 60 requests per minute per user. Which mechanism
is this?

**Q2.** A file download is capped at 2 MB/s. Which mechanism is this?

**Q3.** A service slows or queues incoming work to avoid overwhelming a
dependency. Which concept best describes this behaviour?

**Q4.** A Token Bucket has a capacity of 3 tokens and starts full. Four
requests arrive immediately, each costing one token. Under a
reject-when-empty policy, how many are allowed?

**Q5.** Why can a fixed-window limiter allow a burst around the boundary
between two windows?

### Answers

1.  Rate limiting.
2.  Bandwidth limiting.
3.  Throttling (with queueing or flow control, depending on the
    implementation).
4.  Three requests are allowed; the fourth is rejected because no token
    remains.
5.  The counter resets at the window boundary, so requests just before
    and after that boundary can fall into separate windows.

------------------------------------------------------------------------

## 10. Interview-ready summary

> "Rate limiting controls how many requests a client can make over a
> period of time. Throttling is the broader practice of controlling
> incoming work to protect service capacity, for example by slowing,
> queueing, or rejecting excess work. Bandwidth limiting controls how
> much data can be transferred per second. One common rate-limiting
> algorithm is Token Bucket: tokens refill at a configured rate, each
> request consumes tokens, and requests without enough tokens are
> rejected or delayed according to policy. Token Bucket is useful
> because it supports short bursts while limiting sustained traffic. In
> a distributed system, I would also decide the scope of the limit and
> how state is coordinated across instances."

------------------------------------------------------------------------

## Suggested Bitbucket repository layout

Keep the notes in Markdown so they are easy to read in Bitbucket and can
evolve as you study.

``` text
system-design-patterns/
├── README.md
└── rate-limiting/
    └── README.md
```

Suggested `README.md` topics for this folder: - Rate limiting,
throttling, and bandwidth limiting - Token Bucket worked example - Other
rate-limiting algorithms - Design trade-offs and distributed-system
considerations - Self-test and interview summary

**Next study topic:** Backpressure versus load shedding versus rate
limiting. These mechanisms often work together, but they solve different
problems.
