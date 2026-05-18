# Compiled Injection

Live demo: https://compiled-injection-demo.aurora-sentient.net/

**Deterministic Activation Engineering for Precise Behavioral Control in Pretrained LLMs**

Gradient-free. No fine-tuning. No SGD. Targeted, reversible, and auditable modifications to model behavior via activation-space injection.

---

## Overview

This repository documents **Compiled Injection** — a suite of techniques for surgically editing the behavior of pretrained large language models by directly manipulating activations in the residual stream.

Instead of retraining models with gradient descent (which causes catastrophic forgetting), these methods compile specific behaviors, facts, and response protocols directly into the model's forward pass using steering vectors and lightweight wrappers.

## Core Innovations

- **Per-Step Token Injection**: Solves the *compound-probability collapse* problem that plagues static steering methods, enabling reliable multi-token exact outputs.
- **Composed Injection**: Combines Attention injection (for behavioral protocol/mode shifts) with FFN injection (for token-level precision).
- **Positive/Negative Anchoring**: Persistent protocol-level control (e.g. consistent JSON formatting, assertiveness, etc.).
- **Trigger Design**: Context-aware activation of injections, preventing unwanted side effects.
- **Persistence & Baking**: Methods for making selected modifications survive model saving/loading.

## Key Advantages

- Preserves original model capabilities
- Highly deterministic and auditable
- Works on already-trained models (including quantized)
- Reversible via context managers
- No gradient descent required

## Repository Status (May 2026)

This is the public home for the theoretical foundation and high-level methodology of Compiled Injection techniques.

**Full technical implementation details are currently withheld** pending responsible disclosure and collaboration with AI safety / control research organizations.

## Contents

- `COMPILED_INJECTION_MANUAL.md` — Comprehensive theory, history, and mechanism overview
- `mneme/` — Core concept compilation pipeline (rank-k delta synthesis via pseudo-inverse + SVD)
- `examples/` — High-level demonstrations and proof-of-concept code

## Philosophy

Knowledge and behaviors can be **compiled** into transformer weights and activations in a targeted, reversible manner — much like writing firmware rather than retraining the entire system.

## Citation / Credit

If you reference this work, please use:

```bibtex
@misc{compiled-injection-2026,
  author = {Douglas Rawson},
  title = {Compiled Injection: Deterministic Activation Engineering for Pretrained LLMs},
  year = {2026},
  url = {https://github.com/DRawson5570/compiled-injection}
}
```

**Note**: The core techniques were developed in May 2026. This repository serves as the public timestamp and documentation of that work.

---

## Responsible Disclosure

I am actively reaching out to leading AI safety and control organizations (Apollo Research, xAI, Anthropic, FAR AI, Redwood Research, etc.) for responsible discussion and potential collaboration.

Interested researchers and organizations are welcome to open an issue or reach out via email for further discussion under appropriate terms.

---

**This is early-stage public documentation of ongoing research.**

Star the repo if you're interested in activation engineering, mechanistic interpretability, or AI control.

---

## Contact

- GitHub Issues (preferred for public discussion)
- Email: rawson.douglas@gmail.com

---

*Built with curiosity and a healthy respect for the power of what's possible in the residual stream.*
