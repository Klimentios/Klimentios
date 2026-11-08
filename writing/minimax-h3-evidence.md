# MiniMax H3 field study: published measurement ledger

This ledger accompanies the [field study](minimax-h3-field-notes.md). It exposes the recorded numbers, input settings, controls and disagreements used in the article. The source footage and generated MP4s are not redistributed here; job IDs are internal, and visual acceptance is a first-party editorial decision.

## Production cache A/B: job `c3966a23`, 6 November 2026

Source record: KADRIA / H3 Studio `logs/ab-2026-11-06-160629-ka-cache-0611/report.md`, checked against `docs/DECISIONS.md` entry of 6 November. Both variants: 672×1216, 158 frames, 24 steps, seed **501680375309334**, `animation3d`, same prompt hash prefix `2c75292e67ca`. Only `cache_mode` changed.

| Variant | Render status | Total seconds | Sampling seconds | Seconds/step | Peak GPU memory |
|---|---|---:|---:|---:|---:|
| baseline, cache off | success | 934.5 | 882.9 | 36.79 | 93.3 / 95.01 GB |
| Safe cache | success | 717.8 | 669.3 | 27.89 | 94.68 / 95.01 GB |

Calculated total-time change: **(934.5 − 717.8) / 934.5 = 23.2%**. Recorded frame comparison: **PSNR 38.99 dB**. User decision in the source journal: Safe looked equivalent for this specific KLIM & ARIS segment. This is a one-scene A/B; it does not establish a universal speed or quality claim. The accepted *original* segment was rendered with cache off; the workflow was changed after this A/B.

## Shot timing: source edit versus written cut

Source record: `docs/reference/H3_ENCODER_RESEARCH.md`, “Формат склеек” and “Доказательство на рабочем дубле”, with A/B folders `logs/ab-2026-11-06-154433-night-range` and `logs/ab-2026-11-06-155425-night-official`.

| Scene / written form | Target cut(s), s | Observed cut(s), s | Error, s |
|---|---|---|---|
| Synthetic, bracketed 2.5 s range | 2.50 | 3.04 | +0.54 |
| Synthetic, bracketed 2.0 s range | 2.00 | 3.75 | +1.75 |
| Synthetic, explicit “At 00:02.500” | 2.50 | 2.54 | +0.04 |
| Production repair `97d46b7c`, ranges | 1.63 / 3.40 | 2.58 / 4.04 | +0.95 / +0.64 |
| Same production repair, explicit “At” | 1.63 / 3.40 | 1.58 / 3.12 | −0.05 / −0.28 |

The production comparison retained the seed and references, changing the timing wording. Two other production takes (`51702081`, `9140992b`) matched source-video cuts within approximately ±0.02 s **despite using ranges**. Therefore the published conclusion is limited: explicit “At” improved these two local comparisons; source edits can also dominate timing. A multi-seed study remains open.

## Accepted and rejected take boundaries

Source record: `docs/reference/MINIMAX_H3_RESEARCH.md` §7 and `docs/DECISIONS.md`. These are user-reviewed outcomes, not public model benchmarks.

| Job | Input setup | Observed outcome |
|---|---|---|
| `26fb9bf6` | Face-only character input | Head changed; source body/costume remained |
| `9896b951` | One full-character image, source video, 608×1056, 10 s, seed 22022216 | Single-character replacement accepted |
| `41cb1d3c` | Separate face and costume inputs, no complete temporal plan | Source performers remained through much of clip |
| `19715bfb` | One full-character image per performer, 3,682-character shot plan, 640×1152, seed 21919721 | Two-character replacement accepted; about 27 min |
| `c3966a23` | Source motion/camera, eight images, guide frames at 0 / 1.63 / 3.40 s, 672×1216, 24 steps | First world-change segment accepted; remainder unverified |
| `c683def1` | 4:3 continuation with an extra guide at 6.04 s | Cleared to continue; some props simplified |
| `cbd750eb` | Incorrect prepared guide frame | Guide already depicted three legs; rejected |

The two character contrasts are *not isolated image-count experiments*: shot guidance and other inputs changed too.

## Instrumented local graph

Source record: `docs/reference/H3_ENCODER_RESEARCH.md` § “Зонд на рабочем дубле”, five sampled denoising steps × six transformer blocks, approximately 512 sampled video positions. Accepted job `c3966a23` had **75,408** packed tokens: 12,310 Qwen encoder; 37,506 target-video; 14,112 source-video; 526 audio; remainder reference and guide latents. Sampled attention to the Qwen code: 10.6%; each 798-token shot guide: ~1.7%; source video: 4.3%; each character face/body image: 0.3–0.5%. These are probe readouts, **not causal effects** or a complete attention census.

Another encoder probe compared the same image under different later prompts and found cosine similarity **0.9999993**. It supports the narrower observation that the causal encoder has not read later text while encoding earlier image tokens; it does not imply text has no influence on generation.

## Overnight failure analysis

Source record: `docs/reference/MINIMAX_H3_RESEARCH.md` §8.1. The production review says **24–26 rejected outputs** across about seven GPU hours. In a 24-item defect taxonomy: environment errors 23; shifted timing 17; invented moments 17; wrong character identity 12; source performers remained 11; costume errors 10; missing moments 10; anatomy errors 3. Categories overlap and the 24-versus-26 source discrepancy is unresolved. The run was queued before viewing a complete first segment.

## Source boundaries

Official architecture, checkpoint limits, input modes and hosted/local split come from [MiniMax's model card](https://huggingface.co/MiniMaxAI/MiniMax-H3) and [repository](https://github.com/MiniMax-AI/MiniMax-H3). Every number and verdict in this ledger is from first-party Studio records. This published ledger is a transparent extract; it is **not** an independently reproducible media dataset.
