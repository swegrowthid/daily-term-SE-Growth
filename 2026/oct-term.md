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

day - 3

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