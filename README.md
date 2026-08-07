---
title: Local LLM Hardware Explorer
emoji: 🖥️
colorFrom: indigo
colorTo: gray
sdk: static
app_file: index.html
pinned: false
license: mit
---

The webpage is hosted on Huggingface Spaces: 
https://huggingface.co/spaces/HWu0101/Local-LLM-Hardware-Explorer

# Local LLM Hardware Explorer

Which open-weight models can this machine actually run, how fast, and what would an upgrade change?

A single self-contained HTML page. Model quality on the vertical axis, measured or modelled decode
speed on the horizontal, bubble size as Q4 file size, and a hardware picker that repositions every
model when you change machines. Click any model for its own price-vs-speed staircase.

## Running it

Open `index.html` in a browser. That is the whole procedure — no build step, no server, no network
access, no dependencies. There is not a single `<script src>` or `<link>` tag in the file.

If you prefer to serve it:

```bash
python3 -m http.server 8000
```

## What it models

```
t/token = bytes_read × [ f / BW_fast + (1 − f) / BW_slow ] + overhead
```

where `f` is the fraction of active bytes resident in fast memory. Speed-up over CPU-resident experts
goes as `1/(1 − f)`, which is why the last few percent of residency are worth far more than the first.

Bandwidth figures are **effective batch-1 decode throughput back-calculated from published token
benchmarks** — never spec sheets, and never streaming rooflines, which are roughly twice what MoE
decode converts. Against 14 published measurements the model has a leave-one-out median error of
**5.8%** (worst case 34%).

Solid dots are published measurements; dashed rings are modelled. Hollow rings are proxy quality
scores. Each machine's detail panel carries a bottleneck audit — memory capacity, residency, KV
cache, GPU and system bandwidth, PCIe, GPU and CPU compute, kernel efficiency, per-token overhead —
derived from the same numbers that produce the speed, so the audit cannot disagree with the chart.

## Quality data and attribution

Quality scores come from the **Arena leaderboard dataset**, three lenses:

| Lens | What it measures | Models rated here |
|---|---|---|
| Arena text | style-controlled human preference on ordinary chat | 24 of 30 |
| Arena WebDev | the same method on front-end build tasks | 16 of 30 |
| Arena Agent | standardised score on agentic sessions, 0 = field average | 10 of 30 |

Coverage is shown beside the metric selector, and models unrated on the active lens are hidden and
counted under the chart. There is deliberately **no blended score**: the lenses disagree, and that
disagreement is the useful signal.

> Rating data from [`lmarena-ai/leaderboard-dataset`](https://huggingface.co/datasets/lmarena-ai/leaderboard-dataset)
> by **Arena Intelligence, Inc.**, used under [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/).
> **Changes made:** a subset of models was selected; values were reformatted for plotting and rounded
> for display; no rating was altered. Latest split read 27 July 2026.
> Cite: Chiang et al., *Chatbot Arena*, [arXiv:2403.04132](https://arxiv.org/abs/2403.04132).

These are **Bradley–Terry** ratings, not Elo, despite the common label. Taken from the dataset
release rather than scraped from the site.

Two Qwen3.6 entries carry their identically-sized Qwen3.5 sibling's rating as a labelled proxy,
drawn hollow, because no Qwen3.6 open-weight model has been rated. `GLM-4.7-Flash` deliberately has
no WebDev rating: the board carries `glm-4.7`, which is a different model.

**This page is not affiliated with or endorsed by any benchmark operator.** Scores remain the work of
their respective operators.

### Note on what is absent

An earlier private build used the Artificial Analysis Intelligence Index, which rates all 30 models
here rather than 24. It is not in this version because AA's Terms of Use grant personal,
non-commercial use only and forbid redistributing content from their site. Individual scores are
uncopyrightable facts, so the binding constraint is contract rather than copyright — which is why
removing them is sufficient, and why data taken from primary sources is unaffected. If you want that
lens, use AA's own [embed](https://artificialanalysis.ai/embed/llm-leaderboard).

## Known limits

- Batch-1, Q4-class quantisation only. Sub-Q4 usually costs more quality than the speed is worth.
- **Prefill figures assume a tuned batch** (`-ub 8192 -b 8192`). llama.cpp ships `-ub 512`, which pays
  the offload transfer sixteen times more often; where that matters the panel prints both numbers and
  the stock-settings one can be several times lower on identical hardware. Decode is unaffected.
- Clustering unified boxes (two DGX Sparks over ConnectX, Macs over Thunderbolt) is modelled as
  **capacity only**: at batch 1 pipeline parallelism runs one node at a time, so the cluster reads the
  model at a single node's bandwidth. It runs models no one box holds; it does not run them faster.
- One averaged figure is used for idle system reserve (14 GB RAM, 2 GB VRAM) rather than a per-OS one.
- Prices are sourced as **AUD retail** and switchable to USD at a dated constant rate (0.6973, 24 Jul
  2026) so the page works offline. AU retail includes 10% GST while US list prices normally exclude
  sales tax, so a converted figure sits above the US sticker you would compare it against.
- **Memory is the volatile line.** Server DDR5 ECC RDIMM is priced at A$62/GB (re-checked 7 Aug 2026;
  8×32 GB quotes at A$15–17k) and is still climbing as AI-server demand takes the high-capacity
  modules. On a large-memory build this is usually a bigger number than the CPU and the cards
  combined, so re-check it before deciding anything. Used DDR4 sits on a separate market at
  A$12.5/GB, which is the entire reason second-hand EPYC 7003 boxes dominate the value end.
- Many cards carry `bw est`, `price est` or `used market` flags in the picker, shown per card. Cards
  without a published decode benchmark have their effective bandwidth estimated from nominal at the
  conversion rate the calibrated cards exhibit (~55% consumer, ~45% RTX PRO); that rule lands within
  ~12% on the two cards where a real measurement exists.
- Kernel work moves decode benchmarks fast — one RTX PRO 6000 figure gained 43% in five months.
- Threadripper and Xeon memory figures are **modelled, not measured**: no published llama.cpp
  MoE-offload token benchmark for those platforms could be found as of 27 July 2026.
- Tensor-parallel gains assume working peer-to-peer, which GeForce cards do not have.
- Two card bandwidths are flagged `(est)` in the picker.

Every figure carries its date in the provenance table at the bottom of the page.

## Licence

Code and the performance model: **MIT** (see `LICENSE`).
Compiled data and layout: **CC BY 4.0**.
Underlying benchmark scores remain the property of their operators and are used under the terms above.
