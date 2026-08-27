# Findings

One claim per document. Each states the claim, the evidence behind it, and what it cost to
learn.

| # | Claim | Confidence |
|---|---|---|
| [01](01-single-agent-wrapper.md) | The multi-agent framework was wrapping a single agent. The coordination was hand-written. | **High** — grep-verifiable |
| [02](02-evidence-vs-testimony.md) | Numbers written by LLM-authored code cannot be adjudicated on. A fixed code layer must own the evidence. | **High** — measured across 29 runs |
| [03](03-silent-success.md) | The dangerous failures report success. `exit code 0` is not a success signal. | **High** — five distinct instances |
| [04](04-papers-without-results.md) | Every failed experiment still produced a paper: 16 / 16. | **High** — complete enumeration |
| [05](05-long-horizon.md) | Keeping multi-hour runs alive was the bulk of the engineering, and is absent from short-horizon benchmarks. | **Medium** — single-operator observation |
| [06](06-why-this-stopped.md) | The comparison was untestable by construction, not merely unfinished. | **High** — follows from 01 |

## How to read these

Every quantitative claim names where it came from. Where a number is weak — small n, single
operator, uncontrolled conditions — it says so in place rather than in a footnote.

Findings 01 and 06 are the same story told forwards and backwards: 01 is what the code
turned out to be, 06 is what that did to the research question.
