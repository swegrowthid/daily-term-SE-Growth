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