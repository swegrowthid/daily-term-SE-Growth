day - 2

## Deterministic Simulation Testing (DST)

### Definition:

**Deterministic Simulation Testing** runs the whole system inside one simulated
world. The test controls time, scheduling, network, disk, and randomness. A
single seed number drives all of them. When a test fails, you rerun the same
build with the same seed. The failure happens again, at the same
microsecond, on the same node. This is the whole point: **a bug you can
replay is a bug you can fix.**

DST attacks one problem. Distributed bugs need a rare order of events. A
leader election timer fires at the same instant a heartbeat arrives. A network
partition heals during a leadership transfer. A disk write completes after the
process already restarted. Real clusters produce those orders at random, once
per million hours. A test harness that waits with `sleep(5)` never sees them.
Even when it does see one, the next run gives different timing, and the bug
disappears.

DST removes the randomness instead of sampling it. Two ideas do the work:

1. **One thread, one event queue.** All components run as events in a priority
   queue ordered by simulated time. No OS scheduler decides the order. The
   queue decides. Same seed, same queue, same order, every time.
2. **Every I/O call is an interface.** Clock, network, disk, and RNG sit behind
   injectable interfaces. Production binds real syscalls. Simulation binds
   fakes that add delays, drops, reorders, and errors on purpose.

```text
WITHOUT DST — real cluster                  WITH DST — one seed, one process
════════════════════════════════════════    ══════════════════════════════════════════════
                test code                                      test code
                    │                                              │
                    ▼                                              ▼
┌──────────────────────────────────────┐    ┌────────────────────────────────────────────┐
│ real kernel scheduler <- not yours   │    │ SIMULATOR (single thread)                  │
│ real NIC + switch     <- not yours   │    │ seed -> PRNG -> fault schedule             │
│ real disk + cache     <- not yours   │    │ event queue by simulated time              │
│ real wall clock       <- not yours   │    │ sim clock, net, disk, RNG                  │
└──────────────────────────────────────┘    └────────────────────────────────────────────┘
                    │                                              │
                    ▼                                              ▼
   3 nodes, 3 VMs, 3 OSes                      6 nodes, 2 clients, all in RAM
   sleep(5) "wait for election"                same binary as production
   PASS 99% of the time                        runs 10,000 seeds a night
   the 1% is the bug                           each seed = one exact timeline
   fail? no idea why                           fail? seed + git commit
   rerun? maybe never again                    rerun: byte-identical replay
   debug cost: days                            debug cost: minutes
```

**The determinism boundary comes first.** Before you write a simulator, list
every source of nondeterminism. Each one is either wrapped or banned.

```text
SOURCE OF NONDETERMINISM    IN SIMULATION                  WHY IT BREAKS REPLAY
──────────────────────────  ─────────────────────────────  ──────────────────────────
wall clock / time.Now()     SimClock.Advance(dt)           real time never repeats
thread scheduling           one thread + event queue       interleaving is random
goroutine / thread pool     cooperative, deterministic     OS decides the order
random numbers              seeded PRNG, one stream        crypto/rand is not seeded
network send/recv           SimNet: delay, drop, reorder   kernel timing varies
disk read/write/fsync       SimDisk: delay, fail, corrupt  page cache + hardware vary
process crash / restart     fault in the seeded schedule   real crashes kill the test
DNS, NTP, /dev/urandom      stubbed, fixed answers         external and unstable
hash map iteration order    ordered map, or sort keys      Go maps iterate randomly
goroutine ID, pointer addr  never used as a key or seed    changes between runs
```

Pointers and map iteration deserve a warning. Go randomizes map iteration on
purpose. A struct that iterates a map of peers sends messages in a different
order each run. The simulation is then deterministic only in the parts you
wrapped, not in the parts that decide the bug. Sort the keys, or use an ordered
structure.

**The runner is small.** The hard part is the code above it, not the loop
below it.

```text
simulate(seed, ticks)
  rng     <- PRNG(seed)
  world   <- World(clock = t0, net = SimNet(rng), disk = SimDisk(rng))
  nodes   <- [Node(id) for id in 1..6]           # production binary
  faults  <- FaultSchedule(rng)                  # seeded, pre-computed

  for tick in 1..ticks
    clock <- clock + 1ms                         # time jumps; it does not wait
    if faults.due(tick)                          # partition, crash, drop, corrupt
      world.apply(faults.next())
    world.deliver_pending(rng)                   # reorder and delay live here
    for node in nodes
      node.tick()                                # one step, no preemption
    check_invariants(world)                       # after EVERY tick, not at the end

  return PASS
```

`check_invariants` is what makes DST a correctness tool, not just a fuzzer.
Check after every tick. A ledger that double-spends for three ticks and then
repairs itself still passed the final state. Only a mid-run check catches it.

**Reproduction is a tuple, not a story.** The whole failure record is
`(git commit, seed, config)`. TigerBeetle states this directly: its VOPR
simulator replays any failure from the seed and the commit hash. The same
property removes the worst part of distributed debugging, the part where a
teammate says "I cannot reproduce it".

**Provenance, briefly.** FoundationDB built the practice around 2010 with its
Flow runtime and the Simulation harness, and reports tens of thousands of
simulation runs each night. AWS implemented a comparable harness for an early
internal lock service in the same period. Will Wilson described the technique
at Strange Loop in 2014. TigerBeetle built VOPR on those ideas, and states
that one minute of VOPR time covers days of real-world testing. Antithesis
packaged a different variant: unmodified binaries run under a deterministic
hypervisor, so a team can adopt DST without rewriting its I/O layer.

```text
COST AND LIMIT LEDGER — DST IS NOT FREE
┌──────────────────────────────────┬────────────────────────────────────────────────┐
│ PRO                              │ CON                                            │
├──────────────────────────────────┼────────────────────────────────────────────────┤
│ exact replay of a rare bug       │ whole I/O layer must be injected               │
│ time compresses: 1 min of sim    │ the simulator is production code too,          │
│ covers days of cluster time      │ and it carries its own bugs                    │
│ thousands of seeds per night     │ a mocked dependency that lies                  │
│ faults are cheap: no VMs, no     │ hides a real bug behind a fake                 │
│ cloud bill per run               │                                                │
│ invariant checks run every tick, │ determinism ends at the boundary:              │
│ not only at the end              │ one stray time.Now() breaks replay             │
│ a found bug becomes a permanent  │ models no real hardware: page                  │
│ regression seed in CI            │ cache, fsync latency, kernel                   │
│ no flaky reruns, no              │ upfront cost in weeks; many teams              │
│ "works on my machine"            │ abandon it at the first refactor               │
└──────────────────────────────────┴────────────────────────────────────────────────┘

WHAT IT DOES NOT REPLACE
  • staging and canary runs      -> real hardware, real traffic, real latency
  • chaos engineering in prod    -> finds environment bugs the model omits
  • load tests                   -> throughput and cost, not correctness
```

### Example:

A payments team runs a ledger on a 6-replica consensus cluster. The service
moves money between accounts. Its safety invariant is one sentence: **the sum
of all balances never changes, and no account goes negative.** The team wraps
clock, network, disk, and RNG behind interfaces. The same binary then runs
under `sim` or under `prod`. They keep the old cluster test in CI, and it
passes every night. It has never found a real bug.

The simulator runs 5,000 seeds per night in about 40 minutes of wall clock.
Seed `0x9f3c1a` fails on tick 2,884,102, at simulated time `T+1h 47m`. The
failure record is three fields:

```text
SEED REPLAY — the whole bug report is one line
══════════════════════════════════════════════════════════════════════════════

  commit  a41c9e7   seed 0x9f3c1a   replicas 6   clients 2

  $ ./ledger sim --seed 0x9f3c1a --commit a41c9e7
   ... same tick, same node, same message order, every single run
```

The trace prints the last 20 events before the invariant check fails:

```text
T+1h 47m 03s  loop tick 2,884,102  (network: 41 ms delay, 3% drop)
══════════════════════════════════════════════════════════════════════════════

  replica 1   leader for term 12    log: [... t12:apply(transfer 40)]
  replica 2   follower              log: [... t12:apply(transfer 40)]
  replica 3   follower              log: [... t12:apply(transfer 40)]
  replica 4   follower  DOWN        crashed at T+1h 31m, restarted T+1h 44m
  replica 5   follower  DOWN        in the minority partition, still term 11
  replica 6   follower              log: [... t12:apply(transfer 40)]

    T+1h 47m 02.998s   partition heals: {5} rejoins
    T+1h 47m 02.999s   replica 5 sends its term-11 log tail to replica 1
    T+1h 47m 03.000s   replica 1 accepts the append (index already committed)
    T+1h 47m 03.000s   replica 5 replays entries 1..N locally, including t12
    T+1h 47m 03.001s   replica 5 applies transfer 40  -> SECOND TIME
    T+1h 47m 03.001s   INVARIANT FAIL: sum(balances) = -40  (expected 0)
                       at sim tick 2,884,102, node 5, seed 0x9f3c1a
```

Read the sequence. Replica 5 was partitioned in term 11. It never learned that
term 12 committed. On rejoin, it treated its own stale log as authoritative for
the overlapping range. The leader accepted the append because the code checked
"index in range" and did not check "term older than my commit index". The
stale entry then replayed, and the transfer applied twice.

```text
THE SHAPE OF THE BUG — why no cluster test found it
══════════════════════════════════════════════════════════════════════════════

  3-node CI cluster, 200 runs           6-replica sim, seed 0x9f3c1a
  ──────────────────────────────────    ────────────────────────────────────
  crash a node at a random moment       crash replica 4 at T+1h 31m
  wait 10 s                             partition {5} for 16 min of sim time
  assert the cluster elects a leader    heal it at a 1 ms window between
                                        "accept append" and "replay"
  probability of that order:            probability of that order:
  ~1 in 10^6 runs                       found in 5,000 seeds, one night

  the bug is a DEMAND, not a lucky race. The simulator schedules the window.
```

The fix is one guard. The reviewer rejects any append whose term is below the
local commit index. That is a two-line change, and a normal test cannot prove
it. A replay can.

```text
BEFORE                           AFTER
─────────────────────────────    ──────────────────────────────
if entry.index <= lastIndex      if entry.term < commitIndex
  append(entry)                    reject(entry, staleTerm)
                                 else if entry.index <= lastIndex
                                   append(entry)
```

The seed goes into the permanent suite. CI runs it on every pull request
against the consensus package. Total cost of the bug: one failed nightly, one
afternoon of reading a trace, two lines of code. The same bug in production
would have been a duplicate debit, found by a customer, with no way to
reproduce it.

The honest part. This team spent six weeks building the I/O interfaces before
the first useful seed ran. Two of those weeks went into replacing
`time.Now()` and `rand.Intn` calls that had leaked into the consensus path.
The simulator also hides whole classes of bugs. It does not model page cache
eviction, fsync latency on real NVMe, or a kernel that reorders writes. A
torn-write bug needs a real disk or a fault injected at the right layer. Treat
DST as the cheapest way to explore the *logic* of failure, not as proof that
the system is correct. Pair it with a canary and a chaos run in staging, and
state in the design doc which layer each one covers.

The punchline: a distributed bug is a specific order of events. DST turns that
order into an argument you can pass to the program. Pass the seed, and the bug
comes back on demand. Once a rare bug is reproducible, it stops being a
mystery and becomes a test.

---

day - 3

## Service Mesh

### Definition:

A **service mesh** is a dedicated layer that carries service-to-service traffic. It does four jobs for every call between workloads:

1. **Identity.** Each workload gets a cryptographic ID. In Kubernetes this is usually a SPIFFE ID derived from the service account.
2. **Encryption.** Proxies use that ID for mutual TLS. Both sides prove who they are before a byte of payload moves.
3. **Policy.** The mesh allows or denies a call by identity, not by IP address. A pod that restarts with a new IP keeps the same identity.
4. **Traffic control and telemetry.** Retries, timeouts, weighted splits, and one metric set for every hop.

The mesh adds no new feature to your application. It moves existing features out of the application. Before the mesh, `orders` (Go) and `billing` (Java) each carry a retry loop, a TLS client, a timeout setting, and a metrics SDK. Two teams write those four things twice. The two versions disagree, and nobody knows which one is correct.

After the mesh, both apps speak plain HTTP. A proxy next to each pod does the rest. The application code shrinks. The behaviour becomes the same for every language in the cluster.

```text
WITHOUT A MESH — each app owns its networking      WITH A MESH — one proxy layer owns it
═══════════════════════════════════════════════    ═══════════════════════════════════════════════

   orders (Go)          billing (Java)                 orders           billing
   ┌────────────────┐   ┌────────────────┐             ┌─────────┐      ┌─────────┐
   │ retry loop     │   │ retry loop     │             │  app    │      │  app    │
   │ TLS client     │   │ TLS client     │             │  only   │      │  only   │
   │ metrics SDK    │   │ metrics SDK    │             └────┬────┘      └────┬────┘
   │ timeout config │   │ timeout config │             ┌────▼────┐      ┌────▼────┐
   └───────┬────────┘   └───────┬────────┘             │ proxy   │      │ proxy   │
           │                    │                      └────┬────┘      └────┬────┘
           └─────────┬──────────┘                           └──────┬───────┘
                     ▼                                             ▼
      plain TCP · no identity · 2 retry policies        mTLS · SPIFFE identity · 1 policy
      "who called me?" = an IP address                  "who called me?" = a workload ID
      new language = new library to wire                new language = no work at all
```

**Two halves, two failure modes.** The **data plane** moves the packets: the proxies. The **control plane** (`istiod`, Linkerd's destination controller, Cilium's operator) watches the Kubernetes API and pushes configuration to those proxies. When the control plane dies, existing connections keep working and new routes stop. When the data plane dies, requests fail. Keep the two failure modes separate in your runbooks, because the fixes are different.

**Where the proxy sits is the main design choice in 2026.** Three shapes exist:

- **Sidecar per pod.** Istio classic mode and Linkerd. One proxy container in every pod. Inbound and outbound traffic pass through it. L7 features work everywhere, with no extra hop.
- **Per-node L4 proxy plus L7 waypoints.** Istio ambient. A `ztunnel` on each node handles mTLS and L4 policy. An Envoy **waypoint** runs only for the namespaces or services that need HTTP-level routing. Cilium does L4 in eBPF and runs a per-node Envoy for L7.
- **Native sidecars.** Kubernetes made sidecar containers stable in v1.33. This fixes startup order and Job termination. It does not reduce cost, because there is still one proxy per pod.

The cost follows a multiplier. Sidecars scale with **pods**. ztunnels scale with **nodes**. Waypoints scale with the **namespaces that need L7**.

```text
THE MULTIPLIER — cost depends on where the proxy sits
══════════════════════════════════════════════════════════════════════════════

 SIDECAR                              AMBIENT / eBPF
 ──────────────────────────────       ─────────────────────────────────────────
 node  (40 pods)                      node  (40 pods)
 ┌────────────────────────────┐       ┌───────────────────────────────────────┐
 │ ┌──┐┌──┐┌──┐┌──┐┌──┐       │       │ ┌───────────────────────────────────┐ │
 │ │p ││p ││p ││p ││p │  ...  │       │ │ ztunnel  (1 per node, L4 + mTLS)  │ │
 │ └──┘└──┘└──┘└──┘└──┘       │       │ └───────────────────────────────────┘ │
 │  40 proxies per node       │       │   1 proxy per node                    │
 └────────────────────────────┘       └───────────────────────────────────────┘
   600 pods -> 600 proxies              600 pods on 40 nodes -> 40 ztunnels
   L7 everywhere, in-path               L7 only where a waypoint exists

 MEASURED P99 COST — one GCP cluster, mTLS on, 3,200 RPS, baseline 0.22 s
 (Tel Aviv University + Red Hat, Nov 2024)
 ─────────────────────────────────────────────────────────────────────────────
   no mesh          baseline
   Istio ambient    +8%
   Linkerd          +33%
   Cilium           +99%
   Istio sidecar    +166%
```

The numbers do not rank the products. They show the shape: **every proxy in the request path costs something, and the count of proxies is a design decision.** Read the benchmark with its own caveat. The study ran one workload on one cluster, and a different traffic mix moves the numbers.

**What a mesh gives you, and what it does not:**

```text
MESH REPLACES                          MESH DOES NOT REPLACE
────────────────────────────────────   ──────────────────────────────────────────
per-app TLS client libraries           NetworkPolicy at the CNI layer
per-app retry and timeout code         a retry budget you never tuned
per-app metrics instrumentation        application-level idempotency
IP allowlists between services         secret management
manual cert rotation                   someone who can read the data path at 3 a.m.
```

### Example:

A payments platform runs 140 services and 600 pods on 40 nodes. Three languages: Go, Java, and Python. Compliance needs encrypted service-to-service traffic, and each service must prove its identity. The platform team starts with Istio sidecars.

The first month works. The second month brings the bill. Each pod reserves a proxy. At a planning figure of about 50 MB per sidecar, 600 pods reserve roughly 30 GB of memory for proxies alone, spread across the cluster. Every hop adds latency. Pod startup slows, because the proxy must be ready before the app receives traffic.

The team moves to ambient mode. The result is one ztunnel per node, so 40 proxies instead of 600. Six namespaces need HTTP routing, so six waypoints cover them. The other 134 services keep L4 mTLS and identity policy with no L7 proxy at all.

```text
MIGRATION — same cluster, same 600 pods
══════════════════════════════════════════════════════════════════════════════

 BEFORE  sidecar mode                     AFTER  ambient mode
 ────────────────────────────────────     ──────────────────────────────────────
 600 proxies   (one per pod)              40 ztunnels  (one per node)
 ~30 GB reserved for proxies              6 waypoints  (L7 namespaces only)
 every hop in-path for L7                 L4 path stays in the node
 p99 added: +2 to +5 ms per hop           p99 added: +0.5 to +1 ms (L4 only)
 pod start: proxy gates the app           no proxy in the pod
 cert rotation: 600 identities            node agents + waypoints

 rollout order that worked
 ────────────────────────────────────────
 1. install control plane, ambient profile      (no traffic change)
 2. label one non-critical namespace            (ztunnel takes over its traffic)
 3. verify mTLS: istioctl x describe pod        (peer identity = SPIFFE)
 4. move namespaces one at a time, watch p99    (rollback = remove the label)
 5. delete the sidecar injection webhook        (last step, not first)
```

Step 5 comes last for a reason. Two data planes can run at the same time. Removing the injection webhook first strands every pod that still expects its sidecar.

Then the failure that teaches the most. The team writes an `AuthorizationPolicy` that allows `POST /v1/charge` only for the `checkout` identity. The object applies without error. `kubectl get` shows it as accepted. The traffic test passes, and the policy is wrong: the `billing` service still reaches `/v1/charge`.

```text
THE QUIET FAILURE — L7 policy with no L7 proxy
══════════════════════════════════════════════════════════════════════════════

  AuthorizationPolicy  rules: method POST, path /v1/charge, from checkout
        │
        ▼
  applied to namespace "payments"   ──►  accepted, status: no error
        │
        ▼
  does this namespace have a waypoint?
        │
        ├── YES ──► Envoy waypoint evaluates method + path      policy enforced
        │
        └── NO  ──► ztunnel sees L4 only: src identity, dst port
                    method and path are not visible at L4
                    ─────────────────────────────────────────
                    result: connection ALLOWED, rule silently skipped
```

The ztunnel cannot read an HTTP method. It works on connections. A policy with HTTP fields needs an L7 proxy to evaluate it. The fix is one label on the namespace:

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: payments
  labels:
    istio.io/use-waypoint: waypoint
```

A waypoint pod starts, the policy takes effect, and `billing` now receives a 403 on `/v1/charge`. The lesson is general: **a policy object that applies is not a policy that runs.** After every ambient migration, test one call that must be denied. Do not test only the calls that must pass.

The last piece is portability. The team writes the canary split as a Gateway API `HTTPRoute` bound to a Service, not as an Istio `VirtualService`. Istio, Cilium, and Linkerd all read this API family. The next mesh change becomes a data plane migration, not a rewrite of every route.

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: checkout-canary
  namespace: payments
spec:
  parentRefs:
  - group: ""          # parent is a Service, not a Gateway (GAMMA)
    kind: Service
    name: checkout
    port: 8080
  rules:
  - backendRefs:
    - name: checkout-v1
      port: 8080
      weight: 95
    - name: checkout-v2
      port: 8080
      weight: 5
```

The honest part. The mesh did not fix the retry storm that hit them in week three. Each app still had its own retry count, and the mesh added its own retries on top. Three retries at two layers is nine requests for one user click. They set a retry budget in the mesh and cut the app-level retries to zero for meshed calls. The mesh also does not replace `NetworkPolicy`, and it does not remove the need for idempotent handlers. And at 20 services with one team, a mesh is usually the wrong trade: a gateway, OpenTelemetry, and sane client timeouts cover most of the value at a fraction of the surface.

The punchline: a service mesh is a platform decision, not a library. You pay one more component in the request path. In return, every pod in every language gets the same identity, the same encryption, the same policy, and the same telemetry. That trade is good at 140 services and 40 nodes. It is bad at 12 services and one team. Measure your pod count before you measure the vendor's feature list.

---

day - 4

## ColBERT (Late Interaction Retrieval)

### Definition:

**ColBERT** is a retrieval model that stores one small vector for every token of a
document, not one vector for the whole document. It scores a query with a rule
called **MaxSim**: each query token picks its best matching document token, and
the best scores add up. The mechanism has a name: **late interaction**.

The name answers one question. **When do the query and the document meet?**

- **Never** — a bi-encoder. Encode the query alone. Encode the chunk alone. Then
  compare two vectors with a dot product. The document side is precomputed, so
  search is fast. But the chunk was already squashed before the query existed.
- **At every layer** — a cross-encoder. Feed `[query, chunk]` through one BERT
  stack. Full attention between both texts. Accurate. But you need one forward
  pass per candidate pair, so you can only afford to score a shortlist.
- **At the end** — ColBERT. Encode each side alone. Keep a vector per token.
  Then interact cheaply at scoring time.

ColBERT is the middle path. It keeps the precomputation of a bi-encoder and
recovers much of the precision of a cross-encoder.

**What one pooled vector costs you.** Pooling is a lossy decision made before any
query exists. The model must guess, once and forever, which parts of the chunk
future queries will care about. The guess fails in three places:

1. **Multi-topic chunks.** A section that covers both connection pooling and retry
   policy becomes a blur that sits near neither query.
2. **Exact tokens.** An error code, a function name, or a parameter is a few
   tokens inside hundreds. Its signal averages away. This is the failure that
   keeps BM25 in production.
3. **Out-of-domain text.** An unfamiliar word maps to a region of the space that
   the training data suggested. The whole chunk vector moves with that error.

A better pooling function does not fix this. The problem is pooling itself.

```text
WHAT THE INDEX STORES
one vector per chunk  vs  one vector per token

  SINGLE-VECTOR (bi-encoder)
  ─────────────────────────
    chunk: "Refund gagal, kode E402"
              │
              ▼
      ┌───────────────┐
      │    encoder    │
      └───────┬───────┘
              │
              ▼
       [ 1 x 768 ]       one pooled vector
              │
              ▼
    "E402" is 4 chars inside a pool
    of 180 tokens. It averages away.
    No edge left to match on.


  LATE INTERACTION (ColBERT)
  ──────────────────────────
    chunk: "Refund gagal, kode E402"
              │
              ▼
      ┌───────────────┐
      │    encoder    │
      └───────┬───────┘
              │
              ▼
    [ 180 x 128 ]    one vector per token
              │
              ├── "Refund"  -> [128]
              ├── "gagal"   -> [128]
              ├── "kode"    -> [128]
              ├── "E402"    -> [128]
              └── "timeout" -> [128]

    "E402" keeps its own vector.
    It can win a MaxSim alone.
```

**MaxSim is the whole scoring rule.** It is a procedure, not a formula to
memorise. Take one query token. Compare it against every token vector in the
document. Keep the highest score. Repeat for each query token. Add the winners.

```text
MAXSIM — each query token keeps its best match, then the scores add

  query tokens      refund  gagal  kode   E402  gateway timeout
  ────────────────────────────────────────────────────────────────────
  "refund"           0.91   0.42   0.30   0.11   0.22    0.18
  "gagal"            0.38   0.93   0.35   0.20   0.41    0.29
  "timeout"          0.12   0.28   0.19   0.15   0.44    0.90

  best per row  ->  0.91 + 0.93 + 0.90  =  2.74   this document


WHY THIS BEATS ONE DOT PRODUCT — same query, three documents

  doc A   all three ideas      0.91 + 0.93 + 0.90  = 2.74   WINS
  doc B   loud on two          0.97 + 0.95 + 0.21  = 2.13   loses
  doc C   weak everywhere      0.41 + 0.38 + 0.40  = 1.19   loses

  A pooled dot product ranks B above A: two loud terms
  dominate the pool. MaxSim charges B for the token it
  cannot match. Sum-of-max behaves like an AND, built
  from nothing but max and sum.
```

This is also why late interaction holds up out of domain. A single-vector model
must learn how to **summarise** your passages. A late-interaction model only needs
its token vectors to **align pairwise**. That is a much easier ask on vocabulary
the model never saw.

**The bill arrives as storage.** One vector per token is roughly 180 times more
vectors than one per chunk. Compression decides whether that bill is payable.

```text
STORAGE MATH — 1.2M chunks, 180 tokens each, 128 dimensions

  token vectors            1.2M x 180        = 216,000,000
  raw fp16, 256 B each     216M x 256 B      = 55.3 GB
  ColBERTv2, 20-36 B each  216M x 20..36 B   =  4.3 - 7.8 GB
  one dense vector, 768d   1.2M x 3,072 B    =  3.7 GB
  ─────────────────────────────────────────────────────────
  raw token vectors vs a dense index             15x
  ColBERTv2 vs a dense index                   1.2 - 2.1x

  The 15x, not 180x, is the honest number: each vector
  is 128d, not 768d. After v2 compression the index
  costs about the same as a dense one.
```

**ColBERTv2 (2021) made the bill small.** Two changes:

- **Residual compression.** Cluster all token vectors into centroids. Store the
  centroid id plus a few low-bit residual values. The paper reports a 6–10x
  space cut with quality held.
- **Denoised supervision.** Train with scores from a cross-encoder, not only with
  labelled pairs. The student learns the teacher's ranking, not just its labels.

**PLAID (2022) made the search fast.** It treats each passage as a light bag of
centroids. That bag gives a cheap upper bound. Passages below the bound are
dropped before any token-level work. PLAID reports up to 7x lower latency on a
GPU and 45x on a CPU against vanilla ColBERTv2, at 140M passages, with quality
unchanged.

```text
INDEX ONCE, SEARCH MANY — the shape of the code

index(chunks)
  for chunk in chunks
    toks <- tokenizer(chunk)                    # cut at 180
    vecs <- encoder(toks)                       # [180, 128]
    vecs <- residual_compress(vecs, centroids)  # 20-36 B each
    store(chunk.id, vecs)
  build_ivf_over_centroids()                    # PLAID pruning

search(query, k)
  q     <- encoder(tokenizer(query))[1:]        # drop [CLS]
  cands <- prune_by_centroid_bag(q, 4 * k)      # cheap upper bound
  for cand in cands
    score <- 0
    for qt in q
      score <- score + max(dot(qt, dt) for dt in cand.vecs)
    heap.push(cand.id, score)
  return heap.top(k)                            # exact rescore
```

```text
COST AND LIMIT LEDGER — LATE INTERACTION IS NOT FREE
┌─────────────────────────────────────┬───────────────────────────────────────┐
│ PRO                                 │ CON                                   │
├─────────────────────────────────────┼───────────────────────────────────────┤
│ token match: an error code or a     │ storage: ~180x more vectors than      │
│ function name survives the index    │ one-per-chunk, before compression     │
│ document vectors are precomputed:   │ needs a multi-vector engine: PLAID,   │
│ no forward pass per candidate pair  │ Vespa, Qdrant, Weaviate               │
│ one index serves retrieval AND      │ not a drop-in embedding model: a      │
│ reranking                           │ pooled vector throws the point away   │
│ multi-topic chunks work: every      │ query cost grows with query length,   │
│ query token finds its own home      │ not only with corpus size             │
│ out-of-domain text degrades slowly  │ a cross-encoder still wins on final   │
│ PLAID: tens of ms on GPU at 140M    │ precision over a shortlist            │
│ per-candidate cost far below a      │ new index format, new rebuild cost,   │
│ cross-encoder                       │ new ops surface to own                │
└─────────────────────────────────────┴───────────────────────────────────────┘
```

**What it does not replace.**

- **BM25.** Lexical matching still wins on fresh vocabulary, rare identifiers, and
  text the model never trained on. Run both legs and fuse.
- **A cross-encoder at the top.** If the last 20 candidates must be ordered well,
  a cross-encoder still scores them better. ColBERT is cheaper per candidate, not
  more accurate than full attention.
- **A reranker's job at small scale.** Under about 100k chunks, a pooled vector
  plus a cross-encoder over the top 50 is simpler to run. Late interaction earns
  its keep when the corpus is large and the queries hinge on exact tokens.

### Example:

Consider a support team that runs RAG over 1.2M documentation chunks: API guides,
error-code tables, and changelogs. Traffic is 40,000 queries a day. The baseline
is BM25 plus a pooled 768d vector, fused with reciprocal rank fusion.

The baseline fails on one query class. A user pastes an error code.

```text
THE FAILURE — "why does E402 appear on refund timeout"

  query: "why does E402 appear on refund timeout"
    │
    ├─► BM25 leg
    │     "E402" is a rare term
    │     -> code table page ..................... HIT
    │
    └─► dense leg
          180 tokens pooled,
          a 4-char token averages away
          -> refund overview page ................... MISS

  fused by RRF
    │
    └─► one strong leg, one weak leg
        the code table page lands at rank 9
        below the answer window of 5
        the user sees a generic refund article
        then files a ticket
```

The team adds ColBERT as the **reranker**, not as a replacement. BM25 and the
dense index stay in place to produce 100 candidates. ColBERT rescores them.

```text
AFTER — three stages, each with one job

  query
    │
    ├─► BM25 ........... exact terms, identifiers, fresh text
    ├─► dense 768d ..... paraphrase, semantic recall
    │
    └──────────────┐
                   ▼
    ┌─────────────────────────────┐
    │ fuse (RRF)                  │  -> top 100 candidates
    └──────────────┬──────────────┘
                   ▼
    ┌─────────────────────────────┐
    │ ColBERT MaxSim              │  -> token-level precision
    │ on 100 pairs                │     over precomputed docs
    └──────────────┬──────────────┘
                   ▼
            top 10 to the LLM

  latency per query, modelled from the published PLAID figures
  ─────────────────────────────────────────────────────────
  BM25 + dense recall, 1.2M chunks      18 ms
  ColBERT rescore of 100 candidates     34 ms
  query encode, shared by all pairs      6 ms
  ─────────────────────────────────────────────────────────
  added cost over the baseline          40 ms at p50
  p95, modelled at 2.1x the p50 add     84 ms

  exact-code queries, 900 hand-labelled cases
  ─────────────────────────────────────────────────────────
  baseline    recall@10   0.71        nDCG@10   0.58
  + ColBERT   recall@10   0.89        nDCG@10   0.74
  generic queries, no exact token       +0.02 nDCG
```

The second result matters as much as the first. ColBERT did almost nothing for
generic questions. That is expected: no token needed rescuing. The gain is
concentrated in the query class that was already failing, which is also why the
team could justify the cost with one metric.

**The migration is the hard part, not the model.** Their steps:

```text
1. keep BM25 + dense as the recall stage      (no user change)
2. build the ColBERT index offline            1.2M chunks -> 6.9 GB at 32 B
3. replay 5,000 logged queries, diff ranks    (no serving change yet)
4. turn on rescoring for 5% of traffic        (watch p95, not p50)
5. raise to 100%, keep BM25 + dense for recall
```

Step 3 caught the real problem. Their chunker split error-code tables at 180
tokens, and one table row lost its header. MaxSim scored the row against the
query but could not see the column name. The fix was in the chunker, not the
model. **A better retriever cannot repair a chunk that lost its context.**

**The honest part.** Four limits showed up in production:

- **Query length moves the cost.** MaxSim runs one dot product per
  query-token x document-token pair. A 6-token query is trivial. A pasted 200-token
  stack trace is 30x the work. They cap the query at 32 tokens after the first
  incident.
- **Rebuilds are real.** A new embedding model means a full reindex of 216M
  vectors. The team treats that like a schema migration, with a shadow index and
  a cutover, not like a config flag.
- **Compression is a knob with a quality edge.** 20 B per vector saved space and
  lost a little recall on numeric tables. 32 B was the point where the loss
  disappeared from their eval.
- **The published numbers are not your numbers.** The 7x and 45x figures come
  from PLAID's benchmark. The team measured 3.1x over their own vanilla ColBERT
  baseline, because their candidate set was small and their query encoder was
  already warm.

The numbers above model that workload from the ColBERTv2 and PLAID papers. They
are a worked example, not a public benchmark of a named company.

The punchline: **late interaction moves one decision later.** Not whether the
query and the document interact, but when. Documents stay precomputable, so the
index stays cheap to serve. The interaction stays token-level, so exact words
keep their weight. You pay in storage and in one more engine to operate. Measure
how many of your queries hinge on a specific token before you pay it.

---

day - 5

## Change Data Capture (CDC)

### Definition:

**Change Data Capture (CDC)** moves every data change out of a database as it
happens. The connector does not query the tables. It reads the database's own
change log: the MySQL binlog, the PostgreSQL write-ahead log (WAL), the Oracle
redo log. Every committed INSERT, UPDATE, and DELETE appears there in commit
order. The connector turns each log record into one event and publishes it.

CDC solves one structural problem. Several systems must agree on the same
table data. Re-reading the whole table is slow, late, and blind to deletes.
CDC reads the log once, and every downstream system reads the same stream.

**Three ways to find what changed.**

```text
THREE CAPTURE METHODS — same goal, different source of truth
══════════════════════════════════════════════════════════════════════════════

  method        finds a change by              blind spot          cost to source
  ────────────  ─────────────────────────────  ──────────────────  ──────────────
  polling       SELECT ... WHERE updated_at    deletes             full scans on
                > :last_run                    old rows            every run
                needs a timestamp the app      rows that never     reads live
                writes correctly               get touched         tables

  trigger       a trigger writes each          changes inside      one extra
                change to a side table,        the trigger's own   write inside
                in the same transaction        blind spot          every tx

  log-based     the connector reads the        nothing that        near zero: a
                transaction log itself         committed           stream read
                                               has a log record    of the log
```

Log-based is the default meaning of CDC today. The other two methods still
exist where no log is reachable: managed SaaS databases, or third-party
systems that expose only a table and a timestamp column.

**WITHOUT CDC vs WITH CDC.**

```text
WITHOUT CDC — the batch job guesses           WITH CDC — the log knows
═══════════════════════════════════════════════════════════════════════════════

  app writes ─► SELECT every 15 min           app writes ─► WAL at commit
                 │                                            │
                 ▼                                            ▼
  ┌──────────────────────────────┐            ┌───────────────────────────────┐
  │ WHERE updated_at > :last_run │            │ connector reads the log       │
  │ runs every 15 minutes        │            │ one event per change          │
  └──────────────┬───────────────┘            └───────────────┬───────────────┘
                 ▼                                            ▼
  misses deletes          the row is gone     search · cache · warehouse · services
  misses old rows         no timestamp        shipping · analytics
  spikes the source       big scans
  always 15 min late      app's timestamp
```

**One event, one envelope.** The event carries the old row and the new row.
Downstream code picks what it needs.

```text
THE ENVELOPE — one change event as it lands in Kafka
══════════════════════════════════════════════════════════════════════════════

  {
    "op":     "u",                         c = create, u = update,
                                           d = delete, r = snapshot read
    "before": { "id": 40, "qty": 1 },      old row (null on create)
    "after":  { "id": 40, "qty": 2 },      new row (null on delete)
    "source": {
      "db":     "shop",
      "table":  "orders",
      "lsn":    "0/16B3A7C8",              position in the log = the cursor
      "txid":   88421,                     the transaction it belongs to
      "ts_ms":  1762348800123
    }
  }
```

**Ordering: one key is safe, two keys are not.** The log has a global commit
order. A streaming pipeline usually splits rows across partitions by primary
key. Order survives inside a partition. Order does not survive across them.

```text
ORDERING GUARANTEE — what CDC promises, and what it does not
══════════════════════════════════════════════════════════════════════════════

  same key  →  same partition  →  order kept
  ─────────────────────────────────────────────────────────────────────────
    commits on orders id=40             events for id=40
      UPDATE qty = 2    ─┐
      UPDATE qty = 5     ├──►     ┌──────────────────────┐
      DELETE            ─┘        │ partition 3          │  2 → 5 ✓
                                  │ key = id % N         │  then gone ✓
                                  └──────────────────────┘

  different keys  →  maybe different partitions  →  no global order
  ─────────────────────────────────────────────────────────────────────────
      orders id=40  →  partition 3      (qty 2, then 5)
      orders id=41  →  partition 7      (qty 9)
      "did id=40 change before or after id=41?"   the pipeline cannot say
      one transaction spanning two tables lands in two topics
      a consumer that needs atomicity must use txid to group the events
```

**Delivery: at-least-once by default.** The connector saves its log position
(offset) after it publishes. If it crashes in between, it replays the last
events. Duplicates are normal. A consumer that counts on "exactly once"
without doing work returns wrong results after a replay.

```text
DELIVERY SEMANTICS — where duplicates come from
══════════════════════════════════════════════════════════════════════════════

  connector                     broker                      consumer
  ─────────                     ──────                      ────────
  read event (lsn X)  ──────►   publish event
  save offset X       ──┐
                        │ CRASH HERE (before offset saved)
                        ▼
  restart at saved offset X-1   ──►  same event published again

  fix that works: the consumer is idempotent
    key          = primary key of the row
    dedupe rule  = keep the event with the highest LSN per key
    result       = duplicates become invisible (effective exactly-once)
```

**The bill and the limits.** CDC is cheap to the source, but it is not free.

```text
COST AND LIMIT LEDGER — LOG-BASED CDC
┌──────────────────────────────────┬────────────────────────────────────────────┐
│ PRO                              │ CON                                        │
├──────────────────────────────────┼────────────────────────────────────────────┤
│ near-zero load on the source     │ a stopped connector keeps its replication  │
│ (a stream read, not a scan)      │ slot: PostgreSQL WAL grows until the disk  │
│ sees deletes and old rows        │ fills. Monitor slot lag, or lose the node  │
│ commit order, sub-second lag     │ DDL does not travel: logical replication   │
│                                  │ does not carry ALTER TABLE statements      │
│ the app writes to one system     │ one transaction spanning two tables may    │
│ only: the database               │ arrive on two topics, split and reordered  │
│ downstream reads the same log,   │ downstream must be idempotent: replay is   │
│ so every consumer agrees         │ a feature, not an error                    │
│ no dual-write problem: no app    │ snapshot + backfill of huge tables costs   │
│ code writes to DB and bus twice  │ and it runs against production data        │
│ works on any schema: row images  │ you now own a pipeline: connector, broker, │
│ need no app instrumentation      │ schema registry, offsets, monitoring       │
└──────────────────────────────────┴────────────────────────────────────────────┘
```

**Snapshot and stream must join without a gap.** On first run, the table has
history the log does not. The connector takes a snapshot, remembers the log
position (LSN) where the snapshot started, then reads forward from that same
LSN. Rows changed during the snapshot arrive again from the log. The
consumer's idempotent dedupe rule makes the second copy harmless.

```text
INITIAL SNAPSHOT → STREAM — the join point is one LSN
══════════════════════════════════════════════════════════════════════════════

  table (200M rows)                 transaction log
  ┌──────────────────────┐          ──────────────────────────────────►
  │ snapshot read in     │          LSN X ──► new changes stream from here
  │ chunks, low priority │              ▲
  └──────────┬───────────┘              │
             │  records LSN X           │
             └──────────────────────────┘
  snapshot ends                     stream starts at the same LSN: no gap, no overlap

  rows changed DURING the snapshot:
    snapshot read them at old state  ─┐
    log replays them at new state    ─┴─► consumer keeps highest LSN per key
```

**CDC vs the transactional outbox pattern.** Both fix the same enemy: the
dual-write problem, where one request must update a database and publish an
event, and only one of the two can succeed.

```text
DUAL WRITE — the bug both patterns remove
══════════════════════════════════════════════════════════════════════════════

  request handler
       │
       ├──── write to DB   ─── OK
       └──── publish event ─── FAILED (broker down, timeout)
       result: the row exists. No event exists. The systems disagree.
       Nothing in the code records that it happened.

  fix A  transactional outbox   the app writes the event row inside the same
                                transaction. A relay publishes it later
                                → you control the event: names, not row images
  fix B  CDC                    the app writes only to the DB. The connector
                                reads the committed log and publishes it
                                → zero app code. You get row images, not names

  choose outbox when you need domain events or no log is reachable
  choose CDC when row state is enough and the database already logs it
```

**Production traps, in the order they hit teams.**

```text
TRAP LEDGER — what actually breaks
══════════════════════════════════════════════════════════════════════════════

  1. slot left behind      connector stops over the weekend.
                           PostgreSQL keeps the WAL for the slot.
                           The disk grows from 12 GB to 400+ GB in 3 days.
                           fix: alert on replication slot lag and
                           WAL disk usage, not only on connector status

  2. DDL during streaming  logical replication does not carry DDL.
                           RENAME or type change breaks the consumer
                           while the connector reports "healthy"
                           fix: schema registry with compatibility rules
                           (BACKWARD) + freeze DDL during cutover windows

  3. duplicate handling    broker retries, connector restarts, and
                           re-snapshots all replay events
                           fix: dedupe on (primary key, LSN) in the
                           consumer, before anything else writes

  4. full table as one     a 200M-row snapshot holds resources for hours
     snapshot              fix: chunked snapshot at low priority,
                           start streaming from the snapshot's LSN first

  5. trust in "the          CDC moves rows, not intent: a correction, a
     latest row"           merge, and a revert can look identical downstream
                           fix: keep op + LSN + txid in the event, and let
                           the consumer replay history instead of guessing
```

The honest part. CDC copies row state. It does not copy transactions as one
atomic unit across topics, it does not carry DDL, and it does not replace
application-level events when the meaning matters more than the row image.
A pipeline that reads the log still needs offsets, idempotent consumers,
schema rules, and alerts on lag. Treat CDC as the cheapest correct way to
keep secondary systems in sync with a database, not as a complete
integration strategy.

The punchline: CDC moves one decision from "query the table again" to "read
the log once". The database already wrote every change down. Reading that
record gives all downstream systems the same ordered history, at near-zero
cost to the source. The work moves to the consumer: idempotent writes,
schema discipline, and lag alerts.

### Example:

A shop backend runs PostgreSQL. One order row must reach four systems:
the search index, the read cache, the analytics warehouse, and the
shipping service. The team first tried direct writes: the order service
updated the database, then published an event, then updated the cache.

```text
BEFORE — three writers, one truth, no agreement
══════════════════════════════════════════════════════════════════════════════

  order service (one HTTP request)
       │
       ├──── 1. UPDATE orders        OK
       ├──── 2. publish event        FAILED  ← broker timeout, retry lost
       ├──── 3. SET cache key        OK      ← cache now newer than the event
       │
       └──── the systems now disagree:
             search never learns the order exists
             warehouse misses the row until the nightly job
             the cache disagrees with the database for 6 hours
```

The team removes steps 2 and 3 from the request path. The service now writes
only to PostgreSQL. A Debezium connector reads the WAL. Kafka carries the
events. Each downstream system owns one consumer and one job.

```text
AFTER — one writer, one log, four consumers
══════════════════════════════════════════════════════════════════════════════

  order service          PostgreSQL              Debezium connector
       │                     │                            │
       └── UPDATE orders ──► │ commit → WAL record ─────► │ reads WAL stream
                             │                            │ snapshot first,
                             │                            │ then stream (LSN)
                             │                            ▼
                             │                     ┌──────────────┐
                             │                     │ Kafka topics │
                             │                     │ orders (key  │
                             │                     │ = id)        │
                             │                     └──────┬───────┘
                             │                            │
                             │                            ├──► search indexer    (bulk upsert)
                             │                            ├──► cache invalidator (delete on d,
                             │                            │                      upsert on c/u)
                             │                            ├──► shipping service  (consume events,
                             │                            │                      idempotent)
                             │                            └──► analytics sink    (read the same topic)
```

**The join point works the first hour.** The `orders` table holds 40 million
rows. The connector snapshots them in chunks, records LSN `0/16B3A7C8`, and
streams forward from there. During the snapshot a customer places order
`#40,112,001`. The snapshot reads the table before that insert commits. The
log replays the insert right after. The search indexer receives the row
twice. Its dedupe rule keeps the highest LSN per id, so the index shows the
row once.

```text
WEEK ONE IN PRODUCTION — three incidents, three lessons
══════════════════════════════════════════════════════════════════════════════

  MON  the platform team stops the connector for a schema change
       and forgets to restart it.
       ────────────────────────────────────────────────────────────────
       Postgres keeps the replication slot. The WAL grows from 12 GB
       to 400+ GB in 3 days. An alert on WAL disk usage fires before
       the disk fills. Alert list after the fix:
         replication slot lag      > 5 min
         WAL disk usage            > 60% of volume
         consumer lag per topic    > 100k messages
       lesson: alert on what the slot holds, not on "connector: running"

  WED  a migration renames orders.total to orders.grand_total.
       ────────────────────────────────────────────────────────────────
       Logical replication does not carry DDL. The connector keeps
       running. The warehouse consumer starts writing NULL totals.
       The search index keeps the old field name. Nothing fails loudly.
       fix: schema registry with BACKWARD compatibility rejects the
       rename at registration time. A DDL freeze window covers cutover
       lesson: "the pipeline has no errors" and "the data is right" are two
               different claims

  FRI  one transaction updates orders and shipments together.
       ────────────────────────────────────────────────────────────────
       the two rows land in two topics. The shipping consumer sees
       the shipment row before the order row. Its handler reads the order
       from the database, gets the pre-transaction state, and ships to the
       old address
       fix: group events by txid in the consumer, or key both tables
       into one topic when they must move together
       lesson: per-key order is guaranteed. Cross-table atomicity is not
```

The rollout that worked, in order:

```text
ROLLOUT — five steps, each one reversible
══════════════════════════════════════════════════════════════════════════════

  1. enable logical replication on Postgres        no traffic change
  2. start the connector with a limited snapshot   orders only, chunks on
  3. replay 7 days of logged queries against the   compare row counts and
     new topics, diff the results                  spot the diff, no users
  4. move one consumer at a time to the topics     rollback = point it back
  5. remove the old direct writes from the app     last step, not first
```

Step 5 comes last for a reason. Two sources can run at the same time. The
old direct publish and the new log-based publish both write to Kafka. The
consumer's idempotent dedupe rule hides the overlap. Removing the old path
first would leave a window where no path exists.

The honest part. The nightly job still runs. CDC feeds the warehouse in
seconds, but the team keeps one daily reconciliation query that counts rows
in the source and in the warehouse. In the first month it found one real
gap: a table without a primary key. Log-based CDC cannot split that table
into stable partitions, so its events landed unordered. The fix was a
primary key, not a bigger connector. The team also learned the cost they
did not budget: one connector, one broker, one schema registry, and a
person who reads lag charts. The source database stayed nearly untouched
through all of it, which was the point.

The punchline: the database already records every change in commit order.
CDC turns that record into the shared history of every system downstream.
You still owe idempotent consumers, schema rules, and lag alerts. You stop
polling tables, you stop missing deletes, and the request path stops
writing to three places at once.

---

day - 6

## GitOps

### Definition:

**GitOps** is a way to run a system from a Git repository. Git holds the
desired state. The system holds the live state. A controller runs inside
the system. The controller compares both states. It applies the difference.
It repeats this comparison in a loop, not only at deploy time.

GitOps follows four rules:

1. Git holds the desired state. Git is the single source of truth.
2. A controller inside the system pulls state from Git. Nothing pushes.
3. The controller detects drift. It can also fix drift.
4. Every change starts as a Git change: a merge, a tag, a revert.

The key difference from classic CI/CD is who acts on the cluster.

PUSH MODEL (classic CI/CD)       PULL MODEL (GitOps)
─────────────────────────────    ────────────────────────────────
developer                        developer
   │                                │
   ▼                                ▼
git push                         git push
   │                                │
   ▼                                ▼
┌──┬────────────────────────┐    ┌──┬─────────────────┐
│ CI pipeline               │    │ Git repository     │
│ build, test, sign image   │    │ = desired state    │
│ knows cluster credentials │    │ holds full history │
└──┴────────────────────────┘    └──┴─────────────────┘
   │                                │
   ▼                                ▼
kubectl apply ──────► cluster    ┌──┬───────────────────────────┐
   │                             │ controller in cluster        │
   ▼                             │ (Argo CD / Flux)             │
pipeline ends here               │ reads Git, reads live state  │
nobody watches live state        │ holds no cluster credentials │
drift stays until next deploy    └──┴───────────────────────────┘
rollback = rerun old pipeline       │
                                    ▼
                                 cluster follows Git
                                 drift found within minutes
                                 drift fixed by the loop
                                 rollback = git revert

Pull mode moves authority. The pipeline builds and tests the image. It
never touches the cluster. The controller inside the cluster does the
work. This change has three effects:

- **Credentials stay narrow.** CI needs registry access. The controller
  needs cluster access. A stolen CI token cannot reach production.
- **History is one place.** `git log` shows who changed what and when.
  Cluster logs show only that a controller applied a manifest.
- **Rollback is a Git operation.** Revert the commit. The controller
  applies the old state on the next cycle.

The controller runs a reconcile loop. The loop never stops.

   ┌──────────────────────────────────────────────────┐
   │ RECONCILE LOOP — runs inside the system, forever │
   │ step 1: read desired state   (Git commit SHA)    │
   │ step 2: read live state      (cluster objects)   │
   │ step 3: diff the two states                      │
   └─────────────────────────┴────────────────────────┘
                             │
             ┌───────────────┴───────────────┐
             ▼                               ▼
       states match                    states differ
             │                               │
             │  status: InSync               ▼
                              ┌──────────────┬───────────────────────────────┐
                              │ sync        apply the Git state              │
                              │ self-heal   revert manual edits              │
                              │ prune       delete objects Git removed       │
                              │ alert       report failure, retry next cycle │
                              └──────────────┴───────────────────────────────┘
             │                               │
             └───────────────┴───────────────┘
                             │
                             ▼
                   wait, then run again
        Argo CD: every 120 s (+ up to 60 s jitter)
             Flux: your interval, 60 s or more

Two decisions control the loop: **when** it runs and **what it fixes**.

```text
TOOL      GIT POLL         AUTO-SYNC       SELF-HEAL       PRUNE
Argo CD   120 s + jitter   opt-in          opt-in          opt-in
Flux      your interval    on by default   on by default   explicit

Defaults read from docs of Argo CD v3.5.6 and Flux v2.10.0 (Oct 2026).
```

Argo CD reports drift until you enable `syncPolicy.automated`. This is
safe for change-controlled production. Flux corrects drift every cycle
by default. That is strict, and it surprises new teams.

**Drift is not failure. Sync and health are two different signals.**

  ┌───────────────────────────────────────────────────┐
  │ Synced     live state == Git state                │
  │ OutOfSync  live state != Git state                │
  │                                                   │
  │ Healthy    pods ready, traffic fine               │
  │ Degraded   pods failing                           │
  │                                                   │
  │ OutOfSync + Healthy = someone edited the cluster. │
  │ Synced + Degraded    = Git itself is wrong.       │
  └───────────────────────────────────────────────────┘

Because of that split, you must decide who owns each field. A field
with two owners causes endless drift alerts. The ownership map comes
first, before you enable self-heal.

```text
FIELD                          OWNER              GITOPS ACTION
image tag, env, limits         Git (intent)       keep in Git, diff every cycle
uid, resourceVersion, status   the cluster        ignore, never alert
spec.replicas                  HPA / autoscaler   add an ignore rule
applied manifest               the controller     alert on controller failure
```

Three traps appear in real use:

- **Replicas fight.** Git holds 3 replicas. The HPA raises it to 12.
  The loop resets it to 3 every cycle. Fix: remove `spec.replicas`
  from Git, or add an ignore rule for that path.
- **Two controllers, one object.** Argo CD and Flux both manage the
  same Deployment in the same namespace. Each applies on its own
  schedule. The state oscillates and the diff never settles. Rule:
  one owner per object per namespace.
- **Secrets in Git.** Plain text in Git is a leak. Encrypt the file
  (SOPS, sealed secrets), or store only a reference in Git.

When you must break the loop — for an emergency edit — pause the
controller instead of fighting it. Argo CD 3.4 added a cluster-level
pause: `argocd cluster pause production-cluster`. The loop runs again
after `argocd cluster resume`. Flux has `flux suspend kustomization NAME`.

### Example:

A payments team runs checkout-api on one production cluster. Argo CD
manages it. The desired state lives in a Git repository. This is a
worked scenario, not a named company.

```text
  TIME   EVENT                                 SIGNAL
  t0     merge PR #482: image 1.5.0 -> 1.5.1   Git holds the new state
  t1     Argo CD polls the repo (<= 120 s)     OutOfSync / Healthy
  t2     sync runs a rolling update            live = 1.5.1, Synced
  t3     02:41 on-call edits the Deployment    live != Git, OutOfSync
  t4     next cycle runs with self-heal on     edit reverted in <= 120 s
  t5     on-call pauses, fixes Git, resumes    Git holds the hotfix
```

Step t3 is the lesson. The engineer applies an emergency fix with
`kubectl set image`. It works for two minutes. The next cycle reverts
it, because Git does not know it. The fix vanishes during an incident.

The correct path keeps Git as the only writer:

```text
1. pause     argocd cluster pause production-cluster
2. apply     kubectl set image ... (hotfix now, incident stops)
3. commit    backport the same change to Git, merge the PR
4. resume    argocd cluster resume production-cluster
5. verify    app shows Synced, and git log shows who and why
```

The incident exposed two gaps. The team fixes both:

```text
BEFORE (push, manual)                   AFTER (GitOps)
────────────────────────────────────    ──────────────────────────────────
hotfix: kubectl on prod only            hotfix: PR + merge to Git
audit:  cluster logs, partial           audit:  git log, complete
drift:  found at next incident          drift:  found in <= 120 s
drift:  ignored, nobody watches         drift:  alerted and reverted
rollback: rerun old pipeline            rollback: git revert
access:  CI holds prod cluster token    access:  CI holds no cluster token
```

The migration cost is small, and the order matters:

```text
1. declare state in Git          (one repo per environment)
2. install the controller        (read-only first)
3. run diff mode for a week      (no auto-sync, read the reports)
4. set the ownership map         (replicas, status, generated fields)
5. enable auto-sync + self-heal  (dev, then staging, then prod)
6. enable prune                  (after diff mode looks clean)
7. add alerts on controller health, not on every retry
```

Step 3 catches most problems before they touch production. Step 4
prevents the replica fight. Step 7 keeps alerts useful.

GitOps does not replace CI. CI builds, tests, and signs the artifact.
GitOps decides what runs where. The boundary is simple: **CI produces
the artifact. Git states the intent. The controller makes reality match
intent.** If reality and intent differ, the loop reports it. The team
then fixes Git, not the cluster.

---

day - 9

## KV Cache Compression

### Definition:

**KV cache compression** makes the key-value cache of an LLM server smaller.
The server keeps this cache in GPU memory while it generates text. The cache
grows with every token of every session. When it fills the GPU, the server
stops accepting sessions.

Start with the cache itself. Attention compares a new token with every earlier
token. The model stores the key (K) and the value (V) of each past token, in
each layer. Each decoding step reads them. This reuse avoids a full recompute
of the prefix.

WHY THE CACHE EXISTS — compute once, store, reuse
────────────────────────────────────────────────────────────────────────────────

  prompt tokens   t1   t2   t3  ......  t32768
                    │
                    ▼  prefill: one forward pass, compute-bound
  ┌──────────────────────────────────────────┐
  │   layer L stores K and V for every token │
  │     t1     ->  k1      v1                │
  │     t2     ->  k2      v2                │
  │     ......                               │
  │     t32768 ->  k32768  v32768            │
  └──────────────────────────────────────────┘
                    │
                    ▼  decode step 32769, memory-bound
  q32769 attends to k1 .. k32768        no prefix recomputed
  the step appends k32769 and v32769    cache grows by 1 K + 1 V per layer
                    │
                    ▼
  every new token of every session adds to the same GPU bill

The cache size follows one product. One formula gives it.

SIZE FORMULA — one product decides the bill
══════════════════════════════════════════════════════════════════════════════

  bytes = 2 x layers x kv_heads x head_dim x seq_len x batch x bytes_per_value
          ^^ K and V are both stored      fp16 = 2 bytes per value

  worked numbers: 7B chat model, fp16, multi-head attention
  ────────────────────────────────────────────────────────────────────────────
  layers 32    kv_heads 32    head_dim 128

  per token        2 x 32 x 32 x 128            = 262,144 B   = 0.262 MB
  one 32K session  0.262 MB x 32,768            = 8.6 GB
  model weights    7B params x 2 bytes          = 14 GB

  same model with grouped-query attention (8 KV heads)
  per token        2 x 32 x 8 x 128             = 65,536 B    = 0.066 MB
  one 32K session  0.066 MB x 32,768            = 2.15 GB     (4x smaller)

The same 80 GB card, two cache settings:

SAME GPU, SAME MODEL — only the stored cache differs
────────────────────────────────────────────────────────────────────────────────
┌────────────────────────────┐   ┌─────────────────────────────────────────┐
│ NO COMPRESSION (MHA, fp16) │   │ WITH COMPRESSION (GQA + SnapKV + 2-bit) │
│                            │   │                                         │
│ weights            14 GB   │   │ weights            14 GB                │
│ KV per 32K session 8.6 GB  │   │ KV per 32K session 0.03 GB              │
│ sessions at 32K    6       │   │ sessions at 32K    200+                 │
│ limit at 32K       the GPU │   │ limit at 32K       compute-bound        │
└────────────────────────────┘   └─────────────────────────────────────────┘

Compression moves one number in that formula, or removes tokens from it. Four
moves exist. They cut at different places.

FOUR MOVES — same goal, different place to cut
────────────────────────────────────────────────────────────────────────────────

MOVE       WHAT IT DOES                            WHAT IT COSTS
────       ────────────────────────────────────────────────────────────────
evict      drop low-importance tokens              long-context recall
quantize   store fewer bits per value              accuracy at the tail
low-rank   project K,V through small matrices      one extra matmul in path
architect  build a small cache: MQA, GQA, MLA      model choice at training

**Eviction trades recall for space.** H2O (NeurIPS 2023) keeps recent tokens
plus the few "heavy hitter" tokens that collect most attention mass.
StreamingLLM keeps the first tokens of the sequence, called attention sinks,
plus a recent window. SnapKV votes on prefix positions with a small
observation window at the end of the prompt. PyramidInfer gives each layer its
own budget and reports over 54% less KV memory and 2.2x throughput against
Hugging Face Accelerate. Eviction needs no retraining. The cost shows up as
lost recall on long prompts.

**Quantization stores fewer bits per value.** Keys and values behave
differently, so good schemes treat them differently. KIVI quantizes keys per
channel and values per token at 2 bits. It reports 2.6x lower combined peak
memory (weights plus cache), up to 4x larger batches, and 2.35x to 3.47x
throughput on real workloads. KVQuant calibrates the quantization grid and
separates outliers, and was evaluated up to a 10 million token context.
TurboQuant reports at least 6x memory reduction and up to 8x faster attention
on H100 GPUs at 3 bits. Low-rank methods (Palu, LoRC) project K and V through
small matrices instead.

**Architecture builds the cache small from the start.** MQA gives all query
heads one KV head. GQA shares KV heads across groups of query heads: Llama-3
70B uses 64 query heads and 8 KV heads, so its cache is 8x smaller than
multi-head attention at the same quality target. MLA (DeepSeek-V2) stores one
latent vector instead of full K and V, and reports 93.3% less KV cache than
its multi-head predecessor. These moves cost nothing at runtime. They are a
model choice, so you cannot add them to a model you already run.

Compression has a few neighbours that solve different problems. Keep them
apart.

EVICTING BREAKS SHARED PREFIXES — a trap, not a bug
══════════════════════════════════════════════════════════════════════════════

  request A   system prompt (3,000 tokens)  ─┐
  request B   system prompt (3,000 tokens)  ─┴─►  prefix cache reuses it
                                                   (prefill runs once)

  budget pressure: eviction drops tokens 1..2,500 of the shared range
        │
        ▼
  request C carries the same system prompt
        │
        ├─► cached range is incomplete  ->  prefill recomputes 3,000 tokens
        └─► latency spike on the exact traffic you wanted to make cheaper

  rule: pin the shared prefix, evict only the private tail of each session

One more neighbour: **PagedAttention** (vLLM) removes memory waste from
fragmented allocation. It does not shrink the cache. A paged cache of 8 GB is
still 8 GB. Paging and compression stack: paging wastes less of what is
allocated, compression allocates less.

The honest limits. Compression is lossy, and the loss hides in the tail of the
distribution. Average benchmark scores barely move. A fact buried at token
20,000 moves. Three rules keep you honest:

1. Test the long tail, not the average. A needle-in-a-haystack set at your
   real context length is the smallest useful test.
2. Pin shared prefixes. Eviction can drop tokens from a system prompt that
   many requests reuse. The next request then pays prefill again.
3. Stop when memory stops being the limit. Past a point, batch size follows
   compute and scheduling. More compression buys nothing there.

### Example:

A team runs a customer-support assistant on one H100 (80 GB). The model has
7B parameters, fp16 weights, 32 layers, 32 KV heads, head_dim 128. Context is
32K tokens. Sunday traffic is light. Monday morning brings twelve chats at
once.

SESSION LEDGER — 7B model, 32K per session, weights 14 GB, activations 6 GB
══════════════════════════════════════════════════════════════════════════════

  SESSION  KV CACHE   RUNNING TOTAL   FREE ON 80 GB     RESULT
  ──────────────────────────────────────────────────────────────────────────
  #1       8.6 GB     8.6 GB          51.4 GB           accepted
  #2       8.6 GB     17.2 GB         42.8 GB           accepted
  #3       8.6 GB     25.8 GB         34.2 GB           accepted
  #4       8.6 GB     34.4 GB         25.6 GB           accepted
  #5       8.6 GB     43 GB           17 GB             accepted
  #6       8.6 GB     51.6 GB         8.4 GB            accepted
  #7       8.6 GB     60.2 GB         -0.2 GB           CUDA out of memory
  ──────────────────────────────────────────────────────────────────────────
  session 7 needs 80.2 GB of GPU. The GPU has 80 GB. The weights never
  changed.

The service rejects requests for two minutes until chats close. The team
reads the numbers and picks the cheapest fixes first, in order.

FIX LADDER — each step is independent and reversible
────────────────────────────────────────────────────────────────────────────────

  STEP  CHANGE                                     KV PER SESSION  SESSIONS
  ──────────────────────────────────────────────────────────────────────────────
  0     baseline: multi-head attention, fp16       8.6 GB          6
  1     swap to a grouped-query model (8 KV heads) 2.15 GB         27
  2     enable SnapKV, budget 4K tokens past 8K    0.27 GB         200+
  3     add KIVI 2-bit on the retained tokens      0.03 GB         compute-bound
  ──────────────────────────────────────────────────────────────────────────────
  past step 3, memory stops being the limit. Batch size then follows
  compute and scheduling, not gigabytes.

Step 1 needs no runtime flag. The team moves to a grouped-query checkpoint and
re-runs the same ledger: 2.15 GB per session, 27 sessions inside the same
budget. Step 2 turns on SnapKV only for sessions that pass 8K tokens, so
short chats keep a full cache. Step 3 adds 2-bit quantization on the retained
tokens.

Then the validation that matters. The team builds a prompt set with an order
id buried at position 20,000 and asks for it.

THE TEST THAT DECIDES IT — a needle at token 20,000
────────────────────────────────────────────────────────────────────────────────

  prompt (28,000 tokens)
  [ system | docs ... A-7734 at position 20,000 ... tail ]
                                       │
  question: "what is order id A-7734?"
                                       │
                           ┌───────────┴───────────┐
                           ▼                       ▼
                   flat eviction, 4K budget          sinks + window + SnapKV
                   position 20,000 dropped           kept by attention sinks
                           │                       │
                           ▼                       ▼
                   answer: not found                 answer: A-7734

  short prompts never show this. test the long tail before you ship.

Flat eviction fails the test. The sinks-and-SnapKV combination passes it. The
team keeps the second configuration and adds the needle set to CI, so a future
budget change fails the build instead of a customer.

What this scenario does not claim. The ledger figures come from the size
formula above, not from a published benchmark of a named company. The
eviction results illustrate the failure mode. Your model, prompt mix, and
budget move the numbers. The team still runs a full re-index of eval prompts
whenever the budget changes, and they keep one baseline run without any
compression, so they can measure the loss instead of assuming it is small.

The punchline: the KV cache is the price of not recomputing the past.
Compression lowers that price. It never removes it. Every token you drop or
shrink is a fact the model can no longer read at full precision. Pay the
smallest price your long-context eval can prove is safe, and re-measure it
when the context grows.

---

day - 10

## Trunk-Based Development

### Definition:

**Trunk-Based Development** is a branching model. Developers merge small
changes into one long-lived branch called trunk (usually `main`). Every
change lands within hours. Incomplete work hides behind feature flags. The
trunk stays releasable at every commit.

The core rule is simple: no branch lives longer than a day. A developer
starts work in the morning and merges before going home. Review happens
before the merge. CI runs on every merge. If a change is too big, the
developer splits it into small steps.

WITHOUT vs WITH — when the team sees the change
──────────────────────────────────────────────────────────────────────────────

┌──────────────────────────────────┐      ┌──────────────────────────────────┐
│ WITHOUT: long-lived branch       │      │ WITH: trunk + flags              │
│                                  │      │                                  │
│ start work on a branch           │      │ start work                       │
│ commit for 1 - 3 weeks           │      │ merge a small change, same day   │
│ merge to main at the end         │      │ CI runs the full test suite      │
│ fix conflicts under time         │      │ a flag hides incomplete code     │
│ pressure, then release           │      │ release any day                  │
└──────────────────────────────────┘      └──────────────────────────────────┘

BRANCH LIFETIME — the same feature in both models
──────────────────────────────────────────────────────────────────────────────
  long branch   ●═════════════════════════════════════════════════════════●
                day 1                                                day 14
                team sees the change once: at the merge

  trunk         ●─●─●─●─●─●─●─●─●─●─●─●─●─●
                one merge per day, every diff stays small

Three practices make this work together. Remove any one of them and the
model fails.

 1. SMALL CHANGES. One task = one merge. Big changes split into steps.
    Each step keeps the system green.
 2. FEATURE FLAGS. New code merges into trunk while it is still
    incomplete. A flag turns the code on for users. A flag also turns it
    off in one second when a defect appears.
 3. STRONG CI. Every merge runs the full test suite and deploys to a
    test environment. A red build blocks the next merge.

Why teams pick this model. Long-lived branches hide work and hide risk.
The team sees the change only at the merge. By then the code is weeks old
and other people have moved the same files. Trunk-Based moves the pain
forward: you fix a small conflict on day one instead of a huge conflict
on day thirty. It also supports continuous delivery, because the trunk
can ship any day.

The cost is real, so keep the trade-offs honest. Every developer needs
the whole test suite green before merge, so slow CI hurts. Feature flags
create debt: a forgotten flag rots and confuses the next reader. Teams
schedule flag clean-up work and treat an old flag as a defect. Small
teams feel this cost less. Large teams feel it more, and they buy speed
with automation.

A quick way to know where you stand: measure the age of your branches.
Branches that live for hours fit this model. Branches that live for
weeks do not.

### Example:

Scenario: a payment team rebuilds the checkout page. The work takes two
weeks. The team ships it with one flag, `checkout_v2`, set to OFF at
first merge.

FLOW — two weeks, zero long-lived branches
──────────────────────────────────────────────────────────────────────────────

  time     action on trunk                         flag state
  ─────    ─────────────────────────────────────   ──────────
  day 1    dev1: new page skeleton, merge          OFF
           CI: 412 tests pass, staging deploy
  day 1    dev2: cart widget, merge                OFF
           CI: 418 tests pass
  day 2    dev3: payment form, merge               OFF
           CI: 420 tests pass
  ─────    ─────────────────────────────────────   ──────────
  day 6    team review on staging, found 3 bugs    OFF
           dev4 fixes each bug in one merge
  ─────    ─────────────────────────────────────   ──────────
  day 8    enable flag for 5% of users             ON 5%
           watch: error rate flat, conversion flat
  day 9    raise to 50%                            ON 50%
  day 10   raise to 100%                           ON 100%
  ─────    ─────────────────────────────────────   ──────────
  day 12   delete the flag and the old page        (removed)
           dead code leaves the codebase

Compare the alternative. In a branch model, the same two weeks look like
this:

  main     ●───────────────────────────────────────────────────────●
           │ ╲                                                     │
           │  ╲                                                    │
           │   ●═══════════════════════════════════════════════════●
           │      all conflicts and hotfixes land on day 14

In the trunk model the team sees the change every day. The risky merge
disappears. The risky part becomes the flag rollout, and that part has an
OFF switch. That is the whole point: merge small, ship dark, turn on
slowly, remove the flag fast.

---

day - 11

## Eval-Driven Development (EDD)

### Definition:

**Eval-Driven Development (EDD)** is a way to build LLM applications. You
first write down what "good" behavior looks like in numbers. Then you
measure every change against those numbers. You keep a change only when
its score goes up or stays equal.

An eval is a test case for a model. It has an input, a rule that checks
the output, and a pass or fail result. The rule can be a string check, a
program assertion, or a second model that grades the answer. A set of
evals forms a scoreboard for your agent.

EDD behaves like Test Driven Development, but for probabilistic output.
A unit test has one right answer. An eval grades an answer that varies
between runs. So EDD compares scores of whole runs instead of asserting
one exact string.

WITHOUT EDD vs WITH EDD — how a change earns trust
──────────────────────────────────────────────────────────────────────────────

┌────────────────────────────────┐    ┌────────────────────────────────────┐
│ WITHOUT: demo-driven           │    │ WITH: eval-driven                  │
│                                │    │                                    │
│ change the prompt              │    │ write the change                   │
│ try 3 examples by hand         │    │ run the full eval set (200+ cases) │
│ "looks good"                   │    │ score = pass rate + assertions     │
│ merge to main                  │    │ compare with saved baseline        │
│                                │    │                                    │
│ weeks later: a regression      │    │ score drops 92% -> 88%             │
│ nobody knows which change      │    │ block the merge, find the case     │
│ debug guesswork, days          │    │ fix or revert, hours               │
└────────────────────────────────┘    └────────────────────────────────────┘

The core of EDD is one short loop. You run it after every change.

THE EDD LOOP — every change passes through the same five steps
──────────────────────────────────────────────────────────────────────────────

  ┌──────────┐    ┌──────────┐    ┌──────────┐    ┌──────────┐    ┌────────────┐
  │   SPEC   │───▶│ DATASET  │───▶│   RUN    │───▶│  SCORE   │───▶│  COMPARE   │
  │  "good"  │    │  inputs  │    │  agent   │    │  pass /  │    │  vs saved  │
  │ defined  │    │ + rules  │    │  on all  │    │   fail   │    │  baseline  │
  └──────────┘    └──────────┘    └──────────┘    └──────────┘    └──────┬─────┘
        ▲                                                                │
        │ score >= baseline ?  yes -> merge        no: fix or revert     │
        └────────────────────────────────────────────────────────────────┘

Four pieces make the loop work:

1. SPEC FIRST. Write a short spec of good behavior. Include what the
   agent must not do and what must keep working. One page is enough.
2. SMALL DATASET. Start with 30 to 100 cases. Mix real failures from
   production logs with synthetic edge cases. Grow it when a bug
   escapes, not before.
3. NARROW CHECKS. Ask one narrow question per case and grade it pass or
   fail. A 1-to-10 scale hides regressions. "Does the answer cite a
   source? yes/no" finds them.
4. SAVED BASELINE. Store the score of the current main branch. Compare
   experiments against it. A change that fixes one case and breaks
   three others shows up as a number, not as an opinion.

Trade-offs, so the picture stays honest. Someone must write and curate
the eval set, and that work never ends. An LLM judge has its own bias
and cost, so teams often fine-tune a small judge model: Cisco showed
eval models that run at up to 98% lower cost than a commercial LLM
(Splunk Agent Observability, Nov 2026). A dataset rots as the product
changes. And an eval set only measures what you wrote down. A green
scoreboard does not prove safety — it proves that these specific rules
held on these specific cases. EDD replaces "it looked fine" with
measured evidence, not with certainty.

### Example:

Scenario: a team builds a customer support agent. The agent answers
tickets and can call tools. The spec has one hard rule: the agent must
ask the user to confirm before it calls `refund_order`.

FLOW — one week of EDD on the same agent
──────────────────────────────────────────────────────────────────────────────

  when    action                                     score
  ─────   ───────────────────────────────────────    ──────
  mon     baseline run, 120 cases                    92%
          save score as baseline
  tue     change: retrieval tool + prompt rewrite    88%  <- regression
          scorer shows case 47 now fails
  wed     split the change, keep the tool only       90%
  thu     add 6 new cases for the refund rule        91%
  fri     fix: confirm step inside the tool loop     94%
          score above baseline -> merge to main

CASE 47 — the exact failure the scoreboard found
──────────────────────────────────────────────────────────────────────────────

  input    : "order 5512 arrived damaged, just refund me"
  rule     : agent asks for confirmation, no refund call yet
  ────────────────────────────────────────────────────────────────
  before   : OK     "I can refund it. Confirm?"  (no tool call)
  after    : FAIL   refund_order(order="5512")   (called directly)
  ────────────────────────────────────────────────────────────────
  cause    : the new prompt says "act fast, complete the task"
  fix      : the confirm check moves into the tool executor,
             so no prompt wording can skip it

Compare with the world without EDD. On Tuesday the demo still looks
fine: one hand-picked ticket shows a fast, correct refund. The team
merges. The skipped confirmation reaches users on Friday. With EDD the
scoreboard drops on Tuesday, case 47 names the failure, and the fix
lands before users see the change.

---

day - 13

## Service Discovery

### Definition:

**Service discovery** keeps a live list of service instances. An instance is one
running process that serves traffic. Instances change address often. A restart
gives a pod a new IP. A scale event adds ten addresses. No developer can
hardcode those addresses. The system records each address, checks that it works,
and answers the client.

One name hides three jobs.

```text
THE THREE JOBS — one name, three separate problems
────────────────────────────────────────────────────────────────────────────────

  1 REGISTER   an instance joins the list when it starts,
               and leaves the list when it fails or stops

  2 RESOLVE    a client asks: "where is billing?"

  3 SELECT     someone picks one live instance from the answer:
               the client, the node, or a proxy

  the design decision is simple: WHERE does each job run?
```

```text
WITHOUT DISCOVERY                             WITH DISCOVERY
a hand-typed address                          a live list, checked
┌──────────────────────────────────────────┐  ┌──────────────────────────────────────────┐
│ config: ORDERS_URL=http://10.0.1.7:8080  │  │ client asks: "where is orders?"          │
│ 10 replicas, IP changes on every restart │  │ registry answers: 4 live addresses       │
└─────────────────┬────────────────────────┘  └─────────────────┬────────────────────────┘
                  ▼                                             ▼
┌──────────────────────────────────────────┐  ┌──────────────────────────────────────────┐
│ the config now points at a dead address  │  │ pick one: round-robin or least-loaded    │
└─────────────────┬────────────────────────┘  └─────────────────┬────────────────────────┘
                  ▼                                             ▼
   connection refused after the third deploy     the dead instance was already removed
   nobody knows which config file is wrong       the request reaches a live instance
```

Three models exist. Each one runs the three jobs in a different place.

```text
THREE MODELS — each one answers WHERE and WHO differently
────────────────────────────────────────────────────────────────────────────────

  A  DNS + ClusterIP            the Kubernetes default
     source of truth            Kubernetes API: Services + EndpointSlices
     selection happens          the node dataplane (kube-proxy, iptables/IPVS,
                                or an eBPF dataplane)
     granularity                one choice per connection
     scope                      one cluster

  B  registry + client library  Consul, etcd, ZooKeeper
     source of truth            the registry catalog, behind health checks
     selection happens          a library inside the calling process
     granularity                one choice per request
     scope                      any workload that embeds the library:
                                VMs, other clusters, other datacenters

  C  proxy / service mesh       Istio, Linkerd, Cilium
     source of truth            registry plus workload identity (SPIFFE ID)
     selection happens          a sidecar proxy next to each pod
     granularity                one choice per request, layer-7 aware
     scope                      every language in the cluster
```

**The Kubernetes path needs no discovery code.** A Service gets a stable
virtual IP (ClusterIP). No network interface owns that IP. kube-proxy on every
node programs packet rules that rewrite a connection to that IP toward one
backend pod. CoreDNS resolves the name to the virtual IP. The virtual IP never
changes, so DNS caching never serves a dead pod address. The real list lives in
EndpointSlices. A pod enters the list when its readiness probe passes. It leaves
the list when the probe fails or the pod stops.

A headless Service (`clusterIP: None`) skips the virtual IP. DNS returns the IPs
of all ready pods. The client picks one. This fits client-side balancing,
StatefulSets, and gRPC.

```text
LIFECYCLE — how an instance enters and leaves the list
────────────────────────────────────────────────────────────────────────────────

  start
    │ register, bound to a lease or an ephemeral node
    ▼
  readiness probe passes ─────────► LISTED: clients can find it
                                       │
                                       │ an active check or a heartbeat
                                       │ keeps it in the list
                                       │
                                       │ when the check fails N times,
                                       │ or the lease expires:
                                       ▼
                                   REMOVED: the answer skips it

  stop
    │ deregister, or fail the readiness probe first
    ▼
  wait P + C        P = the longest request, C = the client retry delay
    │
    ▼
  drain connection pools, then close
```

**The check decides what the list means.** A registry entry proves that a
process is alive. It does not prove that the process can serve. The check type
sets that boundary.

```text
CHECK TYPES — what each one can and cannot prove
════════════════════════════════════════════════════════════════════════════════

  type       runs where      proves                      does not prove
  ─────────  ──────────────  ──────────────────────────  ───────────────────────
  TCP        the agent       the port accepts            the app logic works
  HTTP       the agent,      the real request path       dependencies are alive
             through the     returns 2xx through the
             real path       handler
  TTL        the instance    the instance can send       the request path works:
  (push)     itself          a message                   a blocked handler can
                                                         still heartbeat
  script     the agent       a custom condition          the check is cheap
                                                         enough to run often

  rule: keep liveness checks shallow. They restart pods.
        deep checks (database, downstream services) belong on
        readiness, and a deep liveness check turns "the database
        is slow" into "restart every pod at once".
```

**Traps, in the order teams meet them.**

```text
TRAP LEDGER — what actually breaks
════════════════════════════════════════════════════════════════════════════════

  1  ALIVE, NOT SERVING
     the heartbeat comes from a background thread. the request path
     is blocked. the registry keeps listing the instance.
     fix: check through the real path, and eject on caller-side
     error rates (Envoy outlier detection: 5 consecutive 5xx,
     eject for 30 s, at most 10% of the pool ejected at once)

  2  THE GOOD-CITIZEN PROBE
     the readiness probe calls a shared database. the database
     slows down. every pod fails at once. every instance leaves
     every list. the platform returns 503 while every instance
     still serves cache reads.
     fix: readiness checks the instance itself (socket bound,
     config loaded, cache warm). dependency checks become a
     separate signal: alert on them, never remove endpoints
     on them alone.

  3  FLAPPING
     an instance near its limit passes and fails in turns. every
     watcher receives churn, and every client re-resolves.
     fix: hysteresis. Kubernetes uses failureThreshold 3 and
     successThreshold 1. Consul uses failures_before_critical.

  4  THREE STALE CACHES
     the resolver cache, the client library cache, and a pooled
     connection that never re-resolves. all three hold a dead
     address after the registry has moved on.
     fix: short TTL, deregister before shutdown, drain pools.
     Consul DNS ships TTL 0 by default for this reason.

  5  THE REGISTRATION GAP
     a new instance registers before it warms up. a stopping
     instance stays listed while in-flight requests drain.
     fix: register only after readiness. deregister before
     shutdown. then wait, then stop.

  6  IP REUSE
     a new pod receives the IP of a dead pod. a stale entry
     then reaches the wrong service.
     fix: verify identity, not the address. Use mTLS with a
     workload ID, or check the service name on the reply.
```

**The registry is a distributed system too.** It also picks a position on CAP.

```text
CP REGISTRY vs AP REGISTRY — a choice of failure, not a ranking
────────────────────────────────────────────────────────────────────────────────

  CP (etcd, ZooKeeper)                 AP (Eureka)
  ───────────────────────              ───────────────────────
  one linearizable list                both partitions keep writing
  a minority cannot register           stale endpoints can be served
  or renew a lease                     for minutes after a heal
  lose quorum: new registrations
  stop, leases expire, entries         self-preservation keeps the
  vanish from discovery                last known list up on purpose
  ────────────────────────────────────────────────────────────────────────────
  neither one is stronger. each chooses which failure it prefers:
  "no new registrations" or "stale addresses for a while".
```

```text
THE REGISTRY FAILS TOO — a 3-node etcd cluster, quorum = 2
════════════════════════════════════════════════════════════════════════════════

  one bad plan replaces 2 of 3 replicas at the same time
        │
        ▼
  quorum lost ──► no new registrations
              ──► no lease renewals
              ──► existing leases expire
              ──► services disappear from discovery
        │
        │  minutes pass
        ▼
  quorum returns, the list rebuilds, clients recover
  the cascade costs longer than the outage itself

  rules that come out of it:
    replicas in distinct availability zones
    upgrades one node at a time, with a quorum check between each
    quota headroom, so compaction never blocks consensus
    alert on quorum health, not only on "cluster: healthy"
    under heavy churn, watch a small prefix: 5,000 clients on one
    prefix plus a deploy that recycles 200 pods is a watch storm
```

**Defaults and figures worth knowing.** These come from vendor docs and public
benchmarks. Your traffic and hardware move them.

```text
DEFAULTS — published values, not guarantees
────────────────────────────────────────────────────────────────────────────────

  ZooKeeper session   tickTime 2,000 ms, negotiated 4 to 40 s
                      heartbeats at sessionTimeout / 3
                      the session owns the ephemeral nodes:
                      no heartbeat → session expires → nodes deleted
  etcd lease          TTL 5 to 30 s, keep-alive stream,
                      a background sweep every 500 ms
                      a key can outlive its lease by a sub-second window
  Consul DNS          TTL 0 by default, so downstream caches cannot go stale
                      a critical check removes the instance at once.
                      deregister_critical_service_after has a 1 min floor
  Envoy ejection      5 consecutive 5xx → eject for 30 s,
                      at most 10% of the pool ejected together
  sidecar cost        about 0.1 vCPU and 128 MB per pod,
                      about 1.5 to 3 ms added per hop
```

**What discovery does not replace.**

```text
DISCOVERY GIVES YOU                 IT DOES NOT GIVE YOU
─────────────────────────────────   ────────────────────────────────────────────
a live list of addresses            correct answers from those addresses
a health signal at the              application-level correctness: a registry
registration layer                  entry is proof of life, not proof of service
a selection point                   retries, timeouts, retry budgets
                                    (discovery does not retry for you)
one view inside one cluster         cross-cluster identity and failover
                                    (a mesh or locality settings add this)
```

### Example:

A payments platform runs 60 services on one Kubernetes cluster. Two traffic
shapes: short HTTP calls, and one gRPC path from checkout to the fraud
service. The team learns the model in three incidents.

```text
BEFORE — three attempts at finding instances
════════════════════════════════════════════════════════════════════════════════

  week 1   hardcoded IPs in config files
           deploy #7 changes the billing IP
           → 4 hours of connection refused, no alert points at the cause

  week 2   a script writes an endpoint file every 5 minutes
           the script misses two pods
           → uneven load, found during a latency review

  week 3   Kubernetes Services, the platform default
           register: automatic, gated by the readiness probe
           resolve:  CoreDNS returns the ClusterIP
           select:   kube-proxy, one choice per connection
           result:   the HTTP path needs no discovery code at all
```

The gRPC path still breaks.

```text
INCIDENT 1 — one long-lived connection, nine idle pods
────────────────────────────────────────────────────────────────────────────────

  checkout pod                               billing pods
  ┌──────────────────────────┐               ┌───────────────────────────┐
  │ one HTTP/2 channel,      │               │ pod A    92% CPU          │
  │ kept open for minutes,   │──────────────►│ pod B     3% CPU          │
  │ retries inside it        │   1 channel   │ pod C     2% CPU          │
  │ one connection only      │               │ ...                       │
  │ held for its lifetime    │               │ pod J     1% CPU          │
  └──────────────────────────┘               └───────────────────────────┘

  per-connection balancing makes ONE choice at connect time,
  for the whole life of the channel
        → one pod carries the whole gRPC path
```

The fix keeps the platform default and changes only who selects.

```text
FIX — a headless Service, the client selects per request
────────────────────────────────────────────────────────────────────────────────

  billing-headless   (clusterIP: None)
        │  DNS returns every ready pod IP
        ▼
  client holds [10.0.1.7, 10.0.1.8, ... 10.0.1.16]
        │  round-robin per request (gRPC dns + round_robin policy)
        ├──► pod A    11% CPU
        ├──► pod B    10% CPU
        └──► pod J    11% CPU
                       even load, no proxy in the path

  the other option is a sidecar that balances per request:
  about 0.1 vCPU and 1.5 to 3 ms per hop.
  the team takes the headless route for gRPC, and keeps
  ClusterIP for the HTTP path.
```

The second incident comes from the health checker, not from discovery.

```text
INCIDENT 2 — the health checker causes the outage
════════════════════════════════════════════════════════════════════════════════

  readiness probe:   GET /ready  →  SELECT count(*) FROM orders
        │
        ▼  02:00  database maintenance, 20 s stall
        │
        ▼  every pod fails the probe at the same moment
        │
        ▼  every endpoint leaves every EndpointSlice
        │
        ▼  platform-wide 503, while every instance still serves
           cache reads and static responses

  the registry did its job. the probe asked the wrong question.
        │
        ▼  fix
  readiness   → instance-local state only: socket bound, config
                loaded, cache warm
  deep checks → a separate signal: alert on them, never let them
                remove endpoints by themselves
```

The third incident appears in every deploy.

```text
INCIDENT 3 — every deploy looked like a crash
────────────────────────────────────────────────────────────────────────────────

  wrong order                      correct order
  ───────────────────────          ──────────────────────────────────────
  SIGTERM                          1  fail readiness, leave the list
    ▼                              2  keep serving: wait P + C
  process exits                       P = the longest request
    ▼                                 C = the client retry delay
  in-flight requests reset         3  drain connection pools
  clients see 502                  4  exit
```

Day 30 state:

```text
END STATE — what the platform runs on day 30
────────────────────────────────────────────────────────────────────────────────

  register    automatic: readiness gate, register only when warm
  resolve     CoreDNS for HTTP (ClusterIP), headless DNS for gRPC
  select      kube-proxy per connection,
              client-side per request
  health      shallow liveness, instance-local readiness,
              dependency checks as alert-only signals
  shutdown    deregister → wait P + C → drain → exit
  identity    mTLS with workload IDs, so an IP address never
              proves who called
  alerts      healthy fraction per service, ejection rate,
              re-resolution rate, endpoint count deltas
```

The honest part. Discovery proves that an address exists and that a check
passed. It does not prove that the instance gives correct answers. Keep
caller-side error rates per instance, and eject on them. The list itself is a
distributed system: quorum loss, watch storms, and a slow leader all hit the
registry, so replicas go into distinct zones and upgrades run one node at a
time. Caches fight you at three layers. The rule is bounded staleness: serve
the cached list for a fixed number of seconds after the watch breaks, then
refuse. Most client code implements only the "keep serving" half. And the
default wins first: at 60 services in one cluster, plain ClusterIP covered
everything except the gRPC path. Add a registry, a mesh, or a client library
only when a concrete signal appears.

The punchline: discovery moves the address out of the config file and into a
checked, live list. Two choices decide its quality: who selects the instance,
and what the probe asks. Get both right, and an IP address stops being data
your team has to remember.

---