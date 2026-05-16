# Compiled Injection — Technical Manual

**Deterministic Activation Engineering for Pretrained LLMs**

*Douglas Rawson — May 2026*

---

## Theory: How It All Works

### The Residual Stream: Where Everything Flows

A transformer decoder layer computes:

```
hidden_state → [self_attention] → + → [layer_norm] → [FFN/MLP] → + → next layer
                ↓                          ↓
           (residual connection)      (residual connection)
```

At the final position (the last token), the hidden state encodes everything the model
knows about the sequence. The **lm_head** projects this state to vocabulary logits:

```
logits = lm_head(layer_norm(hidden_state[-1, :]))
next_token = argmax(logits)
```

This is why we inject at the **last position**: all token predictions flow through that
single vector. Modifying it modifies what the model predicts next.

### Why Unembedding Rows Are Steering Vectors

The lm_head is a linear projection: `logits = W @ h` where `W` is [vocab_size, hidden_size].
Each row `W[t, :]` is the **unembedding vector** for token `t`. When the hidden state `h`
aligns with `W[t, :]`, the logit for token `t` is maximized.

If we add `alpha * W[t, :]` to the hidden state:

```
h' = h + alpha * W[t, :]
logits' = W @ h' = W @ h + alpha * W @ W[t, :]
```

The term `W @ W[t, :]` adds `alpha * ||W[t,:]||^2` to the logit of token `t`. The model
strongly prefers `t`. This is the **mathematical justification** for using unembedding
rows as steering vectors.

### The Compound-Probability Collapse

Why static injection fails for multi-token sequences:

A static delta applies the SAME correction at every decode step. For a sequence of
N tokens, the probability of generating the full target is:

```
P(correct_sequence) = P(tok_1 | context) × P(tok_2 | context+tok_1) × ... × P(tok_N | ...)
```

Each factor is independently < 1 for any injection weaker than complete dominance.
The compound product decays **exponentially** with sequence length.

This was empirically confirmed: low alpha keeps the baseline behavior; high alpha
collapses to the same token winning every position (e.g., `"getgetgetget...`)
because the unchanging delta keeps winning the argmax for the same token.

### Why Per-Step Injection Solves It

Per-step injection replaces the compound product with N independent single-token problems.
Each delta only has to win at ONE position. The swap between forward passes prevents
the previously-injected token from re-winning. The probability of the full sequence
is limited by the MINIMUM per-token probability, not the product — linear degradation
instead of exponential.

### Attention vs. FFN: Two Axes of Control

| Property | Attention Injection | FFN Injection |
|----------|-------------------|---------------|
| Operation | Mixes information across positions | Transforms information at each position |
| Effect | Sets "protocol class" (mode of response) | Provides per-position token precision |
| Temporal characteristic | Static delta works (same mode for whole generation) | Needs per-step delta (different output per position) |
| Best layer | Middle (8–18) | Late (18–23) |

Attention injection shifts **branch selection** ("am I writing JSON or prose?").
FFN injection provides **token precision** ("this exact token at this exact position").

### The Selective Injection Principle

An injection mechanism has three critical properties:

1. **WHAT** is injected (the steering vector)
2. **WHERE** it's injected (the layer and position)
3. **WHEN** it fires (the trigger condition)

Neglect any one and the mechanism breaks. The trigger is the most commonly neglected.

### Why It Doesn't Need Gradient Descent

Gradient descent modifies ALL parameters to minimize a loss function across ALL examples.
It's undirected — the optimizer doesn't know which parameters matter for which behavior.

Compiled injection modifies EXACTLY the parameters that matter at EXACTLY the right
positions with EXACTLY the right delta (computed from the target, not discovered
through optimization). This is deterministic, bounded, reversible, and auditable.

---

## Mechanism Catalog

### 1. Per-Step Token Injection

**Purpose**: Make the model output an exact multi-token sequence.

**How**: At each decode step, add a different steering vector to the model's activations
at the last position, guiding the next token toward the corresponding target token.

**Key parameters**: Target layer, alpha (steering strength), trigger condition.

**Effective for**: Exact phrase output, API recall, fact injection.

### 2. Positive/Negative Anchoring

**Purpose**: Shift the model's behavioral protocol — not specific tokens, but the class
of output (prose vs. JSON, hesitant vs. assertive, etc.).

**How**: Static attention deltas (no per-step swapping) at specific layers. Positive
anchors pull toward desired behavior. Negative anchors push away from undesired behavior.
Combined, they provide complete protocol control.

**Effective for**: Consistent formatting, response style, mode selection.

### 3. Text Injection (CodingEngine Pattern)

**Purpose**: Interrupt the model mid-generation and inject corrective text.

**How**: A sliding-window detector identifies trigger patterns in the model's output.
When triggered, the system appends corrective text to the context. The injected text
ends with the same pattern that triggered it, allowing seamless resumption.

**Effective for**: Code fix redirection, methodology injection during analysis.

### 4. Composed Injection

**Purpose**: Combine protocol-level and token-level control.

**How**: Install both Attention injection (static, sets mode) and FFN injection
(per-step, provides tokens) simultaneously. Co-tune alphas to prevent one mechanism
from dominating the other.

**Effective for**: Complex behaviors requiring both correct format and exact content.

### 5. Logit Biasing

**Purpose**: Simple token-level control via logit vector manipulation.

**How**: Boost or suppress specific token IDs in the output logits before sampling.
Per-step biasing avoids the compound-probability collapse.

**Effective for**: Safety filtering (suppress dangerous tokens), guaranteed token output.

### 6. Compiled FFN Rules

**Purpose**: Permanent weight-level trigger→response rules.

**How**: Expand FFN intermediate dimensions with gate (trigger detection), up (value
projection), and down (output projection) matrices that implement deterministic
lookup rules.

**Effective for**: Knowledge bases, diagnostic lookup tables. Empirically holds
50,000+ rules with zero cross-talk.

### 7. Concept Compilation Pipeline

**Purpose**: Systematic delta computation from multiple (prompt, response) examples.

**How**: Capture FFN input residuals for prompt-only and prompt+target runs. Solve for
optimal low-rank weight delta via ridge-regularized pseudo-inverse + SVD truncation.

**Effective for**: Compiled knowledge from structured datasets (facts, rules, procedures).

### 8. The Plugin System

**Purpose**: Stackable, reversible hooks for combining multiple injection mechanisms.

**How**: Each plugin implements hooks for `pre_generate`, `on_logits`, and `post_generate`.
Plugins are ordered, composable, and reversible via context managers.

**Effective for**: Production systems combining safety, knowledge, and formatting plugins.

---

## Layer & Alpha Tuning

### Layer Selection

| Layer Range | Role | Best For |
|-------------|------|----------|
| 0–7  | Low-level token features | Rarely used |
| 8–15 | Semantic representation, protocol class | Attention injection, behavioral shifts |
| 14–18 | Transition zone | Composed injection |
| 16–23 | Close to output | Per-step FFN injection |
| 20–24 | Final prediction refinement | Last-resort reinforcement |

### Alpha Ranges

| Alpha | Effect |
|-------|--------|
| 0.1–1   | Subtle nudge (protocol shifts) |
| 1–10    | Noticeable bias |
| 10–100  | Moderate steering |
| 100–500 | Strong steering (exact tokens) |
| 500+    | Over-fire risk |

### Model Size Scaling

| Model | Typical Alpha (per-step) |
|-------|--------------------------|
| 0.5B  | 200–400 |
| 1.5B  | 64–256  |
| 7B    | 128–512 |
| 7B 4-bit | ~2× fp16 (quantization noise floor) |

---

## Common Failure Modes

| Symptom | Cause | Fix |
|---------|-------|-----|
| Target tokens don't appear | Alpha too low | Increase by 50–100 |
| Model loops ("the the the...") | Alpha too high | Halve. Verify per-step vectors differ. |
| All prompts produce target output | No trigger gating | Add trigger check |
| First token correct, rest wrong | Context shifts away from steering | Try earlier layers |
| Garbled output / extreme token collapse | Extreme alpha corrupting residual | Reduce alpha by 75% |
| Output unchanged | Injection not installed or cleared | Verify hook placement |

---

## Deployment Options

### Option A: Fine-Tune Validation

Validate fact via compiled injection first (proves learnability). Then apply minimal
gradient steps to make it permanent for standard serving.

*Trade-off*: Full compatibility, but causes some forgetting on single examples.

### Option B: Persistent Buffers

Store injection configuration as persistent model buffers. On load, re-apply wrappers
via a custom loader.

*Trade-off*: Needs a non-standard loading function.

### Option C: Proxy Wrapper

Deploy a proxy in front of the serving endpoint that intercepts requests and applies
injection before forwarding.

*Trade-off*: Adds latency. Requires separate proxy deployment.

### Option D: Baked FFN Rules

Expand FFN weights with trigger-gated dimensions. Survives standard loading.

*Trade-off*: Architecture-specific. Complex to implement.

---

## History: How We Figured It Out

### The Eight Failed Approaches

The initial question: can we make a pretrained model say a specific multi-token
sequence without retraining? Eight static approaches were tested and failed:

1. Static FFN delta at one layer
2. Static attention delta at one layer
3. Static lm_head bias
4. Static embedding bias
5. Prompt injection (append target to context)
6. Logit biasing (boost target tokens globally)
7. Static delta at multiple layers simultaneously
8. Static delta with per-token weighting

**All eight failed** for multi-token targets. The consistent failure mode: at low alpha,
no effect. At high alpha, the same injected token won every position.

**Diagnosis**: A static delta must win the argmax at every position simultaneously.
Nothing varies it across decode steps. This is the *compound-probability collapse*.

### The Breakthrough: Per-Step Injection (May 2026)

**Hypothesis**: Making the delta a function of the decode step would break the collapse.

**Result**: A stateful wrapper that swaps the injected delta between forward passes
produced exact 4-token sequence matches where all static approaches failed. Scale
validation confirmed the mechanism works across 1.5B fp16 and 7B 4-bit models.

This established that **per-step delta swapping is the missing degree of freedom**
for multi-token sequence injection.

### Attention Injection for Protocol Control (May 2026)

Extended the wrapper pattern to attention layers. Found that static attention deltas
are sufficient for protocol-level shifts (language, formatting, mode). Unlike FFN,
attention controls branch selection — so it doesn't need per-step swapping.

### Composition: Both Axes Together (May 2026)

Combined attention (protocol) + FFN (tokens) in one generation pass. Discovered
that solo-tuned alphas over-dominate when composed. The co-tuning rule: reduce
attention alpha by 4× in the composed regime.

### Concept Compilation (May 2026)

Extended from hand-tuned steering vectors to systematic delta computation via
pseudo-inverse + SVD. Proved that rank-k FFN patches change output behavior.
Demonstrated positive/negative anchoring for complete behavioral control.

### Multi-Token Behavioral Injection (May 2026)

The CodingEngine pattern: sliding-window trigger detection + stateful text injection.
Redirected a model from writing incorrect code fixes to correct ones by detecting
the fix pattern and injecting methodology guidance.

### Plugin System & Persistence (May 2026)

Generalized the wrapper pattern into a stackable, reversible plugin architecture.
Extended to persistent storage: compiled deltas survive model save/load roundtrips.

---

## References

| Resource | Content |
|----------|---------|
| `docs/COMPILED_INJECTION_MANUAL.md` (private repo) | Full implementation manual with code examples |
| `DeepSeek_README.md` (private repo) | Complete mechanism toolbox and session state |
| `docs/EXPERIMENT_LOG.md` (private repo) | Full experiment history from Entry 140 onward |
| `compiled_inject.py` (private repo) | Working end-to-end AttentionInject example |
| `mneme/` (private repo) | Concept compilation pipeline |

---

*This document establishes prior work on Compiled Injection techniques as of May 2026.*
*Douglas Rawson — rawson.douglas@gmail.com*
