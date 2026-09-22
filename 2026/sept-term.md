day - 1

## CQRS (Command Query Responsibility Segregation)

### Definition:

CQRS is an architectural pattern that separates the operations that **change** state (Commands) from the operations that **read** state (Queries), giving each its own model, its own code path, and often its own data store and scaling strategy.

It rests on a simple observation from 1970s database theory: a system does two very different jobs — a write (a "command") is a one-way action that mutates the system and has side effects; a read (a "query") is a side-effect-free lookup that returns data. When you force both through the *same* model, that model becomes a compromise that is good at neither. CQRS deliberately splits them so each half can be optimized for its own job.

A quick word on the name: "Responsibility Segregation" just means "give each responsibility its own class/module/component" — it's the same idea as Separation of Concerns applied to reads vs. writes.

TRADITIONAL (single model) vs CQRS (split models):
═══════════════════════════════════════════════════════════════

  TRADITIONAL CRUD — ONE model does everything:
  ─────────────────────────────────────────────────────────────

            ┌──────────────────────────┐
   write ──►│                          │
   (POST)   │    ONE DOMAIN MODEL      │──► reads hit the SAME
            │    ONE DATABASE          │    structure as writes
   read  ──►│    ONE SERVICE           │
   (GET)    │                          │
            └──────────────────────────┘
              • Model is a compromise: it must serve both
                mutation and query shapes.
              • Read scaling = write scaling (coupled).
              • A heavy write lock blocks reads, and vice versa.


  CQRS — TWO models, each optimized for its own job:
  ─────────────────────────────────────────────────────────────

          ┌──────────────┐         ┌──────────────┐
   write ─►│  COMMAND SIDE │        │  QUERY SIDE   │◄─ read
   (POST)  │  (Write Model)│        │ (Read Model)  │   (GET)
   ──────►│  validates    │        │  denormalized │
          │  business rules│        │  pre-joined   │
          │  persists     │        │  fast reads   │
          └──────┬───────┘         └──────────────┘
                 │      separate stores,
                 │      separate scale
                 ▼         ↑ sync (sync call or async event)
          [WRITE DB]       [READ DB / READ-ONLY REPLICA / CACHE]

              • Commands: strict, validated, consistency-first.
              • Queries: loose, denormalized, speed-first.
              • Each side scales and is deployed independently.

  ┌────────────────────────────────────────────────────────────┐
  │  KEY IDEA: the write path and read path are DIFFERENT       │
  │  shapes, so give them DIFFERENT models instead of one       │
  │  bloated compromise.                                        │
  └────────────────────────────────────────────────────────────┘

When to use it (it is NOT for every app):
  ┌────────────────────────────────────────────────────────────┐
  │  GOOD FIT (use CQRS):                                      │
  │  • Read-heavy systems: 10×–1000× more reads than writes    │
  │  • The read shape is very different from the write shape   │
  │    (write tiny/normalized, read wide/denormalized)         │
  │  • Write and read need different scaling (bursty writes,   │
  │    heavy analytic reads)                                   │
  │  • Teams need to optimize each side independently          │
  │                                                            │
  │  BAD FIT (skip it):                                        │
  │  • Simple CRUD CRUD — a single model is simpler            │
  │  • No read/write asymmetry → CQRS adds complexity          │
  │  • You need strong, immediate read-your-writes consistency │
  └────────────────────────────────────────────────────────────┘

CQRS is often (but not always) paired with **Event Sourcing** — commands produce events that are replayed to build the read model. They are separate patterns that complement each other; CQRS can exist without Event Sourcing.

### Example:

A ticketing platform where 99% of traffic is people *viewing* event seats (reads) and only a tiny fraction is actually *booking* (writes).

```
WITHOUT CQRS — every page-view and every booking hit ONE table
═══════════════════════════════════════════════════════════════

            ┌────────────────────────────────────────┐
 1000/s     │         [events] + [seats] JOIN        │
 reads ───► │    ONE relational model + indexes      │
            │                                        │
  2/s       │   • Seat lookups run heavy JOINs.      │
 writes ──► │   • Every read pays the write-model    │
            │     cost (normalization, locking).     │
            │   • A long booking transaction can     │
            │     block page-view reads.             │
            └────────────────────────────────────────┘
            PROBLEM: reads are slow, writes are rare,
            but they're stuck in the same bottleneck.


WITH CQRS — reads get a model built just for viewing
═══════════════════════════════════════════════════════════════

  Book seat (POST /book)                 View seats (GET /seats)
       │                                        ▲
       ▼                                        │
  ┌────────────────┐            ┌──────────────────────────┐
  │ COMMAND SIDE   │            │ QUERY SIDE               │
  │ write model    │            │ read model               │
  │ • validates    │            │ • pre-joined, flat rows  │
  │   seat exists  │            │ • e.g. "seat: A-12,      │
  │ • checks price │            │   row: front, price: 80, │
  │ • holds lock   │            │   status: FREE"          │
  │ • persists     │            │ • served from cache /    │
  └───────┬────────┘            │   read replica           │
          │                     └────────────▲─────────────┘
          │  emits event                        │ subscribed
          ▼  "SeatBooked"                        │ to events
  ┌──────────────────────────────────────────────────────┐
  │   ASYNC SYNC: write DB → (event) → read DB/cache     │
  │   (event bus / CDC / message queue)                  │
  └──────────────────────────────────────────────────────┘

  RESULT:
  • 1000/s page-views are served from a flat, cacheable,
    denormalized read model — sub-millisecond, no JOINs.
  • The rare 2/s bookings use strict validation and locking
    on the write side, without contending with reads.
  • The read model can be scaled out to N replicas freely;
    the write side stays small and correct.
  • Cost: tiny lag between booking and the read model
    catching up (eventual consistency on the read side).
```

The team gets fast, cacheable reads AND strict, correct writes — each tuned for its own workload instead of one model compromising both.

---

day - 2

## Test-Time Compute (Test-Time Scaling)

### Definition:

Test-Time Compute is the practice of deliberately spending **more computation at inference time** (when the model is answering) to get a better answer — as opposed to spending more computation at **training time** (when the model is built).

For most of AI's recent history there was only ONE dial to turn. To make a model smarter you made it bigger and fed it more data — that computation happened once, upfront, inside the training run, and got baked into the weights. At inference time the model did a single fast forward pass: prompt in, one answer out. More thinking = you had to re-train a bigger model.

Reasoning models (o1-class and successors, which became mainstream through 2025–2026) opened a SECOND dial. Instead of answering in one shot, they generate an internal "chain of thought" — they pause, explore, backtrack, and verify — and the **length and thoroughness of that hidden thinking** is itself a knob you can turn per-request. That extra per-query reasoning is exactly what "test-time compute" means. The same model, given more compute budget at runtime, produces a measurably better answer. Quality now scales with *thinking time*, not just with parameters.

CLASSICAL (training-time only) vs REASONING (test-time scaling):
═══════════════════════════════════════════════════════════════

  CLASSICAL SCALING — one dial, all spent upfront:
  ─────────────────────────────────────────────────────────────

        [TRAINING TIME — expensive, one-time]     [INFERENCE — fixed]
        ┌──────────────────────────────────┐      ┌──────────────┐
        │  MORE PARAMS        ┌──────────┐ │      │  1 fast pass  │
   dial ►│  MORE DATA    ───► │ trained  │ │ ───► │  prompt → out │
        │  MORE GPU-days      │ weights  │ │      │  (no thinking)│
        └──────────────────────────────────┘      └──────────────┘
          smarter model                    SAME cost per query,
                                            no per-question knob

  RESULT: if one answer is wrong, your only fix is to
  retrain something bigger. Expensive, slow, can't adapt
  per question.


  REASONING / TEST-TIME SCALING — a second dial at runtime:
  ─────────────────────────────────────────────────────────────

                          ┌───────────────────────────────┐
   prompt ───────────────►│  TEST-TIME COMPUTE BUDGET     │
                          │                               │
                          │   chain-of-thought ──┐        │
                          │   • try a path        │        │
                          │   • notice error      │◄───────┤  budget
                          │   • backtrack         │  up   │  spend
                          │   • try another route │       │  more =
                          │   • verify            │       │  better
                          └───────────────────────┴───────┘
                                            │  answer
                                            ▼

  • Same trained weights, but you control how "hard" it
    thinks PER REQUEST.
  • Math/code/logic → crank budget up → better accuracy.
  • Simple chit-chat → keep budget tiny → fast & cheap.
  • The trade-off moves to inference time: per-query cost
    rises because the model emits many more tokens (its
    hidden reasoning) even when unit price per token falls.

  ┌────────────────────────────────────────────────────────────┐
  │  KEY IDEA: smarter is no longer ONLY "bigger model".       │
  │  It's also "think longer on the hard questions". Two       │
  │  orthogonal scaling axes — training compute and            │
  │  test-time compute.                                        │
  └────────────────────────────────────────────────────────────┘

The dial is tunable at several levels: per deployment (a "deep reasoning" vs a "fast" model endpoint), per request (an API flag asking for more effort), or algorithmically via methods such as best-of-N sampling, majority voting (self-consistency), or letting the model search/verify before committing to an answer. The economics matter: total inference cost for hard tasks can rise sharply because a reasoning model spends many tokens thinking — so systems must decide WHEN high test-time compute is worth it.

### Example:

A math tutoring app (fitting — exactly the kind of thing you'd build) where the SAME model must handle two very different requests: "what is 7×8?" (instant) vs a hard word problem that trips up one-shot answers.

```
THE SAME MODEL, ONE REQUEST, A PER-REQUEST THINKING BUDGET
═══════════════════════════════════════════════════════════════

  REQUEST A: "7 × 8 = ?"
  ───────────────────────
     budget: LOW  (it's trivial)

        ┌──────────────────────────────┐
        │  single fast pass            │
        │  "7 × 8 = 56"                │  ~few tokens
        └──────────────────────────────┘   → cheap, ~instant

  REQUEST B: hard word problem
  ───────────────────────
     budget: HIGH (it trips up one-shot)

   prompt ──► ┌──────────────────────────────────────────────┐
              │  CHAIN-OF-THOUGHT (hidden reasoning tokens)  │
              │                                              │
              │  attempt 1: sets up wrong equation ──✗        │
              │    "wait, that double-counts the overlap"    │
              │  attempt 2: backtrack, re-model              │
              │    builds correct equation ✓                 │
              │  verify: plug answer back in, consistent ✓   │
              └──────────────────────────────────────────────┘
                                             │  commits "42"
                                             ▼
        more tokens spent → higher accuracy on THIS hard query

  WITH one-shot fast pass (no test-time compute):
     this word problem likely gets the SAME error the model
     always makes on it.
  WITH test-time compute (budget cranked up):
     the model searches internally, self-corrects, verifies,
     and lands the right answer — no retraining needed.
```

The punchline: the app didn't need a bigger, more expensive model. It needed a **budget-aware router** — spend test-time compute only where one-shot accuracy fails, and keep the cheap fast path everywhere else. That's the real engineering superpower of test-time scaling: you buy intelligence on demand, question by question, instead of buying it once in a monolithic training run.

---

day - 3

## Serverless Cold Start

### Definition:

Serverless Cold Start is the latency penalty a serverless platform pays — and your users feel — when it must **create and initialize a brand-new execution environment** (sandbox, runtime, and your code) before it can run a function that was just invoked. A *warm* request finds an environment that is already alive and skips straight to running the handler; a *cold* request has to build the whole thing from zero first.

The important framing: a cold start is **not a bug — it is the deliberate price of scale-to-zero economics**. Because the platform reclaims idle environments after minutes of silence, you pay exactly nothing while your function sits unused. The first request after that silence is the one that pays the full setup bill. Latency and the pay-per-use billing model are two sides of the same coin: the platform can only charge you for what actually runs if it is free to destroy what isn't running.

That also tells you the three triggers — every cold start is one of these:
1. **Idle reclamation** — environment killed after N minutes without traffic; the next request rebuilds it.
2. **A fresh deploy** — a new code version means new environments, so releases can trigger a burst of cold starts.
3. **Scale-out (the sneaky one)** — N concurrent requests arrive but only M < N environments are warm; the extra N−M cold-start *all at the same time*, in the middle of your busiest moment — precisely when extra latency hurts the most.

COLD vs WARM — what actually happens on each path:

```
COLD START vs WARM START
════════════════════════════════════════════════════════════

  COLD START — no environment ready, request pays full setup:

   request ──► ┌───────────────────────────────────────────────┐
               │  1. ALLOCATE a sandbox (micro-VM / container) │  50–100 ms
               │  2. DOWNLOAD your deployment package          │  50–500 ms
               │     (grows with bundle size!)                 │
               │  3. BOOT the language runtime                 │  50–1,000 ms
               │     (interpreter fast, JVM/.NET slow)         │
               │  4. RUN YOUR init code                        │  0 ms – seconds
               │     imports, SDK clients, DB connections      │  ◄── your lever
               │  5. INVOKE the handler ────────────────► done │
               └───────────────────────────────────────────────┘
                 total added latency: ~100 ms to several seconds
                 THE USER WAITS FOR ALL OF IT BEFORE THE
                 FIRST BYTE OF ACTUAL WORK HAPPENS.

  WARM START — environment kept alive, same request later:

   request ──► ┌────────────────────────────────┐
               │  thaw (~1–10 ms)               │
               │  step 4 results (DB pool, SDK  │
               │  clients) are STILL ALIVE      │
               │  5. INVOKE handler ────► done  │   single-digit ms
               └────────────────────────────────┘
                 init code runs ONCE per env,
                 then is reused across requests
```

How bad is it, honestly? It depends mostly on runtime and bundle size. Interpreted runtimes (JavaScript, Python) typically cold-start in **200–400 ms**; compiled ones (Go, Rust) can stay under ~100–300 ms; VM-based runtimes (JVM, .NET) take **500 ms to several seconds** — which is why snapshot-restore features (freeze a booted runtime, restore it in milliseconds instead of re-booting) exist and cut that figure dramatically. And *frequency* is the half of the story most explainers skip: steady production traffic sees cold starts on under 1% of invocations — but a function invoked once per hour cold-starts almost every single call, and development environments can see cold rates of 30–90%. Tail latencies (p99) run 2–3× the median, which is exactly what your latency budget actually cares about.

Because every cold start is a small bill in *time*, teams climb a **mitigation ladder** — cheapest first:

```
THE MITIGATION LADDER (cost order)
══════════════════════════════════
  FREE (code-level)      shrink the deployment bundle & prune
                         dependencies  ← biggest lever most teams
                         never pull; lazy-load heavy imports;
                         open DB connections in init scope so
                         warm requests reuse them
  CHEAP (config)         raise memory (CPU scales with it);
                         enable snapshot-restore / fast-snapshot
                         features where the platform offers them
  FRAGILE (folk remedy)  keep-warm "ping" every few minutes —
                         keeps ONE environment warm, does nothing
                         on scale-out, and lies to dashboards
  PAID (definitive)      provisioned concurrency / minimum
                         instances: N environments always ready,
                         cold starts eliminated up to N — but you
                         reintroduce always-on cost into a
                         pay-per-use model (the irony is priced in)
  ARCHITECTURAL          isolate-based edge runtimes (V8 isolates:
                         single-digit ms to spawn) trade a full
                         runtime for a sandboxed web-standard env
```

The whole discipline boils down to one question: **is a human waiting on this call?** If yes, cold starts are a UX bug and deserve the ladder. If the work is async — a queued job, a webhook, a scheduled task — the tail is absorbed invisibly and you should spend nothing on it.

### Example:

A ticketing platform ("TiketKilat") runs a flash sale at 15:00 sharp. The checkout function is serverless, pay-per-use — and had zero traffic for the 30 minutes before the sale, so the platform reclaimed every idle environment.

```
FLASH SALE — ONE ENDPOINT, THREE MOMENTS
════════════════════════════════════════════════════════════

  14:30–15:00  no traffic → platform reclaims ALL environments

  ┌─ 15:00:00.000  request #1 arrives ── COLD ─────────────────┐
  │  allocate sandbox → download package → boot runtime →      │
  │  init SDKs + DB pool → finally run handler                 │
  └────────────────────────────────────────────────────────────┘
              user #1 waits 1.3 s (spinner, rage, refresh)

  ┌─ 15:00:00.300  requests #2–#10 arrive — THE SPIKE ────────┐
  │  env #1 is warm now BUT handles one request at a time     │
  │  → #2–#10 each need their OWN environment                 │
  │  → 9 MORE cold starts, all simultaneous                   │
  │    ◄── scale-out: cold starts land exactly when           │
  │        traffic is highest (worst possible timing)         │
  └───────────────────────────────────────────────────────────┘
              first 30 s of the sale: p99 ≈ 1.4 s
              (users abandon carts, sale page trends on X
               for the wrong reason)

  ┌─ 15:00:45  autoscaler finally has 40 warm envs ──────────┐
  │  requests #200+ ── WARM ── 30–50 ms each                 │
  └──────────────────────────────────────────────────────────┘
              p99 five minutes later: ~45 ms — smooth,
              but the first 30 seconds already happened
```

The fix is a direct application of the ladder. TiketKilat *knows* the sale is at 15:00 — that's not a surprise, it's a schedule. So before launch day they:

- **Schedule-based provisioned concurrency**: warm 30 environments at 14:59:30 so the first wave of requests lands on ready envs (predictable traffic → pre-warm ahead of the peak instead of reacting to it).
- **Slim the bundle**: the checkout function was importing a whole SDK for one call — tree-shaken down from 41 MB to 6 MB, shaving hundreds of ms off the unavoidable cold starts.
- **Move the non-blocking work off the hot path**: the confirmation email is pushed to a queue and processed by a separate function where a 1-second cold start is invisible — nobody is staring at it.

```
WITH THE FIX — same flash sale, same spike:
═══════════════════════════════════════════
  14:59:30  provisioned envs spin up (N = 30) ── paid, ready
  15:00:00  request #1 ──► WARM env ──► 45 ms      ✓ no spinner
  15:00:00  requests #2–#100 ──► scale-out absorbs on warm pool
  15:00:05  a few cold starts only if traffic exceeds N
            (rare, and ~300 ms now, not 1.4 s)
```

The punchline: serverless never removes latency — it *relocates* it from idle time to the first request after idle. Cold starts can't be eliminated for free; they can only be (a) shrunk with leaner code, (b) skipped on the paths where a human waits, and (c) paid away with always-warm capacity exactly where the spike is predictable. The engineering skill is knowing which of the three applies per endpoint — and never trusting a keep-warm ping to save you.

---

day - 4

## Structured Concurrency

### Definition:

Structured Concurrency is a programming model that guarantees **no concurrent task can outlive the code block that created it**: whenever a block spawns child tasks, those children must finish — or be cancelled — before the block is allowed to exit. It applies the same nesting discipline that *structured programming* gave to control flow back in the 1960s (no more `goto`, everything nests inside `if`/`while`/function calls) to the world of threads, goroutines, and coroutines, which never got that discipline — they were the last `goto` in modern code.

In the classic unstructured model, a task's lifetime is best described as "whoever spawned it *hopes* it finishes." A child routinely outlives its parent (a leak), or its parent dies first and leaves it an orphan that happily writes to closed databases and logs after the response was already sent, or it fails and the error vanishes because nothing is attached to it to hear the scream. Structured Concurrency makes all three impossible by construction — not by discipline, but by the language runtime:

1. **Scope**: tasks can only be spawned inside a delimited scope — a `StructuredTaskScope` (Java), a `coroutineScope { }` (Kotlin), an `errgroup.Group` (Go), a Trio/AnyIO *nursery* (Python).
2. **Fork-join**: the scope cannot return until *every* child has joined. No code after the scope runs while children are still in flight.
3. **Cancel-on-failure**: the first child error automatically cancels all remaining siblings, then propagates up like a normal exception — no zombie fan-out silently half-succeeding.

UNSTRUCTURED vs STRUCTURED — what happens to the children:

```
UNSTRUCTURED CONCURRENCY            STRUCTURED CONCURRENCY
(fire-and-forget)                   (scoped fork-join)
══════════════════════════          ══════════════════════════

 main() ─ spawn ──►┌ worker A ┐     main() ──► ┌──────────────────────────┐
        │          │ (slow)   │                │ scope {                  │
        │          └──────────┘                │   fork A ──┐             │
        │          ┌ worker B ┐                │   fork B ──┼─┐           │
        │          │ (fails!)  │               │   fork C ──┼─┼─┐         │
        │          └──────────┘                │            │ │ │         │
        ▼          nobody hears                │   join A ◄─┘ │ │         │
   main RETURNS    B's error                   │   join B ◄───┘ │         │
   (frame gone)                                │   join C ◄─────┘         │
        │                                     │ }  ── error? cancel       │
        │   worker A STILL RUNNING ──►        │      remaining siblings   │
        │   writes to a DB pool the           └──────────────────────────┘
        │   parent already closed                     │
        ▼                                            ▼
   GHOST WORK after "done":               main() returns ONLY after all
   leaked memory, late panics,            children are joined — the task
   silent partial failure                 tree mirrors the call stack:
                                          no orphans, no ghosts, no
   CHILD LIFETIME: unbounded              swallowed errors.
   CHILD LIFETIME: bounded by scope ──►   ▲
                                          └─ error handling in ONE place
```

The mental model: **the tree of running tasks should look exactly like the tree of the call stack** — a parent should never be able to move on, or die, while its descendants are still alive somewhere in the dark. If a piece of work genuinely must outlive its caller (background telemetry, a confirmation email), structured concurrency doesn't forbid it — it forces you to make that detachment *explicit* (hand it to a queue or a separate process) instead of letting it happen by accident.

Structured Concurrency went mainstream through Project Loom: previewed in Java since Java 19, it was finalized as **JEP 507 in Java 26 (2026)** via `StructuredTaskScope`. Kotlin (coroutines), Swift (task groups), Python (Trio/AnyIO nurseries) and Go (`errgroup` + `context`) ship the same idea — each with its own flavor.

### Example:

"TokoOnline" adds a checkout endpoint. Before returning an order confirmation it must do three independent remote calls in parallel: reserve inventory (RPC), charge the payment (RPC), and run a fraud check. The tempting first version is three fire-and-forget goroutines:

```
NAIVE VERSION — handler exits while workers are still flying:

 request ──► handler
              ├─ go reserveInventory() ──(150 ms)──► done
              ├─ go chargePayment()    ──(300 ms)──► done
              └─ go fraudCheck()       ──(fails @ 200 ms,
              │                          nobody is watching)
              ▼
   ~15 ms later : handler returns "200 OK ✓"  ← LIES,
                  nothing actually finished
   200 ms later : fraud check FAILS → order ships anyway,
                  money moves, no one ever learns
   310 ms later : workers touch the request-scoped DB pool
                  the handler already closed → PANIC
                  in the logs AFTER the response was sent
                  (the ghost crash — your on-call pager
                   ringing about a request that "succeeded")
```

All three classic failures in one screenshot: parent returned before children (lie), a child failed silently (swallowed error), and children outlived the parent (ghost work). The fix is to put the fan-out inside a scope — in Go, `errgroup` from `golang.org/x/sync`:

```go
g, ctx := errgroup.WithContext(r.Context())   // scope begins

g.Go(func() error { return reserveInventory(ctx) })  // fork
g.Go(func() error { return chargePayment(ctx) })     // fork
g.Go(func() error { return fraudCheck(ctx) })        // fork

if err := g.Wait(); err != nil {   // join: waits for ALL children
    return 502                      // first error cancels the rest
}                                   // via ctx before it returns
return 200                          // "OK" is now a TRUE statement:
                                    // all three really completed
```

```
WITH ERRGROUP — the scope cannot exit until every child is joined:

 scope ──► ┌────────────────────────────────────────────────┐
           │ g.Go(reserve) ──(150 ms)──► ok ───────────────┐ │
           │ g.Go(charge)   ──(300 ms)──► ok ────────────┐ │ │
           │ g.Go(fraud)    ──(200 ms)──► ERROR ────► ┌──┘ │ │
           └──────────────────────────────────────────┴────┘ │
                      │  errgroup cancels ctx ────────────────┘
                      ▼
           g.Wait() returns the fraud error
                      ▼
           handler → 502, order rolled back,
           no ghost goroutines, no silent success
```

Same three properties, now enforced by the runtime instead of by hope: the handler's `return 200` physically cannot run before all three children joined; the fraud error is impossible to lose — `g.Wait()` either returns `nil` (all OK) or the first failure; and on failure the context cancels the two siblings still in flight, so no half-charged order limps on.

The punchline: structured concurrency is "**the call stack, but for parallel work**." Its three guarantees — no orphaned tasks, no swallowed errors, cleanup in exactly one place — turn concurrency bugs from a class of mystery (races you debug at 2 AM) into plain control flow you can read top to bottom. The remaining honest use of fire-and-forget isn't "we'll deal with it later" — it's work you *deliberately* detach to a queue or background process where outliving the request is the correct, visible design. Concurrency stopped being special the moment we stopped letting it escape the block that owns it.

---

day - 7

## LoRA (Low-Rank Adaptation)

### Definition:

LoRA is a **parameter-efficient fine-tuning (PEFT) technique** that adapts a large pre-trained model (an LLM, diffusion model, or any Transformer) to a new task *without touching the original weights*. It freezes the full pre-trained model and injects tiny **trainable "adapter" matrices** beside the big weight matrices, so only a minuscule fraction of the network — often under 1% of parameters — actually learns.

The trick comes from an observation about fine-tuning: when you adapt a huge pre-trained model, the learned *change* to its weights is surprisingly **low-rank**. A full weight matrix W (say 4096×4096) barely moves during adaptation — the adjustment ΔW needed to capture a new skill lives on a much smaller number of effective dimensions. So instead of learning all ~16.7M entries of ΔW directly, LoRA *factorizes* it into two skinny matrices: A (rank r × d) and B (d × rank r). Their product B·A reproduces a low-rank approximation of ΔW with only 2·d·r parameters instead of d². With r = 8, that's 65,536 vs 16.7M — about 250× fewer learnable parameters for that layer.

The forward pass becomes: **h = W₀x + (α/r)·BAx** — the frozen path carries all the pre-trained knowledge, and the small learned branch adds a task-specific correction. After training, the correction can be *merged* back into W (W′ = W₀ + (α/r)·BA), leaving a single normal model with zero extra inference cost.

FULL FINE-TUNING vs LoRA:
═══════════════════════════════════════════════════════════════

  FULL FINE-TUNING — every weight learns, everything must fit in memory:
  ─────────────────────────────────────────────────────────────

          [W: d×d] ─── gradient ──► [optimizer state]      GPU MEMORY
           ALL 4B·L             ALL weights move           = weights × ~16
           params trainable     (even if most barely do)     (fp16 + grads
                                                              + Adam)
          • New copy per task: one full model per skill        ▲
          • Checkpoint = the whole model (GBs per save)        │
          • Needs many big GPUs — the optimizer alone          │
            is 2× the model size.                              │
                                                               ▼
                                                        7B model ≈ 110–140 GB
                                                        → several 80GB GPUs

  LORA — frozen base + a tiny learnable detour per layer:
  ─────────────────────────────────────────────────────────────

          [W: d×d]  FROZEN ──► h = W₀x + (α/r)·BAx
               ▲                    │
               │            ┌───────┴────────┐
          no gradient    [A: r×d]      [B: d×r]   ◄── ONLY these learn
               │         random init   zeros init      (2·d·r params)
               │                                        per layer
               │                        GPU MEMORY
               │                        ≈ weights × ~2–3
               ▼                        + a few MB of adapters
        never updated                  → 7B model fits on ONE
                                        24GB consumer GPU
                                         (QLoRA: even one 8–12GB)

          • ONE base model + MANY adapters = many specialist
            skills, each a few MB on disk.
          • Merge B·A into W (W′ = W₀ + (α/r)·BA) → identical
            speed at inference, no adapter overhead.
          • Or keep adapters separate → hot-swap skills live.

  ┌────────────────────────────────────────────────────────────┐
  │  KEY IDEA: the pretrained model already knows 99.9% of     │
  │  what it needs. LoRA learns only the small, low-rank       │
  │  "delta" that points that knowledge at your task.          │
  └────────────────────────────────────────────────────────────┘

Key mechanics worth knowing:

- **r (rank)** — the width of the adapter matrices. Typical values 8–64. Higher r = more capacity (and more memory); lower r = cheaper, often almost as good. r is the main dial you tune.
- **α (alpha)** — a scaling factor applied as α/r. It controls how strongly the adapter correction is applied, not the rank itself. Common lore: set α ≈ 2×r and tune from there (e.g., r=16, α=32).
- **Which layers to target** — originally the attention projections (q, v). In practice people adapt q, k, v, o and often the feed-forward layers too; more targets = more capacity.
- **LoRA vs QLoRA** — QLoRA (2023) adds 4-bit quantization of the *frozen* base model while training the LoRA adapters in higher precision. That collapses the memory of the frozen part (7B base from ~14GB fp16 to ~4–5GB), which is why fine-tuning even 7B–70B class models is now possible on a single consumer GPU.
- **DoRA and the rest** — DoRA (Weight-Decomposed LoRA, 2024) decomposes weights into magnitude and direction and applies the low-rank update only to direction, beating plain LoRA on accuracy at the same rank; the field keeps iterating, but LoRA is the foundation they all build on.
- **Why it matters for serving** — because the base stays untouched, one GPU can hold a single frozen model and swap between many adapters per request (multi-LoRA serving: skill per tenant, per language, per task) — the adapter is the product, the base is shared infrastructure.

### Example:

RumahKode, a small studio, runs a customer-support copilot on a 7B open model for their Indonesian e-commerce client. It must answer in casual Indonesian ("santai", code-switching with English) and follow a strict refund policy. They own one 24GB consumer GPU and cannot rent a cluster for every experiment.

Option A (full fine-tuning) is a non-starter: a 7B model needs ~110–140 GB of VRAM just for weights + gradients + optimizer — 6× their whole GPU. Option B is to fine-tune with LoRA (they use Hugging Face PEFT; with r=16, α=32 on the attention + MLP projections, ~0.2% of parameters train).

```
THE PIPELINE — one frozen base, a small learnable detour
═══════════════════════════════════════════════════════════════

                ┌────────────────────────────────────────────┐
   1000 support │  DATASET: "user says X → agent replies Y"  │
   tickets/hour │  casual Indonesian + refund policy rules   │
                └────────────────────┬───────────────────────┘
                                     ▼
   ┌───────────────────────────────────────────────────────────┐
   │  BASE MODEL 7B — FROZEN (never updated, zero gradient)    │
   │                                                            │
   │  each layer:  h = W₀x + (α/r)·BAx                         │
   │                    ▲          ▲                            │
   │                    │          └── [A][B] ← ONLY TRAINABLE │
   │                    └─ original knowledge, untouched       │
   └──────────────────────────┬────────────────────────────────┘
                              ▼
                    train 3 epochs on the 24GB GPU
                    (weights ~14GB fp16 + adapter few MB)
                              ▼
                  LoRA adapter ≈ 26 MB on disk
                  (vs a ~14 GB full-model checkpoint)
                              ▼
              ┌───────────────────────────────┐
              │  MERGE:  W′ = W₀ + (α/r)·BA   │  ← deploy as ONE
              │  → zero extra latency, same    │    model, no special
              │    model file format           │    runtime needed
              └───────────────────────────────┘
```

After training, the copilot holds the refund policy and the right tone — without the studio ever training more than a few MB of parameters, and without renting a GPU cluster.

The merge step is optional and that's where LoRA gets *really* interesting. Their second product is a fasting-app companion bot that needs three very different voices: a strict medical-accuracy mode, a relaxed "gym bro" motivator mode, and a Bahasa-Jawa-lite casual mode. Instead of three full fine-tunes (three × 14GB+ models), they train **three 26 MB LoRA adapters on the same frozen base** and load the right one per request — or even per user:

```
MULTI-LORA SERVING — one base model, three specialist skills
═══════════════════════════════════════════════════════════════

         ┌─────────────────────────────────────────────────┐
         │            ONE FROZEN 7B BASE (14 GB)          │
         │            loaded once in GPU memory           │
         └──────▲──────────────────▲──────────────────▲───┘
                │                  │                  │
        ┌───────┴──────┐   ┌───────┴──────┐   ┌───────┴──────┐
        │ adapter 1    │   │ adapter 2    │   │ adapter 3    │
        │ "medical"    │   │ "gym bro"    │   │ "casual jv"  │
        │ 26 MB        │   │ 26 MB        │   │ 26 MB        │
        └───────┬──────┘   └───────┬──────┘   └───────┬──────┘
                │   hot-swap per request:             │
                └───────── W + B₁A₁  /  W + B₂A₂ ─────┘
                                             │
                        user asks → route → right adapter → answer
```

Storage saved: 42 GB of full checkpoints become 78 MB of adapters. GPU saved: one model resident instead of three. Iteration saved: retraining one voice's adapter never touches the other voices — no regression risk across skills. And each adapter trains in a fraction of the time of a full fine-tune, so the studio can experiment daily instead of weekly.

The punchline: LoRA separates the two things fine-tuning used to conflate — *knowledge* (expensive, lives in the big frozen weights) and *behavior* (cheap, lives in the tiny adapter delta). When adapting a model stopped meaning "re-train everything" and started meaning "attach a few MB of learned direction," fine-tuning went from a cluster-scale, weekly operation to a single-GPU, per-product one. For anyone running models on a modest box, LoRA isn't an optimization trick — it's the difference between fine-tuning being possible at all and not.

---

day - 8 

## Idempotency Keys

### Definition:

An **idempotency key** is a client-generated unique identifier sent with a mutating HTTP request (typically as an `Idempotency-Key` header on `POST`/`PATCH`) that lets the server recognize retries of the *same logical operation* and answer them with the stored result — instead of executing the side effect again.

It exists because of a brutal asymmetry in networks: **a lost response does not mean the request never arrived.** HTTP gives you safe methods (`GET`, `PUT`, `DELETE` are idempotent by spec — repeating them is harmless) but `POST` is a "fire once" verb with no such guarantee. When a client times out, it cannot know whether the server committed the charge, booked the seat, or sent the email. So it retries — and "retry" is where duplicates are born. At-least-once delivery is the default of the real world; idempotency keys are how you survive it.

The name comes from algebra: an operation is *idempotent* when doing it twice equals doing it once (`f(f(x)) = f(x)`). The pattern does not literally make the operation run once — it makes the **system converge on one visible result**, no matter how many times the request arrives. That distinction ("deterministic convergence", not "exactly-once") is the whole game in distributed systems.

NON-IDEMPOTENT POST — same request sent twice = side effect twice:
════════════════════════════════════════════════════════════════════

  CLIENT                          SERVER
    │ 1. POST /charge $50          │
    │────────────────────────────► │
    │                              │  charge $50    ┌───────────────┐
    │                              │───────────────►│ bank: -$50    │
    │ 2. response LOST             │                └───────────────┘
    │◄───────── network ✂ ──────── │  (client times out)
    │                              │
    │ 3. retry POST /charge $50    │
    │    (no key — looks identical)│
    │────────────────────────────► │
    │                              │  charge $50 AGAIN  ┌───────────────┐
    │                              │───────────────────►│ bank: -$50    │
    │◄──────────────────────────── │  200 OK            │ TOTAL: -$100  │
    │                              │                    └───────────────┘
    RESULT: one intent, two charges. The retry was "safe" from the
    client's view — it never saw a response — but the server could
    not tell the two requests apart.


IDEMPOTENT POST — same key = same logical operation:
════════════════════════════════════════════════════════════════════

  CLIENT                          SERVER
    │ 1. POST /charge $50          │
    │    Idempotency-Key: K-3f9a   │
    │────────────────────────────► │
    │                              │  key K-3f9a seen before?
    │                              │      │ NO
    │                              │      ▼
    │                              │  charge $50   ┌──────────────────────┐
    │                              │  save:        │ IDEMPOTENCY STORE    │
    │                              │  K-3f9a → 200 │ K-3f9a │ 200 {charge} │
    │ 2. response LOST             │               └──────────────────────┘
    │◄───────── network ✂ ──────── │  (client times out)
    │                              │
    │ 3. retry POST /charge $50    │
    │    SAME Idempotency-Key      │
    │────────────────────────────► │
    │                              │  key K-3f9a seen before?
    │                              │      │ YES ──► replay saved response
    │◄──────────────────────────── │  NO second charge!
    │  200 OK (replayed)           │
    RESULT: one intent, one charge, retries are free.

  ┌──────────────────────────────────────────────────────────────────┐
  │  KEY IDEA: the server remembers the OUTCOME of a key, so a       │
  │  retry with that key is answered from memory, never re-executed. │
  └──────────────────────────────────────────────────────────────────┘

How it works in practice:

- **One key per intent.** The client mints a fresh key for each new logical operation (each checkout, each booking) and reuses *that same key* for every retry of it. A UUIDv4 is the convention; never use PII or predictable counters (guessing keys would let callers collide with — and replay — other people's operations).
- **The server stores key → result.** First request with a key executes and records the outcome (status code + response body) in a dedupe store. Any later request carrying the same key short-circuits and gets the stored outcome back — Stripe even replays stored `500`s, so retries after failures behave exactly like the original failure did.
- **Key + different payload = conflict.** If a client reuses a key but sends a different body, the server must reject with `409 Conflict` — that is a bug in the client (new intent needs a new key), not a retry.
- **Keys expire.** Stripe prunes keys after ≥24 hours; a key reused after pruning starts a *new* operation, which is safe only because the original is long settled. TTL is your garbage collector — the store otherwise grows forever.
- **Standardization:** the `Idempotency-Key` header is an IETF draft (`draft-ietf-httpapi-idempotency-key-header`) and a de-facto convention that payment APIs (Stripe, Adyen, PayPal) have shipped for over a decade.

The subtle part — where naive implementations leak duplicates:

```
THE CRASH WINDOW — the store alone is not enough
════════════════════════════════════════════════
  1. business commit        2. write dedupe record    3. reply 200
  ┌──────────────────┐        ┌──────────────────┐      ┌────────┐
  │ charge $50       │        │  K-3f9a → 200    │      │  200   │
  │ tx COMMITS       │  crash │  (never written) │      └────────┘
  └──────────────────┘ ────►  └──────────────────┘
        ▲                        ▲
        └─ if the process dies between these two steps, the retry
           arrives, the key is unknown, and the charge runs again.

  Fixes (defense in depth):
  1. Write the dedupe row in the SAME database transaction as the
     business commit → both happen or neither.
  2. Backstop: a UNIQUE constraint on the natural business key
     (e.g. payment_ref = order-4711). A duplicate insert then
     collides → return the existing row instead of charging twice.
  3. Concurrency: two same-key requests in flight (double-tap).
     The store's primary key on the idempotency key is the lock —
     one insert wins, the other waits or gets a 409.
```

### Example:

Kyomel's fasting-bot launches a paid "Pro" tier. Users pay through a checkout API, and the payment service is written in Go behind a load balancer. It is 11 PM, and a user on a flaky mobile connection taps "Upgrade to Pro" on a slow bus — the request reaches the server, the charge commits, but the response dies somewhere in the tunnel before coming back. The phone retries.

```
RETRY-SAFE CHECKOUT — idempotency middleware + DB backstop
═══════════════════════════════════════════════════════════

  CLIENT (fasting-bot app)                PAYMENT SERVICE (Go)
  ┌──────────────────────────────┐
  │ POST /v1/subscriptions       │
  │ { plan: "pro" }              │
  │ Idempotency-Key:             │
  │   550e8400-e29b-41d4-a716-…  │   ← new UUID minted per checkout
  └──────────────┬───────────────┘
                 │  attempt #1
                 ▼
  ┌─────────────────────────────────────────────────────────────┐
  │ IDEMPOTENCY MIDDLEWARE (dedupe gate, runs before handler)   │
  │                                                             │
  │   key "550e84…" in store?                                   │
  │      │ NO                                                   │
  │      ▼                                                      │
  │   run handler ──► charge $49 ──► INSERT subscription        │
  │                      │              │                       │
  │                      └── SAME TX ──┴─► INSERT dedupe row    │
  │                          (fix #1: commit together)          │
  └──────────────────────────────┬──────────────────────────────┘
                                 │
                    ┌────────────┴────────────┐
                    │ DB                      │
                    │ subscriptions            │
                    │   UNIQUE (client_ref)   │ ← fix #2: natural-key
                    │ idempotency_keys        │   backstop
                    │   PK (key), TTL 24h     │
                    └─────────────────────────┘

  ── the 200 response is lost in the tunnel; client times out ──

  CLIENT                               PAYMENT SERVICE
    │  attempt #2 (auto-retry, SAME key, SAME body)
    │────────────────────────────────►
    │                                 key "550e84…" in store?
    │                                       │ YES
    │                                       ▼
    │                                 replay stored 200 + sub id
    │◄──────────────────────────────── no handler, no new charge

  RESULT: user sees one successful upgrade. Bank shows one $49
  charge. Even if the server had crashed inside the crash window,
  the second attempt's INSERT would hit the UNIQUE(client_ref)
  backstop and return the existing subscription instead of
  creating a duplicate.
```

Three small details make this production-grade rather than demo-grade. First, the dedupe gate must be **atomic**: two retries arriving at the same millisecond (double-tap on the pay button) both check the store, both see "unknown", and both run the handler — so the store's primary key on the idempotency key is what serializes them: one INSERT wins, the other blocks and then replays. Second, the middleware should hash the request body and store it next to the key, so a same-key-different-payload retry is caught with a `409` instead of silently returning someone else's result. Third, idempotency keys are for *your* retries — a background worker, a webhook redelivery, a user mashing the button — all of them must agree on one key per intent or the pattern quietly stops working.

The punchline: idempotency keys move the burden of duplicate-safety from "hope the network behaves" to "design for the network misbehaving." They don't eliminate duplicate execution — nothing can, once a process can die between two side effects — but they guarantee that no matter how many times a request arrives, the user is charged once, the seat is booked once, and the system's answer never changes. That single property is why no serious payment, booking, or messaging API ships without them, and why the pattern is quietly becoming the default answer to every "what if the retry double-fires?" question in distributed systems.

---

day - 9

## Transactional Outbox Pattern

### Definition:

The Transactional Outbox Pattern makes **publishing an event as reliable as committing a database transaction** — by not publishing from the application at all. Instead of sending a message to a broker inside your request handler, you write the event as a row into an **outbox table, in the *same* database transaction** as the business change. A separate *relay* process later reads those rows and forwards them to the broker. Your database transaction becomes the single point of atomicity: business data and its events either commit together or not at all.

The pattern exists because of the **dual-write problem**. In any event-driven system, some operation must do two writes that no single transaction can span: (1) change rows in your database and (2) publish an event to a broker. A relational transaction cannot reach into Kafka; a broker transaction cannot reach into Postgres. So the two writes are never atomic, and whichever order you pick, you expose a window where the two systems disagree — the two classic failure modes:

- **Save-then-publish**: the order commits, the process crashes before the event goes out → the event is *lost forever*. The order exists but nothing downstream ever hears about it, and that inconsistency never heals on its own.
- **Publish-then-save**: the event goes out, then the business write rolls back → a *phantom event*. Consumers eagerly send a confirmation email, decrement stock, and charge a card for an order that does not exist.

The outbox closes both windows with one trick: the event stops being a side effect and becomes **data**. You move the publish outside the transaction and outside the request path entirely, and the message broker becomes just one more *consumer of your database*.

NAIVE DUAL-WRITE vs OUTBOX — where atomicity lives:
════════════════════════════════════════════════════════════════════════

```
NAIVE — two writes, two systems, NO shared atomicity:

  handler
    │  ① write business row        ② send event to broker
    ▼                                ▼
  ┌───────────────┐              ┌──────────────┐
  │  ORDERS  DB   │              │   BROKER     │
  └───────────────┘              └──────────────┘
      no transaction can span both ──► either write fails alone:
        • ① ok, crash before ②  → event LOST (order exists,
                                     nobody notified, never heals)
        • ② ok, ① rolls back    → PHANTOM event (consumers act on
                                     an order that doesn't exist)


OUTBOX — the event is a ROW, committed atomically with the change:

  handler
    │  BEGIN TX
    │    INSERT INTO orders (...)
    │    INSERT INTO outbox (id, aggregate_id, type,
    │                        payload, created_at)
    │  COMMIT                      ← one transaction: both or neither
    ▼
  ┌───────────────────────────────────────┐
  │            ORDERS  DB                 │   the DB is now source of
  │  ┌──────────────┐  ┌───────────────┐  │   truth for state AND events
  │  │ orders       │  │ outbox        │  │
  │  │              │  │ OrderPlaced…  │──┼──► ② relay reads new rows
  │  └──────────────┘  └───────────────┘  │     (polling OR CDC tailer)
  └───────────────────────┬───────────────┘
                          │ ③ publish — relay retries forever,
                          ▼    crash/restart loses nothing
                  ┌───────────────┐
                  │    BROKER     │   at-least-once delivery
                  └───────┬───────┘
                          │ ④ consumers MUST be idempotent
                          ▼    (duplicates are expected, not bugs)
```

  ┌────────────────────────────────────────────────────────────────┐
  │  KEY IDEA: you never "send" an event in the request path —     │
  │  you SAVE it. Delivery becomes a separate, retryable job       │
  │  owned by a relay, fully decoupled from the business write.    │
  └────────────────────────────────────────────────────────────────┘

How the relay works — two flavors, same job (read new rows → publish → mark dispatched):

```
POLLING PUBLISHER                          TRANSACTION LOG TAILING (CDC)
─────────────────────                      ─────────────────────────────
  app process, your code                    Debezium / CDC connector
  every N ms (or on commit hook):           reads the DB's write-ahead
                                            log / replication stream
  SELECT ... FROM outbox
  WHERE dispatched_at IS NULL               ┌───────────────────────┐
  ORDER BY id                               │  outbox table          │
  FOR UPDATE SKIP LOCKED                    │  INSERT ... ──► WAL    │
        │                                   └───────────┬───────────┘
        ▼                                               │ tails binary
  publish each row ──► broker                           ▼
        │                                     ┌───────────────────────┐
        ▼                                     │  Debezium / CDC       │
  UPDATE outbox                               │  ──► broker           │
  SET dispatched_at = now()                   └───────────────────────┘
        │                                     • zero app code, no
  + easy to write, test, debug                  polling latency
  + full control (batching, retry)            • but read-only window
  – adds latency (poll interval)                on your DB internals
  – watch out: MULTIPLE relay instances        • ordering preserved by
    can double-publish → make the relay         the log itself
    idempotent (UNIQUE(id)) or run ONE
```

Non-negotiable companions of the pattern:

- **At-least-once, never exactly-once.** A relay can crash after publishing but before marking the row dispatched, so the same event *will* sometimes arrive twice. Consumers must dedupe (store `processed_event_id` with a UNIQUE constraint, or make the consumer's own write idempotent by natural key). This is not a flaw — it's the honest contract that makes the whole system simple.
- **Ordering is a choice, not a given.** If per-aggregate ordering matters (events for the *same order* must arrive in sequence), either run a single relay or partition the topic by `aggregate_id` — and remember multiple relay instances polling in parallel can hand rows to the broker out of order.
- **The outbox table needs a janitor.** Dispatched rows accumulate forever if nobody deletes them. Common policy: delete rows older than N days/hours (after confirming the broker took them), or keep them briefly as an audit/replay trail. Retention is a business decision — storage is cheap, ordering guarantees are not.
- **It is not Event Sourcing.** Event sourcing stores *all* state as events and rebuilds aggregates from them; the outbox is just a delivery queue that mirrors your normal writes. They coexist happily — and so do the outbox and CQRS (day - 1): the outbox is often the reliable pipe that feeds a CQRS read model.

When you should *not* reach for it: purely synchronous request/response with no events; workloads where a lost event is genuinely tolerable (metrics, best-effort notifications) — there the pattern is pure overhead; and systems that need a broker message *before* the DB commit becomes visible. Otherwise, for anything where "this happened" must eventually reach other services exactly once per occurrence, the outbox is the standard answer — which is why it shows up in every serious microservices guide, and why CDC-based relays (Debezium + Kafka Connect) turned it into mainstream production practice through 2024–2026.

### Example:

"KopiKode" — an online coffee-subscription shop — splits checkout into services. The `orders` service is the source of truth for orders. When a customer checks out, three other services must learn about it: `billing` (charge), `inventory` (reserve beans), and `notifications` (send "order received" email). The first version did the naive dual-write — and production found both failure modes within a week:

```
NAIVE VERSION — the week from hell:

  checkout handler
    │  INSERT order (ok)
    │  kafka.send("OrderPlaced")        ── ① 3 AM: crash between the
    ▼                                      two lines → order stored,
  ┌────────────┐   ┌──────────┐            email never sent, stock
  │ orders DB  │   │  broker  │            never reserved. Support
  └────────────┘   └──────────┘            tickets: "I ordered, no
                                           confirmation, beans never
    and the reverse: a retry sent the      shipped."
    event twice for ONE order after a
    timeout → customer charged 2×,         ── ② double-send on retry:
    two emails, beans reserved 2×.             no dedupe anywhere.
```

They rebuilt it with an outbox. The checkout handler now does exactly ONE transactional write; a polling relay (their Go service, `SELECT … FOR UPDATE SKIP LOCKED`, every 100 ms) does the publishing; and consumers dedupe on `event_id`:

```
WITH THE OUTBOX — order → event → three consumers, no lost or phantom events:

  CUSTOMER taps "Checkout"
        │
        ▼
  ┌─────────────────────────────────────────────┐
  │ ORDERS SERVICE — ONE local transaction      │
  │                                             │
  │   BEGIN;                                   │
  │     INSERT INTO orders (id, sku, qty, …)   │   ← business change
  │     INSERT INTO outbox (id, aggregate_id,  │
  │       type, payload)                       │   ← the event, same tx
  │       VALUES (evt_9f2c, 'order_4711',      │
  │               'OrderPlaced', '{…}');       │
  │   COMMIT;                                  │
  │                                             │
  │   ┌──────────────┐   ┌───────────────────┐ │
  │   │ orders       │   │ outbox            │ │
  │   └──────────────┘   │ evt_9f2c OrderPl… │ │
  │                      └───────────────────┘ │
  └─────────────────────────────┬───────────────┘
                                │
                  RELAY (Go, polls every 100 ms:
                  SELECT … FOR UPDATE SKIP LOCKED)
                                │
                                │  publish evt_9f2c ──► topic "orders"
                                ▼
                     ┌──────────────────────┐
                     │        KAFKA         │   at-least-once
                     └──┬───────┬───────┬───┘
                        │       │       │
            ┌───────────┴──┐ ┌──┴──────┐ └──────────┐
            ▼              ▼          ▼            ▼
     ┌──────────────┐ ┌──────────┐ ┌──────────────┐
     │   BILLING    │ │INVENTORY │ │ NOTIFICATIONS │
     │  charge $12  │ │reserve 1 │ │  send email   │
     └──────┬───────┘ │bag "Gayo"│ └──────┬───────┘
            │         └──────────┘        │
            └─────────── all dedupe on ───┘
                 event_id: INSERT INTO processed (event_id)
                 … UNIQUE(event_id) → duplicate delivery of
                 evt_9f2c is swallowed silently, never re-applied
```

```
THE CRASH TEST — why this survives what the naive version didn't:

  relay publishes evt_9f2c ──► broker ACKs ──► crash
        │                                        │
        │         relay dies BEFORE marking       │
        ▼         outbox row dispatched           ▼
  outbox: evt_9f2c  dispatched_at = NULL    (row still pending)

  relay restarts → re-reads evt_9f2c → publishes AGAIN
        │
        ▼
  billing receives evt_9f2c twice:
     1st: INSERT processed(evt_9f2c) ✓ → charge $12
     2nd: INSERT processed(evt_9f2c) ✗ UNIQUE VIOLATION
          → caught, skipped, NO second charge

  RESULT: the event is delivered at least once, the CHARGE happens
  exactly once (thanks to the consumer's dedupe), and no order is
  ever silently missing its events. Even a crash in the relay is
  just "publish again" — the DB never lies about what happened.
```

The team's final shape: orders DB holds state + outbox; the Go relay (or, later, Debezium tailing the WAL) moves events to Kafka; billing, inventory, and notifications all dedupe by `event_id`; a nightly cleanup job deletes outbox rows dispatched more than 24 h ago. The checkout path gained a few microseconds writing one extra row — and lost an entire class of "it happened but nobody knows" incidents.

The punchline: the Transactional Outbox Pattern is the pragmatic answer to the oldest lie in distributed systems — "I'll write to my DB and then tell everyone about it." You cannot make two systems commit atomically, so you stop trying: you make the *event itself* part of the one transaction you do control, and you treat the broker as a downstream consumer that lags a little. Events become durable facts that survive crashes, restarts, and deploys by construction — and the only price is a relay you can restart freely and consumers that must tolerate (and dedupe) a duplicate now and then. That trade — atomicity where it's free, idempotency where it's needed — is why the outbox, not distributed transactions, became the default backbone of event-driven systems.

---

day - 10

## Confidential Computing

### Definition:

Confidential Computing is a **hardware-enforced runtime isolation** model: a workload runs inside a *Trusted Execution Environment (TEE)* — a cryptographically sealed region of CPU and GPU memory whose encryption keys are generated **inside the silicon and never leave it**. The consequence is the whole point of the pattern: everyone who *operates* the machine — the hypervisor, the host kernel, the cloud provider's privileged staff, a fully root-compromised host OS — sees only ciphertext where your model weights, your prompts, and your patient records used to be.

The framing that makes it click. Security has covered **two states of data** for thirty years, and quietly skipped the third:

- **at rest** → disk / object-storage encryption
- **in transit** → TLS
- **in use** → …nothing, by design

To compute on data, a CPU *must* decrypt it into DRAM — and DRAM belongs to whoever owns the host. Disk encryption ends at the bootloader; TLS ends at the socket. The moment the bytes have to be understood, they are plaintext owned by the operator. That is why every cloud workload has always carried an unstated assumption: **trust the operator**.

Confidential computing moves that assumption down into the silicon. Instead of trusting the operator, you trust the chip vendor's hardware — plus a cryptographic proof that the right code is running. That proof is **remote attestation**, and it is the second half of the pattern:

1. **Runtime isolation** — the TEE's memory is encrypted in place; the host cannot read it, and (in 2026 silicon) cannot silently re-map, replay, or roll it back either.
2. **Attestation** — the TEE can produce a hardware-signed *Evidence* artifact (an SEV-SNP report / TDX Quote / NVIDIA GPU attestation report) proving exactly which firmware, kernel, and application measurements were loaded, and that debug mode is off.

Neither half works alone. Isolation without attestation is unverifiable (you cannot tell a genuine TEE from a simulator, or from a legitimate TEE running the attacker's image). Attestation without isolation is a notarized promise that nobody enforces. Together they are the product: **"prove what is running, then hand it secrets that even its own host cannot read."**

WITHOUT CONFIDENTIAL COMPUTING — a standard VM: the host reads everything
════════════════════════════════════════════════════════════════════════════

```
                    ┌─────────────────────────────────────────────┐
   patient data ───►│  CLOUD PROVIDER'S PHYSICAL MACHINE          │
   model weights    │                                             │
   (TLS in transit) │   ┌─────────────────────────────────────┐   │
                    │   │  YOUR VM / CONTAINER                │   │
                    │   │   ┌───────────────────────────────┐ │   │
                    │   │   │  DRAM  —  PLAINTEXT           │ │   │
                    │   │   │   • prompt: "pasien Budi, …"  │ │   │
                    │   │   │   • weights: 7B fp16          │ │   │
                    │   │   │   • KV-cache: full reasoning  │ │   │
                    │   │   └───────────────────────────────┘ │   │
                    │   └─────────────────────────────────────┘   │
                    │        ▲              ▲              ▲      │
                    │   hypervisor     host kernel   provider ops │
                    │   ═════════ ALL OF THEM CAN READ ═════════► │
                    └─────────────────────────────────────────────┘

   Disk encryption and TLS BOTH END HERE — at the doorstep of the
   machine. The instant the CPU needs the bytes to compute on them,
   they are plaintext, and the operator owns the plaintext.
```

WITH CONFIDENTIAL COMPUTING — a Confidential VM (CVM) + GPU TEE
════════════════════════════════════════════════════════════════════════════

```
                    ┌─────────────────────────────────────────────┐
   patient data ───►│  CLOUD PROVIDER'S PHYSICAL MACHINE          │
   model weights    │  (operator: untrusted, and now irrelevant)  │
                    │   ┌─────────────────────────────────────┐   │
                    │   │  CONFIDENTIAL VM  ── TEE ──         │   │
                    │   │   ┌───────────────────────────────┐ │   │
                    │   │   │  DRAM — ENCRYPTED (per-page)  │ │   │
                    │   │   │  0x9f3a…  0x41bc…  0x77de…    │ │   │
                    │   │   └───────────────────────────────┘ │   │
                    │   │            ▲ decrypts ONLY here     │   │
                    │   └────────────┼────────────────────────┘   │
                    │        ┌───────┴────────┐                   │
                    │        │  CPU / GPU     │ keys are born in  │
                    │        │  SILICON       │ silicon, never    │
                    │        └───────┬────────┘ handed to the host│
                    │   AMD SEV-SNP · Intel TDX · ARM CCA ·       │
                    │   NVIDIA C-CAP (H100 / H200 / B200)         │
                    │                                             │
                    │  hypervisor ─┐                              │
                    │  host kernel ┼─► see CIPHERTEXT + MEASURE-  │
                    │  provider ops┘   MENTS, never the contents  │
                    └─────────────────────────────────────────────┘

  ┌────────────────────────────────────────────────────────────────┐
  │  KEY IDEA: the trust boundary moves from "the party running    │
  │  the hardware" to "the chip vendor's silicon + a signed proof  │
  │  of what is loaded." You stop asking the operator to be good   │
  │  and start verifying, cryptographically, that they can't peek. │
  └────────────────────────────────────────────────────────────────┘
```

The four pillars (as the Confidential Computing Consortium frames them) are a checklist you can audit against:

- **Hardware root of trust** — encryption keys are fused into the chip; the host cannot derive them.
- **Attestation** — signed Evidence of identity, initial state, and TCB (firmware microcode) version.
- **Sealed storage** — data encrypted to the TEE's *identity*, so it can only be unsealed by a TEE matching the same measurement. This is "encryption at rest" rebound to *code* instead of to a machine.
- **Secure channels** — a session key negotiated *after* attestation, so the wire is pinned to the verified enclave rather than to a hostname.

The 2026 hardware landscape, and why "the TEE" is usually several TEEs stitched together:

```
PROTECTING A FULL AI INFERENCE STACK — isolation at both ends of the bus
════════════════════════════════════════════════════════════════════════

  ┌──────────────────────────── CONFIDENTIAL VM ──────────────────────────┐
  │                                                                       │
  │  AMD SEV-SNP / Intel TDX / ARM CCA                                    │
  │  ┌───────────────────────────────────────┐                            │
  │  │ CPU TEE: guest DRAM encrypted,        │   · whole-VM isolation     │
  │  │ pinned launch measurement,            │   · 3–6% overhead on       │
  │  │ REVERSE MAP TABLE blocks remap/replay │     inference in 2026       │
  │  └───────────────┬───────────────────────┘                            │
  │                  │ PCIe — bus traffic encrypted by the CPU             │
  │  ┌───────────────▼───────────────────────┐                            │
  │  │ NVIDIA C-CAP GPU TEE (H100/H200/B200) │   · VRAM encrypted in HBM3e│
  │  │ weights + activations + KV-CACHE      │   · ~7% throughput cost    │
  │  │ live encrypted in VRAM                │   · has its OWN attestation│
  │  └───────────────────────────────────────┘     report, bound to the   │
  │                                                 CPU's — "composite    │
  │  AWS Nitro Enclaves: no persistent storage, no  attestation"          │
  │  networking by default — you must proxy every                         │
  │  byte through the parent, by design.                                  │
  └───────────────────────────────────────────────────────────────────────┘

  Rule of thumb: encrypt the CPU side only and your weights still sit in
  plaintext VRAM. Confidential inference means a CVM PLUS a CC-capable
  accelerator, with an attestation that covers BOTH.
```

What changed by 2026 — this is the reason the term moved from "compliance checkbox" to "default":

- **Overhead collapsed.** First-generation TEEs cost 30–40% throughput. Current SEV-SNP/TDX silicon sits at **3–6%** on inference, and NVIDIA C-CAP at roughly **7%** now that the memory-encryption engines live inside the HBM3e controllers. Below the variance between two cloud regions, the cost stops being a business decision.
- **Frameworks caught up.** By 2026 vLLM, TGI, and Triton ship first-class confidential modes: pass a flag, weights load across an encrypted path into a CC GPU, the KV-cache stays sealed, tokens stream out over an attested channel. Before this, every team hand-instrumented its own inference server.
- **Consumer-scale confidential AI went mainstream.** Apple's Private Cloud Compute, Meta's Private Processing, and Azure Confidential VMs / Google Confidential Space made "the operator cannot read it" a consumer-visible claim, not a B2B SKU.
- **Procurement language hardened.** Compliance text now literally reads: *model weights and inference inputs must be protected against access by the infrastructure provider, attested at workload launch, and never appear in plaintext in host memory.* That sentence is a TEE requirement written as a purchase order.

Where it fits — and where it does not:

```
┌──────────────────────────────────────────────────────────────────────┐
│  GOOD FIT (a TEE is doing real work)                                 │
│  • Regulated data you cannot show the host: health, finance, gov     │
│  • Proprietary weights + untrusted/foreign infrastructure            │
│  • Multi-party data clean rooms (two banks, one model, no raw data   │
│    ever visible to either counterparty or the cloud)                 │
│  • Cloud bursting / colo where you do not own the rack               │
│  • Sovereignty rules that forbid foreign staff touching the data     │
│                                                                      │
│  NOT WORTH IT (the operational tax buys nothing)                     │
│  • Any workload you would happily run in your own datacenter         │
│  • Public data + public model: no secret exists to protect           │
│  • Teams that will not run attestation verification — a TEE with     │
│    no verifier is decoration                                         │
│  • Latency-hard budget with autoscaling churn (attestation is part   │
│    of cold start; budget it, and pre-warm)                           │
└──────────────────────────────────────────────────────────────────────┘
```

Being honest about the threat model is what separates the deployments that hold from the ones that get breached. A TEE does **not** defend against **side channels** (microarchitectural leakage is relocated, not eliminated), **output leakage** (memory isolation does not stop a jailbroken or poisoned model from exfiltrating through its own answers), **compromised supply chain** (a poisoned weight file loads perfectly — and privately), **a lazy verifier**, or a **debug/swap path left enabled** (SEV-SNP's `DEBUG` flag and TDX debug mode must be off; the host must not be able to core-dump guest memory).

One honest comparison worth pinning, because the two get blurred constantly — and because this journal already covered **Homomorphic Encryption (FHE)**:

```
TEE (CONFIDENTIAL COMPUTING)   vs   FHE (HOMOMORPHIC ENCRYPTION)
════════════════════════════════════════════════════════════════

  TRUST ANCHOR
    TEE  a hardware vendor's silicon (AMD / Intel / ARM / NVIDIA)
    FHE  nobody — the security is mathematical, not physical

  DATA IN USE
    TEE  plaintext, but only inside a sealed + attested box
    FHE  ciphertext end to end — never decrypted anywhere

  vs MALICIOUS ADMIN / HYPERVISOR
    TEE  ✓ blocked (host sees ciphertext)
    FHE  ✓ blocked (there is nothing to see)

  vs SIDE-CHANNEL ATTACK
    TEE  ⚠ residual risk — leakage is relocated, not removed
    FHE  ✓ structurally immune (no hardware is trusted)

  POST-QUANTUM
    TEE  ⚠ AES-based, breakable by a large quantum computer
    FHE  ✓ lattice-based, believed quantum-safe

  CODE CHANGE TO ADOPT
    TEE  ~none — deploy the same binary into a TEE
    FHE  full rewrite: arithmetic must become FHE arithmetic

  TIME TO PRODUCTION
    TEE  days to weeks
    FHE  months to years, with cryptography specialists

  COST
    TEE  ~1.0–1.1× baseline (3–7% overhead in 2026)
    FHE  ~100×–1,000×+ — usually fatal for interactive work
```

The pragmatic reading: **FHE is the stronger security claim, TEEs are the only one you can actually ship this year.** They are complements — and FHE is quietly useful in exactly the niches a TEE cannot serve, like when no one at all may hold a decryption key.

### Example:

"KlinikSehat", a Jakarta health-tech, wants to launch an LLM triage assistant that reads patient records. Its constraints are brutal but normal: the hospital contracts forbid the cloud provider from ever accessing patient data; Indonesian personal-data law requires demonstrable technical controls; and the model is a fine-tuned, million-dollar asset that must not leak to the provider either. They refuse to build a datacenter. So they deploy a **confidential inference stack** — and the shape of the system changes from an architecture diagram into a *handshake*.

```
CONFIDENTIAL INFERENCE, END TO END — nothing readable until attestation passes
══════════════════════════════════════════════════════════════════════════════

   CLINIC (verifier side)                      CLOUD (untrusted operator)
  ┌───────────────────────┐
  │ client SDK / KMS      │  1. send a NONCE (freshness / anti-replay)
  │ reference values      │──────────────────────────────►  starts a CVM
  │  pinned image hash    │                               with the pinned
  │  min TCB version      │                               image hash
  │  debug = OFF          │
  └──────────┬────────────┘
             │                             2. TEE asks its HARDWARE for Evidence
             │                                ├─ SEV-SNP attestation report (VCEK-signed)
             │                                ├─ Intel TDX Quote
             │                                └─ NVIDIA GPU CC attestation report
             │                                …the GPU's report is bound to the CPU's
             │                                → the two are ONE verified composite
             │  ◄─────── signed Evidence ──────┘
             │
             ▼  3. VERIFY against policy (OPA/Rego, not a hard-coded ==):
                ├─ measurement == our image hash?             ✓
                ├─ TCB ≥ minimum microcode version?           ✓
                ├─ nonce matches the one we just sent?        ✓ (no replay)
                ├─ debug flag OFF, no core-dump path?         ✓
                └─ signature chains to AMD/Intel/NVIDIA root  ✓
                        │
                        │  ✗ ANY FAILURE → abort, release nothing.
                        │    That is the entire point of the pattern.
                        ▼
  ┌──────────────────────────────────────────────────────────────────┐
  │  4. UNSEAL — key release is GATED BY ATTESTATION                 │
  │                                                                  │
  │   KMS policy: kms:RecipientAttestation == <expected measurement> │
  │        │                                                         │
  │        │  KMS refuses to hand over the model-decryption key to   │
  │        │  anything that cannot prove what it is. A host that     │
  │        └─ steals the encrypted weights still gets 0x41bc…  ✓     │
  └──────────────────────────────────────────────────────────────────┘
                        │
                        ▼
  ┌──────────────────────────────────────────────────────────────────┐
  │  5–6. LOAD + SERVE, all inside the TEE                           │
  │                                                                  │
  │   prompt "pasien Budi, 54, diabetes, metformin…"                 │
  │        │                                                         │
  │        ▼  plaintext exists ONLY inside encrypted DRAM/VRAM       │
  │   [ CPU TEE: prompt + tokenizer ]──PCIe(encrypted)──►[ GPU TEE:  │
  │                                                        weights + │
  │                                                        KV-cache ]│
  │        │                                                         │
  │        ▼  response encrypted to the clinic's session key         │
  │   "Saran triage: …"  ──►  clinic decrypts                        │
  └──────────────────────────────────────────────────────────────────┘

  MEANWHILE, THE CLOUD ADMIN'S DASHBOARD SHOWS:
  ┌──────────────────────────────────────────────┐
  │  CVM-7f3a   ATTESTED  ✓   TCB 3.1.2          │
  │  DRAM  0x9f3a: 9d 41 bc 77 de 00 3a …        │
  │  VRAM  0x41bc: c0 ff ee 1a 9f 3a 44 …        │
  │  egress: 2.1 GB   cpu: 41%   net: ok         │
  │  ── no prompt, no weights, no records ──     │
  └──────────────────────────────────────────────┘
```

Two design decisions carry most of the weight here, and both are easy to get wrong:

**The verifier is not the cloud.** If KlinikSehat accepts the provider's own attestation service verdict, it has merely moved the trust, not removed it — the operator can lie about a quote it validates itself. The IETF RATS framework (RFC 9334) names the roles precisely: the **Attester** (the TEE) produces **Evidence**; the **Verifier** checks it against **Reference Values**; the **Relying Party** (KlinikSehat's KMS) decides whether to release the key. Two standard topologies fall out:

```
BACKGROUND CHECK MODEL — every relying party verifies independently
────────────────────────────────────────────────────────────────────

   TEE ──Evidence──► Relying Party ──Evidence──► Verifier
                          ▲                          │
                          └───── Attestation Result ─┘
   decision: the RELYING PARTY releases the key, on a FRESH result
   cost:     every service that must decide needs verifier access

PASSPORT CHECK MODEL — verify once, then present a token
────────────────────────────────────────────────────────

   TEE ──Evidence──► Verifier ──► Attestation Result (signed token)
     │
     └──presents token──► Relying Party ──► validates the SIGNATURE only
   decision: any service can decide, with NO verifier access
   cost:     a reusable token exists — so bind it to a short TTL + audience

GOOD FOR:  few, high-stakes consumers     GOOD FOR:  many services / proxies
           key release to one KMS                      fleet-wide rollout gates
```

KlinikSehat self-hosts the verifier (Trustee-style) inside its own VPC for the high-stakes key release, and lets downstream proxies accept passport-style tokens so every microservice does not need to re-verify a hardware quote.

**Patching is an attestation event.** This is the operational trap nobody sees in the architecture diagram: every routine kernel update changes the launch measurement, so the *reference values* must be provisioned to the verifier **before** the updated guests boot — otherwise a fleet-wide patch at 02:00 becomes a fleet-wide attestation outage at 02:01. The mature pattern is policy, not equality: pin the measurement for the tight deployments, allow a *minimum TCB version* window during rollouts, and keep an explicit allow-list of hardware models and a hard rule that debug mode is off. "Measurement churn" is a first-class release-engineering concern the day you adopt confidential computing, not an afterthought.

Finally, the part KlinikSehat gets right by *not* trusting the TEE for everything. The TEE protects memory, not meaning. So the triage bot keeps three controls *outside* the enclave: output filtering and redaction on the response path (because a poisoned or jailbroken model exfiltrates happily through an attested channel — the pipe is private, the payload is not), rate limits and per-tenant quotas, and ECC memory plus disabled swap as a baseline, since memory corruption inside an enclave is far harder to recover from when the host is not allowed to intervene. And because attestation adds latency, they budget it into autoscaling — pre-warming confidentially, the same lesson as any cold start, with a verifier handshake stapled to the front.

The punchline: confidential computing is the first time the phrase *"trust no one"* became an actual machine instruction. For thirty years the cloud forced a trade — get scale and elasticity, and in exchange let the operator see your plaintext. TEEs plus attestation break that trade: the operator keeps the hardware, you keep the secrets, and a signed quote from the chip decides who is lying. It does not make a system safe — a TEE is a memory-isolation primitive, not a security programme, and teams that deploy one without a verifier, without output controls, and without a patch plan are buying a certificate rather than a control. But by 2026 the overhead has fallen below the noise floor and the frameworks ship it as a flag, which is why the honest framing is no longer "should we adopt confidential computing?" but "which of our workloads can we still afford to have the provider read?"

---

day - 11

## Continuous Batching

### Definition:

**Continuous batching** — also called **iteration-level scheduling**, **in-flight batching** (TensorRT-LLM) or **persistent batching** (LMDeploy) — is the LLM-serving scheduler design in which the batch sitting on the GPU is **re-composed on every single decode iteration**. The instant a sequence emits its stop token, its slot and its KV-cache memory are freed; the instant a slot is free, a waiting request is admitted into it and folded into the running batch for the next forward pass. There is no such thing as a "batch lifetime" — the batch's shape changes every token.

It exists because of a structural defect in the naive approach. In classic **static batching** (which is what a hand-rolled `model.generate()` loop does), you collect N prompts, run them together, and return results only when *all N* are done. The batch therefore runs for as many decode steps as its **longest** member needs. Every shorter request finishes early and then occupies a dead slot — padded tensor row, KV cache still reserved — contributing nothing but holding resources. Since real output lengths vary by 50x or more (a 20-token classification vs. a 2,000-token reasoning trace), most slots are idle most of the time. The dashboard says "GPU busy"; the throughput says otherwise.

The intuition to hold onto is that LLM decoding has two phases with opposite physics, and only one of them parallelizes well on its own:

```
PREFILL  (the prompt)      : all prompt tokens processed at once
                             → big matrix multiplies, COMPUTE-bound,
                               high arithmetic intensity, GPU loves it

DECODE   (one token at a time) : 1 new token per sequence per step, but the
                             model must re-read the entire KV cache to do it
                             → tiny compute, HUGE memory traffic,
                               MEMORY-BANDWIDTH-bound, GEMM with batch=1
                               is an almost empty GPU

CONSEQUENCE: decoding 1 sequence alone wastes ~95% of an H100.
             You need MANY sequences in flight to saturate the hardware.
             Static batching cannot keep many in flight, because finished
             work cannot leave and new work cannot enter.
```

The fix is to change the **unit of scheduling** from a request to a **decode iteration**. This was formalized by the **Orca** paper (OSDI 2022), which named it iteration-level scheduling; **vLLM** (2023) made it runnable at scale by pairing it with **PagedAttention** — a paged KV cache where each sequence holds a list of fixed-size blocks instead of one contiguous reservation. That pairing is not optional, it is co-designed: dynamic admission/eviction means sequence memory is constantly being allocated and returned, and with naive contiguous allocation you get fragmentation plus up-to-60% memory waste, which in turn caps how many sequences you can keep in the batch. Continuous batching is the *scheduling* innovation; PagedAttention is the *memory* innovation that makes it possible.

```
STATIC BATCHING — the batch is FROZEN until the slowest member finishes
═══════════════════════════════════════════════════════════════════════════
slot │                     decode iterations →
     │  t0      30       300        900             1500
─────┼────────────────────────────────────────────────────────────────
 R1  │ ██████████████████████████████████████████████  done (1500 tok)
 R2  │ ██████ done ░░░░░░░ idle ░░░░░░░░░ idle ░░░░░░░ idle
 R3  │ ██████ done ░░░░░░░ idle ░░░░░░░░░ idle ░░░░░░░ idle
 R4  │ ██████ done ░░░░░░░ idle ░░░░░░░░░ idle ░░░░░░░ idle
 R5  │ ██████ done ░░░░░░░ idle ░░░░░░░░░ idle ░░░░░░░ idle
 R6  │ ██████ done ░░░░░░░ idle ░░░░░░░░░ idle ░░░░░░░ idle
─────┴────────────────────────────────────────────────────────────────
 response time: t1500 for EVERYONE  ·  slot-time wasted: ~86%
 new request arriving at t=31 → puts on a queue and waits for t1500
 KV cache for all 6 slots stays RESERVED the whole time


CONTINUOUS BATCHING — the batch is re-composed at every iteration
═══════════════════════════════════════════════════════════════════════════
slot │  it0     it30      it31          it300      it301       it1500
─────┼────────────────────────────────────────────────────────────────
 R1  │ ███████████████████████████████████████████████████████ done
 R7  │            ▲ADMIT  ███████████████ done
 R8  │                              ▲ADMIT ████████████████████ done
 R9  │                                          ▲ADMIT ██████████ done
─────┴────────────────────────────────────────────────────────────────
 short requests return at it30 (not it1500)
 every freed slot is refilled on the NEXT iteration
 throughput ≈ 5x–23x static batching at comparable p50 latency
```

The mechanism, spelled out, is a three-step loop the engine runs forever:

```
EVERY DECODE ITERATION — the scheduler executes this, endlessly
┌──────────────────────────────────────────────────────────────────────┐
│                                                                      │
│  1. STEP    one forward pass for every sequence currently running    │
│             (one new token each — this is the GPU's real work)       │
│                                                                      │
│  2. EVICT   any sequence that just emitted its stop token or hit     │
│             max_tokens → its slot AND its KV blocks are released     │
│             in THIS iteration, not at the end of a batch             │
│                                                                      │
│  3. ADMIT   pull waiting requests from the queue into the freed      │
│             slots (run their prefill, or a chunk of it), so the      │
│             next iteration starts with a fuller batch                │
│                                                                      │
└──────────────────────────────────────────────────────────────────────┘
        ▲                                                    │
        └──────────── preempt if the KV pool runs dry ────────┘
             (swap the whole sequence out, or drop and RECOMPUTE
              its KV later — Oracle/preemption policy, not luck)

TOKEN BUDGET: the scheduler does not admit everything it could. Each
iteration is capped by a token budget (max_num_batched_tokens), which is
the real dial between "many short requests" and "a few long ones".
```

**Chunked prefill** is the companion scheduling decision. A prefill is *not* free: if a 30,000-token prompt is processed in one giant pass, every already-decoding sequence on the GPU stalls for the duration — one user's long document spikes the inter-token latency of everyone else, which shows up as a p99 disaster while p50 looks great. Chunked prefill (SARATHI, 2023; the paper calls the resulting mixed batch **decode-maximal batching**) slices the prompt into token-budget-sized chunks and interleaves them with ongoing decode steps, so prefill chunks supply the parallel work needed to saturate compute while decode tokens ride along for free.

```
WITHOUT chunked prefill          WITH chunked prefill
─────────────────────────        ─────────────────────────
 it 100: 30k-token prefill        it 100: 4k prefill + 127 decodes
 it 101: (blocked)                it 101: 4k prefill + 127 decodes
 ...                              it 102: 4k prefill + 127 decodes
 every decode client sees a       decodes keep streaming between
 200–800ms ITL spike              chunks · ITL stays smooth
 TTFT: best possible              TTFT: slightly worse (more steps)
 p50 great, p99 wrecked           p50 / p99 both acceptable
```

The trade-off, stated honestly, is a knob war rather than a free lunch:

| Dial | Turn it up | Turn it down |
|---|---|---|
| `max_num_seqs` | higher throughput, fuller GPU; more KV pressure, slower per-token latency, more preemption | snappier individual responses, cheaper memory, GPU underused |
| `max_num_batched_tokens` | bigger batches (better throughput, better TTFT for long prompts) | finer-grained scheduling, lower ITL jitter |
| `enable_chunked_prefill` | smooth p99 ITL under mixed prompt lengths | best possible TTFT; risks head-of-line blocking on long prompts |
| Admission policy | throughput-first (fill every slot) | fairness-first (per-tenant caps, priority queues) — otherwise a burst of long generations starves short interactive requests |

The pattern's limits are worth naming so it does not become cargo cult. It optimizes **throughput and p50**, not tail latency: any real fleet still needs admission control, per-tenant quotas and load shedding in front of it, because continuous batching will happily let an unbounded queue turn into an unbounded TTFT. It also converts GPU memory into the binding constraint — once the paged KV pool is exhausted, the engine preempts (swap out or recompute), and recompute shows up as mysterious latency cliffs under load. And it is a *serving-engine* property: if you are calling a hosted API, you never configure it, but every per-token price you pay is computed by somebody else's scheduler running this loop. That is why the same model on the same GPU can differ 5x in cost per million tokens between a naive vendor and a good engine.

Finally, one clarification that saves a lot of confusion: continuous batching is **not** classic **batch processing**. Batch processing is a *data-engineering* mode — take a static dataset, process it in a long offline job, forget latency. Continuous batching is a *live traffic scheduler* — requests arrive unpredictably, each has its own latency budget, and the "batch" is a momentary, ever-changing grouping of in-flight work. Same word, opposite intent.

### Example:

TokoKita runs a customer-support RAG bot ("KirimChat") on **one H100** with vLLM, serving ~40 req/s at peak. The prompt is a RAG context (~3,000 tokens) plus history, and answer lengths are wildly mixed: 60% are 20–60 token "status pesanan saya?" replies, 30% are 200–500 token explanations, and 10% are 1,500+ token policy walkthroughs. Their first version used a hand-rolled loop with static batches of 8. The Flash Sale at 20:00 WIB is where it fell over.

```
STATIC BATCH OF 8 — 20:03 WIB, measured on the same H100
════════════════════════════════════════════════════════════════════════
  R1 (policy walkthrough, 1,600 tok)  ████████████████████████████ ...
  R2 (order status,        24 tok)    ██ done ░░░░ idle ░░░░ idle ░░░
  R3 (order status,        31 tok)    ██ done ░░░░ idle ░░░░ idle ░░░
  ...R4–R8 (12–48 tokens)             ██ done ░░░░ idle ░░░░ idle ░░░
════════════════════════════════════════════════════════════════════════
  p50 latency: 11.4 s  ·  p99: 13.1 s  ·  throughput: 610 tok/s
  GPU util: 38%        ·  KV cache pinned at 61% while mostly idle
  Cost: $2.90 / 1M output tokens  ·  queue depth grew to 300 during peak
```

Two things made it structurally bad: the 7 short users waited ~11 s for a one-line answer they should have had in under a second, *and* the engine had no free slot to admit the 300 queued requests into, because every slot's KV memory was reserved until the long request drained. Buying a second GPU would have cost $2/hour and fixed nothing structural — the slots would still have been parked.

The fix was a config change, not a rewrite: keep vLLM's default continuous batching, then tune it.

```
CONTINUOUS BATCHING + CHUNKED PREFILL — 20:03 WIB, same single H100
════════════════════════════════════════════════════════════════════════
 iteration │ running batch (max_num_seqs=192)
───────────┼────────────────────────────────────────────────────────────
 it 1000   │ 192 slots: 178 decodes + a 4k chunk of P-9912's long prefill
 it 1001   │ R-4021 emits EOS ─► evicted ─► queued R-7710 ADMITTED
 it 1002   │ R-7710 prefill chunk (2k) │ 191 others keep decoding
 it 1003   │ R-3318 hits max_tokens ─► evicted ─► R-8102 admitted
 ...       │ (mix changes every iteration, exactly as designed)
════════════════════════════════════════════════════════════════════════
  p50 latency: 1.9 s   ·  p99: 4.2 s   ·  throughput: 3,480 tok/s
  GPU util: 87%        ·  KV cache 92% occupied, blocks recycled per step
  Cost: $0.51 / 1M output tokens      ·  queue drained within 90 s
```

```
vLLM flags actually used
┌────────────────────────────────────────────────────────────────────┐
│ --max-model-len 32768          keep the context they trained for   │
│ --max-num-seqs 192             how many sequences may run at once  │
│ --max-num-batched-tokens 8192  per-iteration token budget          │
│ --enable-chunked-prefill       stop long prompts stalling decodes  │
│ --gpu-memory-utilization 0.92  grow the paged KV pool → more slots │
└────────────────────────────────────────────────────────────────────┘
```

The order of operations matters and is worth copying: continuous batching is already on by default in vLLM, so the first wins came from *removing* static batching and letting the engine admit continuously; the next win was raising the KV pool so more sequences could be resident; chunked prefill came last, because it trades a little TTFT for tail stability and only matters once prompts are long and traffic is mixed. Their p99 before chunked prefill was 9.8 s even with continuous batching on — a textbook case of "continuous batching fixed the p50 and exposed the p99": a single 30k-token contract upload could stall every decoding request behind it until the prompt was chunked.

Two operational notes from the post-mortem. First, `max_num_seqs=192` was found empirically, not from a blog post: pushing it to 512 raised throughput another 8% and pushed p95 latency past the 3 s SLO, because slot count is a *memory* decision — the pool ran dry, preemption kicked in, and evicted sequences were recomputed, which looks exactly like random slowness in the logs. Second, GPU utilization stopped being a meaningful dashboard signal: it now reads 85–90% whether the service is healthy or drowning, so they scale on **batch occupancy and queue wait time** instead — the scheduler's own signal is the honest one.

The punchline: continuous batching is what turned "serve an LLM" from "rent a GPU and hope" into a scheduling problem with a dial on it. The same weights on the same hardware went from 610 to 3,480 tokens per second — a 5.7x cost cut — because the scheduler stopped treating a batch as a fixed group of requests and started treating it as a living set of in-flight sequences, evicted and admitted at the granularity of a single token.

---

day - 14

## Cell-Based Architecture

### Definition:

**Cell-Based Architecture** (also called *cellular architecture*, *cell-based design*, or *deployment stamps* in Azure's vocabulary) is a fault-isolation pattern in which a workload is split into multiple **complete, independent copies of itself — cells** — where each cell serves a slice of traffic chosen by a **partition key**, and a thin **cell router** in front maps every request to exactly one cell. Cells share *nothing*: their own compute, their own database, their own cache, their own queues, their own secrets. A crash, an overload, a bad deploy, or a poison-pill request inside one cell **cannot escape it**.

The failure modes cells exist for are the ones redundancy does *not* fix. "A machine died" is not the target — that is what load balancers and replicas have handled for twenty years. Cells target the **correlated** failures: the ones where every copy of your stack falls over at the same instant because they are all running the same bad thing:

- a **bad deployment or a bad schema migration** (every instance gets the same bug at the same time)
- a **poison-pill request or tenant** — one query shape, one customer, that saturates a shared resource everyone else depends on
- hitting a **hard quota ceiling** (connection pool, partition throughput, API rate limit) that no amount of horizontal scaling raises
- a **black-swan / metastable failure** — a retry storm or a cache stampede that feeds on itself fleet-wide

In a single shared pool all of these are fleet-wide events. AWS's Well-Architected guidance frames the design question better than any paraphrase: *"Is it better for 100% of customers to experience a 5% failure rate, or 5% of customers to experience a 100% failure rate?"* Cell-based architecture answers **the second, on purpose** — it trades a small, *bounded*, *known* group of users being fully degraded for the guarantee that nobody else notices. That bounded number is called the **blast radius**, and shrinking it is the entire point.

SINGLE SHARED STACK vs CELL-BASED — where the blast radius is measured:
═══════════════════════════════════════════════════════════════════════

  WITHOUT CELLS — one stack, every tenant in the same pool:

                    all tenants, all traffic
                              │
                              ▼
            ┌─────────────────────────────────────────┐
            │           ONE SHARED STACK              │
            │  ┌────────┐  ┌────────┐  ┌───────────┐  │
            │  │  API   │  │ Redis  │  │ workers / │  │
            │  │ hosts  │  │ cache  │  │  queues   │  │
            │  └───┬────┘  └───┬────┘  └─────┬─────┘  │
            │      └───────────┴─────────────┘        │
            │                   │                     │
            │          ┌────────▼─────────┐           │
            │          │  ONE DATABASE    │           │
            │          │  one conn pool   │           │
            │          │  one quota       │           │
            │          └──────────────────┘           │
            └─────────────────────────────────────────┘

        ONE bad deploy · ONE poison-pill tenant ·
        ONE quota ceiling · ONE hot partition
                 │
                 ▼
        EVERY tenant is hit at once
        BLAST RADIUS = 100% of tenants


  WITH CELLS — traffic sliced, the whole stack replicated per slice:

                          all tenants
                               │
                     ┌─────────▼─────────┐
                     │    CELL ROUTER    │   stateless, thin,
                     │ hash(tenant_id)%N │   statically stable
                     │ (deterministic!)  │   the ONLY shared part
                     └──┬─────┬─────┬────┘
          ┌─────────────┘     │     └─────────────┐
          ▼                   ▼                   ▼
    ┌────────────┐      ┌────────────┐      ┌────────────┐
    │  CELL 0    │      │  CELL 1    │ ...  │ CELL N-1   │
    │ ┌────────┐ │      │ ┌────────┐ │      │ ┌────────┐ │
    │ │  API   │ │      │ │  API   │ │      │ │  API   │ │
    │ │ Redis  │ │      │ │ Redis  │ │      │ │ Redis  │ │
    │ │ queue  │ │      │ │ queue  │ │      │ │ queue  │ │
    │ │  DB    │ │      │ │  DB    │ │      │ │  DB    │ │
    │ └────────┘ │      │ └────────┘ │      │ └────────┘ │
    │ tenants    │      │ tenants    │      │ tenants    │
    │ 0, 8, 16…  │      │ 1, 9, 17…  │      │ 7, 15, 23… │
    └────────────┘      └────────────┘      └────────────┘
      no state shared between cells — ever, by design

        ONE bad tenant / ONE bad deploy is CONTAINED
        BLAST RADIUS = 1 cell = 1/N of tenants
        (N = 4 → 25%,  N = 8 → 12.5%)
        and it is bounded BY CONSTRUCTION, not by luck

  ┌────────────────────────────────────────────────────────────────┐
  │  KEY IDEA: redundancy protects you from things that BREAK (a   │
  │  disk, an AZ). Isolation protects you from things that        │
  │  BREAK EVERYONE — a bad deploy, a poison pill, a quota wall.  │
  │  Cells are not "more servers"; they are a smaller number of   │
  │  users who can be sacrificed to keep the rest correct.        │
  └────────────────────────────────────────────────────────────────┘

**The three components.** Every cell-based system decomposes into exactly these, and getting the boundaries between them right is the whole design:

1. **The cells** — a *template instantiated with a cell identifier*. A cell is a complete stack: API/compute, data store, cache, queues, and whatever else the request path touches. The critical property is that it owns its state. The most under-appreciated line in the AWS whitepaper, and the one that gets this pattern past a budget holder: *"Building a cell-based architecture doesn't necessarily mean having to double, triple, or more your application's infrastructure. It might be that your application has 30 hosts, and in a cell-based architecture it has the same 30 hosts, but with a cell router and with tasks that are distributed or grouped between cells."* You are not buying N times the fleet — you are regrouping the fleet you have and paying extra only for the stateful parts (N databases instead of one).
2. **The cell router** — the hardest component, because it is the *one thing that cannot be cellularized*. It is the only element holding shared state about all cells, so it must be brutally thin, stateless, deterministic, and unavailable-free: ideally a hash of the partition key against a mapping that is cached in-process and replicated (S3/CDN/Route 53/API-Gateway routing rules/edge worker), **never a call to a live control plane on the request path**. Two rules matter more than any implementation detail: the mapping must be **deterministic** (same tenant → same cell, always; hash the *tenant*, never the *request*, or you lose data locality and cache hits and start splitting a tenant's writes across cells), and the router must be **statically stable** — it must keep serving correctly with the control plane completely down.
3. **The control plane** — everything that *manages* the fleet rather than serving traffic: provisioning cells, maintaining the tenant→cell mapping, deploying code, migrating tenants between cells, running migrations. It is deliberately **off the request path**. Cells move the deployment problem from "shipping to production" to "shipping to N productions", which is exactly why the control plane has to be fully automated: if creating cell 51 involves a manual step, you do not have a cell-based architecture, you have N snowflakes.

```
CONTROL PLANE vs DATA PLANE — static stability is the requirement
═══════════════════════════════════════════════════════════════════

  CONTROL PLANE (off the request path, may be down)   DATA PLANE (always on)
  ┌────────────────────────────────────────────┐     ┌───────────────────┐
  │ • provision / de-provision cells           │     │  cell router      │
  │ • own the tenant → cell mapping            │     │      +            │
  │ • deploy code, run migrations              │     │  cells 0..N-1     │
  │ • migrate tenants between cells            │     └─────────┬─────────┘
  └───────────────────┬────────────────────────┘               │
                      │ pushes mapping + config (cached,        │
                      │ replicated, versioned)                  │
                      └────────────────────────────────────────►│
                                                      requests never wait
                                                      on the control plane

  If the control plane dies, the data plane must KEEP SERVING with the
  mapping it already has. Otherwise your "isolation" has a single point
  of failure sitting right in front of it.
```

**Shuffle sharding: isolation *inside* a cell.** Cells and shuffle sharding are constantly confused, and the distinction is precise. Shuffle sharding (AWS, 2014 — published by Colm MacCarthaigh, built for Route 53's *Infima* library, framed as a deck of cards) gives **each tenant its own unique combination** of the fungible workers: with M workers and a shard size r, a tenant is assigned a distinct *r*-sized subset, so two tenants overlap only if they happened to draw the *identical* combination. With 8 workers in shards of 2 there are C(8,2) = 28 distinct shards, so the chance two random tenants collide is ~3.6%; scale to 100 workers in shards of 2 and there are 4,950 shards — a collision probability of **0.02%**. In shards of 3: 161,700 shards, **0.0006%**.

```
FAN-OUT (naive)                    SHUFFLE SHARDING (M=8, r=2 → 28 shards)
════════════════════════════       ═══════════════════════════════════════
 every tenant hits every worker     each tenant gets a UNIQUE pair

   tenant A ──►┌───────────┐          tenant A ──► {w0, w1}
               │  w0 .. w7 │          tenant B ──► {w4, w5}
   tenant B ──►│   ALL     │          tenant C ──► {w2, w6}
               └───────────┘          ...all 28 combinations in use...

 one bad tenant poisons the      two tenants share a fate only if they
 WHOLE pool              →       drew the IDENTICAL shard: 1/28 = 3.6%
                                 (M=100, r=2 → 4,950 shards → 0.02%)

  SPREAD THE RISK  vs  GIVE EVERY ONE OF THEM A DIFFERENT RISK
```

The boundary rule AWS states outright: **no overlap *between* cells, shuffle sharding *inside* a cell.** Cells bound the impact of a bad deployment or a poison pill on *stateful* things; shuffle sharding bounds the impact of a noisy tenant on a cell's *fungible* resources — worker pools, queues, rate limiters, sender slots. Letting a shuffle shard span cells would destroy the cell's independence, which is the one thing the pattern cannot give up: *"in a cell-based architecture, a cell should be self-contained, not share its state."* A cell is defined by sharing nothing; a shuffle shard is defined by deliberately overlapping.

**What cells are *not* — the four mix-ups, and why each one matters:**

- **Not a failover domain.** This is the most consequential misunderstanding. A cell is a fault-isolation boundary, not a spare that tenants get moved onto. If your disaster-recovery plan is "fail over to another cell", you do not have a cell-based architecture — you have an **active-passive pair with extra steps**, and your tenants' state does not live where you are sending them. When a cell fails, its tenants are *degraded or down*; the win is that everyone else is untouched. Cells do not make you highly available, they make you **fault-isolated**.
- **Not the same as multi-AZ.** Multi-AZ gives you redundant copies inside a *provider-defined* failure domain. Cells create isolation boundaries *you* define with a partition key — and they compose, because the boundary is yours to place: a cell can be **zonal, regional, or global**. Slack's implementation (aligned to Availability Zones after a 2021 AZ networking incident) puts one complete, siloed backend deployment in each AZ, assigns workspaces to cells by workspace ID, and leaves only the low-traffic admin plane (workspace creation, billing) shared.
- **Not the same as the Bulkhead Pattern** (already in this journal). A bulkhead partitions *resource pools* inside one shared stack — a separate thread pool and connection pool per downstream dependency, so one slow dependency cannot starve the others. It still shares the app, the database, and the schema with everyone. A cell partitions the **entire stack** by traffic slice, all the way down to the database. Bulkhead: *"don't let dependency X starve dependency Y."* Cell: *"don't let tenant A's bad day ever be tenant B's bad day."*
- **Not free.** Running N copies of the stateful layer costs real money. In practice the published rule of thumb is **~40% added infrastructure cost, not N×** — compute scales sub-linearly because each cell holds fewer tenants — plus a permanent complexity tax. Every operational procedure grows a *"which cell?"* step: migrations, schema changes, config rollouts, secrets, and dashboards all become cell-aware; deprovisioning or rebalancing eventually demands **online tenant migration** between cells; and global features (cross-tenant search, analytics, billing, admin) need a separate aggregation layer that reads from all cells — which must be thin, read-only from the cell's perspective, and highly available, or it becomes the shared failure domain you just spent all that money removing.

**Operating a cell fleet** has three disciplines that teams learn the hard way. First, **deploy in waves with a canary cell**: one cell always receives code and migrations first, is watched for a defined window, and is rolled back alone if a signal appears — turning a fleet-wide 45-minute outage into a 5-minute degradation for 1/N of users. Second, **observability must be cell-scoped**: tag every metric with a `cell` label and publish the **worst cell's** health, not the fleet average, because the aggregate dashboard lies — a fleet-wide error rate of 0.4% looks healthy and can be one cell sitting at 100% with the rest at 0%. Third, **audit capacity and drift per cell** on a schedule: quota headroom against service limits, configuration drift between cells, and whether each cell can absorb the load placement might send it. A cell has a *published, tested maximum* — that is the number that makes all of this tractable.

The honest decision rule, which is also where to start: **use cells when a single failure domain affecting all users is an existential risk to the business**, and your workload is multi-tenant with a stable natural partition key. Below that bar, circuit breakers, bulkheads, load shedding and multi-AZ are cheaper and enough. And when you do build it: **start with 4 cells, not 40.** Each cell is a full copy of the stack; more cells means finer isolation and exponentially more operational overhead. Teams that start at 8 and shrink to 4 once the monitoring burden becomes real are following the standard path.

### Example:

Kyomel's **fasting-bot** outgrew the friend group. What started as one WhatsApp group became **"Komunitas"** — 12,000 community workspaces (gyms, mosques, offices, alumni groups) with ~1.4M members, each community with its own leaderboard, streaks, and reminder scheduler. Architecturally it was the obvious v1: one Go service, one Postgres, one Redis, one worker pool, and one notification scheduler, all shared. Traffic is brutally rhythmic — a **sahur burst at 04:00–05:00 WIB** and a **buka-puasa burst at 17:30–18:15 WIB** — when every community's membership wants the same "jangan lupa sahur" ping within the same twenty minutes.

Then **KomunitasGymBesar** joined. One workspace, 40,000 members, and a leaderboard feature that runs `SELECT member, COUNT(*) FROM fasting_logs ... GROUP BY member ORDER BY count DESC LIMIT 100` plus an "export my streak" CSV for every member. At **04:12 WIB** it saturated the shared connection pool, and 12,000 *unrelated* communities learned about it instantly.

```
BEFORE — 04:12 WIB, ONE POISON PILL, EVERY COMMUNITY HIT
═══════════════════════════════════════════════════════════════════

  KomunitasGymBesar ──►┌──────────────────────────────────────────┐
  (40k members:        │  ONE SHARED STACK                        │
   leaderboard ×40k,   │                                          │
   CSV export, GROUP   │   ┌──────────────┐                       │
   BY over 900 days of │   │ Postgres     │ max_connections = 200 │
   fasting logs)       │   │ 200/200 BUSY │ 8s waits, then 500s   │
        │              │   └──────┬───────┘                       │
        │              │          │                               │
        │              │   ┌──────▼───────┐                       │
        │              │   │ Redis +      │ cache misses pile up, │
        │              │   │ worker pool  │ 1 shared queue        │
        │              │   └──────┬───────┘                       │
        └─────────────►└──────────┼───────────────────────────────┘
                                  │
        ┌─────────────────────────┴──────────────────────────┐
        ▼                                                    ▼
  04:00 sahur reminder job                     streak / leaderboard API
  → queued behind 40k exports                  → 30s timeouts, HTTP 500
  → delivered 41 MINUTES LATE                  → members think streaks broke

              ALL 12,000 KOMUNITAS  ◄──────────────────────┘
              missed or got a useless sahur ping

  BLAST RADIUS: 100% of tenants  ·  DURATION: 41 min
  A week later the same shape repeated with a BAD DEPLOY: an index
  migration on fasting_logs took a table lock and held it for 45 min —
  every instance ran the same migration, so every instance was down.

  ROOT CAUSE: no isolation boundary anywhere between a tenant
              and the shared state every tenant depends on.
```

They rebuilt it as a **cell-based architecture** — but with the same 12 app hosts they already ran. The stack was regrouped into **8 cells**, each cell being 1–2 app hosts plus its own small Postgres, its own Redis, and its own worker pool, with a stateless **cell router** at the WhatsApp gateway assigning every workspace deterministically via `hash(workspace_id) % 8`.

```
AFTER — same poison pill, same bad deploy: 1/8 of the fleet
═══════════════════════════════════════════════════════════════════

  WhatsApp inbound ──►┌─────────────────────────────────────────────┐
  (wa-gateway)        │  CELL ROUTER — stateless, statically stable │
                      │  mapping: workspace_id → cell               │
                      │  cached in-process + replicated (versioned) │
                      │  hash the WORKSPACE, never the message      │
                      └──┬────┬────┬────┬────┬────┬────┬────┬──────┘
        ┌────────────────┘    │    │    │    │    │    │    └────────┐
        ▼                     ▼    ▼    ▼    ▼    ▼    ▼             ▼
  ┌───────────┐        ┌───────────┐   ...   ...   ...   ...   ┌───────────┐
  │  CELL 0   │        │  CELL 1   │                           │  CELL 7   │
  │  ◄ canary │        │           │                           │           │
  │  API      │        │  API      │                           │  API      │
  │  Postgres │        │  Postgres │                           │  Postgres │
  │  Redis    │        │  Redis    │                           │  Redis    │
  │  workers  │        │  workers  │                           │  workers  │
  │ ~1,500    │        │ ~1,500    │                           │ ~1,500    │
  │ tenants   │        │ tenants   │                           │ tenants   │
  └─────┬─────┘        └───────────┘                           └───────────┘
        │
        │  GymBesar lives HERE — it is just one of ~1,500 tenants
        ▼
  04:12 its own conn pool saturates (200/200 busy, 8s waits)
        • its own members' reminders late, its own API 500s
  ──────────────────────────────────────────────────────────────
  CELLS 1–7:  sahur reminders delivered ON TIME, no 500s,
              no idea anything happened. p99 unchanged.
              The support channel gets messages from ONE
              community, not twelve thousand.

  BLAST RADIUS: 12.5% of tenants  ·  7/8 of the fleet unaffected

  DEPLOY WAVES (fixes the bad-migration failure mode too):
    cell 0 (canary) ──► watch 5 min ──► cell 1 ──► … ──► cell 7
    bad migration on cell 0?  ROLLBACK CELL 0 ALONE.
    5 minutes, 12.5% of tenants, other 87.5% never saw it.
```

Shuffle sharding came in one layer *inside* each cell, for the piece that is genuinely fungible: the WhatsApp **sender slots**. Each cell has 64 sender slots against the WA gateway, and a community that bursts (or gets rate-limited) must not clog the slots everyone shares — so each community was assigned a **unique pair** of senders out of the cell's 64: C(64,2) = **2,016 distinct shards**, meaning two random communities collide with probability ~0.05%. GymBesar flooding its own two senders costs GymBesar latency, not the cell.

```
THE SHAPE THEY ENDED UP WITH
────────────────────────────────────────────────────────────────
  cells          8 cells (1–2 app hosts each) on the SAME 12 hosts
                 that already existed — only the stateful layer
                 multiplied (8 small Postgres + 8 Redis instead of 1)
  partitioning   hash(workspace_id) % 8, deterministic, cached at the
                 router; a workspace NEVER spans two cells
  canary         cell 0 permanently receives deploys + migrations first
  inside a cell  64 WA sender slots, shuffle-sharded r=2 per community
  global view    cross-community leaderboard = nightly batch job over
                 all cells → thin read-only API (NOT on the request path)
  observability  every metric tagged cell="0..7"; the dashboard shows
                 the WORST cell's availability, not the fleet average
  cost           +38% infrastructure (extra databases, caches, per-cell
                 dashboards + tooling) for blast radius ÷ 8
────────────────────────────────────────────────────────────────
```

Three notes from their migration that generalise. **They cellularized the template before they cellularized anything real** — cell 0 was stood up from IaC as a throwaway and rebuilt from scratch four times until provisioning a cell was one boring command. Without that step, 8 cells would have become 8 hand-tuned snowflakes and the whole exercise would have made the fleet *fragile*. **They moved the global leaderboard off the request path** before opening the second cell, because a cross-cell feature is the one place where the isolation quietly leaks back in: an on-demand query that reads all 8 cells would have made the aggregation layer the new single point of failure. And **they resisted going wider**: the plan started at 16 cells, they shipped 8, and after a month of running per-cell dashboards they were glad they had not — the monitoring and migration work was already the dominant cost, and 12.5% was a blast radius they could live with.

The punchline: cell-based architecture is the admission that **not all failures can be made rare, so we make them small instead.** For two decades the answer to "a bad thing happened to everyone" was stronger engineering — safer migrations, better reviews, more capacity, more retries. Cells accept that some of these will get through anyway and change the *shape* of the damage: from a vertical cliff where the whole product is down to a horizontal slice where 1/N of users have a bad ten minutes and everyone else keeps doing what they were doing. It is not free, it is not for everyone, and it does not make you highly available — it makes you **isolated**. But for any multi-tenant system where a table lock or one customer's runaway query can reach every user at once, the question stops being "should we do cells?" and becomes the one AWS asks: *whose bad day are you willing to let it be?*

---

day - 15

## Circuit Breaker Pattern

### Definition:

The **Circuit Breaker Pattern** is a resilience pattern in which every outbound call to *one specific dependency* is wrapped in a small **state machine** that watches recent outcomes and, once that dependency's failure rate (or its latency) crosses a threshold, **stops calling it at all** — failing instantly and locally instead of letting requests pile up on a service that is already falling over.

The name is borrowed straight from electrical engineering, and the analogy is unusually honest: when a wire shorts, the breaker pops so the *house* does not burn down. The wire is already lost. What the breaker protects is everything downstream of it. Software circuit breakers protect exactly the same thing — not the failing dependency, but the caller, its thread pool, its connection pool, its latency, and every user who never asked for that dependency in the first place.

To understand why it exists, you have to notice an asymmetry that fools almost everyone the first time:

```
A DEPENDENCY THAT FAILS FAST  ── almost harmless
   connection refused in 1 ms → you hold a connection for
   1 ms, return an error, move on. Nobody notices except the
   one user who got the error.

A DEPENDENCY THAT FAILS SLOW  ── a loaded gun pointed at YOU
   the same dependency at 10 s instead of 100 ms:
   • every waiting call holds a goroutine/thread,
   • a socket, and
   • a slot in a pool that has a HARD CEILING.
   Little's Law: in-flight requests ≈ arrival_rate × latency.
   Latency went up 100×, traffic did not change,
   so in-flight requests went up 100× → THE POOL IS GONE.
   → your service now returns 5xx to requests that never
     touched the sick dependency at all.
```

That last line is the whole disaster. The checkout page dies because a recommendation widget's API got slow. Cascading failure is not "the bad service took everyone down" — it is *"the bad service made its callers look bad."*

And naive retries make it strictly worse. This is the counter-intuitive part: retry logic, the thing you added to survive blips, is what turns a blip into an outage. Three retries across a dozen replicas multiplies the load hitting a dependency that was already too weak to answer — a self-inflicted DDoS. AWS's own guidance (Builders' Library, *Timeouts, retries and backoff with jitter* / *Good Retry, Bad Retry*) is blunt about it: a retry loop without a budget is a load amplifier, not a resilience feature.

So the design problem is: **how do I stop paying the cost of waiting, when I already know waiting is hopeless?** A timeout caps a *single* call. It does nothing about the ten thousand calls that will each pay that timeout. The circuit breaker is the missing piece: it lets the caller *learn* from recent outcomes and refuse to wait at all once the evidence says the dependency is down.

The comparative picture:

```
WITHOUT A CIRCUIT BREAKER  vs  WITH A CIRCUIT BREAKER
(retry + timeout only)         (retry + timeout + breaker)
═══════════════════════════    ═══════════════════════════════════

  request ──► [slow dep 10 s]    request ──► [ BREAKER: OPEN ]
                    │                             │
        no state, nobody remembers          "I already know this
        that the last 5,000 calls           dep is dead — from the
        also timed out                      last 20 samples"
                    │                             │ 0 ms
                    ▼                             ▼
     ┌────────────────────────────┐   ┌────────────────────────────┐
     │ POOL (cap 200)             │   │ POOL (cap 200)             │
     │ ████████████████████ 200/200│   │ ██               18/200    │
     │ 199 waiting on a corpse,   │   │ 182 slots still serving    │
     │ the 200th is checkout      │   │ everything else normally   │
     └────────────────────────────┘   └────────────────────────────┘
              │                                  │
              ▼                                  ▼
     p99 = 10 s (the timeout)          p99 = 10 s (only for the
     + retries ×3 → the sick dep       few calls in flight);
       receives 3× MORE traffic        breaker-rejected calls
     + cascade: checkout 5xx →           return in ~0 ms with a
       cart 5xx → gateway 5xx            clear, countable error
     = one slow dep takes the          = one slow dep degrades ONE
       whole product down                FEATURE, product stays up
```

The state machine is the core of the pattern, and it has three normal states — plus, importantly, **edges that are deliberately missing**:

```
CLOSED  ──failure rate ≥ threshold──►  OPEN
OPEN    ──waitDurationInOpenState──►   HALF_OPEN
HALF_OPEN ──probes succeed──► CLOSED
HALF_OPEN ──any probe fails──► OPEN

MISSING EDGES (this is where the design lives):
  CLOSED ──✗──► HALF_OPEN   half-open only means "recovering FROM
                            open" — it is meaningless as an entry
                            point, so the transition must not exist
  OPEN   ──✗──► CLOSED      recovery is never assumed from the
                            passage of time; it must be PROVEN
                            by a probe
Keeping the state graph this small is what makes a breaker testable.
```

What each state actually means:

- **CLOSED** — normal operation, the breaker passes every call through and records the outcome. A healthy system spends effectively all its time here. "Closed" = circuit closed = current flows.
- **OPEN** — the breaker short-circuits: it does not call the dependency, it throws immediately (`CallNotPermittedException` in resilience4j). Rejecting *everything* is what gives the dependency room to recover — it also means the breaker now has **zero information** about whether recovery happened, which forces the third state.
- **HALF_OPEN** — after the cool-down, a **bounded number of probe calls** are let through while everything else is still rejected. The naive alternative (flip back to CLOSED after a fixed cooldown) fires the full traffic load at a service that may still be broken — the exact thundering herd the breaker was built to prevent. Probes decide: enough successes → CLOSED; any failure → straight back to OPEN and the timer restarts.

How you make it *trip* (the part people get wrong):

- **The window is what you measure over.** A count-based sliding window is a circular array of the last N outcomes with incremental aggregation (an O(1) snapshot, which is why it beats storing tuples); a time-based window is a ring of per-second buckets — e.g. "the last 10 seconds", which is what Hystrix used by default. Both are *sliding*, not fixed buckets, so a breaker cannot be fooled by a window boundary.
- **Two thresholds, not one.** Failure rate (errors) *and* slow-call rate (latency). A dependency that answers 200 OK in 8 seconds is functionally down; a failure-rate-only breaker will happily keep hammering it. This is the single most common production miss.
- **Minimum throughput is mandatory.** Without it, "1 call, 1 failure" is a 100% failure rate and the breaker opens on one unlucky request. Hystrix required ≥20 requests in the 10 s window before it would even evaluate; resilience4j's defaults are `slidingWindowSize: 100`, `minimumNumberOfCalls: 100`, `failureRateThreshold: 50`, `slowCallRateThreshold: 100` with `slowCallDurationThreshold: 60s`, `permittedNumberOfCallsInHalfOpenState: 10`, `waitDurationInOpenState: 60s`, `automaticTransitionFromOpenToHalfOpenEnabled: false`. Defaults are a *starting point* for a slow internal RPC — not for a checkout path.
- **Timeouts belong INSIDE the breaker.** The timeout's job inside the chain is not to improve your latency; it is to guarantee that **every admitted attempt eventually settles**, so the window never fills with calls that are still hanging. Retry placement is a real design choice, but the invariant (timeout innermost) is not.
- **Stale results are a real bug.** Calls admitted while the breaker was HALF_OPEN may settle several transitions later; a late success counted as-is can close a breaker that a newer probe just re-opened. Mature implementations carry a **generation counter** (bumped on every transition) and discard results whose generation no longer matches.
- **Special states exist so you can control it in operations:** `METRICS_ONLY` (record but never open — the shadow mode you use to *tune* thresholds before trusting them), `DISABLED`, `FORCED_OPEN` (kill switch for a dependency you know is poisoned).

Now the part that makes this a genuinely 2026 term rather than a 2007 one: **"circuit breaker" now names three different mechanisms**, and teams get burned by assuming they are the same thing.

```
THREE DIFFERENT THINGS CALLED "CIRCUIT BREAKER"
══════════════════════════════════════════════════════════════════

1. LIBRARY-LEVEL  (resilience4j, Polly, gobreaker, Hystrix)
   ── a state machine over a sliding window, KEYED BY DEPENDENCY
   ── OPEN/HALF_OPEN/CLOSED, error-rate + slow-call-rate tripping
   ── lives IN your process (or in a shared store — see below)
   → this is what the rest of this entry is about

2. ENVOY / SERVICE-MESH "circuit_breakers"  ← NOT a state machine
   ── hard CONCURRENCY CEILINGS on a cluster:
        max_connections · max_pending_requests
        max_requests    · max_retries             (defaults: 1024!)
   ── exceed it → instant 503 local reply, NO queueing, NO probe
   ── counters: upstream_cx_overflow, upstream_rq_pending_overflow,
                upstream_rq_overflow, upstream_rq_retry_overflow
   → it is a load-shedding gate, not a health state machine.
     A 1024-request ceiling on a service that peaks at 30 concurrent
     requests is not a circuit breaker — it is decoration.

3. ISTIO/ENVOY "OUTLIER DETECTION"  ← per-HOST ejection
   ── consecutive_5xx (default 5), interval, base_ejection_time
   ── ejects the sick POD from the load-balancing pool, then
      re-introduces it after the ejection period (that cycle is the
      closest thing the mesh has to HALF_OPEN)
   ── Envoy itself calls this PASSIVE HEALTH CHECKING, deliberately
      not "circuit breaking" — the vocabulary matters when you
      debug it at 3 AM
```

The mesh variant is the dangerous one, because it looks like free safety and behaves like a feedback loop. A real incident (Mercari, presented at CloudCon 2026: *"When your circuit breaker backfires — outlier detection in Istio and Envoy"*) went exactly like this: outlier detection started ejecting misbehaving pods, the mesh shifted that traffic onto the *remaining* pods, those pods crossed their own saturation point, got ejected too — and traffic concentrated onto fewer and fewer endpoints until they crashed. Errors spread to services that had nothing to do with the change, and the setting had passed every automated policy check. Each component did its job; together they produced a cascade.

There is a deeper version of that trap, and it is the one that should make you cautious about breakers in *sharded* and *cell-based* systems: a breaker aggregates outcomes into a single verdict. If 10% of your dependency's shards are hot, your window sees a 10% error rate, trips, and your caller short-circuits **100% of its traffic — including the 90% of requests that would have succeeded.** Marc Brooker's framing is the sharpest one available: *circuit breakers can misinterpret a partial failure as a total failure and inadvertently bring the system down.* Cells and shards (see day - 14, Cell-Based Architecture) are exactly the topologies where this bites, because partial failure is the normal state of a sharded system. The workaround — the server tells the client *which* shard is overloaded and the client keeps per-shard mini breakers — works and is genuinely painful: more state, more keys, more things to monitor.

So the honest pro/con ledger:

```
┌────────────────────────────────────────────────────────────────┐
│  ✅ WHAT YOU GET                                               │
│  • Converts "wait 10 s, then fail" into "fail in 0 ms"         │
│  • Frees the bounded resource (threads, conns, goroutines)     │
│    that every waiting call was holding                         │
│  • Gives a still-recovering dependency QUIET — no hammering    │
│  • Makes the failure a COUNTABLE, ALERTABLE event              │
│    (rejected_count / calls_total rises before your SLO dies)   │
│  • Turns an unbounded cascade into a bounded, local outage     │
│                                                                │
│  ❌ WHAT YOU PAY / WHAT IT DOES NOT DO                         │
│  • New tuning surface: window type, size, min throughput,      │
│    two thresholds, open duration, probe count — all coupled    │
│  • Does not make requests succeed — it makes them FAIL; you    │
│    still owe the user a fallback, a cached answer or a 503     │
│  • Local breaker state = N replicas each must learn on their   │
│    own → slow collective reaction, and every replica can be    │
│    wrong independently (fixed by a shared window)              │
│  • Can misfire on PARTIAL failure (hot shard, bad tenant,      │
│    single bad region) and short-circuit healthy traffic        │
│  • Without an accompanying RETRY BUDGET, retries outside the   │
│    breaker re-amplify exactly what the breaker stopped         │
│  • A breaker that trips constantly is a SYMPTOM, not a fix —   │
│    the real bug is the dependency, and the breaker is why you  │
│    still have a product while you fix it                       │
└────────────────────────────────────────────────────────────────┘
```

Scaling it beyond one process — this is where the modern design work happens. A breaker per replica sees only the traffic the load balancer happened to send it, so with 20 replicas each needs its own `minimumThroughput` observations before it reacts, and they react at different times. The fix is to move the **window** into a shared store every replica can reach (Redis is the common choice, with the check-and-record done in Lua so it stays atomic):

- **Choose the share key deliberately.** One window per *dependency* (`global`) reacts fastest, because every replica's failures land in the same window — and it is also the most dangerous, since one bad tenant or region opens the breaker for everyone. `region:<x>` or `tenant:<y>` is the middle ground, and scope keys should never contain sensitive data because they live in the coordinator's keyspace.
- **The threshold math must survive the trip.** Compare in integer thousandths (`windowFail * 1000 >= threshold * windowTotal`) — a float threshold rounded to `1.0` becomes "every single observation must fail", i.e. a breaker that never opens. Looks fine in a config file, undebuggable at runtime.
- **The probe budget has to be global**, claimed with a lease. Otherwise every replica independently transitions to HALF_OPEN and races for the slots — recreating the herd in miniature (and the lease must be longer than the slowest possible probe, which is why the timeout inside the breaker matters again).
- **Generation becomes epoch, and the epoch must not move on the OPEN→HALF_OPEN edge** — keeping the epoch *is* how a window is preserved; moving it is how a window is thrown away.
- **A breaker that is not CLOSED must never be allowed to expire.** A missing state record reads as CLOSED, so a silently-TTL'd OPEN state admits all traffic at once — the failure the breaker existed to prevent, delivered on schedule.

And it composes with the patterns this journal already covered: a **Bulkhead Pattern** (isolate the pools so a slow dependency cannot eat the checkout pool at all), the **Retry Budget Pattern** (cap retries as a *fraction* of total traffic — AWS Builders' Library suggests a client can also disable retries entirely once its error rate passes a threshold like 10%), **Load Shedding Architecture** (drop the least valuable work when the pool *is* full), **Tail Latency** (a slow-call-rate threshold is the breaker's latency SLO made actionable), and **Graceful Degradation** / **Cell-Based Architecture** for what the user actually sees while the breaker is open.

The one-line summary: **a circuit breaker is a memory.** Timeouts make a single call cheap to abandon; the breaker makes the *next ten thousand calls* free to abandon, because somebody finally wrote down what everyone already knew.

### Example:

"GymFolks" — an Indonesian fitness app whose `workout-service` (Go, 12 replicas on Kubernetes) calls a third-party **exercise-catalog API** to hydrate each workout card with exercise names, muscle groups, and media URLs. Normal: 300 ms p50, no errors. Then, at 19:00 during the gym rush, the vendor pushes a bad deploy and their API degrades — 40% of calls return 503, the rest crawl at 8–12 s.

The first version of `workout-service` had a 10 s timeout and `retryPolicy: 3 attempts`. Here is what that version does:

```
WITHOUT THE BREAKER — 19:00–19:06, twelve replicas, one slow vendor
════════════════════════════════════════════════════════════════════

  19:00:00  vendor degrades (40% 5xx, else 8–12 s)
  19:00:30  workout-service goroutine usage: 41/200  → 188/200
            (Little's Law: same traffic × 12 s latency)
  19:00:45  ┌──── POOL SATURATED: 200/200 ────────────────────────┐
            │ 196 goroutines: waiting on the vendor              │
            │   4 goroutines:  actual user traffic               │
            │ → endpoints that NEVER touch the vendor now queue:  │
            │   GET /sessions (history)   GET /profile            │
            │   POST /log-set (the core action!)                  │
            └────────────────────────────────────────────────────┘
  19:01:00  retries ×3 → the dying vendor receives ~3× more
            traffic than it was already too weak to serve
            (self-inflicted DDoS on a service that is DOWN)
  19:01:30  cascade: gateway sees 5xx from workout-service,
            its OWN retries fire, the notification worker
            retries, the home-feed aggregator retries…
            ⇒ users cannot log a set. The vendor's outage has
              become GymFolks' outage.
  19:06:00  vendor recovers — but GymFolks stays down another
            4 minutes, because the saturated pool has to drain.
```

The team keeps the timeout and the retries (they are still correct) and adds a **breaker per dependency**, plus a fallback. Go, `gobreaker`-style settings that they actually tuned from metrics rather than guessed:

```
breaker "exercise-catalog":
  window          COUNT_BASED, 20 outcomes   (fast reaction)
  minimumCalls    10        ← without this, 1/1 = 100% → instant open
  errorThreshold  50%       ← trips on the vendor's 40%+ 5xx
  slowThreshold   30% of calls slower than 2000 ms
                            ← catches the SOURCE of the problem:
                              even "successful" vendor calls are
                              eating the pool. Latency threshold
                              fires FIRST here — that is the point
  openDuration    15 s (+0–5 s jitter per replica
                        so 12 replicas do not probe in lockstep)
  probes          3 in HALF_OPEN
  timeout(inside) 2500 ms   ← guarantees every admitted attempt
                              settles and leaves the window
  scope           "dep: exercise-catalog" (NOT a global breaker for
                   all dependencies — the vendor must not trip the
                   payment breaker)
```

```
WITH THE BREAKER — the same vendor outage, six minutes later
════════════════════════════════════════════════════════════════════

  t+0s   vendor degrades
         │
         ▼
  ┌───────────────────── BREAKER: CLOSED ───────────────────────┐
  │ window (last 20 outcomes, sliding)                          │
  │  ✓ ✓ ✓ ✗ ✓ ✗ ✗ ✓ ✗ ✗ ✓ ✗ ✗   ← 8/13 fail (62%) + 9 slow   │
  │  errorRate 62% ≥ 50%   AND   slowRate 69% ≥ 30%             │
  └───────────────────────────┬─────────────────────────────────┘
                              │  TRIP
                              ▼
  ┌───────────────────── BREAKER: OPEN ─────────────────────────┐
  │ EVERY call short-circuits in ~0 ms (no vendor, no socket)   │
  │ goroutines: 41/200 · pool healthy · logs fill with         │
  │   "circuit open for exercise-catalog, serving fallback"     │
  └──────────┬───────────────────────────────┬──────────────────┘
             │                               │
     FALLBACK PATH                    ALL OTHER PATHS
             ▼                               ▼
  ┌──────────────────────────┐   ┌──────────────────────────────┐
  │ local Redis cache of the │   │ /log-set, /sessions,         │
  │ exercise catalog         │   │ /profile, leaderboards       │
  │ (names + muscle groups)  │   │ unaffected, full speed       │
  │ media URLs → placeholder │   │ 182 free goroutine slots     │
  │ card still renders with  │   └──────────────────────────────┘
  │ a "details unavailable"  │
  │ chip — DEGRADED, not     │            ⇒ ONE FEATURE IS
  │ broken (Graceful         │              DEGRADED
  │ Degradation)             │            ⇒ PRODUCT STAYS UP
  └──────────────────────────┘

  t+15s (vendor still sick)
                              ▼
  ┌────────────────── BREAKER: HALF_OPEN ───────────────────────┐
  │ 3 probe calls admitted (only replica #7 wins the lease;    │
  │ the other 11 replicas ask the shared Redis window and are   │
  │ told "probe budget exhausted — keep rejecting")             │
  │   probe 1: 9.8 s ✗  → probe 2 skipped                       │
  │   → back to OPEN, timer restarts (+jitter)                  │
  └──────────────────────────┬──────────────────────────────────┘
                             │  … 4 cycles later, vendor fixed …
                             ▼
  ┌────────────────── BREAKER: HALF_OPEN ───────────────────────┐
  │   probe 1: 287 ms ✓   probe 2: 301 ms ✓   probe 3: 276 ms ✓ │
  │   successRate 100% ≥ threshold → CLOSED                     │
  └──────────────────────────┬──────────────────────────────────┘
                             ▼
  full traffic back to the vendor — returning the FULL payload
  (media URLs, not placeholders) automatically, no deploy, no
  feature flag, no human
```

Two things they got right on the second attempt, both of which are where teams usually lose:

**The breaker state was shared, not local.** With a pure in-process breaker, each of the 12 replicas needs 10 of its *own* outcomes before it reacts — so if the load balancer sends a given replica only 5 vendor calls while the vendor is down, that replica keeps paying 2.5 s timeouts while its neighbours have already given up. They moved the window into Redis, keyed `dep:exercise-catalog`, with admission + recording in one Lua script and a global probe lease — and they kept the scope **per dependency** rather than global precisely because a `global` key means the vendor's bad day also opens the breaker for payments.

**They paired it with a retry budget instead of leaving retries unbounded.** The retry policy now has a budget of 10% of total requests, plus a client-side rule that disables retries entirely when the dependency's error rate passes 10%. That is the difference between "the breaker noticed" and "the breaker noticed *while 11 replicas were still hammering*" — and it is the piece the Mercari incident was missing: outlier detection ejected pods, traffic concentrated on the survivors, and the retries that clients were still sending turned a partial failure into a full one.

And the ending that makes this pattern worth internalising: they deliberately did **not** set the breaker on the *paid* path (`payments`). A breaker there would be wrong the other way around — short-circuiting a payment call fails a user's intent silently and non-retryably, which is worse than paying a few seconds of latency. The rule they wrote into the ADR: **a breaker is for calls you can afford to not make; for calls that must land, you buy latency, not fast failure.**

The punchline: almost all resilience advice is about making individual calls more patient — longer timeouts, more retries, more replicas. The circuit breaker is the one pattern that says the opposite: **patience, applied to a dead dependency, is how you kill yourself.** It does not heal anything and it does not save a single request; it preserves the one resource that lets you survive — the capacity to keep serving the requests that still have a chance. Get the window, the two thresholds and the minimum throughput right, put the timeout inside, share the state once you have more than a few replicas, and pair it with a retry budget; skip any one of those and you have built a device that reliably converts a vendor's bad afternoon into exactly the outage you were trying to avoid.

---

day - 16

## Semantic Caching

### Definition:

**Semantic Caching** is a cache that matches requests by *meaning*, not by text. The cache turns every incoming prompt into an embedding vector. It then searches for the nearest prompt it has answered before. If the similarity score passes a threshold, the cache returns the stored answer and the model never runs.

The pattern exists because real traffic repeats itself. Users do not ask a thousand unique questions. They ask the same five questions in a thousand different words. An exact-match cache sees five distinct prompts and pays five model calls. A semantic cache sees one meaning and pays one.

The request path has five steps:

1. Embed the prompt with an embedding model (for example `text-embedding-3-small`, 1536 dimensions).
2. Search a vector index for the nearest stored prompt (approximate nearest neighbour, usually HNSW).
3. Compare the similarity score to a threshold.
4. Score ≥ threshold → return the stored answer. This is a **hit**. No model call.
5. Score < threshold → call the model, then store the new prompt vector and the new answer. This is a **miss**.

The cache lives at the application layer or at the AI gateway layer. The gateway layer wins in practice, because the gateway already sees every provider, every key, and every request metric. Published gateway guides report 30–50% cost cuts for repetitive support and FAQ traffic. The real number depends on how diverse the questions are.

Semantic caching is not the only cache in a modern LLM stack, and the three layers answer different questions. Provider **prompt caching** keys on a byte-identical prompt prefix (usually the system prompt). It reduces prefill compute and the model still generates a fresh answer, so it carries no correctness risk. **Exact-match caching** keys on the full prompt text and reuses the whole answer. **Semantic caching** keys on meaning and reuses the whole answer.

```
EXACT-MATCH + PROMPT CACHING  vs  SEMANTIC CACHING — what actually gets reused
═════════════════════════════════════════════════════════════════════════════

FOUR MEMBERS ASK ONE QUESTION IN ONE HOUR:

  Q1  "cara refund transaksi gagal?"            ← original wording
  Q2  "transaksi gagal, gimana cara refundnya?" ← same meaning
  Q3  "uang saya balik kapan kalau gagal?"      ← same meaning
  Q4  "refund dong, transaksi gagal"            ← same meaning


WITHOUT SEMANTIC CACHING — the text is the key
┌───────────────────────────────────────────────────────────────────┐
│  question ──► [ cache key = the exact prompt bytes ]              │
│                        │                                          │
│      Q1 HIT            │   Q2 MISS   Q3 MISS   Q4 MISS            │
│      (stored earlier   │                                          │
│       from the same    ▼                                          │
│       wording)   ┌──────────────┐                                 │
│                  │   the model  │   4 prompts ──► 4 calls         │
│                  │   (900 ms)   │   3 of them redundant           │
│                  └──────────────┘                                 │
├───────────────────────────────────────────────────────────────────┤
│  provider prompt caching helps a little here: the shared system   │
│  prompt prefix is cheap for all four, but all four still generate │
│  a full answer. Cost per answer drops. Calls do not.              │
└───────────────────────────────────────────────────────────────────┘


WITH SEMANTIC CACHING — the meaning is the key
┌────────────────────────────────────────────────────────────────────┐
│  question ──► [ embed + ANN search + threshold ]                   │
│                        │                                           │
│      Q1 HIT            │   Q2 HIT   Q3 HIT   Q4 HIT                │
│      (cold cache:      │                                           │
│       the model ran    ▼                                           │
│       once)      ┌────────────────────┐                            │
│                  │  1 model call for  │   stored vector + answer   │
│                  │  all four meanings │   goes back into the index │
│                  └────────────────────┘                            │
├───────────────────────────────────────────────────────────────────┤
│  the price of the win: similarity is a GUESS. A wrong guess       │
│  returns a fluent, confident, well-formatted, WRONG answer —      │
│  and the user has no way to tell it came from a cache.            │
└───────────────────────────────────────────────────────────────────┘
```

The threshold is the whole design. One number controls the hit rate and the false-positive rate at the same time. No single value gives both a high hit rate and few wrong answers.

```
ONE KNOB, TWO METRICS — the shape every threshold sweep produces
════════════════════════════════════════════════════════════════════

  threshold   hit rate   precision (500 sampled hits, human/judge graded)
  ─────────────────────────────────────────────────────────────────────
    0.75        61%        79%   ✗ cheap but reckless: 1 in 5 answers wrong
    0.86        38%        97%   ← the shipped operating point
    0.92        12%       99.6%  ✗ safe but weak: almost no reuse left
  ─────────────────────────────────────────────────────────────────────

  quality falls as you loosen the threshold, savings fall as you tighten it
  → so the cache is a QUALITY decision first and a cost decision second

  the measurement set you need (a hit rate alone hides quality problems):
    · hit rate        — how often the cache fired
    · precision       — of the hits, how many were actually correct
    · recall          — of the reusable answers, how many you captured
    · F1              — the balance point you tune toward
    · expected latency = (hit rate × cache latency)
                       + (miss rate × full pipeline latency)
    · cost per successful answer, and cost per tenant
```

Four failure modes matter, and the last one survives every threshold setting:

- **Negation blindness.** Embedding models place a sentence and its negation close together. A 2025 study in *Scientific Reports* measured this directly. "You must keep the subscription" and "you must not keep the subscription" score as highly similar. No threshold fixes this, because the vectors really are close.
- **Context dependence.** Two prompts look similar and deserve different answers, because one carries a conversation history, an account state, or a retrieved document. Agentic and RAG traffic is context-sensitive by definition, so a prompt-only key is too weak. The key must include the model, the system prompt version, the tenant, and any retrieved context hash.
- **Staleness.** The world moves and the cached answer does not. A price change, a policy change, or a fixed bug leaves the old answer in the index until the TTL expires.
- **Cross-tenant leakage.** A shared index makes one member's answer available to another member's matching question. Any answer that contains personal, health, or financial data must not enter a shared namespace.

The safe configuration uses strict rules:

- One index per tenant for anything member-specific. One shared index only for public, static, factual answers.
- Exact-match lookup first (normalize the text, then hash it). Only make the embedding call when the exact key misses.
- A strict threshold per category. Health, money, and legal answers get the strict value, or no cache at all.
- A validator on the ambiguous band (for example 0.86–0.94). A small, cheap model checks the candidate before the answer is served.
- A TTL per category, and a flush hook that every content change calls.
- An adversarial "must not hit" test set in CI: hundreds of pairs of near-identical questions with opposite answers.
- Sampled hits graded offline, every week, with the grade feeding back into the threshold.

### Example:

Kyomel's **fasting-bot** grew an LLM assistant called **"Tanya Ustadz"**. Members ask it about sahur, buka, travel, illness, and the fast of pregnant members. The bot answers in Indonesian. Volume is 1.8M questions per month, and the traffic pattern is harsh: at **04:00–05:00 WIB**, one hour carries **9,000 questions**, with a peak of about 900 questions per minute. The conversation provider rate-limits the workspace at 400 requests per minute, so the burst used to return 429s to real members.

The first version called the model for every question. p50 latency was 900 ms per answer, and the bill was about **$0.0019 per answer**, or **$3,420 per month**.

The second version added a semantic cache at the gateway: one embedding call per question (about **8 ms**, **$0.00002**), one HNSW index, threshold **0.86** on cosine similarity, TTL 24 hours.

```
THE REQUEST PATH AFTER THE CACHE — and the numbers it produced
═══════════════════════════════════════════════════════════════════════

  member question
        │
        ▼
  ┌─────────────────┐        ┌──────────────────────┐
  │ embed (8 ms)    │───────►│ ANN search (HNSW)    │
  │ $0.00002        │        │ nearest stored prompt│
  └─────────────────┘        └───────────┬──────────┘
                                         │
                                         ▼
                              ┌──────────────────────┐
                              │ similarity ≥ 0.86 ?  │
                              └───┬──────────────┬───┘
                            yes   │              │  no
                                  ▼              ▼
                     ┌────────────────────┐  ┌────────────────────┐
                     │ return stored      │  │ call the model     │
                     │ answer  (9 ms)     │  │ 900 ms, $0.0019    │
                     └────────────────────┘  └─────────┬──────────┘
                                                       │ store vector
                                                       │ + answer + TTL
                                                       ▼
                                             ┌────────────────────┐
                                             │ vector index       │
                                             └────────────────────┘

  RESULT (measured over one month, 1.8M questions)
  ─────────────────────────────────────────────────────────────────────
  hit rate, all day          38%     p50 latency   900 ms → 561 ms
  hit rate, sahur burst      71%     p50 in burst  900 ms → 267 ms
  burst model calls          9,000 → 2,610 per hour at a 71% hit rate;
                             900 q/min peak → 261 calls/min, under the
                             400/min provider limit
  cost                       $3,420 → 1,116,000 misses × $0.0019
                             + 1.8M embeddings × $0.00002 = $2,156
  saving                     ≈ $1,264 per month, before tune-up
  ─────────────────────────────────────────────────────────────────────
```

Then the cache answered a question it should have refused.

Two members asked questions that a human reads as opposites. `"Batal nggak puasanya kalau lupa makan siang?"` (a genuine mistake) and `"Batal kan puasanya kalau sengaja makan?"` (a deliberate act). The answers differ: one is "the fast stands, continue your day", the other is "the fast is broken, you must make it up". The embeddings scored **0.91**. Member two received the cached answer for member one, in flawless Indonesian, in 9 ms. The member escalated, and the escalation is what found the bug, not any dashboard.

The fix list is the same list every team ends up with. They split the index by category and gave the **fiqh** category a threshold of **0.95**. They added an exact-match tier in front, so repeated identical questions never pay an embedding call. They added a small validator model on hits between 0.91 and 0.95, and they accepted the extra ~300 ms on that thin band. They built a **"must not hit" CI suite** of 240 adversarial pairs — lupa versus sengaja, sahur versus iftar, pregnant versus menstruating, travel by air versus travel by land — and every pair must score below the category threshold or the build fails. And they scoped member-specific answers, such as cycle tracking and medication notes, to a per-member namespace with the cache disabled for those categories.

Three lessons generalise. **A hit is a claim about correctness, not a cache statistic**, so precision — never hit rate — is the number that decides whether the cache stays. **Thresholds belong to categories, not to the system**, because "what is the refund policy" is a safe question to fuzzy-match and "does my fast count today" is not. And **the embedding call is paid on every request, hit or miss**, so a semantic cache that fires rarely is a straight cost increase: at a 12% hit rate the assistant would have paid $3,046 against a $3,420 baseline, which means the team optimised a metric and paid more money.

The punchline: every cache before this one reused *bytes*, and bytes are safe to compare. A semantic cache reuses *meaning*, and meaning is a similarity score with a decimal point. That decimal point now sits on the request path of a system where a wrong answer costs trust, so the work is no longer cache configuration — it is a labelled test set, a per-category threshold, and the discipline to serve a slower answer when the fast one might be wrong.

---

day - 17

## Copy-on-Write (COW)

### Definition:

**Copy-on-Write (COW)** is a memory-sharing rule. Two owners point to the *same* physical page. The kernel marks that page read-only for both owners. The first write makes a private copy for the writer only. The other owner keeps the original page.

The rule does not remove a copy. It moves the copy in time. A clone becomes cheap now and expensive at the first write.

- **Eager copy** pays once, up front: O(memory). The clone waits for a full duplicate of RAM.
- **Copy-on-Write** pays O(metadata) at clone time, then O(pages written) over the lifetime.

```
EAGER COPY  vs  COPY-ON-WRITE — one 400 MB parent, one clone
══════════════════════════════════════════════════════════════════════

EAGER COPY — the clone waits, the memory is committed at once
┌────────────────────────────────────────────────────────────────────┐
│                                                                    │
│  parent 400 MB ──► [ read all 400 MB ] ──► [ write all 400 MB ]    │
│                                                     │              │
│                                                     ▼              │
│                                          child 400 MB (private)    │
│                                                                    │
│  clone runs after      ~1.5 s      page that nobody touches:       │
│  memory now            800 MB      still copied, still paid        │
└────────────────────────────────────────────────────────────────────┘

COPY-ON-WRITE — the clone starts at once, the pages stay shared
┌────────────────────────────────────────────────────────────────────┐
│                                                                    │
│  parent 400 MB ──► [ copy PAGE TABLES only ] ──► clone starts now  │
│        ▲                        │                                  │
│        │                        ▼                                  │
│        │             ┌───────────────────────┐                     │
│        └─────────────┤ ONE physical set of   │◄── child points to  │
│                      │ 400 MB pages          │    the SAME pages   │
│                      │ PTE bit: read-only    │                     │
│                      └───────────────────────┘                     │
│                                                                    │
│  clone runs after   ~1–10 ms     memory now   400 MB shared        │
│                                                                    │
│  then the writes arrive, one page at a time:                       │
│    parent writes page 12   ──► fault ──► copy 4 KB ──► parent own  │
│    child  writes page 12   ──► fault ──► copy 4 KB ──► child  own  │
│    page 99 nobody writes   ──► nothing copied, nothing paid        │
└────────────────────────────────────────────────────────────────────┘
```

Every first write runs the same four steps in the kernel:

```
ONE COPY-ON-WRITE FAULT — what a single first write costs
═══════════════════════════════════════════════════════════════════════

  ① store instruction hits a page marked read-only
        │                          (protection fault, not a slow disk)
        ▼
  ② kernel allocates one free physical page
        │
        ▼
  ③ kernel copies the page content into the new page
        │
        ▼
  ④ kernel flips the PTE to read-write for this process, then sends a
     TLB shootdown to every other core that maps the same page

  4 KB copy + one fault + one IPI  →  in the critical path of the write
```

COW is not a fork feature. It is a general sharing rule, and it carries these systems:

- **`fork()`** — Redis background saves and AOF rewrite, gunicorn/uWSGI prefork workers, shell pipelines, CRIU checkpoints.
- **MicroVM snapshots** — Firecracker and similar sandboxes restore a machine from a memory file with COW. Clone cost stays O(metadata) instead of O(RAM).
- **Container image layers** — `overlayfs` holds lower layers read-only. The first write to a lower-layer file triggers **copy-up**: the whole file moves into the container layer.
- **Filesystem snapshots** — Btrfs and ZFS snapshots copy block references, not blocks.
- **`mmap(MAP_PRIVATE)`** — a file mapped private onto memory. The page cache stays shared until the mapping is written.

The unit of the copy is the page. That one fact controls the size of the bill.

```
PAGE SIZE IS THE UNIT OF THE COPY — the setting that decides how big the bill gets
═════════════════════════════════════════════════════════════════════════════════

  page size   who sets it                  one first write copies
  ─────────────────────────────────────────────────────────────────────────────
  4 KB        default, THP = madvise/never      4 KB   (the small page)
  2 MB        THP = always (default on many distros)  2 MB (the whole huge page)
  ─────────────────────────────────────────────────────────────────────────────

  one byte written inside a 2 MB huge page copies the full 2 MB
  → on a write-heavy box, THP turns a 2x memory spike into 4x or worse
```

Four properties of COW are easy to miss, and each one has produced an outage:

- **Page tables are still copied.** `fork()` writes out the paging metadata for the whole mapped address space. A multi-GB process pays milliseconds at fork time, even when it writes nothing. Kernel work (shared last-level page tables) exists to cut this cost.
- **Sharing is coarse.** Two owners that write two different variables inside the *same* 4 KB page pay two copies. A shared allocator header or a hot lock in a shared page defeats the sharing fast.
- **Memory accounting lies.** A container limit is enforced on RSS, not on logical data size. Shared pages are charged to the first writer, so a snapshot can push RSS far above the data size and invite the OOM killer.
- **Reference counting breaks the sharing.** CPython bumps a reference count inside the object header, so even a *read* of a shared object writes to its page. Prefork Python workers therefore share less memory than the theory promises. Runtimes without per-object counters share more.

Correct use comes down to measurement and one setting:

- **Measure the copy, do not guess it.** Redis reports `rdb_last_cow_size` and `aof_last_cow_size`. A value above roughly 50% of `used_memory` means the box is copying enough pages to threaten itself.
- **Size memory for RSS, not for the dataset.** A dataset that forks for a snapshot needs headroom near 1.5x, on top of the data itself.
- **Turn THP off for fork-heavy workloads** (Redis, forked database backups). Set both `transparent_hugepage/enabled` and `transparent_hugepage/defrag` to `never`, and make the change survive reboot.
- **Set `vm.overcommit_memory=1`** when a fork must reserve address space for the eventual copy. Caution: this trades a clean failure for a possible OOM kill.
- **Keep image layers immutable** in containers, so copy-up runs once per file instead of in a loop.

### Example:

Kyomel's **fasting-bot** runs on one VPS with **1 vCPU and 1 GB RAM**. A small Redis on the same box holds the leaderboard, the streak counters, and the reminder queue. Redis keeps RDB snapshots on, so the data survives a restart.

The numbers below are the shape of this failure class, not a measurement of that box. The commands at the end let Kyomel check his own instance.

```
THE SAVE THAT KILLS THE BOX — one snapshot, one fork, one OOM
═══════════════════════════════════════════════════════════════════════

  dataset = 300 MB (used_memory 300 MB)      cgroup limit = 512 MB

  03:58  cron fires SAVE ──► Redis calls fork()
         │
         ├── parent  ──┐
         └── child   ──┴──► SAME 300 MB of pages, PTE read-only
                            extra memory at fork time: ~2 MB of page tables

  04:00  the sahur burst starts
         │   reminder jobs fan out, every streak update writes a counter
         ▼
  ┌──────────────────────────────────────────────────────────────┐
  │ parent writes → the child needs a stable view of the data    │
  │ → every dirty page is copied for the PARENT and charged      │
  │   to the parent RSS                                          │
  └──────────────────────────────────────────────────────────────┘
         │
         ▼
  04:07  rdb_last_cow_size = 205 MB   (68% of used_memory — past the
         safe line, the workload copied two thirds of the dataset)
         parent RSS  300 MB ──► 505 MB
         cgroup used 470 MB ──► 505 MB   > 512 MB limit
         │
         ▼
  ┌────────────────────────────────────┐
  │ OOM killer kills Redis             │
  │ child dies with it — the snapshot  │
  │ never lands on disk                │
  │ bot restarts with a COLD cache     │
  └────────────────────────────────────┘
  ─────────────────────────────────────────────────────────────────────
  used_memory during the whole event: flat at 300 MB
  → the dashboard that watches the dataset shows nothing wrong
  → the dashboard that watches RSS shows the whole event
```

The failure needs three ingredients at the same time: a forked child, a write-heavy window, and memory sized from the data instead of from RSS. Remove any one of them and the box survives.

The fix list is short and each item maps to one cause:

- **THP off** on that VPS. With 2 MB pages, one byte written inside a huge page copies 2 MB. This is the single largest multiplier.
- **Memory sized for RSS**: 300 MB dataset → at least 450 MB of headroom, or a bigger cgroup limit.
- **The snapshot moved out of the burst window** (the burst is 04:00–05:00 WIB, so the save runs in the quiet afternoon).
- **An alarm on `rdb_last_cow_size`** above 150 MB, so the next write storm pages Kyomel instead of surprising him.
- **`vm.overcommit_memory=1`**, or an explicit `maxmemory` policy, so the fork cannot be refused halfway.

```
VERIFY ON THE BOX — three commands, no guessing
═══════════════════════════════════════════════════════════════════════

  $ cat /sys/kernel/mm/transparent_hugepage/enabled
    [always] madvise never        ← must read "never" for a forking store

  $ redis-cli info persistence | grep cow_size
    rdb_last_cow_size:205000000   ← compare with used_memory from
                                    "info memory"

  $ redis-cli info stats | grep latest_fork
    latest_fork_usec:...          ← 3-digit microseconds = healthy,
                                    six digits = page-table copy pain
```

Three lessons generalise. **COW moves the copy into the write path**, so the writer pays at fault time instead of the cloner paying at clone time — latency moves with the bill. **Any system that promises a fast clone ships a copy that arrives later**, so the correct question is always "who writes, and how much?" And **the page is the atomic unit of the trade**, which is why one kernel setting (THP) can turn a survivable 2x spike into a fatal 4x one.

The punchline: `fork()`, container start, microVM snapshot, and filesystem snapshot all promise the same thing — an instant copy. None of them makes a copy. They postpone it to the first write, and they hand the cost to whoever writes first. Read that contract before sizing the box, because the box gets OOM-killed by RSS, and RSS counts the pages that COW copied.

---

day - 18

## CRDT (Conflict-Free Replicated Data Type)

### Definition:

A **CRDT (Conflict-Free Replicated Data Type)** is a data type that many replicas update at the same time, without a central coordinator. Any two replicas merge into one identical result. The merge order does not matter.

The guarantee has a name: **strong eventual consistency**. Any two replicas that received the same set of updates hold the same state. A replica that received fewer updates is only behind. It is never wrong.

The reason CRDTs work is algebra, not consensus. The merge function must obey three laws:

- **Commutative** — `merge(A, B) = merge(B, A)`. Replicas may exchange updates in any order.
- **Associative** — `merge(merge(A, B), C) = merge(A, merge(B, C))`. A replica may merge in batches.
- **Idempotent** — `merge(A, A) = A`. A duplicate update changes nothing.

These three laws define a **join-semilattice**. That is the whole trick. The network may drop, duplicate, delay, or reorder messages. The state still converges. No lock, no leader, no two-phase commit, no quorum.

```
NAIVE REPLICATION  vs  CRDT — two replicas edit the same data, no coordinator
════════════════════════════════════════════════════════════════════════════════

NAIVE LAST-WRITE-WINS — the whole record is overwritten
┌──────────────────────────────────────────────────────────────────────────────┐
│                                                                              │
│   phone A (offline)              server                 phone B (online)     │
│   ─────────────────              ──────                 ────────────────     │
│   row = {streak: 12}     ...     row = {streak: 12} ◄──  row = {streak: 13}  │
│   user taps "done"               (B's write lands)                            │
│   row = {streak: 13}                                                          │
│        │                                                                      │
│        │  network returns                                                     │
│        └──────► pushes WHOLE row {streak: 13} ──► server row = {streak: 13}   │
│                                                                              │
│   RESULT: B's edit is gone. No error appears. The row still looks correct.    │
│   Two concurrent edits to ONE record = one survives, the other is lost.       │
│   A stale client can also overwrite NEWER data (clock skew, retry, replay).   │
└──────────────────────────────────────────────────────────────────────────────┘

CRDT — merge, never overwrite
┌──────────────────────────────────────────────────────────────────────────────┐
│                                                                              │
│   replica A                       replica B                                  │
│   ─────────                       ─────────                                  │
│   counter = {A: 7, B: 6}          counter = {A: 6, B: 7}                     │
│                    \             /                                           │
│                     \           /   merge = take the MAX per entry           │
│                      ▼         ▼    (no clock, no order, no coordinator)     │
│             ┌───────────────────────────────┐                                │
│             │ {A: 7, B: 7}  →  total = 14   │                                │
│             └───────────────────────────────┘                                │
│                                                                              │
│   RESULT: both edits survive. Both replicas hold the same value.             │
│   Merge the same pair 10 times  → same value (idempotent).                   │
│   Merge in any order            → same value (commutative).                  │
└──────────────────────────────────────────────────────────────────────────────┘
```

This is a different place in the consistency spectrum than consensus. Raft and Paxos order every write through a leader, so a read is always current. A CRDT removes the ordering step, so a read is only eventual. The trade is deliberate: no leader means no failover stall and no cross-region round trip.

```
THE TWO FAMILIES — what travels on the wire
═══════════════════════════════════════════════════════════════════════════════

  STATE-BASED (CvRDT)               OPERATION-BASED (CmRDT)
  ─────────────────────             ──────────────────────────
  sends the FULL state              sends each OPERATION
  merge = join of two states        merge = apply the operation
  delivery: any time, any order     delivery: causal order, exactly once
  robust, bandwidth-heavy           efficient, transport must be solid
        ▲                                     ▲
        └──────────── DELTA-STATE CRDT ───────┘
                      sends only the delta,
                      keeps the idempotent join

  Delta-state is the 2026 default. Yjs and Automerge both ship deltas,
  not full documents, on every update.
```

The family of types is small. Each type answers one shape of question:

- **G-Counter** — grow-only counter. One slot per replica. Value = sum of all slots. Merge = max per slot.
- **PN-Counter** — two G-Counters, one for increments and one for decrements. Value = plus minus minus.
- **LWW-Register** — one value plus a timestamp. The later write wins. Simple, and it silently drops the other write.
- **OR-Set (Observed-Remove Set)** — add and remove concurrently. A remove tombstones only the tags that replica observed. A concurrent add survives.
- **Sequence CRDT (RGA, YATA, Logoot)** — text. Every character gets a stable ID and sits between two neighbours, so two inserts at one spot both land.
- **OR-Map** — a map whose values are themselves CRDTs.

Where this runs in production today:

- **Collaborative editors** — Yjs and Automerge for text and JSON documents. Reported users include Jupyter, Evernote, and Sanity Studio. Figma and Apple Notes both built CRDT-flavoured models for offline edits.
- **Geo-replicated databases** — Redis Enterprise Active-Active replicates counters and sets across regions with CRDT merge, so two regions accept writes at the same time. Riak DT and AntidoteDB belong to the same class.
- **Sync engines** — the local-first stack of 2026 pairs a CRDT document with a sync engine for transport and storage.

Published framework numbers (2026 vendor comparison, so treat them as a shape, not a promise):

```
  Yjs                              Automerge v2
  ───                              ────────────
  load a 50K-op document ~40 ms    ~80–120 ms
  memory ≈ 2x raw text             ≈ 3–5x, history kept by default
  ~10K ops/s per document          ~3–5K ops/s per document
  compaction aggressive            full history by default (a feature
                                   for version-controlled documents)
```

Four properties of CRDTs are easy to miss, and each one has a bill attached:

- **The bill arrives as metadata.** A CRDT converges only because it keeps enough history to merge with any future replica. Deleted items stay as **tombstones**. A G-Counter keeps one slot per replica that ever wrote. A 1,000-character document edited hard can hold roughly 50,000 tombstones. That figure comes from published CRDT measurements, not from your box.
- **Garbage collection needs coordination.** A tombstone may go only after every replica acknowledges the delete. The one thing a CRDT avoids — coordination — comes back for cleanup. Until then, the replica keeps paying.
- **Merge cannot enforce a global invariant.** Merge asks "both edits are valid, how do I keep both?". An invariant asks "only one of these edits is valid".
- **Presence and cursors are not the same thing.** Awareness data (who is online, where the cursor sits) belongs outside the CRDT. It is ephemeral and it must expire.

```
CRDT MERGE  vs  GLOBAL INVARIANT — the job a CRDT cannot do
═════════════════════════════════════════════════════════════

  example: a unique username
    replica A claims "kyomel"        replica B claims "kyomel"
             │                                 │
             └───────────── merge ─────────────┘
                              │
                              ▼
            both claims survive locally
            → two rows hold one name
            → the UNIQUE index fails at write time, or one user
              is silently renamed later

  rule of thumb:
    counters, sets, text, presence, drafts, offline queues → CRDT fits
    balances, unique keys, stock, quotas, payments          → server decides
```

Use a CRDT when replicas must accept writes while the network is bad, and when a lost edit costs more than extra metadata. Skip it when every write must pass one rule (money, stock, unique names), or when one central database is fast enough.

### Example:

Kyomel's **fasting-bot** takes fasting logs from WhatsApp and keeps a leaderboard on one VPS with **1 vCPU and 1 GB RAM**. The bot is small. The network around it is not reliable.

Two normal WhatsApp behaviours break a naive counter:

1. **Webhook retry.** WhatsApp re-sends a webhook event when it does not receive the 200 response in time. The same "fast done" event arrives twice.
2. **Late delivery.** A user turns airplane mode on during a flight or a bad signal, logs "sahur" at 04:30, and the message reaches the bot at 07:12 — after the "buka" message at 06:00.

Neither behaviour is a bug. Both are how the network works. The storage design must absorb them.

```
A FASTING BOT WITHOUT CRDT — one replay, one late message, two wrong numbers
════════════════════════════════════════════════════════════════════════════════

  clock      what arrives at the bot                    row in the database
  ────────────────────────────────────────────────────────────────────────────
  04:00      (user A: airplane mode, offline)            user A {
                                                            status: "fasting"
                                                            hours:  0
                                                          }
  05:10      webhook "fasting done" (attempt 1)
             → UPDATE hours = hours + 14                 hours: 14
  05:13      webhook TIMEOUT, WhatsApp retries
             same message, same event id
             → UPDATE hours = hours + 14                 hours: 28   ◄── WRONG
  06:00      user B logs normally                        user B { hours: 12 }
  07:12      user A lands, 04:30 "sahur" finally arrives
             → UPDATE status = "sahur", hours = 0        A hours: 0  ◄── LOST
                                                            the 14 credited
                                                            hours are gone

  ────────────────────────────────────────────────────────────────────────────
  two silent failures, no error line in any log:
    • replay     → a counter is not idempotent by itself
    • late write → a whole-row overwrite erases newer data
  the leaderboard still renders. it is simply wrong.
```

The same two behaviours against a CRDT-backed log:

```
THE SAME TWO EVENTS WITH A CRDT — replay and late arrival are absorbed
══════════════════════════════════════════════════════════════════════════════

  user A holds an offline log. The bot holds its own copy.
  Every event carries a stable ID: (user, session, event kind, timestamp)

  ┌─────────────────────────────┐         ┌───────────────────────────────┐
  │ phone A  (offline replica)  │         │ bot      (server replica)     │
  │                             │         │                               │
  │ events: {e-sahur-0430,      │         │ events: {e-fast-0510,         │
  │          e-buka-0600}       │         │          e-buka-0610}         │
  │  (an OR-Set — adds do not   │         │  (an OR-Set)                  │
  │   overwrite each other)     │         │                               │
  └──────────────┬──────────────┘         └───────────────┬───────────────┘
                 │                                        │
                 │  signal returns, phone syncs           │
                 ▼                                        ▼
            ┌──────────────── MERGE (union of event IDs) ────────────────┐
            │  {e-sahur-0430, e-fast-0510, e-buka-0600, e-buka-0610}     │
            │  a replayed webhook adds an ID that is ALREADY a member    │
            │  → the set does not change           (idempotent)          │
            └────────────────────────────┬───────────────────────────────┘
                                         ▼
                     fasting hours are DERIVED from the set, not stored
                     ┌──────────────────────────────────────────────┐
                     │ derived view:                                │
                     │   day - 1   sahur 04:30 → buka 06:00  = 90 m │
                     │   streak   1 day  (both events present)      │
                     │ no counter to double, no row to overwrite    │
                     └──────────────────────────────────────────────┘
```

The derived view is the key move. The bot stores **facts** (events), and it computes **numbers** (hours, streak, rank) from those facts. A replayed fact changes nothing, because a set treats a second identical add as a no-op. A late fact needs no repair job, because it merges into the same set. Only the derived view changes, and it changes on the next read.

The three rules that carry over to any bot like this:

- **Make the event ID stable.** A hash of `(sender, message id, event kind)` works. Always use the sender's message ID, never the arrival time.
- **Store facts, derive numbers.** Counters and streaks are views. They never accept a direct write.
- **Keep the leaderboard in one place.** The leaderboard is a ranking over all users, so it is a global view. It has no business being a replicated CRDT. Compute it on the server from the merged facts.

```sql
-- The idempotency rule, in one line of schema.
-- A replayed webhook hits this constraint and changes nothing.
CREATE TABLE fasting_events (
  event_id   TEXT PRIMARY KEY,   -- (sender, wa_message_id, kind)
  user_id    TEXT NOT NULL,
  kind       TEXT NOT NULL,      -- 'sahur' | 'buka' | 'water' | 'dry'
  event_time INTEGER NOT NULL    -- time from the message, not from the server
);
```

The honest limits of that design on a 1 vCPU box:

- **The `fasting_events` table grows forever.** A tombstone-free event log is cheap per row, but it never shrinks. Add a monthly rollup: keep raw events for 90 days, keep the derived daily totals for years.
- **An event log cannot enforce "one fast per day".** Two machines may both log a start. The rule "one fast per day" is a global invariant, so one server must decide it. The CRDT part removes duplicate loss. It does not remove the rule.
- **Two devices owned by one user is the real test.** If Kyomel later adds a phone app beside the bot, the same user writes from two replicas. Then the merge matters, and the event ID must include the device, not only the WhatsApp sender.

A CRDT replaces coordination with algebra. The return is real: replicas accept writes during a bad network, replays and late messages are harmless, and no leader has to be elected. The price is also real, and it lands in three places — extra metadata that needs garbage collection, weak reads that may lag, and invariants that need a server decision anyway. Choose the CRDT for the part of the data that tolerates all three. Keep one server-owned path for the part that does not.

---

day - 21

## FlashAttention (IO-Aware Exact Attention)

### Definition:

**FlashAttention** is an attention algorithm that never materializes the N x N score matrix in GPU global memory. It computes attention in small tiles that stay on-chip, and it fuses the whole operation into one GPU kernel. The output is exact. The saved data movement is the whole point.

Start with the memory hierarchy, because the algorithm makes sense only there:

```
THE TWO MEMORIES THAT DECIDE ATTENTION SPEED
═══════════════════════════════════════════════════════════════════════

  HBM (global memory)            SRAM (shared memory, on-chip)
  ────────────────────           ──────────────────────────────
  40–80 GB per GPU               192–228 KB per SM
  ~1.5–3 TB/s                    ~19 TB/s
  off-chip, slow, big            on-chip, fast, tiny
        ▲                              ▲
        │                              │
        └──── ~12x bandwidth gap ──────┘

  Attention moves a lot of bytes:
    - FLOPs  : O(N² · d)     (d = head dim, N = sequence length)
    - bytes  : O(N²)         (one score per query-key pair)
  -> few FLOPs per byte moved = MEMORY-BOUND
  -> the GPU cores wait on memory. More FLOPs/s does not help.
```

A naive attention kernel walks the N x N matrix through HBM several times: write the scores, read them back for softmax, write the probabilities, read them back for the final multiply. At N = 8,000 tokens with 8 heads, that matrix alone is 1.07 GB in FP16. At N = 128,000 tokens it is 32.8 GB for a single head. The compute is fine. The traffic is fatal.

FlashAttention removes the matrix instead of compressing it:

```
NAIVE ATTENTION (materialize)  vs  FLASHATTENTION (tile + fuse)
═══════════════════════════════════════════════════════════════════════

NAIVE — the N x N matrix lives in HBM, and the kernel touches it 4x
┌───────────────────────────────────────────────────────────────────┐
│                                                                   │
│   Q, K ──► S = Q·Kᵀ ──► [WRITE S: N x N] ──► [READ S]             │
│                                                 │                 │
│                                                 ▼                 │
│                            P = softmax(S) ──► [WRITE P: N x N]    │
│                                                    │              │
│                                                    ▼              │
│   V ─────────────────────────────────────► O = P·V  ([READ P])    │
│                                                                   │
│   HBM traffic : Θ(N·d + N²)      Memory : Θ(N²)  per head         │
│   Kernel count: 4+ separate passes, each one re-reads the result  │
└───────────────────────────────────────────────────────────────────┘

FLASH — the matrix never exists. Tiles pass through SRAM, once
┌───────────────────────────────────────────────────────────────────┐
│                                                                   │
│   for each block of Q        (Q block stays in registers)         │
│     O = 0;  m = -inf;  l = 0        ← running output, max, sum    │
│     for each block of K, V   (streamed from HBM)                  │
│        S_tile = Q_block · K_blockᵀ       128x128 → ~32 KB in SRAM │
│        m_new  = max(m, rowmax(S_tile))                            │
│        l        = e^(m - m_new)·l + rowsum(e^(S_tile - m_new))    │
│        O        = e^(m - m_new)·O + e^(S_tile - m_new)·V_block    │
│        m        = m_new                                           │
│     write O once ──► HBM        (plus one scalar logsumexp per row)│
│                                                                   │
│   HBM traffic : Θ(N²·d² / M)     Memory : Θ(N·d)  per head        │
│   Kernel count: 1                                                │
└───────────────────────────────────────────────────────────────────┘

  M = SRAM size. With d = 64–128 and M ≈ 100 KB, the paper measures
  4x–20x fewer HBM accesses — close to the IO lower bound for matmul.
```

Four ideas carry the design:

- **Tiling.** Split Q, K, V into blocks that fit in SRAM. The full matrix is never needed at once, so it is never built.
- **Online softmax.** Softmax needs the row maximum and the row sum, and both are known only after the whole row. The online form keeps a running max `m` and a running sum `l`, and rescales the partial output `O` each time a new block raises the max. The rescale is exact, not an approximation. It corrects earlier blocks to the new max, so the final value equals the textbook softmax.
- **Kernel fusion.** One kernel does the matmul, the max, the exponential, the sum, and the second matmul. No intermediate tensor travels to HBM.
- **Recomputation in the backward pass.** Training needs the probability matrix `P` for gradients. Standard training stores `P` (O(N²)). FlashAttention stores only the logsumexp value `L` — one float per query row (O(N)) — and rebuilds `P` from SRAM tiles during backward. That adds FLOPs and removes a large block of HBM traffic. On a memory-bound kernel, that trade wins.

This is not an approximation. Sparse attention and linear attention change the math to cut FLOPs. FlashAttention keeps the math and cuts bytes. Models stay bit-comparable to the reference attention up to floating-point tolerance.

The generations:

- **FlashAttention-1 (Dao et al., NeurIPS 2022, A100).** Tiling plus online softmax plus fusion. Loop order: outer loop over K, V blocks.
- **FlashAttention-2 (Dao, 2023).** Inverted the loop nest, so the output block stays in registers. Added parallelism over the sequence dimension. Better work partitioning.
- **FlashAttention-3 (Shah et al., 2024, Hopper H100).** Asynchronous copies with the Tensor Memory Accelerator. Warp specialization: producer warps load and run softmax, consumer warps run tensor-core matmul. FP8 support with block scaling.
- **FlashAttention-4 (Tri Dao and collaborators, March 2026, Blackwell B200/GB200).** Targets the new bottleneck: tensor cores now outrun shared memory and the exponential unit. Adds emulated exponentials, LPT scheduling of tiles, 2-CTA MMA mode, and CuTe-DSL in Python instead of C++ templates. Reported: up to 1613 TFLOPs/s (about 71% of B200 peak) in BF16, 1.3x over cuDNN 9.13, 2.7x over Triton, and 22x–32x faster kernel compile (2.5 s vs 55 s for the forward kernel).

Adoption is the reason this term is worth knowing: the flash kernel is the default attention path in PyTorch `scaled_dot_product_attention`, and in vLLM, HuggingFace Transformers, TensorRT-LLM, and cuDNN.

### Example:

A coding agent sends one request with a 120,000-token repository context to a single 80 GB GPU. This is a prefill-heavy request: the model must run attention over all 120,000 tokens at once. Compare the two paths.

```
ONE 120K-TOKEN PREFILL REQUEST — WHERE THE MEMORY GOES
═══════════════════════════════════════════════════════════════════════

  NAIVE ATTENTION — the score matrix must exist in HBM
  ─────────────────────────────────────────────────────────────────
    N = 120,000   FP16 (2 bytes per value)

    scores per head        = N x N x 2 B     = 28.8 GB
    with 8 KV heads        = 8 x 28.8 GB     = 230 GB
    ┌──────────────────────────────────────────────────────────┐
    │  RESULT: OOM on one 80 GB GPU before the first matmul     │
    │  ends. Chunked workarounds exist, and they re-read HBM.   │
    └──────────────────────────────────────────────────────────┘

  FLASHATTENTION — only O(N) tensors exist
  ─────────────────────────────────────────────────────────────────
    Q, K, V, O  per head   = 4 x N x d x 2 B = 123 MB  (d = 128)
    logsumexp L per head   = N x 4 B         = 0.48 MB
    with 8 KV heads        = ~0.99 GB total
    ┌──────────────────────────────────────────────────────────┐
    │  RESULT: fits. The 28.8 GB score matrix is never created. │
    │  The 128x128 tiles live and die inside SRAM.              │
    └──────────────────────────────────────────────────────────┘
```

The execution path, block by block:

```
PREFILL PATH — 120K tokens, one Q block at a time
═══════════════════════════════════════════════════════════════════════

  prompt (120K tokens)
        │
        ▼
  ┌───────────────┐   split K, V into blocks of 128 columns
  │ embed + proj  │───────────────┐
  └───────┬───────┘               │
          │ Q block 0 (128 rows)  │
          ▼                       ▼
  ┌───────────────────────────────────────────────────────────────┐
  │  SRAM WORKING SET (per SM, ~200 KB)                           │
  │                                                               │
  │    Q_block ──► ×K_blockᵀ ──► S_tile (128x128, 32 KB)          │
  │                                  │                            │
  │                                  ▼                            │
  │    running max m ──► exp ──► running sum l ──► rescale O      │
  │                                  │                            │
  │                                  ▼                            │
  │                         O += P_tile · V_block                 │
  │                                                               │
  │    K_block, V_block arrive, are used, are dropped.            │
  │    Only m, l, O stay.                      ← O(N) state       │
  └───────────────────────────────┬───────────────────────────────┘
                                  │ last K/V block processed
                                  ▼
                       O written to HBM once
                       L (logsumexp) kept for backward
                                  │
                                  ▼
                       decode loop starts (1 token at a time,
                       now bound by KV-cache bandwidth,
                       not by attention FLOPs)
```

Two notes that separate a correct mental model from a popular half-truth:

- **FlashAttention does not make attention cheap in FLOPs.** It runs the same O(N²·d) math. At short sequences and small batches, the tensor cores are already the limit, so the speedup is small. The gain grows with sequence length, because that is where memory traffic dominates.
- **It does not fix decode.** Token-by-token generation is bound by reading the KV cache, which grows with context. FlashAttention speeds up the prefill pass and the long-context training step. For decode, the levers are different: grouped-query attention, paged KV cache, quantization, speculative decoding.

The honest limits:

- **Kernel support is narrower than plain PyTorch attention.** Arbitrary additive attention bias and exotic masks are not supported by the flash kernel. PyTorch picks a different backend (memory-efficient or math) when the mask is unsupported, and that fallback is slower. Head dimensions are capped (256 in FlashAttention-2), so unusual model shapes fall back too.
- **Non-standard attention shapes need a custom kernel.** Sliding-window, prefix-LM, or block-sparse masks are separate implementations, not a flag.
- **Backward adds FLOPs.** Recomputation costs extra compute. The trade is only good while the kernel stays memory-bound.
- **Numerics stay close, not identical.** FP8 mode in FlashAttention-3 and the emulated exponential in FlashAttention-4 change rounding. Accuracy checks stay mandatory for long-context production models.

The transferable habit is the real lesson. Before optimizing, count the bytes, not only the operations. Then find the biggest tensor that nobody actually needs, keep the live state small, and let the fast memory do the work. FlashAttention is that habit applied to the most expensive layer in every Transformer.

---

day - 22

## Backpressure

### Definition:

**Backpressure** is a control signal that travels backward through a pipeline. A slow consumer sends the signal upstream. A fast producer receives the signal and slows down, blocks, or stops. The consumer sets the pace, not the producer.

The need comes from simple arithmetic. Producers almost always run faster than consumers, at least in bursts. Without a signal, the extra work must go somewhere. It goes into a buffer. The buffer grows with no ceiling. Memory runs out. The process dies, and every request in the buffer dies with it.

Backpressure adds one rule: the buffer has a ceiling, and the producer feels that ceiling.

```
WITHOUT BACKPRESSURE (no ceiling)      WITH BACKPRESSURE (bounded gate)
═════════════════════════════════      ═══════════════════════════════════

  producer  12,000 req/s                 producer  12,000 req/s
      │                                      │
      │  no signal goes up                   │  ◄── "slow down" signal
      ▼                                      ▼
  ┌────────────────────┐                 ┌────────────────────┐
  │ queue   size = ∞   │                 │ queue  max = 10,000│
  │ grows 7,000 req/s  │                 │ held at the ceiling│
  │ + 13.7 MB/s        │                 └─────────┬──────────┘
  └─────────┬──────────┘                           │ 5,000 req/s
            │ 5,000 req/s                         ▼
            ▼                                consumer work
      consumer work
                                            heap stays flat.
  8 GB heap fills in 10 minutes.            New work waits, then
  Then OOM kill. All queued                 gets refused at the gate.
  requests fail together.                   Blast radius stays small.

  RESULT: unbounded queue = a latency       RESULT: latency grows inside
  and memory bug that hides until           a known limit. Refusal is
  traffic spikes.                           explicit and cheap.
```

The signal exists at every layer of the stack. The shape is always the same: credit or space flows back, the sender respects it.

```
ONE MECHANISM, MANY LAYERS — THE SIGNAL ALWAYS TRAVELS BACKWARD
═══════════════════════════════════════════════════════════════════════
  LAYER                SIGNAL                 PRODUCER BEHAVIOR
  ───────────────────────────────────────────────────────────────────
  TCP                  receive window (rwnd)  stops sending when the
                       carried in every ACK   window reaches 0, resumes
                                              on the next window update

  HTTP/2 and gRPC      WINDOW_UPDATE frames   pauses the stream or the
                       per stream, per conn   connection, resumes on
                                              fresh window credit

  Node.js streams      write() returns false  waits for the 'drain'
                       past highWaterMark     event (default 16,384
                       (default 16,384 bytes) bytes per stream)

  Reactive Streams     Subscription           emits at most n items,
                       .request(n)            never more. Demand comes
                                              from the subscriber

  Kafka consumers      consumer lag,          poll() stops fetching,
                       pause() / resume()     the group holds its
                                              position until the
                                              consumer catches up

  Application (LLM)    queue depth + queue    producer holds, or the
                       wait time against      gateway answers 429 or
                       the TTFT budget        503 with Retry-After
  ───────────────────────────────────────────────────────────────────
```

Six design rules carry the pattern:

- Make the signal explicit. An unbounded queue deletes the signal. It converts an overload problem into a slow memory leak that appears only under peak traffic.
- Treat the buffer as a shock absorber, not a reservoir. Size the queue for microbursts of seconds, not for a full traffic spike of minutes.
- Keep the signal cheap. A signal that costs a network round trip makes throughput worse. Local credit counters are cheap. Remote calls are not.
- Add timeouts. A producer that waits for a slow consumer must fail at some point. A wait without a deadline is a deadlock in progress.
- Pair it with shedding. Backpressure slows the arrival rate. Load shedding refuses work. Backpressure alone cannot protect a service that has no spare capacity left.
- Alert on the backlog, not on CPU. Consumer lag, queue depth, and queue wait time show the problem first. CPU stays low while a queue grows, because the consumer is waiting on something else.

What it is not:

```
BACKPRESSURE vs LOAD SHEDDING vs RATE LIMITING
═══════════════════════════════════════════════════════════════════
                WHO ACTS        WHAT IT DOES         WHERE THE
                                                     SIGNAL COMES FROM
  ───────────────────────────────────────────────────────────────
  Backpressure  producer        slows, blocks,       the consumer
                (told by the    or waits             (measured load)
                consumer)

  Rate limiting gateway         caps the arrival     a fixed policy
                (per client)    rate before work     or a quota
                                starts

  Load shedding server          refuses work now     local load or
                (own load)      with 429 or 503      a queue ceiling

  They compose. A gateway rate limit protects the door. Backpressure
  protects the pipe. Load shedding protects the last line of defense.
```

Honest limits of the pattern:

- Backpressure moves the failure to the edge. The producer now waits, so the caller sees higher latency. That latency is the visible price of stability.
- A consumer that is permanently too slow never catches up. Backpressure holds the line. It does not add throughput. Only more capacity or less work does that.
- One stuck consumer can block a shared producer. This is head-of-line blocking. Per-tenant queues, separate connections, and bulkheads contain it.
- Both sides can wait on each other. A blocked request that holds a lock the consumer needs is a deadlock. Timeouts and bounded retries break the cycle.

### Example:

An agent platform runs a self-hosted LLM inference service behind an API gateway. One agent workflow fans out many calls at once, and each call chains 10 to 20 model requests.

Monday 09:00. The fan-out sends 12,000 requests per second to the gateway. The GPU worker pool completes 5,000 requests per second. The autoscaler sees the pressure at 09:02. A new GPU pod needs about 4 minutes to schedule, pull the image, and load the model weights.

The queue degrades much faster than the GPU fleet grows. Queue growth wins the race.

```
THE RACE — 09:00 SPIKE, NO BACKPRESSURE
═══════════════════════════════════════════════════════════════════════

  09:00  arrival 12,000 req/s   workers 5,000 req/s   gap 7,000 req/s
         │
         │  2 KB headers per request → +13.7 MB/s into the queue
         ▼
  ┌──────────────────────────────────────────────────────────────────┐
  │ queue depth                                                      │
  │   0 ──► 420,000 ──► 840,000 ──►        ... ──►  2,100,000        │
  │  09:00      09:01       09:02       (spike for 300 s)            │
  └──────────────────────────────────────────────────────────────────┘
         │
         │  8 GB of queue space fills in 10 minutes
         ▼
  09:10  ┌───────────────────────────────────────────────────────────┐
         │ OOM kill. Every queued request fails together.            │
         │ Clients retry. The retries arrive on top of the backlog.  │
         │ The condition is now self-sustaining.                     │
         └───────────────────────────────────────────────────────────┘

  The new GPU pod lands at 09:06. Useful, but late.
  Draining 2,100,000 queued requests at 5,000 req/s takes 420 s (7 min).
  Requests that waited 300 s already blew a 2 s TTFT budget 150x over.
  Queue growth outran both the autoscaler and the SLO.
```

The fix uses four layers. Each layer answers a failure the layer below cannot.

```
CONTROL LOOP THAT HOLDS — SLO + BACKPRESSURE + SHEDDING + AUTOSCALING
═══════════════════════════════════════════════════════════════════════

  client / agent fan-out
        │  12,000 req/s
        ▼
  ┌─────────────────────────────────────────────────────────────────┐
  │ LAYER 1  API GATEWAY — admission control at the door            │
  │          reads queue state, then decides: queue or refuse       │
  └─────────────────────────────┬───────────────────────────────────┘
             accepted           │            refused
                                │             └──► 429 or 503
                                ▼                  + Retry-After: 2
  ┌─────────────────────────────────────────────────────────────────┐
  │ LAYER 2  BOUNDED QUEUE — max_queued_requests = 10,000,          │
  │          max wait = 0.75 s (40% of a 1.9 s TTFT budget)         │
  │          the ceiling IS the backpressure signal                 │
  └─────────────────────────────┬───────────────────────────────────┘
                                │  workers pull at 5,000 req/s
                                ▼
  ┌─────────────────────────────────────────────────────────────────┐
  │ LAYER 3  GPUs (vLLM) — continuous batching, priority lanes.     │
  │          A P0 interactive request preempts a P2 batch request,  │
  │          keeps its KV cache, and resumes when credit returns.   │
  └─────────────────────────────┬───────────────────────────────────┘
                                │
                                ▼
  ┌─────────────────────────────────────────────────────────────────┐
  │ LAYER 4  AUTOSCALER — scales on queue depth and TTFT, not CPU.  │
  │          Adds supply. Needs 4 min. Backpressure covers those    │
  │          4 minutes without losing the whole queue.              │
  └─────────────────────────────────────────────────────────────────┘
```

The gateway rule is a two-state decision, measured against the wait budget:

```
  STATE   TRIGGER                                     ACTION
  ──────────────────────────────────────────────────────────────────
  GREEN   queue_time < 40% of the TTFT budget         accept normally

  RED     queue_time + expected_scale_up_time         shed now:
          >= the TTFT budget                         429 / 503
                                                     + Retry-After

  Why shed and not queue? A request that waits 4 minutes and then
  times out costs full GPU work, full tokens, and one connection.
  A request refused in 1 ms costs nothing and frees the client to
  pick another route. Failing fast is cheaper than failing slow.
```

A simplified gate, in Python. The `await` on a full queue is the backpressure. The timeout is the point where backpressure becomes shedding.

```python
import asyncio

MAX_QUEUED = 10_000        # ceiling on waiting requests
MAX_WAIT_S = 0.75          # 40% of a 1.9 s TTFT budget

class Overloaded(Exception):
    """The gateway answers 429 or 503 with Retry-After."""

class Gate:
    def __init__(self):
        # maxsize is the backpressure signal. Remove it and the
        # queue grows until the process runs out of memory.
        self.queue = asyncio.Queue(maxsize=MAX_QUEUED)

    async def admit(self, req):
        try:
            # Full queue -> put() waits. The producer feels the ceiling.
            await asyncio.wait_for(self.queue.put(req), timeout=MAX_WAIT_S)
        except asyncio.TimeoutError:
            # The wait budget is gone. Shed the request, do not queue it.
            raise Overloaded("429 Retry-After: 2")

    async def worker(self):
        while True:
            req = await self.queue.get()     # pull at consumer speed
            await run_on_gpu(req)
            self.queue.task_done()
```

Measured on the same 09:00 spike, the two designs diverge:

```
OUTCOME COMPARISON — SAME 12,000 req/s SPIKE
═══════════════════════════════════════════════════════════════════

                        NO BACKPRESSURE      BOUNDED + SHED
  ────────────────────────────────────────────────────────────────
  peak queue depth      2,100,000            10,000 (ceiling)
  queue memory          ~8 GB, OOM at 09:10  ~20 MB, flat
  p99 TTFT              300+ s then fail     inside SLO or refused
  refused requests      0 refused, most      429 in under 1 ms,
                        failed later         client retries safely
  worker work           3.15B tokens spent   tokens spent only on
                        on requests that     requests that can
                        never answered       finish
  recovery              cold restart, then   no restart. The queue
                        retry storm          drains in 2 s at
                                             5,000 req/s.

  Key point: autoscaling adds supply, backpressure caps demand, and the
  SLO decides when the system must act. Backpressure alone does not add
  throughput. It stops the pile-up that hides a capacity shortfall as a
  tail latency problem.
```

---