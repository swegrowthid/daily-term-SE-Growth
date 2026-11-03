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