# TaleMondo Studio: from references to a reviewed video cut

*A public, noncommercial edition of my local AI video production studio. Published 8 November 2026.*

[**Explore the source and installation guide**](https://github.com/Klimentios/talemondo-studio) · [**Read my MiniMax H3 field study**](minimax-h3-field-notes.md)

I built TaleMondo Studio because a video model is only one part of making a scene. A production also needs reference management, a shot plan, a render queue, a way to reject bad takes, and a route from accepted clips to an edit. The [public repository](https://github.com/Klimentios/talemondo-studio) contains the Mac application, GPU-side code, ComfyUI integration, MCP server, setup scripts and documentation. It is a code release, not a claim that every workflow has passed a fresh end-to-end GPU test on another machine.

## The production path

1. **Collect the inputs.** The asset library keeps photos, video and audio with roles such as full character, face, costume, environment, motion, camera and scene guide. A shot can refer to these assets by numbered tags. The distinction matters: a source clip that should supply motion for a world change must not silently become the visual world to preserve.
2. **Build a scene.** Story workspaces hold scenes, shots, reference assignments and keyframes. The creator can plan which character appears in each shot, where the camera is, what action occurs, and how cuts and continuity work. World and character profiles are *user-created data*, not bundled franchises in the public repository.
3. **Prepare and render.** Studio compiles the shot into an H3 job, checks the input and required ComfyUI capabilities, then places it in a local queue. The Mac UI and project data stay local; a connected NVIDIA GPU server runs ComfyUI and H3. A completed queue entry is a saved output, not automatic approval of its visual or audio quality.
4. **Review and edit.** Takes remain attached to their jobs and source settings. The creator can compare, accept or reject them, assemble a scene and work with the montage and export tools. Some editor integrations have separate compatibility limits documented in the repository; I do not treat the presence of an export button as proof of a finished film.

This is why I call it a studio rather than an H3 prompt form: references, story structure, GPU execution, review and editing share one project context.

## Inside the Studio

These are direct screenshots of the public Studio build in English, opened with a separate empty data directory. They show the actual interface without my private projects or render history. An empty queue and an offline GPU in the captures reflect that isolated showcase session, not a completed production run.

**Video generation:** reference roles, the shot prompt, duration, format, quality, variants, director controls, and the local queue in one workspace.

![TaleMondo Studio video generation workspace in English](media/talemondo/studio-video-generation.png)

**Custom workflows:** a ComfyUI graph can be imported from JSON, a connected ComfyUI instance, or an existing job or recipe. The separate showcase data directory has no imported graphs yet.

![TaleMondo Studio custom ComfyUI workflows page in English](media/talemondo/studio-custom-workflows.png)

## ComfyUI without losing the production context

The built-in H3 route exposes quality and director controls, including guides at specific frames, sampling settings, model profiles and optional LoRAs. The separate **Custom Workflows** area can import a ComfyUI API graph, identify inputs, turn them into a Studio form and queue the graph on a connected ComfyUI server. Its graph editor can change values, nodes and links, record versions and bring a graph back from a job or recipe. These are actual modules and endpoints in the public code; compatibility still depends on the installed nodes and models on a live server.

The code includes a first-run Mac setup, GPU installation and verification scripts, and a route for adding pinned ComfyUI nodes and models. Model weights, GPU hardware and private project data are not included in the repository. See its [README](https://github.com/Klimentios/talemondo-studio#readme) for requirements and the [source tree](https://github.com/Klimentios/talemondo-studio/tree/main) for the implementation.

## Coding agents can operate the same studio

TaleMondo exposes an `h3-studio` MCP server for Claude Code and Codex. An agent can inspect the current board and assets, draft a scene, validate shots, prepare a render, inspect jobs and help edit a montage. The agent route also exposes Custom Workflow operations, including graph edits and version history. A Studio task asks for approval before it spends GPU time. The agent's proposed shot or a technically complete render still needs the creator's review.

The UI runs on Mac in English or Russian. It is meant for someone who wants to direct a sequence and also inspect the exact graph and job behind a shot.

## Video gallery

These seven clips come from my video work and show the range of material I am using to develop the studio. They are **visual examples**, not controlled A/B evidence for a model setting. The two measured side-by-side comparisons are in the [H3 field study](minimax-h3-field-notes.md). Click a frame to open its MP4 with audio.

| Clip | Preview | Length |
|---|---|---:|
| Animated character close-up | [![Animated character close-up](media/talemondo/IMG_9868.jpg)](media/talemondo/IMG_9868.MP4) | 18.7 s |
| Animated cat scene | [![Animated cat scene](media/talemondo/IMG_9900.jpg)](media/talemondo/IMG_9900.MP4) | 24.0 s |
| Costumed cooking scene | [![Costumed cooking scene](media/talemondo/IMG_9995.jpg)](media/talemondo/IMG_9995.MP4) | 14.7 s |
| Night motorcycle scene | [![Night motorcycle scene](media/talemondo/IMG_9996.jpg)](media/talemondo/IMG_9996.MP4) | 9.7 s |
| Halloween car scene | [![Halloween car scene](media/talemondo/IMG_9997.jpg)](media/talemondo/IMG_9997.MP4) | 12.5 s |
| Multi-shot action and comedy sequence | [![Multi-shot action and comedy sequence](media/talemondo/IMG_9998_showcase.jpg)](media/talemondo/IMG_9998_showcase.MP4) | 43.7 s |
| Car stunt shot | [![Car stunt shot](media/talemondo/IMG_9999.jpg)](media/talemondo/IMG_9999.MP4) | 11.7 s |

The 43.7-second showcase removes a 49.6-second black gap from the supplied 92.7-second file; the picture and audio on either side were copied without re-encoding. The other six MP4s were remuxed to remove creation-time metadata. The gallery does not assign an exact H3 graph, source-versus-output role or model checkpoint to a clip without its job record.

## Credits and boundaries

[MiniMax AI](https://huggingface.co/MiniMaxAI/MiniMax-H3) created and released MiniMax H3. [Comfy-Org](https://github.com/Comfy-Org/ComfyUI/blob/master/comfy_extras/nodes_minimax_h3.py) provides the native H3 nodes that make the local graph possible. [Kijai](https://github.com/kijai/ComfyUI-KJNodes) has contributed ComfyUI tooling used by the Studio ecosystem, including KJNodes in our optional color-match path; [his H3 model packaging](https://huggingface.co/Kijai/MiniMax-H3_comfy) is also a useful community resource. TaleMondo Studio is my application built around that work, not an official MiniMax, Comfy-Org or Kijai release.

The [public code](https://github.com/Klimentios/talemondo-studio) contains no model weights or private productions. It is released under [PolyForm Noncommercial 1.0.0](https://github.com/Klimentios/talemondo-studio/blob/main/LICENSE); bundled third-party components keep their own licences in the repository's [NOTICE](https://github.com/Klimentios/talemondo-studio/blob/main/NOTICE). Commercial use needs separate permission. Its Mac setup and GPU requirements are documented in the README.
