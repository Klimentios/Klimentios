# What MiniMax H3 taught me about reference video, character identity, and render cost

*Field notes from my own video production workflow, November 2026.*

I use [MiniMax H3](https://huggingface.co/MiniMaxAI/MiniMax-H3) on rented GPUs to make scenes with my own characters and environments. My typical task is more specific than generating a good-looking clip: keep the actions, camera movement, and cut structure of a reference video while replacing the performers and sometimes the visual world. A clip can look impressive and still fail that assignment.

Here are three findings from accepted short clips, failed attempts, and one controlled render comparison. These are observations from my workflow, not universal claims about the model.

**Model vs. workflow terminology.** The released H3-Base checkpoints are **FL2VA** (text and optional first/last frames) and **Ref2VA** (text plus image, video, or audio references). My “character swap” and “world change” labels describe how I arrange references and shot guidance in my Studio around Ref2VA; they are not official H3 model variants. The official H3-Context-IR and H3-Regenerate-2K stages are hosted services and were not part of this local H3-Base A/B. See the [official model card](https://huggingface.co/MiniMaxAI/MiniMax-H3).
## 1. Decide what the source video controls

I use two distinct workflows. When I want to replace a performer while preserving the original location, I feed the source as video and use character swap. When I need a different environment as well, treating that same video as the visual source can keep pulling the result back toward its original room.

For a world change, I use the source for motion and camera guidance, then provide a target-world image at the start of each shot. The important detail is spatial mapping: retain the reference room's layout, camera angle, actor positions, and movement, but change the characters and the appearance of the environment. A freely redesigned room can break the reverse angle or the background behind an actor.

The first six-second segment of one such world-change sequence was accepted for production. That validates this route for that scene; it does not mean the rest of the sequence has been completed or validated.

## 2. A correct face is not a correct character

In a one-character test, a face-only reference changed the head while leaving the original body, paws, and clothing. A later ten-second clip using a single full-character image—face, body, and costume together—was accepted as a complete replacement.

In a two-character scene, separate face and costume references without a shot-by-shot plan left the original performers visible for most of the clip. The accepted version used one full-character image per performer and described when each appeared, what each did, and which shots had no performers.

This does not prove that one image always beats multiple references. It shows that, for these scenes, coherent identity references and a temporal map made the task less ambiguous. I now check body, costume, scale, screen position, and action as carefully as faces.

## 3. Benchmark speed on a clip you would actually use

I repeated an accepted scene with the same input, seed, resolution, and 24 sampling steps, changing only the cache mode. The render log records:

| Cache mode | Total render | Sampling |
|---|---:|---:|
| Off | 934.5 s | 882.9 s |
| Safe | 717.8 s | 669.3 s |

Safe cut total render time by about 23% in this A/B. Frame comparison was about 39 dB PSNR, and I separately accepted the visual result. I kept Safe for this particular production route. I would not extrapolate that percentage or quality result to every scene, GPU, or configuration.

The most expensive lesson was operational. A night batch produced many technically successful files that missed the actual shot requirements: original performers or rooms remained, actions shifted in time, or character identity drifted. Several input and planning errors were involved. Since then, I inspect one complete segment against the source actions and cuts before scaling a batch. A successful render status is only the start of quality control.

I also ran smaller probes around shot timing, optics, and attention controls. Some used one synthetic scene and one seed that I did not accept as a production benchmark. I treat those as leads to investigate, not as reliable general rules.

My working conclusion: the quality of an H3 video transformation depends heavily on assigning clear roles to the reference video, target-world frames, character images, and shot plan. Better wording cannot fully rescue conflicting inputs.

*Disclosure: These results are from my own Studio workflow. The underlying character footage and internal job logs are private; the A/B values and acceptance boundaries above come from those logs.*


**Cache note.** Safe is a cache setting in my Studio inference workflow, not an official H3 model variant. The measured 23% is specific to the accepted scene and setup above.
