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