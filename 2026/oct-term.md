day - 1

## Speculative Decoding

### Definition:

Speculative decoding is an inference optimization for autoregressive text generation. A small **draft** model proposes several tokens. The large **target** model then checks all proposed tokens in one forward pass. The system accepts the tokens that pass the check. It stops at the first token that fails.

The method attacks a hardware problem, not a modeling problem. Decoding is sequential: one token per forward pass. One token needs very little arithmetic. It needs the full weight matrix, loaded from GPU memory (HBM). So the arithmetic units sit idle while memory streams in. This is the memory-bound shape of decode.

Verification has the opposite shape. A forward pass over K tokens costs almost the same as a forward pass over 1 token, because the weights load once. So the target model can score K draft tokens for the price of one step. Arithmetic becomes dense, and the memory wait is amortized over K tokens.

The speed-up is **lossless**. The output distribution stays exactly equal to plain sampling from the target model. Speculative decoding is not quantization. It is not an approximation. A rejection-sampling rule enforces the exactness:

```
accept draft token x with probability   min(1, p_target(x) / p_draft(x))
if rejected, sample from the residual   max(0, p_target(x) - p_draft(x))
```

The residual step matters. Without it, the output drifts toward the intersection of the two distributions, and the text changes character. With it, the result is statistically indistinguishable from the target model alone.

Leviathan et al. (2022) and Chen et al. (2023) introduced the method.

```
WITHOUT vs WITH SPECULATIVE DECODING
═══════════════════════════════════════════════════════════════════════════

  WITHOUT — one full pass per token; memory bandwidth is the wall
  ─────────────────────────────────────────────────────────────────────────

    "The capital of France is"
         │
         ▼
    ┌───────────────────────────────────────────────┐
    │  TARGET MODEL (large)                         │
    │  load ~all weights from HBM    <- slow, 1x    │
    │  do math on 1 token            <- tiny        │
    └────────────────────┬──────────────────────────┘
                         ▼
                      "Paris"          step 1
         │
         ▼
    ┌───────────────────────────────────────────────┐
    │  TARGET MODEL (large)                         │
    │  load ~all weights from HBM    <- slow again  │
    │  do math on 1 token            <- tiny again  │
    └────────────────────┬──────────────────────────┘
                         ▼
                      "."              step 2  ... repeat per token

    cost per token = one full weight read
    decode usually uses a small fraction of peak GPU FLOPs


  WITH — K tokens proposed, ONE verify pass, several tokens accepted
  ─────────────────────────────────────────────────────────────────────────

    "The capital of France is"
         │
         ├──────────────►┌──────────────────────────────────┐
         │               │ DRAFT MODEL (small, cheap)       │
         │               │ proposes K = 5 tokens            │
         │               └───────────────┬──────────────────┘
         │                               │ "Paris" "." "It" " is" " home"
         │                               ▼
         │               ┌──────────────────────────────────────────┐
         └──────────────►│ TARGET MODEL — ONE forward pass          │
                         │ scores all 5 positions in parallel       │
                         │ weights load once, for K positions       │
                         └───────────────┬──────────────────────────┘
                                         ▼
                         verify each token: min(1, p_target / p_draft)
                                         │
                    ┌────────────────────┴─────────────────────┐
                    ▼                                          ▼
          ACCEPT "Paris" "." "It"                  REJECT " is" at position 4
          positions 1-3 pass                       sample from the residual
                                                          -> " a"
                    │                                          │
                    └────────────────────┬─────────────────────┘
                                         ▼
                  4 tokens emitted for roughly the cost of 1 target step

  ┌─────────────────────────────────────────────────────────────────────┐
  │  KEY IDEA: the expensive model stops writing and starts checking.   │
  │  Checking K tokens is nearly free, because the weights load once.   │
  │  The cheap model pays for the extra work, and the expensive model   │
  │  stays the only source of the output distribution.                  │
  └─────────────────────────────────────────────────────────────────────┘
```

The arithmetic of the speed-up is simple to state and hard to predict. Only the drafted tokens that pass the check save time.

```
  expected tokens per target step  =  sum of acceptance probabilities
  wall-clock speed-up              ~  accepted tokens per step
                                      ───────────────────────────
                                      1 + draft cost fraction

  draft cost fraction = cost of drafting K tokens, as a fraction of one
                        target forward pass

  the theory assumes independence between positions. Real text breaks
  that assumption in both directions:
    • boilerplate, code identifiers, JSON keys  -> acceptance is high
    • creative prose, unusual names, math       -> acceptance is low
```

Where the draft tokens come from decides the ceiling and the operational cost:

```
  PROPOSAL METHOD LANDSCAPE (published 2026 figures)
  ┌────────────────────────┬───────────────────────┬──────────────────────────────┐
  │ METHOD                 │ TYPICAL GAIN          │ TRADE-OFF                    │
  ├────────────────────────┼───────────────────────┼──────────────────────────────┤
  │ Vanilla draft model    │ ~1.95x, accept 0.62   │ a second full LLM in VRAM,   │
  │ (small second LLM)     │                       │ plus its KV cache             │
  ├────────────────────────┼───────────────────────┼──────────────────────────────┤
  │ Medusa (extra decoding │ ~2.21x, accept 0.68   │ light heads, lower ceiling;   │
  │ heads on the target)   │                       │ absent from current vLLM docs │
  ├────────────────────────┼───────────────────────┼──────────────────────────────┤
  │ EAGLE-1 / EAGLE-3      │ ~2.52x accept 0.73    │ needs a trained speculator    │
  │ (feature-level draft)  │ ~2.89x accept 0.81    │ head for your exact model     │
  ├────────────────────────┼───────────────────────┼──────────────────────────────┤
  │ n-gram / prompt-lookup │ ~1.38x general,       │ no training, copies from the  │
  │ / suffix decoding      │ best on edit + RAG    │ prompt; weak on open prose    │
  ├────────────────────────┼───────────────────────┼──────────────────────────────┤
  │ Native multi-token     │ model-specific        │ the target model predicts     │
  │ prediction (MTP)       │                       │ several tokens itself         │
  └────────────────────────┴───────────────────────┴──────────────────────────────┘
```

Acceptance of a token is not a quality judgement. Under greedy decoding the check reduces to "does the same argmax appear". Under sampling the check is the ratio rule above, and a low ratio means the two models disagree about the next token.

The go/no-go rule is the acceptance rate, measured on real traffic:

- Below roughly 0.5 acceptance, speculation is net negative. The verify pass costs more than it saves.
- The healthy band is roughly 0.6 to 0.8.
- A draft model must share the tokenizer and the chat template of the target. A mismatch quietly destroys the acceptance rate.
- Speculation helps low-to-mid concurrency. It hurts saturated serving. A 2026 serving study (arXiv 2406.14066) measured the gain eroding past roughly 12 requests per second with 5-token proposals, and turning negative past roughly 16 requests per second with 3-token proposals. Draft work competes with batch throughput.
- If the client uses top-k or nucleus sampling, apply the acceptance rule to the full distribution. A rule applied to a truncated distribution breaks the exactness guarantee.
- Long context is a common surprise. Draft KV cache grows with the prompt, and the draft cost fraction rises with it.

### Example:

"KodeKita" is a coding-assistant API. It serves one 32B-class open model on two H100s with vLLM and continuous batching. Completion is the main product: inline edits, JSON repair, unit-test stubs. Latency per completion is the number the team watches.

The workload is friendly to speculation. Completion continues code that already sits in the prompt. Identifiers, brackets, and JSON keys repeat from the context. A cheap proposer can guess several of them before the big model speaks.

```
STEP-BY-STEP — one completion of a JSON field, K = 5 proposals
═══════════════════════════════════════════════════════════════════════════

  prompt so far:   {"order_id": "A-9931", "status":

  ┌─ 1. DRAFT (cheap model or speculator head, ~7 ms) ─────────────────┐
  │   proposes 5 tokens:   "PAID" , " failed" , "," , " total" , ":"   │
  └────────────────────────────┬──────────────────────────────────────┘
                               ▼
  ┌─ 2. VERIFY (target model, ONE pass over the 5 positions) ──────────┐
  │   scores all 5 in parallel. Weights load once.                     │
  │                                                                    │
  │   pos 1  "PAID"    p_t=0.96  p_d=0.91   accept  min(1, 1.05) = 1.0 │
  │   pos 2  " failed" p_t=0.71  p_d=0.63   accept  min(1, 1.13) = 1.0 │
  │   pos 3  ","       p_t=0.88  p_d=0.84   accept  min(1, 1.05) = 1.0 │
  │   pos 4  " total"  p_t=0.31  p_d=0.72   REJECT  min(1, 0.43) = 0.43│
  │                                                                    │
  │   residual at pos 4 = max(0, p_t - p_d), renormalized             │
  │       -> sample " amount" (from the TARGET distribution)           │
  └────────────────────────────┬──────────────────────────────────────┘
                               ▼
  ┌─ 3. COMMIT + KV HANDLING ──────────────────────────────────────────┐
  │   keep KV entries for accepted positions 1-3, plus the corrected   │
  │   token at position 4. Drop the rejected draft KV entries.         │
  │   Next round drafts from the new position.                         │
  └────────────────────────────┬──────────────────────────────────────┘
                               ▼
  OUTPUT:  {"order_id": "A-9931", "status": "PAID failed, amount"

  4 tokens committed in about the time of 1 target pass
  acceptance length this round = 4 of 5 proposed
```

The team enables an EAGLE-3 speculator head for their model family, one configuration change:

```
SPECULATOR ON — vLLM, K = 5, feature-level draft from a trained head
────────────────────────────────────────────────────────────────────

  vllm serve <target-model> \
    --speculative-config '{
        "model": "<speculator-head-for-this-model>",
        "method": "eagle3",
        "num_speculative_tokens": 5,
        "draft_tensor_parallel_size": 1
      }'

  what the team measures, before and after:
  ┌───────────────────────┬──────────────────┬───────────────────────┐
  │ METRIC                │ BASELINE         │ WITH SPECULATION      │
  ├───────────────────────┼──────────────────┼───────────────────────┤
  │ tokens / second,      │ 1.0x             │ 2.0x - 2.9x           │
  │ single stream         │                  │ (published EAGLE-3)   │
  │ acceptance rate       │ n/a              │ 0.6 - 0.8 target band │
  │ output distribution   │ target model     │ unchanged, exact      │
  │ extra VRAM            │ 0                │ speculator head + KV  │
  │ extra work per step   │ 0                │ one draft pass        │
  └───────────────────────┴──────────────────┴───────────────────────┘
```

Then peak hours arrive, and the same setting backfires.

```
PEAK HOURS — the same GPU serves a batch, and the draft competes with it
═══════════════════════════════════════════════════════════════════════════

  OFF-PEAK                         PEAK
  ─────────                        ────
  2 concurrent requests            64 concurrent requests
  GPU: mostly memory-bound         GPU: compute-bound, batch is full

   ┌──────────┐                     ┌───────────────────────────────┐
   │ target   │← verify helps       │ every step already carries 64 │
   │ pass     │  (idle FLOPs)       │ sequences -> FLOPs are spent  │
   └──────────┘                     └───────────────────────────────┘
        ▲                                    ▲
        │                                    │ draft pass now steals
   ┌──────────┐                              │ batch slots and FLOPs
   │ draft    │ cheap, hides in            ┌──┴───────┐
   └──────────┘ the memory wait             │ draft    │ NOT free anymore
                                            └──────────┘

  measured direction of the effect (arXiv 2406.14066):
     ~12 req/s with 5-token proposals  -> gain gone
     ~16 req/s with 3-token proposals  -> throughput LOWER than baseline
```

The fix is not to remove speculation. The fix is to route it by workload. The team keeps a second deployment without a speculator head and sends bulk traffic there.

```
THE ROUTING RULE THAT SURVIVED CONTACT WITH PRODUCTION
═══════════════════════════════════════════════════════════════════════════

  interactive completions, low concurrency   -> speculation ON  (K = 5)
  background batch jobs, high concurrency    -> speculation OFF
  retrieval / edit-heavy prompts, high concurrency
                                             -> n-gram or suffix
                                                speculation (cheap,
                                                no draft model, no
                                                extra FLOPs at peak)
  code completion, low concurrency           -> EAGLE-3 head (the
                                                largest published
                                                gains sit on code)
```

Two operational details decide whether the gain is real. First, log the acceptance rate per request. vLLM exposes request-level acceptance metrics, and an acceptance rate that drifts from 0.8 to 0.4 means the traffic changed shape. Second, run an A/B comparison on real prompts with a fixed seed. The claim is exactness, so the output must match the baseline token-for-token under greedy decoding.

The honest limits stay on the table:

- The gain is not free throughput. It trades cheap model work for expensive model idle time. Fill the idle time, and the trade disappears.
- The speed-up depends on the workload, not on the hardware alone. A code assistant and a poetry assistant with the same model and the same GPU see different numbers.
- A trained speculator head binds to one target model. Change the base model, and the head must be retrained or replaced.
- The theoretical speed-up model assumes independent acceptance per position and no interference between the two passes. Real serving violates both.

The punchline: speculative decoding does not make the big model faster. It makes the big model wait less. The big model still owns the distribution, and the small model only guesses what the big model will say. So measure the acceptance rate first, then choose a method. Choose the method before the measurement, and the GPU will show you the bill at peak traffic.

---

day - 2

## Consistent Hashing

### Definition:

**Consistent hashing** maps each key to a node without a global lookup table. It uses one rule: hash the keys and the nodes onto the same circle, then send each key to the first node clockwise from the key. The circle is the range [0, 2^32) or [0, 2^64).

The design goal is **churn control**. When one node joins or leaves, only the keys that belong to that node move. Every other key stays on its node. That single property explains why the technique sits under caches, sharded databases, CDNs, and LLM routers.

The naive alternative is modulo hashing: `slot = h(key) mod N`. It is trivial and it breaks on membership change. Change N and almost every key lands on a different node. Every local cache entry, session binding, and warm connection dies at the same time.

```
MODULO HASHING vs CONSISTENT HASHING          (4 nodes, then add 1 node)
═══════════════════════════════════════      ═════════════════════════════════════
 slot = h(key) mod N                          nodes AND keys share one circle
 N lives inside every client                  key → first node clockwise

   slots from 4 nodes                         4 points on the ring, 0 .. 2^32
   ┌─────┬─────┬─────┬─────┐                       B
   │  0  │  1  │  2  │  3  │                     ╱   ╲
   └─────┴─────┴─────┴─────┘                   A       C     key u_104 hashes
    ▲     ▲     ▲     ▲                        ╲       ╱     to the tick mark,
    │     │     │     │                         D  ◀──╯      so it goes to D.
   3 keys 5 keys 4 keys 8 keys
   (evenly by luck, never by                       ┌──────────────┐
   design)                                         │  A─B─C─D     │
                                                   │  clockwise   │
   ADD node 4 → slot = h(k) mod 5                  └──────────────┘

   re-check all 20 keys                         ADD node E on the D→A arc

   key   h(k)  mod 5   was  now                    D ─── A   becomes  D ─ E ─ A
   ───────────────────────────
   u_101  55     0     D    A  ✗ moved              only the keys inside the
   u_102  56     1     A    B  ✗ moved              D→A arc move to E.
   u_103  57     2     B    C  ✗ moved
   u_104  58     3     C    D  ✗ moved              no other key moves.
   ... 80% of all keys move                         at most 1/4 of keys move
                                                    here, and the rest are
   ~every cache entry is now cold                   untouched.
```

The repair is a **hash ring with virtual nodes**. A single point per node gives a node a random and often huge arc. Many points per node (16, 100, 256 — the number is a tuning knob) chop the ring into small arcs and drive the load error down.

```
WHY ONE POINT PER NODE IS NOT ENOUGH
═══════════════════════════════════════════════════════════════════
 ONE POINT PER NODE                   MANY POINTS PER NODE

 arcs are random and lopsided         each node owns many small arcs

   A ████████████████████  61%          A ██████  26%
   B ██                     8%          B ██████  25%
   C ▪                      2%          C █████   24%
   D █████████             29%          D ██████  25%

   A melts. C idles.                    Load spread is even. A node
   Removing C moves 61% of keys.        that leaves moves its ~1/N only.
```

The ring is old and boring in the best way. Karger and his co-authors published it in 1997 for web caches. Amazon's Dynamo paper used it in 2007 with virtual nodes. Cassandra, Riak, Chord, Voldemort, and the memcached client libketama shipped it. Envoy's `ring_hash` load balancer pads a ring to a minimum size (1024 entries by default). Cassandra turns virtual nodes on by default (`num_tokens`, 16 in recent releases, 256 in older ones). The hash is usually xxHash or MurmurHash, not MD5, because the router must stay cheap.

One honest boundary: the ring balances **keys**, not **load**. Hash output is uniform, but key popularity is not. One hot key still lands on one node, and that node still overloads.

```
THE FAMILY — HOW A KEY PICKS ITS NODE
═══════════════════════════════════════════════════════════════════════════
                    needs a sorted   minimal      best use
                    ring?            movement?
 ─────────────────────────────────────────────────────────────────────────
 Modulo hashing     no               no           fixed pool, no resize
                                                   (e.g. 16384 fixed slots in
                                                   Redis Cluster)
 Consistent         yes              yes          caches, DHTs, sharded
 hashing                                           stores, sticky routing
 Rendezvous / HRW   no               yes          pick max(h(key,node)).
 hashing (1998)                                    O(N) per lookup, no ring
                                                   to build or store
 Jump hash (2014)   no               yes          constant memory, but
                                                   numbered buckets only
 Bounded-load CH    yes              yes          ring plus a capacity rule.
 (2018)                                            Kills hot spots. Adds a
                                                   clockwise walk
 ─────────────────────────────────────────────────────────────────────────
 Rule of thumb: any balanced scheme must move about 1/N of the keys when one
 node joins N nodes. The ring reaches that floor and adds no coordination.
```

**Consistent hashing with bounded loads (CHWBL)** closes the hot-spot gap. Mirrokni, Thorup, and Zadimoghaddam proposed it in 2018. Each node gets a hard capacity near the mean:

```
capacity = ceil( c * m / n ),   c = 1 + ε > 1
```

A key that finds its primary node full walks clockwise to the first node with room. Now placement is still sticky, but no node passes the cap. The known cost is **cascaded overflow**: when several nodes in a row are full, all their spill lands on the next free node, and that node fills faster. Later work fixes the cascade with random jumps instead of a clockwise walk.

Honest limits:

- **Balance of keys is not balance of load.** Skew in key popularity, request size, or per-node cost defeats a perfect ring. Measure load per node, not key count.
- **Bounded loads trade cache for safety.** A spilled request lands on a node with a cold cache. That is a real cost. It is the central tension in LLM serving (see Example).
- **Every client must see the same ring version.** A stale ring sends a key to the wrong node. Version the ring and publish it atomically.
- **The ring does not resize cheaply in every design.** Rebuilding a few thousand vnode points is cheap. Rebuilding a full lookup table, as Maglev does, is a global operation.
- **A repartition is not free.** Removing a node does load its successor with the whole departed arc until data moves or the cache refills.

### Example:

A platform serves a 70B model on 24 vLLM replicas behind an LLM gateway. Agent workloads share long prefixes: the same 2,400-token system prompt, the same tool schemas, and a per-tenant LoRA adapter. vLLM keeps the prefill work of those prefixes in a **KV cache** (prefix caching).

Round-robin routing destroys that reuse. The prefix lands on R7, then R2, then R9, then R7 again. Each replica holds only a small piece of the prefix, KV blocks get evicted under pressure, and the shared 2,400 tokens are prefilled again and again. Time to first token (TTFT) stays high and unstable.

The fix is to route on the prefix instead of a random id, and to keep a load cap so affinity does not create a hot replica.

```
THE ROUTER — PREFIX AFFINITY WITH A LOAD CAP
═══════════════════════════════════════════════════════════════════════
 request: prefix P (2,400 tokens) + adapter "bantukode"

   h(P + adapter) ─────────────────────▶ position on the ring
                                              │
                                              ▼
                                        primary replica R3
                                        ┌──────────────────────┐
                                        │ load = 178% of cap   │
                                        └──────────┬───────────┘
                                                   │  cap rule:
                                                   │  walk clockwise
                                                   ▼
                                        R4  load = 91%  ──▶  serve here
                                                             (prefix cold,
                                                              extra prefill,
                                                              but no overload)
 rules in one config file:
   strategy        PrefixHash          (hash prefix + adapter name)
   meanLoadFactor  125                 (cap = 1.25 × mean load)
   replication     256                 (vnode points per replica)
```

Before and after, on the same fleet:

```
RANDOM ROUTING                       RING HASH + 1.25 LOAD CAP
══════════════════════════════       ══════════════════════════════════════
 prefix hits a different replica      prefix always maps to one replica,
 on almost every request              and spills only when that replica is
                                      above the cap
 KV hit rate      ~64%                KV hit rate      ~93%
 TTFT at 1,200    baseline            TTFT at 1,200    ~95% lower
 concurrent reqs                      throughput        ~127% higher
 at saturation    baseline            capacity          ~2.3× at a 3.5s p99
```

The last row is not a toy. A 2026 study (CacheRoute) reports 93.2% served KV hit and 2.3× the capacity of the strongest baseline on 60 H100 GPUs with Llama-3.3-70B in FP8. KubeAI's PrefixHash benchmark reports the 95% TTFT cut and the 127% throughput gain at 1,200 concurrent requests.

The same study also publishes the counterexample, and it matters more than the win. On two 32B workloads, the recoverable prefix work was too small. Affinity then bought almost no KV reuse, while its residual load skew stayed. Capacity fell instead of rising. So the rule is not "always pin keys". The rule is "pin keys only where the workload gives the pin something back".

Six rules carry the pattern:

- **Hash the thing you want to keep together.** Session id, tenant id, prefix, or shard key. Never a random request id.
- **Freeze the hash input.** Add a version tag to the key. One silent change to the key format invalidates the whole cache on deploy.
- **Use virtual nodes.** A ring with one point per node behaves worse than random. Start at 100 to 256 points.
- **Add a load cap.** Pure affinity converts key skew into hot spots. A cap near 1.25× mean load keeps the ring useful at saturation.
- **Watch the right signals.** Served cache hit rate, per-node imbalance, and p99. A drop in a single node's hit rate shows a ring bug early.
- **Test on your own workload.** Published speedups belong to the workload that produced them. Replay your own traffic before you enable affinity.

The punchline: consistent hashing is not a load balancer. It is a **placement** function with a minimal-movement guarantee. It keeps a key on the same node across membership changes, and it costs one hash and one binary search. Everything else — the vnode count, the load cap, the spill rule — is a knob you have to set against real traffic. Get the key right, keep the ring identical on every client, and put a ceiling on each node. Then adding the 25th replica is a boring event, which is the entire point.

---

day - 5

## Vector Clocks

### Definition:

A vector clock is a logical clock made of a vector of counters. The system has N nodes. Each node keeps one counter per node in the system. A node updates its own counter and copies the counters it learns from messages.

The purpose is one question: did event A cause event B, or are the two events independent? A physical clock cannot answer this question. A vector clock answers it exactly.

**Why wall-clock time fails.** Every node has its own clock. Cloud clocks drift apart. Published measurements put the drift between NTP-synced cloud machines at roughly 10 to 250 milliseconds. So a timestamp comparison across nodes is a guess. Two events with no causal link land in an order that is real in the clock and false in the world.

```text
WALL CLOCK — an order that looks true and is false
════════════════════════════════════════════════════════════════════════

  node A  write "kopi"      wall clock 12:00:00.010
  node B  write "gula"      wall clock 12:00:00.030

  naive read: A came first, so B is newer. keep B. delete A.
  truth     : A and B never saw each other. both are valid.
              the delete lost a customer order.
```

**Causality, not simultaneity.** Lamport (1978) defined the happens-before relation. It has three rules:

1. Two events on the same node are ordered by program order. The earlier event happens before the later event.
2. A send event happens before the matching receive event.
3. The relation is transitive. A before B and B before C means A before C.

Two events are **concurrent** when neither happens before the other. Concurrent does not mean simultaneous. It means causally unrelated.

**The scalar clock and its one-way guarantee.** A Lamport clock keeps one integer per node. It gives a total order that respects causality, and it is cheap. The guarantee is one-way only:

```text
LAMPORT CLOCK — cheap, but one direction only
════════════════════════════════════════════════════════════════════════

  rule:   a -> b   implies   L(a) < L(b)
  limit:  L(a) < L(b)  does NOT imply  a -> b

  trace:  node A: a1 send m1 ─────────────────────► a3 recv m2
          node B:         recv m1 ─► b2 send m2 ─►
          node C: c1 local event (C talks to nobody)

  ┌──────────────────────────────────────────────────────────────────┐
  │ LAMPORT CLOCK  (one integer per node)                            │
  │                                                                  │
  │   a1 = 1            m1 carries 1                                 │
  │   b2 = max(0,1)+1 = 2      b3 = 3      m2 carries 3              │
  │   a3 = max(1,3)+1 = 4                                            │
  │   c1 = 1                                                         │
  │                                                                  │
  │   a3 = 4  vs  c1 = 1   ->  clock reports: a3 came later          │
  │   but A and C never exchanged a message. the clock lies.         │
  └──────────────────────────────────────────────────────────────────┘
  ┌──────────────────────────────────────────────────────────────────┐
  │ VECTOR CLOCK  (N counters, one per node)                         │
  │                                                                  │
  │   a1 = [1,0,0]     m1 carries [1,0,0]                            │
  │   b2 = max([0,0,0],[1,0,0]) = [1,0,0], then +1 on B -> [1,1,0]   │
  │   b3 = [1,2,0]     m2 carries [1,2,0]                            │
  │   a3 = max([1,0,0],[1,2,0]) = [1,2,0], then +1 on A -> [2,2,0]   │
  │   c1 = [0,0,1]                                                   │
  │                                                                  │
  │   a3 = [2,2,0]  vs  c1 = [0,0,1]                                 │
  │   component 1: 2 > 0.  component 3: 0 < 1.                       │
  │   neither vector dominates  ->  CONCURRENT                       │
  │   the clock reports the truth.                                   │
  └──────────────────────────────────────────────────────────────────┘
```

The third counter is the whole difference. One integer carries one number. A vector carries a history, one entry per node.

**The clock rules.** Three events update a vector clock:

```text
RULES — local event, send, receive
════════════════════════════════════════════════════════════════════════

  local event on node i     V[i] = V[i] + 1
  send                      V[i] = V[i] + 1, then attach the whole vector
  receive on node i         for every j:  V[j] = max(V[i][j], incoming[j])
                            then        V[i] = V[i] + 1

  the receive rule is the important one. the node absorbs the history of
  the sender before it records its own event.
```

**The comparison rule.** Two vectors compare in three ways. The result is exact, not a heuristic.

```text
THE COMPARISON RULE — three outcomes, no guessing
════════════════════════════════════════════════════════════════════════

  V(a) <= V(b)  means  every counter of V(a) <= the matching counter of V(b)

  ┌────────────────────────┬─────────────────────────────┬─────────────────┐
  │ relation of the two    │ meaning                     │ what to do      │
  ├────────────────────────┼─────────────────────────────┼─────────────────┤
  │ all counters equal     │ same event, same version    │ nothing         │
  │ V(a) <= V(b), one <    │ a happened before b         │ keep b, drop a  │
  │ neither <= the other   │ a and b are concurrent      │ merge or ask    │
  └────────────────────────┴─────────────────────────────┴─────────────────┘

  causal join (merge) = component-wise maximum
     [1,0,0] join [0,1,0] = [1,1,0]   both histories are kept
     the join decides what is known. it does not decide which value wins.
```

The strength of the mechanism is the biconditional: `V(a) < V(b)` if and only if `a` happens before `b`. A scalar clock gives one direction. A vector clock gives both directions, and it exposes concurrency as a first-class answer.

**The cost.**
- Memory: O(N) per event or per stored version, where N is the number of writers. A three-node cluster pays three counters. A fleet of clients pays one counter per client.
- Compare: O(N) integer comparisons. Cheap at 3 to 9 nodes. Non-trivial at hundreds of nodes.
- Metadata: every write carries the vector. For a small value (a flag, a counter), the metadata can be larger than the payload.

**Variants, and the problem each one fixes.**

```text
THE FAMILY — same idea, different scope
════════════════════════════════════════════════════════════════════════

  ┌────────────────────┬──────────────────────┬───────────────────────────┐
  │ MECHANISM          │ WHAT IT ORDERS       │ SCOPE / LIMIT             │
  ├────────────────────┼──────────────────────┼───────────────────────────┤
  │ wall clock + NTP   │ nothing causal       │ drift 10-250 ms, can jump │
  │ Lamport clock      │ total order          │ cannot detect concurrency │
  │ vector clock       │ cause and effect     │ O(N), N = writers         │
  │ version vector     │ stored data versions │ keyed on storage nodes    │
  │ dotted version     │ stored data versions │ dot + server context,     │
  │ vector (DVV)       │                      │ fixes sibling explosion   │
  │ hybrid logical     │ causality + near     │ 64 bit, no concurrency    │
  │ clock (HLC)        │ wall-clock time      │ detection, needs bounded  │
  │                    │                      │ clock skew                │
  └────────────────────┴──────────────────────┴───────────────────────────┘

  a vector clock counts events on processes.
  a version vector counts data versions on storage nodes.
  the structure is the same. the meaning is different.
```

HLC packs 48 bits of physical time and 16 bits of logical counter into one 64-bit value (Kulkarni et al., 2014). The physical part keeps the timestamp readable and sortable. The logical part keeps the causality rule. CockroachDB, YugabyteDB and MongoDB use HLC-style clocks. HLC is one-way like Lamport, so it orders writes but does not detect concurrency.

**Pros and cons.**
- Pro: exact concurrency detection. No false order, no silent loss from a bad timestamp.
- Pro: no central coordinator. Each node decides locally.
- Pro: the merge operation is defined. The component-wise maximum always exists.
- Con: size grows with the number of writers. Unbounded client fleets break plain vector clocks.
- Con: the merge of the vectors is automatic, but the merge of the values is application work. A cart unions. A ledger must not guess.
- Con: garbage collection of dead writers is mandatory. Without it, the vector never shrinks.

### Example:

"Warungku" is an offline-first shopping cart for small shops. One cart, three node groups: `JKT` (Jakarta), `SBY` (Surabaya), `SIN` (Singapore). A shopper works offline on a phone and syncs later. The cart key is `cart-7`. The vector order is `[JKT, SBY, SIN]`.

```text
RUN TRACE — one partition, three writes, one merge
════════════════════════════════════════════════════════════════════════

  t1  shopper on JKT adds "kopi 1 kg"
      V(JKT) = [1,0,0]                          cart-7 = { kopi }

  t2  network between JKT and SBY is down
      shopper on SBY adds "gula 2 kg" to its own copy
      V(SBY) = [0,1,0] on the SBY copy

  t3  sync: SBY sends its version

      ┌──────────────────────────────────────────────────────────────┐
      │ classify the two versions                                    │
      │                                                              │
      │   V(JKT) = [1,0,0]        V(SBY) = [0,1,0]                   │
      │   1 > 0 at JKT, but 0 < 1 at SBY                             │
      │   neither dominates  ->  CONCURRENT  ->  keep BOTH           │
      └───────────────────────────┬──────────────────────────────────┘
                                  ▼
      ┌──────────────────────────────────────────────────────────────┐
      │ merge the values (application rule: union the cart)          │
      │                                                              │
      │   { kopi }  ×  { gula }   ->  { kopi, gula }                 │
      │   V = join([1,0,0],[0,1,0]) = [1,1,0]                        │
      │   a write on JKT records the merge: V = [2,1,0]              │
      └───────────────────────────┬──────────────────────────────────┘
                                  ▼
  t4  SIN receives the merged version and adds "susu"
      V(SIN) = max([0,0,0],[2,1,0]) = [2,1,0], then +1 on SIN
      V(SIN) = [2,1,1]

      classify [2,1,1] against [2,1,0]:  2>=2, 1>=1, 1>0
      -> SIN DOMINATES the merge. no conflict. one causal line of history.

  result
    causally ordered   : t1 -> t3 -> t4        (each saw the previous)
    concurrent         : t1 and t2             (both survived, cart merged)
    lost writes        : none
```

Now the same team replaces the merge rule with last-write-wins on the wall clock, because the merge code is extra work.

```text
THE FAILURE MODE — one rule change, silent data loss
════════════════════════════════════════════════════════════════════════

  rule:  compare wall-clock timestamps, keep the larger one

   JKT "kopi"  12:00:00.010   ─┐
   SBY "gula"  12:00:00.030   ─┴─►  keep "gula".  "kopi" is deleted.
                                    the shopper never sees an error.

  this is not a story about a bad clock. it is a story about a wrong rule.
  a Jepsen test of Cassandra under QUORUM with last-write-wins clock
  timestamps observed 285 of 1,009 acknowledged writes lost, about 28%.
  the number is one measured case, not a universal constant.
```

The next failure is structural, and it comes from the phone fleet.

```text
SIBLING EXPLOSION — when client IDs enter the vector
════════════════════════════════════════════════════════════════════════

  40,000 shopper devices. Each device writes with its own client ID.

  vector at the start  : [JKT, SBY, SIN]             3 counters
  after a week         : [JKT, SBY, SIN, d-1, d-2, ... d-5593]   huge
  every offline write is concurrent with every other offline write
  every read returns a growing set of siblings

  ┌──────────────────────────────────────────────────────────────────┐
  │ read cart-7  ->  [ gula , gula+teh , gula+teh+kopi , ... ]        │
  │ the store must store all of them. the client must merge all.      │
  └──────────────────────────────────────────────────────────────────┘

  the fix: dotted version vectors (DVV)
    context  = vector over STORAGE nodes only     (small, bounded)
    dot      = one (node, counter) pair for the value that was written

    a client that reads a context and writes with that context
    replaces the versions it saw. only a real race creates a sibling.
    Riak hit this problem with client-keyed vectors and moved to DVV
    in version 2.0. after the change, well-behaved clients converge to
    one value, and only genuine concurrent writes stay as siblings.
```

The production rule that survived contact with real traffic:

```text
CHOOSING THE MECHANISM — three questions, in order
════════════════════════════════════════════════════════════════════════

  Q1  must the system DETECT concurrent writes?
      ├─ no  -> do not pay O(N). use HLC or a per-key single writer.
      └─ yes
          Q2  how many writers per key?
              ├─ few (3 to 9 nodes)   -> version vector keyed on nodes
              └─ many (thousands)     -> dotted version vector,
                                         or one writer per key,
                                         or a CRDT with a built-in rule

  monitoring signals that matter after launch
  ┌──────────────────────────────┬────────────────────────────────────┐
  │ conflict rate per key        │ a rise means longer partitions,   │
  │                              │ slower sync, or a bad owner split  │
  │ vector size per key          │ an unbounded climb means client   │
  │                              │ IDs entered the vector             │
  │ sibling count on read        │ more than 1 means the merge path  │
  │                              │ is live and must be tested         │
  │ clock skew between nodes     │ relevant for HLC, which needs a   │
  │                              │ bounded skew to stay correct       │
  └──────────────────────────────┴────────────────────────────────────┘
```

The limits stay visible:

- A vector clock orders events. It does not choose values. The join of two vectors is a fact. The merge of two carts is a product decision.
- A resolved write must carry both histories. If a merge writes only the branch it read, the new version stays concurrent with the other branch, and the conflict returns on the next sync.
- Vector clocks do not make replication fast. A stale node stays stale until it receives the newer version.
- HLC is compact and friendly to wall-clock reasoning, but its causality guarantee lasts only while clock skew stays bounded. Under unbounded drift, the guarantee weakens.
- The counters must be compared as integers, not as serialized field order. The implementation detail decides whether the comparison is correct.

The punchline: a vector clock answers the one question a timestamp cannot answer. It tells you whether two events are related or independent. The price is one counter per writer, and that price is fine at three nodes and dangerous at forty thousand clients. So choose the clock for the question you must answer. Detect concurrency, and pay the vector, keyed on storage nodes. Order writes and stay compact, and take HLC, accepting the one-way guarantee. Pick by the question, and the conflict story becomes boring, which is the entire point.

---

day - 6

## Data Gravity

### Definition:

**Data gravity** is the pull that a body of data exerts on applications, services, and other data. Dave McCrory coined the term in 2010. He copied the shape of Newton's law. Mass attracts mass, and mass is hard to move. A large dataset pulls compute, tools, and people toward it. The dependent set then makes the data harder to move again.

The pull comes from four forces. Say each force out loud before you pick a region.

```text
THE FOUR FORCES — why a dataset holds its compute in place
═══════════════════════════════════════════════════════════════════════════

 1. TRANSFER COST              ingress is free, egress is priced high
                               AWS $0.09/GB out, GCP $0.12/GB,
                               Azure $0.087/GB  (list price)
                               wholesale transit ≈ $0.005/GB
                               → markup of 17x to 24x on the way out
                               → 1 TB out of AWS ≈ $90

 2. DISTANCE                   every read crosses a network
                               a daily scan pays the latency daily
                               a training job pays it per batch

 3. ENTANGLEMENT               pages, schemas, indexes, caches,
                               IAM rules, dashboards, cron jobs,
                               backups, feature tables

 4. LOCK-IN AS A PRODUCT       free in + cheap storage + paid out
                               the exit fee is a choice, not a cost
```

Force 3 carries more weight than people expect. Data does not grow alone. A team builds a whole ecosystem around a table. The ecosystem, not the byte count, decides the price of the next move. A 50 TB table with 40 downstream jobs resists a move more than a 500 TB archive that nobody reads.

**The naive pattern ignores the pull.** A platform team keeps compute in one region and data in another. Every job pays the distance, and every migration pays the fee.

```text
COMPUTE-CENTRIC vs DATA-CENTRIC             (same 500 TB dataset)
═══════════════════════════════════════     ══════════════════════════════════

 COMPUTE-CENTRIC  (fight gravity)            DATA-CENTRIC  (ride gravity)

   data: 500 TB            new GPU cluster     data: 500 TB    GPU cluster
   eu-central-1            us-east-1           eu-central-1    eu-central-1
        │                        │                   │              │
        └──── pull 500 TB ───────┘                   └── read via ──┘
              ~$46,000 egress                            private link
              ~5 days at 10 Gbps                         ~$0 transfer
              every new job repeats both                 2 ms, not 150 ms

   the invoice grows with the traffic            compute adapts to the data
```

The pattern is one rule: **move the compute to the data**. HDFS and MapReduce wrote this rule down in 2004. A mapper reads its input from the local disk when it can. Modern tools keep the same rule. Push the query down to the storage engine. Run the container in the same region as the bucket. Train the model next to the corpus.

```text
STRENGTH OF THE PULL  —  gravity ∝ volume × dependents × growth rate
═══════════════════════════════════════════════════════════════════════════

   SMALL AND YOUNG                    LARGE AND OLD
   ┌────────────────────────┐         ┌──────────────────────────────────┐
   │  12 GB Parquet file    │         │  400 TB lake                     │
   │  1 dashboard           │         │  + 60 jobs + 20 models           │
   │  no other reader       │         │  + IAM + CDC + backups           │
   └───────────┬────────────┘         └───────────────┬──────────────────┘
               │                                      │
      move in 2 hours                    every dependent orbits here
      cost under $2                            ╔═══════════════════╗
               │                               ║   the gravity well ║
               ▼                               ╚═══════════════════╝
        escape is easy                                 │
                                                       ▼
   PRO                                           CON
   • one copy, one source of truth                • switching cost compounds
   • locality removes latency                     • lock-in grows in silence
   • egress on a small set stays small            • blast radius grows with the lake
   • the first placement is free                  • a 50 GB choice becomes a
                                                   3-year commitment at 500 TB
```

Honest limits:

- **Gravity is a tendency, not a law.** You can move the data. The bill and the calendar tell you the price.
- **Decide the placement early.** At 12 GB the choice costs nothing. At 500 TB it is a project with a sponsor.
- **Move the query, not the table.** Federation, query pushdown, and compute over open formats in place (Parquet, Iceberg) remove part of the pull. The storage engine does the work where the bytes already sit.
- **Replicate the derived data, not the lake.** A curated table, a CDC stream, or a materialized view is far cheaper than a full copy.
- **The egress lever weakened.** Google Cloud dropped exit fees for leaving customers in January 2024. AWS and Microsoft followed in part. The EU Data Act took effect in September 2025 and requires providers to phase out switching charges. Read the wording. A waiver often applies only when the customer closes the whole account.
- **Cross-AZ and cross-region traffic is the quiet cost.** Internet egress is occasional. Replication, multi-AZ HA, and analytics pay a transfer fee on every read and every pipeline run. One 2025 analysis puts this internal traffic above internet egress in aggregate for analytics workloads.
- **Regulation can beat gravity.** A residency law forces a copy even when the physics says no. Then you run two lakes and pay for both.
- **Digital Realty publishes the Data Gravity Index (DGx).** It scores this pull across metro areas. Use it as a rough input, not a verdict.

### Example:

A fintech keeps a 400 TB analytics lake in `eu-central-1` (S3 plus Iceberg tables). The ML team wants to fine-tune a model in `us-east-1`, because GPU capacity is cheaper and easier to reserve there. The proposal looks simple. Copy the lake once, then train forever.

The transfer line kills the plan. 400 TB out of AWS at the list price of $0.09 per GB is about **$37,000**, and the copy needs days to finish. A nightly cross-AZ read makes the same point at a smaller scale. A job that reads 20 TB from another availability zone pays about **$410 per night**, or about **$12,000 per month**, at the $0.01 per GB list price in each direction.

The team does not abandon the plan. It shrinks it.

```text
THE DECISION — full copy vs curated subset
═══════════════════════════════════════════════════════════════════════════

  PLAN A  move the lake                    PLAN B  move the training set
  ──────────────────────────────           ──────────────────────────────

  eu-central-1  ─── 400 TB ───▶ us-east-1  eu-central-1
   S3 + Iceberg      $37,000                 ├── 396 TB stays. Jobs keep
                     ~days                   │   running in-region.
                     every refresh            │
                     repeats the fee          └── 4 TB curated set ──▶
                                                  $370 egress, 2 hours
  ┌────────────────────────────────────┐
  │ cost       : ~$37,000 per copy     │     ┌──────────────────────────┐
  │ time       : days, link-bound      │     │ cost   : ~$370 per copy  │
  │ refreshes  : drift between copies  │     │ time   : 2 hours         │
  │ result     : two lakes to govern   │     │ result : one lake, one    │
  └────────────────────────────────────┘     │          training copy    │
                                             └──────────────────────────┘
  PLAN C  keep the data. rent GPUs in eu-central-1. move nothing.

  the rule that survived the meeting:
    shrink the mass BEFORE the pull matters.
    move gigabytes on purpose.
    never move terabytes by reflex.
```

The curated set is the key idea. 4 TB moves in two hours for about $370. The 396 TB stays home, and every existing job keeps its locality. Training reads the copy, and the results travel back as a model file of a few gigabytes.

Two limits stay visible. First, a stale copy has a cost of its own. The training set needs a refresh policy, or the model learns last quarter's truth. Second, GPU price and GPU availability can outweigh the transfer saving. The honest comparison puts the egress quote next to the GPU quote on one page. Then the team picks the region with the lower total, and the smaller copy keeps that choice open.

The punchline: data gravity is a timer, not a wall. It grows with every byte you add and every service you attach. A placement that costs nothing at 12 GB costs a project at 400 TB. So place the data on purpose, push the compute toward it, and move only the small derived piece when a cheaper machine calls.

---

day - 7

## PagedAttention

### Definition:

**PagedAttention** is an attention algorithm that keeps the KV cache in fixed-size blocks. The blocks do not sit next to each other in memory. Each sequence owns a **block table**. The table maps the logical blocks of the sequence to physical blocks in the GPU pool. The attention kernel reads keys and values through that table.

The idea comes from the operating system. Virtual memory gives a process a flat address space while the frames live anywhere in RAM. PagedAttention gives a sequence a flat token order while its blocks live anywhere in the KV cache pool. Tokens are bytes. Blocks are pages. Sequences are processes. The paper is Kwon et al., **"Efficient Memory Management for Large Language Model Serving with PagedAttention"** (SOSP 2023, arXiv 2309.06180). The system built on top of it is **vLLM**.

**Why the naive layout fails.** A decoder generates one token per step. Every step needs the keys and values of all previous tokens. So the KV cache grows one token at a time, and nobody knows the final length when the request arrives.

Classic serving systems store the cache of a request as one contiguous tensor. Deep learning frameworks want contiguous tensors, so the scheduler reserves a chunk sized for the **maximum** length (for example 2048 tokens) before the first token exists. Three kinds of waste follow:

```
THE THREE WASTES OF CONTIGUOUS RESERVATION
════════════════════════════════════════════════════════════════════════════

  NAIVE — one contiguous chunk per request, sized at MAX length
  ─────────────────────────────────────────────────────────────────────────

   request A   max 2048, real length 300
   ┌──────────────────────────────┬─────────────────────────────────────┐
   │ 300 real tokens              │ 1748 slots RESERVED for the future  │
   └──────────────────────────────┴─────────────────────────────────────┘
   request B   max 2048, real length 900
   ┌─────────────────────────────────────────┬──────────────────────────┐
   │ 900 real tokens                         │ 1148 slots RESERVED      │
   └─────────────────────────────────────────┴──────────────────────────┘
   request C   max 512, grows to 512
   ┌─────────────────────────────────────────┐
   │ 512 real tokens                         │   external fragment      │
   └─────────────────────────────────────────┘   around the chunk

   1. RESERVED slots      unused while the request is alive, and the space
                          is blocked for other requests
   2. INTERNAL fragment   over-provision to the max length, never filled
   3. EXTERNAL fragment   the allocator (buddy allocator) leaves holes of
                          the wrong size, so a new chunk does not fit

   the paper profiled this: only 20.4% - 38.2% of the KV cache memory
   held real token states in the systems they measured
```

PagedAttention replaces the one big chunk with a pool of equal blocks and a lookup table:

```
NAIVE CONTIGUOUS vs PAGED BLOCKS
════════════════════════════════════════════════════════════════════════════

  NAIVE — the request owns one chunk, max-length sized, fixed address
  ─────────────────────────────────────────────────────────────────────────

    request A  ─────────►┌───────────────────────────────┬──────────────┐
                         │ 300 real tokens │ 1748 empty   │  one tensor  │
                         └───────────────────────────────┴──────────────┘
    request B  ─────────►┌────────────────────┬──────────────────────────┐
                         │ 900 real          │ 1148 empty               │
                         └────────────────────┴──────────────────────────┘

    contiguity is required by the old kernel, so length must be promised
    up front. the promise is the waste.


  PAGED — one shared pool of 16-token blocks, any order in memory
  ─────────────────────────────────────────────────────────────────────────

    KV cache pool    ┌────┬────┬────┬────┬────┬────┬────┬────┬────┬─────┐
    (physical blocks)│ b0 │ b1 │ b2 │ b3 │ b4 │ b5 │ b6 │ b7 │ b8 │ ... │
                     └────┴────┴────┴────┴────┴────┴────┴────┴────┴─────┘
                       ▲         ▲         ▲              ▲
                       │         │         │              │
    request A  block table:  [ b7 , b2 , b9 ]     3 blocks, 48 slots
    request B  block table:  [ b1 , b4 ]          2 blocks, 32 slots
    request C  block table:  [ b7 , b2 , b9 ]     same blocks as A -> SHARED

    a block is allocated only when the sequence needs it, 16 tokens at a time
    the last block of a sequence keeps free slots
    no full block is ever reserved for the future
    all blocks are the same size, so external fragmentation disappears
```

**One decode step through the table.** The kernel does not stream one tensor. It walks the block table of each sequence and gathers the keys and values of the listed blocks:

```
A DECODE STEP — how the kernel reads a non-contiguous cache
════════════════════════════════════════════════════════════════════════════

   new token arrives, query vector q
        │
        ▼
   ┌──────────────────────────────────┐
   │ BLOCK TABLE of request A         │
   │   logical 0 ──► physical b7      │
   │   logical 1 ──► physical b2      │
   │   logical 2 ──► physical b9      │   last block, 9 of 16 slots full
   └────────────────┬─────────────────┘
                    │
                    ▼
   ┌──────────────────────────────────────────────────────────────────┐
   │ ATTENTION KERNEL — one launch for the whole batch               │
   │   for each sequence: gather K and V from the blocks in its list │
   │   score q against every key in those blocks                     │
   │   write the new K and V into the current block                  │
   └────────────────┬─────────────────────────────────────────────────┘
                    ▼
   the layout is invisible above the kernel. the sequence still behaves
   like one continuous list of tokens.

   vLLM default block size = 16 tokens (CacheConfig.DEFAULT_BLOCK_SIZE = 16)
   the OS analogy holds: block size 16 ≈ page size, block table ≈ page table
```

**Sharing and copy-on-write.** Two sequences that hold the same prefix can point at the same physical blocks. A reference count tracks the users of a block. Read is free. Write is not. When a sequence must write into a shared block, the runtime copies the block for that sequence first, then writes. This is the copy-on-write rule from the OS, applied to the KV cache:

```
COPY-ON-WRITE ON A SHARED PREFIX
════════════════════════════════════════════════════════════════════════════

   prompt "explain this log" + 3 samples (n = 3)
   2,400-token system prompt = 150 full blocks

   ┌────────────── shared blocks, ref_cnt = 3 ───────────────┐
   │  b7  │  b2  │  b9  │ ... 150 blocks, one copy in the pool │
   └───┬───┴───┬──┴───┬──┴────────────────────────────────────┘
       │       │      │
   ┌───┴──┐ ┌──┴───┐ ┌┴────┐
   │ s1   │ │ s2   │ │ s3  │   all three read the same blocks
   └──┬───┘ └──┬───┘ └──┬──┘
      │        │        │
      │  s1 samples a token and the shared block is not full
      │        │        │
      ▼        ▼        ▼
   ┌────────────────────────────────────────────────────────────┐
   │ the runtime COPIES the shared block for s1, then writes    │
   │ b9 -> b9' for s1, ref_cnt(b9) drops by one                 │
   │ s2 and s3 keep the old block. no data is lost.             │
   └────────────────────────────────────────────────────────────┘

   after the copy, s1 and s2 stop sharing blocks. sharing pays only
   while the generated tokens stay equal.
```

**Prefix caching on top of the blocks.** A full block holds a fixed tuple of tokens, so it can be named by a hash. vLLM hashes each block over the block tokens plus the prefix before it. The components are the parent block hash, the block token ids, and extra keys (LoRA adapter id, image hash for multimodal input, cache salt for tenant isolation). A later request with the same prefix hits those blocks and skips the prefill work. Since v0.11 the default hash is `sha256`, chosen to make collisions practically impossible.

Measured numbers from the paper:

```
PUBLISHED RESULTS (vLLM paper, SOSP 2023)
════════════════════════════════════════════════════════════════════════════

  ┌───────────────────────────────────────────────┬───────────────────────┐
  │ MEASURE                                       │ VALUE                 │
  ├───────────────────────────────────────────────┼───────────────────────┤
  │ throughput vs. FasterTransformer and Orca     │ 2x - 4x, same latency │
  │ gain on longer sequences / larger models      │ gain grows           │
  │ KV memory holding real token states, before   │ 20.4% - 38.2%        │
  │ vLLM (i.e. 61.8% - 79.6% wasted)              │                      │
  │ memory saving, parallel sampling, Alpaca      │ 6.1% - 9.8%          │
  │ memory saving, parallel sampling, ShareGPT    │ 16.2% - 30.5%        │
  │ memory saving, beam search w=6, Alpaca        │ 37.6% - 55.2%        │
  │ memory saving, beam search, ShareGPT          │ 44.3% - 66.3%        │
  │ model accuracy                                │ unchanged            │
  └───────────────────────────────────────────────┴───────────────────────┘

  note the pattern: the win grows when sequences share more prefix.
  parallel sampling shares the prompt only. beam search shares more,
  because beams stay identical for many steps.
```

**Pros.**

- Near-zero waste. Only the last block of a sequence has empty slots.
- The batch size rises, and throughput follows the batch size, not the clock speed.
- Sharing is a first-class feature, not a trick. Same-prefix requests, parallel samples, and beams reuse one copy of the blocks.
- Blocks are uniform, so the allocator is a simple free list. No best-fit search for a hole of the right size.
- Model weights stay byte-identical, so the output does not change. PagedAttention is not an approximation.

**Cons and honest limits.**

- Waste is small, not zero. The last block of every sequence keeps up to 15 empty slots at block size 16. Short requests pay a higher share.
- Block size is a trade-off. Small blocks (8) waste less memory and add more table entries and gather steps. Large blocks (128) reduce table overhead and waste more memory.
- Paging fixes **capacity**, not **bandwidth**. Every decode step still reads the whole KV cache of every active sequence. The levers for that traffic are GQA, KV cache quantization, and speculative decoding. Paging does not touch them.
- The indirection is not free. The kernel reads a block table and gathers keys and values from many blocks instead of streaming one tensor. The paper reports a small cost in the isolated attention kernel and a net gain in the full system, because the batch size grows. Measure both, not one.
- A block hash collision returns the wrong cache silently. vLLM moved to `sha256` as the default for this reason. `xxhash` is faster and carries a documented collision risk.
- Sharing ends at the first divergent token. Copy-on-write then costs one copy per broken share. Workloads with early divergence gain little.
- vLLM v1 keeps the block table append-only. A duplicate block can therefore exist for a short time and is cleaned up when the request finishes. This is a deliberate speed trade, not a leak, but it shows in the block accounting.
- Paging raises the ceiling of one GPU. It does not remove the ceiling. Very long contexts still need tensor parallelism plus KV offload to CPU, NVMe, or another node.

### Example:

"BahasaKu" is a customer-support assistant for Indonesian online shops. Agents ask questions in Bahasa Indonesia, and the model answers with shop policy, order status, and canned replies. The service runs one 32B model in FP8 on a single H100 80 GB card. vLLM serves it with continuous batching and paged KV cache.

The team first computes the KV cache budget, because that number decides the whole capacity plan.

```
CAPACITY MATH — one H100 80 GB, 32B model in FP8
════════════════════════════════════════════════════════════════════════════

  weights, FP8                      ≈ 32 GiB
  activations + workspace           ≈  8 GiB
  ────────────────────────────────────────────
  left for the KV cache pool        ≈ 40 GiB

  KV bytes per token
    64 layers x 8 KV heads x 128 head dim x 2 (K and V) x 1 byte
      = 131,072 bytes
      = 128 KiB per token

  one 8,192-token conversation      = 8,192 x 128 KiB = 1 GiB
  one block (16 tokens)             = 16 x 128 KiB   = 2 MiB
  blocks in the pool                = 40 GiB / 2 MiB = 20,480 blocks

  this is arithmetic with round numbers, kept for teaching.
  the point is the shape: capacity is counted in blocks, not in requests.
```

**Before: contiguous reservation.** The old deployment reserved the maximum length for each admitted request. The prompt maximum was 8,192 tokens, and the average real length was about 900 tokens.

```
CONTIGUOUS RESERVATION — memory full, GPU mostly idle
════════════════════════════════════════════════════════════════════════════

  every admitted request reserves its max length up front
     8,192 tokens x 128 KiB = 1 GiB per request
  40 GiB / 1 GiB = 40 admitted requests  <- the reservation sets the cap
  when the allocator cannot find a clean 1 GiB hole, the real cap is lower

  real tokens in use   900 tokens x 128 KiB ≈ 112 MiB per chat
  40 chats x 112 MiB   ≈ 4.4 GiB of real token state
  ┌──────────────────────────────────────────────────────────────────────┐
  │ 4.4 GiB real tokens        35.6 GiB reserved-but-empty               │
  └──────────────────────────────────────────────────────────────────────┘

  util of the reservation ≈ 11%
  the promise of the max length, not the GPU, set the batch size
  extra requests waited in a queue. the queue set the latency.
```

**After: paged KV cache.** The same 40 GiB pool now holds blocks, and the scheduler admits requests by block count.

```
PAGED KV CACHE — the same 40 GiB, used in 16-token steps
════════════════════════════════════════════════════════════════════════════

  one 900-token answer   = ceil(900 / 16) = 57 blocks = 114 MiB
  the last block holds 4 real tokens and 12 free slots

  40 GiB / 114 MiB ≈ 350 conversations of that size

  ┌──────────────────────────────────────────────────────────────────────┐
  │ pool 20,480 blocks                                                   │
  │ ████████████████████  real token slots                               │
  │ ░░░░                  last-block slack (up to 15 slots per sequence) │
  └──────────────────────────────────────────────────────────────────────┘
```

The next gain comes free, because every agent shares the same prefix. All conversations open with the same 2,400-token instruction block: shop persona, tone rules, refund policy, and the tool schema. Prefix caching names those blocks by hash, so the pool keeps one copy.

```
PREFIX SHARING — one copy of the system prompt, many users
════════════════════════════════════════════════════════════════════════════

  shared system prompt    2,400 tokens = 150 full blocks = 300 MiB

  without prefix caching
      40 concurrent chats x 300 MiB = 12 GiB of duplicate KV blocks

  with prefix caching
      ┌──────────────── pool ────────────────┐
      │ 150 shared blocks, ref_cnt = 40      │  300 MiB total
      └───┬───────┬───────┬───────┬──────────┘
          │       │       │       │
        chat 1  chat 2  chat 3  ... chat 40      each appends its own
                                                  private blocks after
                                                  the shared prefix

  prefill skipped for the shared 2,400 tokens -> time to first token falls
  pools stay warm across requests -> the admission gate opens wider

  then chat 7 asks a question the prompt does not cover, and the very
  first divergent token breaks the share.
  ┌──────────────────────────────────────────────────────────────────────┐
  │ copy-on-write: the runtime copies the last shared block for chat 7   │
  │ only. chats 1-6, 8-40 keep the shared blocks. cost = one 2 MiB copy. │
  └──────────────────────────────────────────────────────────────────────┘
```

The operational numbers that matter, before and after:

```
BEFORE / AFTER ON THE SAME CARD
════════════════════════════════════════════════════════════════════════════

  ┌────────────────────────────┬───────────────────┬─────────────────────┐
  │ METRIC                     │ CONTIGUOUS        │ PAGED (vLLM)        │
  ├────────────────────────────┼───────────────────┼─────────────────────┤
  │ admitted conversations     │ ~40 (8k reserve)  │ ~350 short chats    │
  │ reservation util           │ ~11%              │ >96% of block slots │
  │ duplicate prompt copies    │ one per chat      │ 1 shared copy       │
  │ prefill work for the       │ repeated per chat │ skipped on a hit    │
  │ 2,400-token system prompt  │                   │                     │
  │ model accuracy             │ baseline          │ identical           │
  └────────────────────────────┴───────────────────┴─────────────────────┘
```

Three checks keep the win real, and each one came from a real failure mode:

- Watch the block accounting. `vllm:gpu_cache_usage_perc` near 1.0 means the pool is the limit, not the GPU. A pool that is full while GPU compute sits at 40% says the block size or the admission rule is wrong.
- Watch the prefix hit rate. A hit rate near zero after a persona change means the prefix is no longer identical. One edited character in the system prompt invalidates 150 blocks.
- Watch the last-block slack on short traffic. A service of 60-token replies spends 15 of every 16 slots in its final block. At that shape, block size 8 or 16 beats block size 128.

The honest limits stay on the table:

- A very long chat still does not fit. A 128k-token conversation needs 16 GiB of KV cache, and no pool trick changes that. That traffic needs KV quantization, tensor parallelism, or offload to CPU and disk.
- Sharing is not a saving on unique content. A workload with a unique prompt per request gains memory efficiency, not sharing.
- Decode bandwidth is untouched. Every step reads all active KV blocks. Turn on GQA-friendly serving, KV FP8, and speculative decoding for that.
- The prefix cache must be treated as a cache with privacy rules. A cache salt per tenant isolates caches, and without it one tenant can read another tenant's shared blocks. vLLM exposes this as an export hook for prefix cache keys.

The punchline: PagedAttention does not shrink the KV cache. It **stops reserving memory that no token has claimed**. One tensor per request forces a promise of the maximum length, and that promise, not the GPU, was setting the batch size. Replace the promise with a block table, allocate 16 tokens at a time, and the same card serves the same work with a much larger batch. Then add sharing on top, because identical prefixes are the normal case in real traffic. Keep the last-block slack and the copy-on-write cost in view, and the capacity plan becomes boring arithmetic on blocks, which is the entire point.

---