# MiniMax H3 in production: reference roles, identity, timing, and render cost

*A field study from six weeks of KADRIA / H3 Studio work, September–November 2026. Updated 8 November 2026.*

I run the released **MiniMax H3-Base** on rented GPUs for original character projects. My hard task is preserving a source video's actions, camera and edit while replacing performers, or replacing both performers and the world. A plausible clip can still fail the assignment. I consolidated our render logs, code probes, shot reviews and accepted/rejected takes. This article publishes the useful parameters and limits of that evidence.

**Data:** The [measurement ledger](minimax-h3-evidence.md) publishes the recorded settings, timing A/B, take IDs, probe counts and contradictory cases behind this account.

## The model and the actual input path

MiniMax describes the full system as **Context-IR → H3-Base → Regenerate-2K**. Context-IR and Regenerate-2K are hosted components. Our local work uses **H3-Base**, mainly the **Ref2VA** checkpoint. FL2VA is a distinct first/last-frame checkpoint. “Character swap,” “motion and camera,” and “world swap” below are *Studio workflow roles*, not official MiniMax model variants. [MiniMax's model card](https://huggingface.co/MiniMaxAI/MiniMax-H3) specifies at most nine images, three videos (15 seconds combined), three audio clips (15 seconds combined) and 12 mixed files in Ref2VA, with 24 fps output, native 32 kHz stereo audio and 4–15-second clips.

Officially, Qwen3-VL-32B supplies layer-50 encoder states; visual material also enters through a video VAE, audio through its own VAE, and a 33B single-stream transformer generates the output latents. The published inference release uses full attention. Those are **model facts from MiniMax**, not findings of my tests. See the [architecture and example requests](https://huggingface.co/MiniMaxAI/MiniMax-H3) and [released repository](https://github.com/MiniMax-AI/MiniMax-H3).

Our inspection of the local ComfyUI path found that numbered image and video references are assembled *before* the task text for the causal Qwen encoder. Video is sampled at 2 fps for encoder frames; its visual latents also enter generation. Audio has a separate path. In a probe, image encodings under two different later text prompts had cosine similarity **0.9999993**. The image encoder had not read that later wording, although the generation transformer still sees the combined context. This makes explicit reference mapping valuable: name `<Picture N>` and `<Video N>` consistently, then say whether each supplies identity, source scene, motion, target room or voice. MiniMax's [official Ref2VA example](https://huggingface.co/MiniMaxAI/MiniMax-H3) likewise names the numbered video/audio inputs and states what to preserve or reuse.

| Input | Task role in our workflow | Preflight check |
|---|---|---|
| Source video | Preserve performer surroundings for character replacement; provide motion, blocking and cuts for world change | Correct clip and cut boundaries |
| Full-character image | Face, body, costume and scale | Whole silhouette, hands and props |
| Face image | Extra close-up detail when needed | Same identity as body reference |
| Target-world guide frame | Target room and composition at a specific cut | Spatial match, anatomy, frame index |
| Shot text | Actions, timing, expression, reference relationships | Only events inside this segment; numbered tags match files |
| Audio reference | Voice/music when needed | Whether it is necessary; retained source audio can be remuxed later |

The same source video is useful for *preserving* a room in one task and dangerous if the task is to *replace* that room. That distinction produced more practical gain than changing a sampler.

## 1. A correct face is not a correct character

With a face-only reference, take `26fb9bf6` changed the head while leaving the source body, paws and clothing. A ten-second replacement using one coherent full-character image, `9896b951` (608×1056, seed 22022216), was accepted. For two performers, separated face/costume references without a temporal plan (`41cb1d3c`) left source performers visible. The accepted two-character take `19715bfb` (640×1152, seed 21919721, about 27 minutes) used a full-character image per performer and a 3,682-character shot plan for when each appeared.

These are **paired workflow observations**, not controlled proof that one image always beats several: both images and shot plan changed. A replacement campaign recorded four successful and eleven failed attempts. Six attempts using Studio's “motion and camera” role for a performer replacement tended to leave source performers in place and add characters beside them. The successful route mapped source performer to target character while treating the source as the scene to edit. These role labels belong to Studio, not H3.

My acceptance check now includes body, costume, scale, screen position, prop handling and source-person removal. A flawed guide frame can pre-load a failure: rejected frame `cbd750eb` already showed a character with three legs before generation.

## 2. World change requires a spatial contract

For a world change, the source supplies timing, blocking and camera movement. Target-world frames supply the room, characters and style **at each cut**. A guide must retain the source shot's geometry: angle, positions, movement and reverse-angle relationship. A visually attractive but unrelated room breaks continuity. Treating the source as a scene to preserve can pull the output back toward its original room.

The accepted *first segment* `c3966a23` used a six-second source, **eight images plus video**, target frames at **0, 1.63 and 3.40 seconds**, 672×1216 output and 24 steps. This validates that segment, **not the complete film**. It also contradicts our early simplistic rule that more than four images must harm identity. References need clear roles and correct content; count alone is a poor rule. A later 4:3 continuation (`c683def1`) added an interior guide at 6.04 seconds and was cleared to continue, although props simplified.

An instrumented probe of `c3966a23` counted **75,408 packed tokens**: 12,310 encoder, 37,506 target video, 14,112 source video, 526 audio and the remainder image/guide latents. In sampled attention, a 798-token guide at a cut drew about **1.7%** of video attention; a roughly 1,536-token character portrait drew **0.3–0.5%**. Attention is **not** a causal importance score. The probe samples selected positions, steps and blocks. It does justify checking guide frames closely before spending GPU time.

## 3. Cuts: a small timing A/B, plus a counterexample

In a same-scene test, bracketed ranges such as `[Shot 2: 2.5–5 s]` placed a requested 2.5-second cut at **3.04 s** (+0.54 s); another requested 2.0-second cut landed at **3.75 s** (+1.75 s). The form `[Shot 2] At 00:02.500, the shot cuts to ...` landed at **2.54 s** (+0.04 s). A repair of failed take `97d46b7c` moved requested 1.63/3.40-second cuts from **2.58/4.04 s** with ranges to **1.58/3.12 s** with the explicit “At” form. This is **n=2**, and the second repaired cut was still 0.28 s early.

Counterexample: production takes `51702081` and `9140992b` used ranges yet matched source cuts within about 0.02 s. The source edit appears to be a strong timing cue. Overnight failures also showed why segment preparation matters: round-second trims left 0.33–0.68-second tails from neighboring shots, shifting subsequent events. I map and cut the source edit first, then write explicit cut events. Sub-second shots can still disappear.

## 4. What the prompt probes support

Across 10–24 real production jobs, sampled video-token attention was approximately **63–84% to target video**, **11–16% to reference latents**, **3–6% to Qwen visual tokens** and **3–7% to prompt text**. These are descriptive probe shares, not a ranking of causal control. A section analysis of ten jobs found shot/action prose drew about **2.1×** its token-share attention in one sampled block, while character prose drew **0.5×** and later **0.3×**. This supports placing identity in coherent images and using text for actions, shots and expression; it does not prove character words are useless.

In one synthetic same-seed code ablation, removing an expression phrase erased the fear-to-smile arc while largely retaining the scene and lighting. Removing dialogue tags chiefly changed sound. Removing the push-in wording made the view more static. Increasing the expression code by 1.5 had little effect. The synthetic scene and one seed make this an **exploratory lead**.

We also found a Studio input-path bug: on an IR route, `director_config`, `camera_state` and `camera_optics` were discarded before H3; affected camera fields were null. It would be wrong to blame H3 for ignoring controls that never reached it. This was an application-path finding, separate from model behavior.

## 5. Cost: sequence length, hardware and quality

A ten-second replacement estimate contained about **68k source-video**, **58k target audio/video**, **14k text/encoder visual** and **3k still-image** tokens. In our graph, the source was encoded at a relatively large fixed size; reducing output resolution alone did not proportionally speed replacement. Reference complexity can cost as much as output length.

| Local measurement | Result | Limit |
|---|---:|---|
| Synthetic scene, 672×1216, 24 steps: Safe / off | 198 / 256 s | Speed probe, not production-quality benchmark |
| Accepted `c3966a23`, same seed/input/24 steps: off / Safe | **934.5 / 717.8 s** total; 882.9 / 669.3 s sampling | 23.2% faster in this take, ~39 dB PSNR; visual result separately accepted |
| Synthetic 0.8 → 0.98 MP | 198 → 268 s | 35% more time and a different composition, not proven better |
| BF16 vs INT8 / W4A8 on one 300 W Max-Q host | 485 → 648 / 710 s | Quantization slower on that host, not a general GPU result |
| One four-step turbo case | 485 → 140 s | Faster motion was rejected |

The 59 production replacements logged on 22 October ran roughly **7 / 19 / 39 minutes** for 5 / 10 / 15-second jobs, with varying input complexity. A 250–300 W RTX PRO 6000 Max-Q was roughly 2.5× slower than a full-power server version in comparable long replacements. In our BF16 setup a 96 GB card could not retain both H3 and Qwen simultaneously; disk reloads sometimes added 10–12 minutes. Different hosts and graphs prevent a clean universal performance ranking. Cache is one operational detail, **not** the study's thesis; Fast and turbo paths were rejected for relevant production quality.

## 6. The failed overnight batch

On 5–6 November, an overnight run used about seven hours of rented GPU time. Records disagree between **24 and 26 rejected outputs**, so I retain that range. In a 24-item defect review, **23** had environment errors, **17** displaced timing, **17** invented moments, **12** identity errors and **11** original performers remaining; categories overlap. The batch combined an experimental replacement route with world-change requests, overlong whole-film descriptions in short segments, arbitrary time cuts, guide lists that could be empty, and no review of the first complete render before filling the queue.

The production rule that followed is concrete: map source cuts → assign input roles → inspect guides and compiled prompt → render **one** full segment → compare every action, identity, edit and sound with the source → record an explicit human acceptance → scale. A saved MP4 is a pipeline event, not editorial acceptance.

## Evidence ledger and unresolved questions

These identifiers and measurements come from KADRIA / H3 Studio records and are expanded in the [published ledger](minimax-h3-evidence.md). IDs are internal identifiers, not public video links. The source media and full logs are not attached here, so external readers cannot independently reproduce the visual verdicts. The table gives parameters and scope instead of implying a public benchmark.

| Evidence | Claim status |
|---|---|
| `9896b951` vs `26fb9bf6` | Full-character accepted; face-only left body/costume. User-reviewed workflow contrast. |
| `19715bfb` vs `41cb1d3c` | Two coherent characters plus shot plan accepted; separated references failed. Multiple inputs changed. |
| `c3966a23` | Eight images, source video and three guides; **first segment** accepted. |
| `97d46b7c` | Local timing repair; n=2 cuts, one still 0.28 s early. |
| `c3966a23` cache A/B | Same seed/input/24 steps; 934.5 vs 717.8 s, ~39 dB PSNR, visually accepted for this take. |
| Encoder and attention probes | Image-code cosine 0.9999993; 75,408-token accepted job. Instrumented local graph; attention caveat applies. |
| Synthetic control map | 16+ variants but one scene/seed; **not** a production-wide optics/focus claim. |
| Night batch | Approximately 24–26 rejected outputs; 24-item defect taxonomy. Exact count unresolved. |

Two contradictions remain useful. Some earlier accepted clips used Fast cache despite a later rule rejecting it; the media, not a retrospective slogan, decides. Some range-form cuts matched source edits almost exactly despite the small A/B favoring the explicit “At” form. A multi-seed, same-source timing study is still needed. Our optics/focus control map mostly used one synthetic sailor scene later rejected as a production benchmark, so I do **not** claim H3 universally ignores focal length or rack focus.

### Sources

- [MiniMax H3 model card](https://huggingface.co/MiniMaxAI/MiniMax-H3): primary source for architecture, input limits, variants and official request examples.
- [MiniMax H3 repository and prompt references](https://github.com/MiniMax-AI/MiniMax-H3): primary source for released code and guides.
- KADRIA / H3 Studio internal `MINIMAX_H3_RESEARCH.md`, `H3_ENCODER_RESEARCH.md`, timing reports, approved-recipe records and user decisions, consolidated 8 November 2026: first-party source for every local measurement and verdict in this article.

*Kliment / KADRIA Studio. These local A/B runs used rented GPUs and H3-Base, not the hosted Context-IR or Regenerate-2K stages.*
