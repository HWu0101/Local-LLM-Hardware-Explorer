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
decode converts. Against 16 published measurements the model has a leave-one-out median error of
**5.6%**, worst case **34%**.

Both numbers are measured under strict leave-one-out, which an earlier harness was not doing: it
cleared the calibration cache but left the held-out measurement in place, so `calc()` returned the
measurement itself for the pair under test and reported a flattering 34% tail. The median barely
moved when that was fixed (5.3% → 5.9%); the tail did. Treat the median as the useful figure and the
tail as a warning that a small coarse-MoE model on a small card can be badly under-predicted.

### Architecture-aware memory

Two things in a model file do not cost what their size suggests, and both are modelled per model:

- **KV cache from each model's own config.** Only full-attention layers grow a cache with the
  context. Sliding-window layers stop at their window (Gemma 4, gpt-oss, MiMo, Step, Inkling,
  Beam, Muse, Command A+, K-EXAONE); Gated DeltaNet, Kimi Delta Attention and Mamba layers keep a
  fixed state (Qwen3.5 onward, Kimi K3, Ling 3.1, GLM-5.3 Flash, Nemotron 3); MLA stores a
  compressed latent rather than per-head keys and values (DeepSeek V3.2, Kimi K2, Mistral Large 3
  and Small 4, GLM-5, Hy4, A.X-K2), and sparse-attention indexers add their own key. The result
  spans 0.9 KB a token (DeepSeek V4.1 Flash) to 188 KB (GLM-4.7) at a Q8 cache. Every model had
  been sized as a plain full-attention transformer from its parameter count, which was off by up
  to 25x; correcting it cut the leave-one-out 90th-percentile error from 34% to 11% and the worst
  case from 46% to 34%.
- **Lookup tables.** Qwen3.8-Flash-Next is a 125B-A6B model plus a separate 51B n-gram embedding
  table, and DeepSeek V4.1 Flash a 552B backbone plus 196B of "Engram" memory. A token indexes a
  few rows of either, so the table can stay memory-mapped on SSD and only the model proper has to
  sit in VRAM + RAM. NVIDIA's serving recipe for Flash-Next keeps the table in host RAM instead,
  which is faster when RAM is spare.

Reading the configs also corrected Kimi K3 to 104B active (from 50B), which halves its modelled
decode speed.

### Residency is split in two

An MoE has weights that are read on *every* token — attention, embeddings, any shared expert — and
routed experts that are read only when selected. llama.cpp exploits this: `--n-cpu-moe` keeps the
attention stack on the GPU and pushes experts to system RAM. The model therefore tracks two numbers:

- **`f`** — the fraction of *active bytes* in fast memory. This drives speed.
- **`fRouted`** — the fraction of the *routed expert file* resident. This drives capacity, the fit
  colour, and how much has to stream during prefill.

Collapsing those into one number was a real error: on a 16 GB card six models reported `f = 0`,
i.e. "every byte comes from system RAM", when in practice their attention layers sit on the GPU and
only the experts stream. Fixing it moved the worst-case leave-one-out error from 57% to 46% and
brought a user-measured Qwen3.5 122B-A10B on an RTX 5080 from 27% under to 6% under.

Solid dots are published measurements; dashed rings are modelled. Hollow rings are proxy quality
scores. Each machine's detail panel carries a bottleneck audit — memory capacity, residency, KV
cache, GPU and system bandwidth, PCIe, GPU and CPU compute, kernel efficiency, per-token overhead —
derived from the same numbers that produce the speed, so the audit cannot disagree with the chart.

## Quality data and attribution

Quality scores come from the **Arena leaderboard dataset**, three lenses:

| Lens | What it measures | Models rated here |
|---|---|---|
| Arena text | style-controlled human preference on ordinary chat | 46 of 64 |
| Arena WebDev | the same method on front-end build tasks | 41 of 64 |
| Arena Agent | standardised score on agentic sessions, 0 = field average | 21 of 64 |

Coverage is shown beside the metric selector, and models unrated on the active lens are hidden and
counted under the chart. There is deliberately **no blended score**: the lenses disagree, and that
disagreement is the useful signal.

> Rating data from [`lmarena-ai/leaderboard-dataset`](https://huggingface.co/datasets/lmarena-ai/leaderboard-dataset)
> by **Arena Intelligence, Inc.**, used under [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/).
> **Changes made:** a subset of models was selected; values were reformatted for plotting and rounded
> for display; no rating was altered. Latest splits read 22–27 October 2026.
> Cite: Chiang et al., *Chatbot Arena*, [arXiv:2403.04132](https://arxiv.org/abs/2403.04132).

These are **Bradley–Terry** ratings, not Elo, despite the common label. Taken from the dataset
release rather than scraped from the site.

Two Qwen3.6 entries carry their identically-sized Qwen3.5 sibling's rating as a labelled proxy,
drawn hollow, because no Qwen3.6 open-weight model has been rated. `GLM-4.7-Flash` deliberately has
no WebDev rating: the board carries `glm-4.7`, which is a different model.

Nineteen models were added in the 21 Aug 2026 refresh, including Qwen3.8-27B, the GLM-5/5.1/5.3
siblings, MiMo-V2.5 and V2.5-Pro, Hunyuan Hy3, Mistral Large 3 and Medium 3.5, Inkling, Motif 3 and
Qwen3-Coder 480B. Models carrying `est` have their Q4 file size derived from total parameters at the
0.58 ratio the sighted GGUFs in this table exhibit, not from a sighted download — so their bubble
size and capacity verdicts are estimates. Several are rated on only one lens because the boards do
not all move at the same speed.

The 23 Sep 2026 refresh re-read every rating against the 13 Sep Arena split and added nine models:
GLM-5.3 Flash, GLM-4.7, DeepSeek V4.1 Flash, MiMo-V2.6-Pro, Qwen3.8-Flash-Next, Hy4 preview, K2 Horizon
375B-A23B, Step 3.7 Flash and Nemotron 3 Nano 30B-A3B. Four models rated on no board at all were
dropped. Agent ratings for models that have left the Agent board (Gemma 4 31B, Nemotron 3 Ultra,
Kimi K2.6, Kimi K2.7 Code) were removed rather than kept from an older split.

The 28 Oct 2026 refresh re-read every rating against the 22 Oct (text), 23 Oct (Agent) and 27 Oct
(WebDev) splits and added eleven models: MiMo-V2.6-Flash, MiniMax-M3.1-Flash, Step 5 Preview, Muse Spark
Open, Beam 1.0 501B-A23B, Ling 3.1 Flash, Inkling-Small, Nemotron 3.5 Super VL, Nemotron 3.5 Bolt
74B-A9B, Granite 4.2 30B and Laguna M.1. Qwen3.8-Max was dropped: its open-weight release is now
listed separately from the rated proprietary model, and nothing rates the open one. Step 5 Preview
is still labelled proprietary on Arena because it was rated from the API before its weights shipped
on 15 Oct. Ten models carry no Arena rating at all and so do not appear on this page (Gemma 4 E4B,
Gemma 4 12B QAT, Nemotron 3.5 Bolt, Mistral Small 4, Step 3.7 Flash, Command A+, K2 Horizon, Ling 3.1
Flash, A.X-K2, K-EXAONE 2.0).

**This page is not affiliated with or endorsed by any benchmark operator.** Scores remain the work of
their respective operators.

### Note on what is absent

An earlier private build used the Artificial Analysis Intelligence Index, whose v4.3.2 rates 58 of
the 64 models here rather than 46 (AA moved to v5.0 in late October 2026, which so far covers 19). It is not in this version because AA's Terms of Use grant personal,
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
- **Graphics cards are the volatile line now.** On 28 Oct 2026 the RTX 5090 was A$10,499 and the 5080
  A$2,580, up 17% and 23% in five weeks. Memory eased slightly: server DDR5 ECC RDIMM is A$55/GB
  (from A$62 in August) and desktop DDR5 about A$15/GB. On a large-memory build RDIMM is still
  usually a bigger number than the CPU and the cards combined, so re-check it before deciding
  anything.
- **Second-hand server hardware is not the bargain it was.** Refurbished DDR4 is up 30–50% in the
  same squeeze (A$15/GB here, against A$55 for DDR5 RDIMM and about A$15 for consumer DDR5 — the
  discount over ordinary desktop memory has vanished), and used EPYC 7002/7003 prices span
  a genuine 3× between US eBay and AU-sourced. `epycRome2` covers two CPUs plus a dual-SP3 board at
  A$9,600, which is the lower-middle of the AU range on the reasoning that memory bandwidth comes
  from **channels, not cores** — the value build populates all eight channels per socket with the
  cheapest part that will, not the top SKU. This is the widest single uncertainty in the price
  table; if you can import, it drops by half.
- Many cards carry `bw est`, `price est` or `used market` flags in the picker, shown per card. Cards
  without a published decode benchmark have their effective bandwidth estimated from nominal at the
  conversion rate the calibrated cards exhibit (~55% consumer, ~45% RTX PRO); that rule lands within
  ~12% on the two cards where a real measurement exists.
- Kernel work moves decode benchmarks fast — one RTX PRO 6000 figure gained 43% in five months.
- **The coarse-vs-fine MoE kernel factor is the thinnest constant here.** Coarse-expert models
  (gpt-oss) are credited with converting a discrete GPU's bandwidth far better than fine-expert ones.
  Setting the two equal makes the fit clearly worse (mean leave-one-out error 11.5% → 16.7%), so it
  stays — but only four of the fourteen anchors test it, and on unified machines it is not exercised
  at all. gpt-oss-120b really is ~2.5× faster than GLM-4.5-Air on the same Strix Halo, and that much
  is pure active-parameter count (2.76 GB read per token against 6.96); the extra credit it gets on
  a discrete card rests on a single benchmark pair.
- **Apple GPUs convert bandwidth much better on dense models than on MoE.** The first published
  llama.cpp run on an M5 Ultra (Qwen3.8-27B at Q4_K_M, 44.9 t/s at 8k) is 1.7× what the MoE-derived
  bandwidth predicts, while the same model on a DGX Spark lands within 6%. Dense models on Macs carry
  a 1.7× factor as a result; one M3 Ultra measurement suggests the true gap may be larger (2.8×).
  The M5 Ultra's MoE figure is still an estimate at 35% of its 1.2 TB/s nominal; MLX runs of two MoE
  models suggest it may be ~15% high, but those were mixed 4/8-bit rather than Q4.
- Threadripper and Xeon memory figures are **modelled, not measured**: no published llama.cpp
  MoE-offload token benchmark for those platforms could be found as of 27 July 2026.
- **Tensor parallelism is gated on the GPU-to-GPU link, and the link is modelled per card.**
  Cards that can bridge (RTX 3090, A6000, A100, H100/H200 NVL) or share a fabric (HGX B300 on
  NVSwitch) are assumed connected by default, because that is a property of the hardware rather
  than a setting; the custom builder marks those rows with a green edge and lets you switch it off.
  With a coherent link, tensor parallelism pays on **MoE at batch 1** as well as on dense models —
  the per-layer AllReduce costs microseconds instead of tens of them. Without one it pays only on
  dense models. On an 8-way B300 board that distinction is worth 4.5x on GLM-5.2 (81 to 368 tok/s),
  which is most of what the fabric is for. Consumer Blackwell and the RTX PRO 6000 have no NVLink;
  the 3090 is the last GeForce that does.
- Two card bandwidths are flagged `(est)` in the picker.

Every figure carries its date in the provenance table at the bottom of the page.

## Licence

Code and the performance model: **MIT** (see `LICENSE`).
Compiled data and layout: **CC BY 4.0**.
Underlying benchmark scores remain the property of their operators and are used under the terms above.
