# MiniMax H3 in production: reference roles, identity, timing, and render cost

*A field study from six weeks of KADRIA / H3 Studio work, September–November 2026. Updated 8 November 2026.*

I run the released **MiniMax H3-Base** on rented GPUs for original character projects. My hard task is preserving a source video's actions, camera and edit while replacing performers, or replacing both performers and the world. A plausible clip can still fail the assignment. I consolidated our render logs, code probes, shot reviews and accepted/rejected takes. This article publishes the useful parameters and limits of that evidence.

**Data:** The [measurement ledger](minimax-h3-evidence.md) publishes settings, timing A/B, take IDs and probe counts.

**Production system:** These experiments were run through [TaleMondo Studio](https://github.com/Klimentios/talemondo-studio), the MiniMax H3 filmmaking Studio I built around my production workflow. It links franchise and character context, reference roles, scene planning, shot direction, queued GPU jobs, take review and editing. Its MCP tools let Claude Code and Codex work through those stages with me, while story approval, GPU spending and final creative acceptance stay explicit decisions. Its [product walkthrough and seven video examples](talemondo-studio.md) show the wider filmmaking workflow. Those gallery clips are visual context, not additional controlled A/B evidence. The [working-Studio gallery](talemondo-studio.md#in-my-working-studio) shows the populated scene graph, reference library, franchise boards and channel analytics. The H3 path queues work on a connected GPU and separates completed jobs from human-reviewed takes.

## The model and the actual input path

MiniMax describes the full system as **Context-IR → H3-Base → Regenerate-2K**. Context-IR and Regenerate-2K are hosted components. Our local work uses **H3-Base**, mainly the **Ref2VA** checkpoint. FL2VA is a distinct first/last-frame checkpoint. “Character swap,” “motion and camera,” and “world swap” below are *Studio workflow roles*, not official MiniMax model variants. [MiniMax's model card](https://huggingface.co/MiniMaxAI/MiniMax-H3) specifies at most nine images, three videos (15 seconds combined), three audio clips (15 seconds combined) and 12 mixed files in Ref2VA, with 24 fps output, native 32 kHz stereo audio and 4–15-second clips.

Officially, Qwen3-VL-32B supplies layer-50 encoder states; visual material also enters through a video VAE, audio through its own VAE, and a 33B single-stream transformer generates the output latents. The published inference release uses full attention. Those are **model facts from MiniMax**, not findings of my tests. See the [architecture and example requests](https://huggingface.co/MiniMaxAI/MiniMax-H3) and [released repository](https://github.com/MiniMax-AI/MiniMax-H3).

Our inspection of the local ComfyUI path found that numbered image and video references are assembled *before* the task text for the causal Qwen encoder. Video is sampled at 2 fps for encoder frames; its visual latents also enter generation. Audio has a separate path. In a probe, image encodings under two different later text prompts had cosine similarity **0.9999993**. Qwen had not read that later wording while encoding the earlier image tokens, although the generation transformer still sees the combined context. This makes explicit reference mapping valuable: name `<Picture N>` and `<Video N>` consistently, then say whether each supplies identity, source scene, motion, target room or voice. MiniMax's [official Ref2VA example](https://huggingface.co/MiniMaxAI/MiniMax-H3) likewise names the numbered video/audio inputs and states what to preserve or reuse.

### A frame and a prompt do not become a single “Qwen prompt”

The distinction matters because practitioners can easily mistake three different stages for one model call:

1. **Studio prepares the job.** It selects the source segment, still images, optional audio, target length and resolution, and a written shot description. Some Studio routes used a separate Qwen-based *prompt rewriter* before inference. That rewriter is an application feature; it is not the H3-Encoder. We found routes where its extra boilerplate diluted the action text, and routes where form fields were dropped before inference.
2. **The local Ref2VA node assembles encoder input.** It numbers references and places their image/video tokens before the written task text. In this implementation the encoder sees sampled video frames at 2 fps. A literal `<Picture 1>` written in the task text is not itself an image token; its number must match the file order. Audio is represented by a label for this encoder path and carried separately through the audio path.
3. **Qwen3-VL-32B encodes that sequence.** The H3-Encoder exports hidden states from layer 50, as MiniMax documents. Because this path is causal, an earlier image token cannot incorporate words appended after it. Later text tokens *can* incorporate earlier visual context. The local probe's near-identical image states under two later prompts tested exactly that narrow property.
4. **The visual and audio VAEs encode references again as latents.** H3 therefore receives both Qwen's multimodal contextual states and lower-level visual/audio latent sequences. This explains why reducing the text length alone may have little effect on a long reference-video job: the target and source latents can dominate the packed sequence.
5. **The H3 transformer jointly generates target video and audio latents.** The released local path uses a single packed sequence and full attention, then decodes the generated latents into an MP4 with sound. Text remains relevant inside this transformer. The fact that Qwen encoded an image before reading later instructions does **not** mean the final video ignores those instructions.

For a concrete input contract, consider a performer replacement. In Studio, `<Video 1>` identifies the source shot to edit, `<Picture 1>` identifies the replacement character, and the shot text states which source performer maps to that character, what remains in the room, and when the action occurs. For a world change, the same video instead supplies movement and cuts, while numbered target-world images specify the new room at those cuts. These are **two task descriptions over Ref2VA**, not two H3 checkpoints.

An additional local test separated encoder cost from model-loading cost. We saved the complete 48 MB prepared Ref2VA input, removed Qwen and image-loading nodes, and replayed the saved input into H3. On the same warmed host, the resulting MP4 matched byte-for-byte; the run took **193 s** versus **198 s** with Qwen. A colder 243-second run was slower mainly because it loaded weights. This is evidence that *reusing an already encoded input* can reproduce that take and save roughly five seconds of warm encoding. It does **not** show that H3 can understand new text or images without the H3-Encoder; new inputs must be encoded again. The separate Safe-cache A/B later in this article is a denoising optimization, not this encoder-input replay.

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

**Watch the production comparison:** [range syntax on the left, explicit “At” syntax on the right](media/h3-cut-timing-range-vs-at.mp4). Both are rendered outputs from the same local timing experiment, not the untouched source video. The [preview frame](media/h3-cut-timing-preview.jpg) helps identify the two sides, but the measured cut displacement requires watching the video.

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

**Watch the production A/B:** [cache off on the left, Safe on the right](media/h3-cache-off-vs-safe.mp4). The [preview frame](media/h3-cache-preview.jpg) shows the side-by-side layout. Visual similarity in this one take supports the logged review; it does not establish that every frame is identical or that Safe is suitable for every scene.

## 6. The failed overnight batch

On 5–6 November, an overnight run used about seven hours of rented GPU time. Records disagree between **24 and 26 rejected outputs**, so I retain that range. In a 24-item defect review, **23** had environment errors, **17** displaced timing, **17** invented moments, **12** identity errors and **11** original performers remaining; categories overlap. The batch combined an experimental replacement route with world-change requests, overlong whole-film descriptions in short segments, arbitrary time cuts, guide lists that could be empty, and no review of the first complete render before filling the queue.

The production rule that followed is concrete: map source cuts → assign input roles → inspect guides and compiled prompt → render **one** full segment → compare every action, identity, edit and sound with the source → record an explicit human acceptance → scale. A saved MP4 is a pipeline event, not editorial acceptance.

## 7. More accepted takes, and why they matter

An acceptance is a boundary around a *specific* media result, not a transferable preset. Our internal register contains six production takes that were accepted for their stated purpose. They did not all use the same cache, duration, resolution, prompt length or reference count:

| Take | Task and input package | Recorded result and limit |
|---|---|---|
| `5a4cd625` (4 Oct) | Still-image scene; character, face and environment images; 640×1152, 15 s, 25 steps, Fast cache | Accepted as a strong scene. This directly conflicts with the later blanket rejection of Fast cache; it is not evidence that Fast always works. |
| `587a1743` (5 Oct) | Face/costume images and a movement video; 5 s, about 0.9 MP, 24 steps, Safe | Likeness and dance accepted; the camera moved into an unwanted close-up late in the clip. |
| `9896b951` (16 Oct) | Single full-character replacement; 608×1056, 10 s, Safe, seed 22022216 | Accepted; rendered in 17.6 min. Contrast with face-only failure `26fb9bf6`. |
| `19715bfb` (22 Oct) | Two full-character images, shot map; 640×1152, 3,682-character prompt, Safe | Two-character replacement accepted; about 27 min. |
| `c3966a23` (5 Nov) | World change, source motion/camera, eight images, guide frames at cuts, 672×1216, 24 steps, cache off | **Only the first segment** accepted. Later Safe-cache use was a separate controlled A/B. |
| `c683def1` (6 Nov) | 4:3 world change with an additional guide inside a long shot | Cleared for continuation; hand-held props simplified. |

Two other takes are useful *visual* references but not complete recipes. `a60f8972` was praised for graphics, acting and expression, yet its body anatomy was imperfect. `0c31262c` was a look/grade reference while its story continuity failed. Treating either as a wholly accepted scene would erase the user's actual review.

The replacement recipe arose from a small campaign of four useful and eleven failed attempts. Its robust elements are a source-to-target performer map, full-body identity references, a shot map, and segment boundaries at source cuts. The successful prompts were roughly 2,900–3,700 characters; this is an observed band, **not** a hard H3 token limit. A close-up sometimes needed an extra face image. Crowds and three-plus mappings were unreliable in this campaign. These are Studio production rules with specific prior failures, not statements that H3 cannot ever render a crowd.

For world changes, the reference must be treated differently. We detect source edits, prepare a target frame for each new shot, and add a frame inside a long pan or orbit when identity/room mapping drifts. The first guide belongs at frame zero. The guide is checked against the character bible for eyes, markings, height, anatomy and costume before GPU submission. This is how a reference package becomes an actual edit contract rather than a pile of uploaded media.

## 8. Parameter experiments that did *not* justify new defaults

Most controlled parameter tests used one synthetic sailor scene (seed 424242, five seconds, 672×1216, 24 steps) that was later rejected as a production benchmark. It was useful for isolating variables, but visual rankings from it cannot be promoted to universal production advice.

| Change from local baseline | Measured sampling / total | Observed interpretation |
|---|---:|---|
| Baseline with Safe cache | 179 / 198 s | Reference for this scene only |
| Cache off | 237 / 256 s | Similar staging; PSNR 34.4 dB to baseline |
| Beta scheduler | 218 / 237 s | Different take; no clear quality win |
| Shift 8 / shift 16 | 180 / 199 s; 170 / 189 s | Different noise trajectories; no clear quality win |
| Output 768×1344 instead of 672×1216 | 247 / 268 s | More facial detail in one view, changed composition, 35% more total time |
| SageAttention 1.0.6 | 186 s sampling versus 180 s | 3.5% *slower* on this Blackwell setup; no reason to enable from this test |

We also compared `ref_image_size=match` with `max`. Recorded total-time reductions varied with the task: about 22% twice for a five-image character scene, 13% for another replacement, 3.5% for a ten-second replacement, and 6% for the one-image synthetic test. For the accepted eight-image world-change take, a circulated “14%” figure was **not supported by a completed timing log**. File timestamps give only an approximate 10% upper-bound comparison. Close-up identity was not visibly worse in reviewed pairs, but a one-seed visual check does not guarantee that for other scenes. That is why the full accepted recipe remains separately recorded.

On one 300 W Max-Q card, a BF16/24-step character case took 485 s; INT8 took 648 s and W4A8 710 s. Four-step turbo finished in about 140 s but its motion was rejected. These numbers show why “fewer bits” and “fewer steps” cannot be equated with usable throughput. They are host- and implementation-specific.

LoRA tests also have narrow boundaries. In the synthetic round, a face-realism adapter improved apparent human skin detail, while a different realism adapter shifted color toward orange; a camera-motion adapter increased the apparent orbit. Those were **one-scene leads**. Character Swap v1 appeared in accepted replacement recipes, whereas v1.5 failed to replace a source performer in a later batch. No claim here is that one LoRA version is generally superior across subjects.

## 9. Where the money and time went

Our BF16 deployment needed roughly 63 GB of GPU memory for H3, 49–51.5 GB for Qwen3-VL-32B and about 5.8 GB for the video/audio VAEs. The combined working set exceeded a 96 GB RTX PRO 6000, so a job could trigger model swaps; an H200-class host could keep more resident. These are observed deployment footprints of our graph, not MiniMax's universal minimum requirements.

Source-reference length and resolution can dominate the token budget. A longer source also means more frames to encode and more context to attend to. On a comparable 15-second replacement, a low-power Max-Q rental took roughly 48–50 minutes, while full-power server variants were substantially faster. In one host, reloading weights from a slow overlay disk added about **9.6–12.6 minutes** after switching between H3 and another image model. A “high RAM” launch cut loading time on one attempt but exhausted the host's swap and slowed inference badly, so we rolled it back. Grouping queued jobs by warm model and keeping keyframe generation off the H3 GPU were safer fixes.

Rental infrastructure introduced another kind of failure: a preemptible VM stopped several times during production, and some work was lost. A saved model snapshot reduced setup work on the next host, but a queue status alone did not prove the last video existed or was viewable. For cost accounting we separate startup, encoding, sampling, decoding, transfer and human QC. Hardware power limit, source-token budget and cold-start behavior often mattered more than a fashionable sampler option.

## 10. Failure taxonomy, not just a list of bad clips

The overnight batch was the clearest system failure, but earlier campaigns exposed distinct mechanisms. In a 28–31 October review, **63 of 82** takes were rejected and none of 38 planned shots was accepted as a complete shot. A separate 25 October batch of 38 clips was rejected for logic and visual errors. These counts are retrospective Studio records; they are not a controlled model benchmark and the rejection criteria varied by production task.

| Symptom seen in a reviewed take | Source or workflow cause recorded | Production check added |
|---|---|---|
| Original room survives a world change | Character-replacement route was asked to preserve the source scene | Choose world-change role and supply target guides |
| Original performer stays while target appears beside them | Source video assigned “motion and camera” role for a performer-replacement task | Explicit performer map and first-frame check |
| Identity or anatomy drifts | Incorrect or missing guide; inconsistent face/body references | Verify every prepared frame against character bible before render |
| Requested cuts arrive late or disappear | Text ranges, arbitrary trims, sub-second shots | Inspect source edits; cut segments at edits; explicit cut wording |
| Late source angles vanish | Only first seconds of a long source were fed into a short job | Review full source and label segment 1 of N |
| Props or costumes from another segment appear | Whole-film synopsis was inserted into a short-segment prompt | Prompt only the seconds being generated |
| A source person gets a forbidden item | Negated prose repeated the unwanted object or source costume | Describe desired appearance positively and inspect compiled prompt |
| Camera form controls seem ineffective | Studio dropped some IR-route control fields before inference | Trace actual payload and graph, then test model behavior |
| Completed jobs are still unusable | Automated QC watched statuses and thumbnails, not full video/audio | Review a complete first take before queue expansion |

Some rows are strong workflow diagnoses; others are hypotheses from a small number of failures. We keep the rejected job and its input package in the internal ledger so a rule can be revisited. The most damaging mistake was organizational: twenty-plus paid outputs were generated while the first could have been watched and rejected early.

## 11. What is still contradictory or untested

The evidence does not support a single neat “best settings” recipe. An early accepted clip used Fast cache, although later production reviews rejected Fast for eye and motion artifacts. The accepted world-change file records cache off, while a later route used Safe after the user's A/B approval. One successful scene used eight images despite earlier warnings that many images dilute identity. Ranged cuts failed in two direct tests but worked when aligned with source edits. These are distinctions to preserve, not inconvenient entries to delete.

The synthetic control map suggested that lighting, timed acting and camera movement responded more consistently than lens length, depth-of-field wording or rack-focus requests. However the official MiniMax examples themselves describe focus changes, and our map came largely from one rejected synthetic scene. Whether those optical controls work under different staging, references and seeds remains open. The same caution applies to attention maps: a token receiving more sampled attention per position is a useful diagnostic, not proof that it caused a better video.

Next defensible experiments are multi-seed tests on *accepted character footage*: explicit versus range cuts when the source edit agrees or disagrees; guide-frame placement inside long shots; `ref_image_size` quality at close-ups; and source-video preprocessing at fixed output settings. Each needs visual and audio review, not only MD5, PSNR or a successful queue result.

## Evidence ledger and unresolved questions

These identifiers and measurements come from KADRIA / H3 Studio records and are expanded in the [published ledger](minimax-h3-evidence.md). The two side-by-side MP4s above expose the **timing** and **Safe-cache** comparisons. Other job IDs are internal identifiers, not public video links; their source clips and full logs are not attached. Readers can inspect the two published comparisons, but cannot reproduce the other visual verdicts from this repository alone.

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
- [Comfy-Org native H3 nodes](https://github.com/Comfy-Org/ComfyUI/blob/master/comfy_extras/nodes_minimax_h3.py) and [Kijai's ComfyUI-KJNodes](https://github.com/kijai/ComfyUI-KJNodes): upstream tools used in the Studio ecosystem. MiniMax created H3; Kijai's community tooling supports optional Studio paths, including color matching. Neither is an author of this field study.
- KADRIA / H3 Studio internal `MINIMAX_H3_RESEARCH.md`, `H3_ENCODER_RESEARCH.md`, timing reports, approved-recipe records and user decisions, consolidated 8 November 2026: first-party source for every local measurement and verdict in this article.

*Kliment / KADRIA Studio. These local A/B runs used rented GPUs and H3-Base, not the hosted Context-IR or Regenerate-2K stages.*
