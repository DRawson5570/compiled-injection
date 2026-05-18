# Cognitive Grafting: Cross-Model MLP Substitution via Learned Projection Adapters

**Author:** Douglas Rawson
**Date:** May 2026
**Status:** Public results write-up. Full training recipe, hyperparameters, and reference implementation are withheld pending responsible disclosure. See [README.md](README.md#responsible-disclosure).

---

## Summary

We replace a single transformer MLP block in a small same-family frontend model with the corresponding MLP block from a strictly larger backend model, mediated by a pair of small learned projection adapters that bridge the hidden-dim gap. The bridge is trained against a **substitution objective** — making the bridged forward pass behaviorally match the original frontend MLP — rather than against a latent-reconstruction objective. At a single middle layer of both models, **pure substitution costs only +3.1% perplexity** on a held-out language-modeling slice, while a small blend slightly **beats** the unmodified frontend baseline. Greedy generation under pure substitution remains coherent on factual recall, code, and short-story prompts. The bridge is small (tens of megabytes), trains in minutes on a single consumer GPU, and is gradient-free at deployment after the one-time fit.

**Implication.** A small frontend (cheap, fast attention and lm_head) can lease specific MLP blocks from arbitrarily larger same-family backends as drop-in expert modules, without retraining either model. *Small-model speed, big-model knowledge*, plugged in at the MLP boundary.

A live demonstration runs in the **Hybrid Chat** tab of <https://compiled-injection-demo.aurora-sentient.net/>.

---

## 1. Setting

- **Frontend (FE).** A small instruction-tuned transformer (sub-1B parameter class). The FE keeps its own attention, embeddings, layer norms, residual routing, and output head throughout.
- **Backend (BE).** A strictly larger same-family instruction-tuned transformer (1–2B class). The BE is **frozen** at all times.
- **Splice point.** One MLP block, at a single chosen layer index, in both models. The FE's residual stream enters the splice; the BE's MLP at the same relative depth is executed in its native hidden width; the bridge translates back to the FE's residual stream.

The two hidden widths are different (the FE is narrower). A pair of small learned adapters bridges the gap in both directions.

## 2. The Bridge

Two adapters, each a small SiLU MLP:

- A **forward adapter** that maps an FE residual-stream slice to the BE's MLP input space.
- An **inverse adapter** that maps a BE MLP output back into the FE's residual stream.

Per-feature input/output statistics are captured once from the frontend MLP under the training distribution and registered as fixed normalization buffers around the bridge.

Total bridge size: tens of megabytes — independent of either model's checkpoint.

## 3. Training Objective

The key design decision is the loss. We train against a **substitution objective**: the entire `bridge → BE_MLP → bridge` pipeline is asked to behave like the original FE MLP would have behaved on the same residual-stream slice. The BE MLP is frozen; gradients flow only through the two adapters.

We earlier tried, and abandoned, a cycle-consistency autoencoder formulation that achieved low latent reconstruction error but caused the assembled model's perplexity to explode by many orders of magnitude. The methodological takeaway is the central point of this work and is stated bluntly:

> **Bridges between heterogeneous neural components must be trained against the downstream behavior they replace, not against internal reconstruction.**

Latent consistency between two representations is necessary but vastly underdetermined: it leaves the bridged module free to occupy *plausible-looking* but behaviorally arbitrary regions of the host's residual stream. Pinning the bridge to the behavior of the component it replaces removes that freedom.

Full training recipe — exact hyperparameters, normalization scheme, optimizer settings, schedule, dtype handling, and reference implementation — is withheld in this public write-up.

## 4. Runtime: Blended Substitution

The bridge is installed as a wrapper around the FE MLP at the splice layer, exposing a single continuous knob `β ∈ [0, 1]`:

- `β = 0` is the unmodified frontend.
- `β = 1` is pure substitution: the FE MLP's output never participates in the residual stream at that layer — the bridge alone drives layer-14 MLP behavior.
- Intermediate values blend the two outputs linearly.

A NaN guard falls back to the original FE MLP for any token where the bridged forward pass produces a non-finite value (rare; a fraction of a fraction of a percent in practice).

## 5. Results

### 5.1 Bridge-internal quality

On a held-out validation slice of the captured paired vectors:

| Metric | Value |
|---|---|
| Bridge val MSE vs predict-mean baseline | **≈ 4× better** |
| Per-feature Pearson correlation (bridged vs original FE MLP output) | **+0.85** |

Per-feature Pearson is the more honest progress metric: it tracks "does the bridged output point the same direction the original FE MLP wanted to point", independent of the per-feature energy absorbed by the normalization buffers.

### 5.2 Held-out perplexity, assembled model

Held-out language-modeling slice, frontend model with its layer-14 MLP replaced by the bridge:

| Blend β | Δ perplexity vs unmodified frontend |
|---:|---:|
| 0.0 (baseline) | — |
| 0.1 | **slightly better than baseline** |
| 0.3 | +0.07% |
| 0.5 | +0.51% |
| 0.7 | +1.27% |
| **1.0 (pure substitution)** | **+3.11%** |

Two qualitative observations matter more than any individual cell of the table:

1. **A small dose of the larger model's MLP at one layer is a net positive on the small model's own held-out loss.** The big model knows things the small model does not, and a 10% blend at one MLP block is enough to surface some of it.
2. **Pure substitution costs single-digit perplexity.** The small model's own MLP at layer 14 is completely replaced — its output never enters the residual stream at that layer — and the perplexity penalty is in the low single digits.

### 5.3 Generation under pure substitution

Greedy decoding under `β = 1.0` produces coherent free-form text:

- Factual prompts complete correctly ("capital of France" → "Paris"; "chemical symbol for gold" → "Au"; "the mitochondria is the" → "organelle that is responsible for the production of ATP").
- Short narrative prompts produce coherent stories.
- Simple code prompts produce valid Python.

Coherence under pure substitution at single-layer granularity is the qualitative claim that backs the perplexity number.

## 6. Why It Works

Transformer MLP blocks are routinely described in the interpretability literature as approximate position-wise associative memories on top of a shared residual stream. If we accept that framing, an MLP block's interface with the residual stream at a given layer is essentially a position-wise functional map. For two same-family models trained on broadly overlapping data, the map at corresponding layers is closely related, and the relation can be captured by a small learned projection pair fitted against the behavior of the smaller map.

What is **not** claimed here:

- That this works across arbitrary model families without modification. (Cross-family is the obvious next experiment; we expect it to require either a larger bridge, family-aware feature alignment, or multi-layer joint training.)
- That stacking many simultaneous bridges across many layers composes for free. (Joint training is the obvious extension.)
- That the bridge transfers the *full* capability of the larger model. (It transfers what the small model's residual stream is able to read out at one layer; that is enough for the results above, but it is not the full LM.)

What **is** claimed:

- A single MLP block at one layer can be substituted across a hidden-dim gap by a small learned bridge trained against the right objective.
- The right objective is behavioral substitution, not latent reconstruction.
- The cost is in the low single digits of perplexity for pure substitution and is actually favorable at low blend.

## 7. Live Demo

The bridge runs in production at the **Hybrid Chat** tab of <https://compiled-injection-demo.aurora-sentient.net/>. You can ask the same prompt to:

- The small frontend alone.
- The hybrid: small frontend with the bridged backend MLP spliced in at the chosen layer, with a blend slider from 0.0 to 1.0.

Side by side, in the browser, no setup required. Single Tesla M40 GPU on the back end.

## 8. Implications and Responsible Disclosure

A bridge of this kind is a deployment primitive with real economic consequences: it makes it possible to ship a small frontend and trade off cost vs. capability at decode time by leasing specific MLP blocks from a larger backend on demand. It is also a control primitive: the bridge is a small, inspectable artifact that sits at a clean architectural seam.

Full training recipe, exact hyperparameters, normalization scheme, reference implementation, and the multi-layer / cross-family extensions are withheld in this public document. Interested AI-safety, AI-control, or capability-research organizations (we are in active outreach with Apollo Research, xAI, Anthropic, FAR AI, and Redwood Research) are welcome to open an issue or contact <rawson.douglas@gmail.com> for further discussion under appropriate terms.

---

## Citation

```bibtex
@misc{cognitive-grafting-2026,
  author = {Douglas Rawson},
  title  = {Cognitive Grafting: Cross-Model MLP Substitution via Learned Projection Adapters},
  year   = {2026},
  url    = {https://github.com/DRawson5570/compiled-injection},
  note   = {Public results write-up; full methodology withheld pending responsible disclosure.}
}
```
