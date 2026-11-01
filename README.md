# Colab + ComfyUI + MiniMax H3: video notes from an 8GB-VRAM laptop

These are working notes from making a short promo film when my own GPU was too small for the model. My laptop has an RTX 5060 Laptop GPU with **8GB VRAM** and 16GB system RAM. Instead of buying a new GPU, I ran ComfyUI on a **Google Colab A100** and generated the shots there with **MiniMax H3 (Ref2V) + a 4-step Turbo LoRA**.

This is not an official guide and not a one-click installer. It's what we actually used, the numbers we actually got, and the mistakes worth skipping. The example project is [Marooned](https://marooned.app), an interactive music app I'm building, but nothing below depends on it.

## Contents

1. [What you need](#1-what-you-need)
2. [Colab: what to expect](#2-colab-what-to-expect)
3. [Keep everything on Google Drive](#3-keep-everything-on-google-drive)
4. [Models we used](#4-models-we-used)
5. [Workflow and settings](#5-workflow-and-settings)
6. [Our actual results](#6-our-actual-results)
7. [Consistency across shots](#7-consistency-across-shots)
8. [Mistakes and lessons](#8-mistakes-and-lessons)
9. [Licensing: read this before you post anything](#9-licensing-read-this-before-you-post-anything)
10. [Official links](#10-official-links)

---

## 1. What you need

- A Google account with Colab access to a large GPU. I had a **Google AI Pro** subscription, and Google AI plans now include Colab benefits ([Google's announcement](https://developers.googleblog.com/en/colab-is-now-part-of-your-google-ai-plan/)). What you actually get (compute units, GPU types, priority) depends on your plan and changes over time, so check your own Colab resources panel rather than trusting any tweet, including mine.
- Enough **Google Drive storage** for the model files. The H3 files are large (tens of GB depending on which variants you pick), so a free 15GB Drive won't hold them.
- Basic comfort with running notebook cells and ComfyUI's node editor.
- **Not needed:** a powerful local GPU. My 8GB laptop was only used for watching previews and editing.

## 2. Colab: what to expect

- **GPUs vary.** In different sessions I got an **A100 80GB** and an **A100 40GB**. The film in this repo was produced on the **A100 40GB**. You can't pick and guarantee one; Colab's own FAQ says resources "are not guaranteed and not unlimited."
- **It is not free or unlimited.** A100 time burns compute units. Disconnect the runtime when you're done; an idle notebook can keep using units until it times out.
- **Runtimes reset.** VMs are deleted when idle and have a maximum lifetime. Anything stored only on the VM's local disk (`/content`) disappears. This is the single most important thing to plan around (next section).
- **Accessing the UI.** Colab doesn't expose ComfyUI's port directly. You either tunnel the web UI out of the notebook, or skip the UI and send workflows to ComfyUI's HTTP API (`POST /prompt`) from a notebook cell. Our shots were rendered through the API, one saved JSON per shot.

## 3. Keep everything on Google Drive

Treat the Colab VM as disposable and Drive as the hard disk.

```python
from google.colab import drive
drive.mount('/content/drive')
```

What we kept on Drive:

- **Models**: downloaded once to Drive, not on every session. Re-downloading tens of GB each time wastes both time and compute units. ComfyUI can read models from an extra folder through `extra_model_paths.yaml`, or you can symlink Drive folders into `ComfyUI/models/`.
- **Workflows**: every shot's workflow saved as API JSON, one file per shot.
- **Outputs**: written straight to Drive (ComfyUI has an `--output-directory` launch flag), so a disconnect mid-batch doesn't lose finished shots.

The per-project folder layout that worked well for us:

```
MY_PROJECT/
├── prompts/           one text file per shot
├── workflows/         one API-format JSON per shot (exactly what was rendered)
├── source_frames/     reference images and contact sheets for continuity
├── generated_shots/   raw model outputs, never edited in place
├── app_capture/       real screen recordings (see section 7)
├── audio/             real audio recordings and masters
├── edit/              edit script, edit decision list, graphics
├── qa/                frame grabs and probe/loudness checks of exports
├── exports/           final deliverables
└── PRODUCTION_REPORT.md
```

Saving the exact workflow JSON next to each output means you can re-render one shot weeks later without guessing which settings produced it.

Tip: reading big model files from a mounted Drive can be slow the first time. If loading feels stuck, give it time before assuming it crashed.

## 4. Models we used

All file names below are taken from our real workflow and match the files in the Comfy-Org repackaged repo: [huggingface.co/Comfy-Org/MiniMax-H3](https://huggingface.co/Comfy-Org/MiniMax-H3).

| Role | File | ComfyUI folder |
|---|---|---|
| Diffusion model (Ref2V, quantized) | `minimax_h3_ref2va_pruned_int8_convrot.safetensors` | `models/diffusion_models/` |
| Text encoder | `qwen3vl_32b_minimax_h3_nvfp4_awq.safetensors` | `models/text_encoders/` |
| Video VAE | `minimax_h3_video_vae_int8_convrot.safetensors` | `models/vae/` |
| Audio VAE | `minimax_h3_audio_vae_fp32.safetensors` | `models/vae/` |
| Turbo LoRA (4-step, Ref2V) | `minimax_h3_ref2v_turbo_4step_v0.1_comfyui_bf16.safetensors` | `models/loras/` |

Notes:

- **Ref2V** ("reference to video") lets you pass a reference image so characters, wardrobe and setting stay consistent. ComfyUI's docs say it needs ComfyUI 0.30.0 or newer.
- The Turbo LoRA for Ref2V was released by lightx2v ([huggingface.co/lightx2v/Minimax-h3-Turbo](https://huggingface.co/lightx2v/Minimax-h3-Turbo)); the ComfyUI-format file is also in the Comfy-Org repo.
- The Comfy-Org files are **pruned/quantized conversions**, not MiniMax's original checkpoint. MiniMax's own docs note their output can differ from the full model.
- We did **not** try running this stack on the 8GB laptop, so I can't tell you it works locally. MiniMax's deployment guide lists community examples on 24GB and 32GB cards with offloading, which is the honest reason we went to Colab.

## 5. Workflow and settings

[`workflows/h3_ref2v_turbo_4step_api_reference.json`](workflows/h3_ref2v_turbo_4step_api_reference.json) is the API-format graph we rendered one of our shots with. I only replaced the prompt with a placeholder, renamed the reference image and output path, and removed metadata. Everything else is unchanged.

**Please read before using it:**

- It is a **reference, not plug-and-play.** It was only run inside our own Colab + ComfyUI setup. I haven't tested it on a clean install.
- It's **API format** (what `POST /prompt` accepts), not the UI save format, so it won't open as a node graph by dragging it onto the canvas.
- It needs a ComfyUI version recent enough to include the MiniMax H3 nodes (`MiniMaxH3ReferenceToVideo`) plus core nodes like `ComfySwitchNode`, `ComfyMathExpression` and `ResolutionSelector`. If any node shows as missing, update ComfyUI first.
- Put your own reference image in ComfyUI's `input/` folder as `reference_frame.png`, or change node `300`.

Settings that mattered:

| Setting | Our value |
|---|---|
| Resolution | 736 × 1312 (vertical) |
| FPS | 24 |
| Length | 124 frames (about 5.2 s) |
| Sampler / scheduler | `res_multistep` / `simple` |
| Steps | 4 with the Turbo LoRA (the graph has a switch: 20 steps without it) |
| LoRA strength | 1.0 |
| Reference image sizing | `match` |

About the odd frame count: the graph turns "seconds" into frames with this expression, which snaps to a valid length:

```
max(5, round(a * 24)) + (5 - (max(5, round(a * 24)) % 17)) % 17
```

For `a = 5` seconds that gives 124 frames. If you hand-type a frame count that doesn't fit the pattern, expect errors or trimmed output.

## 6. Our actual results

These are **our numbers from one run**, not a benchmark:

- GPU: Colab **A100 40GB**
- 7 shots at 736 × 1312, 24 fps, 124 frames each
- Roughly **65–70 seconds per shot** with the 4-step Turbo LoRA
- Final films edited with FFmpeg and exported at 1080 × 1920

Your timing will depend on the GPU you get, whether models are already loaded, Drive read speed and the settings you change. The first shot of a session is usually slower because everything has to load.

## 7. Consistency across shots

What worked better than one long generation:

- **Short, separate shots.** Each shot is its own ~5 s generation with its own prompt and seed. If one fails, you redo one shot instead of the whole film.
- **One shot becomes the reference for the rest.** We generated the first shot from text only, picked a frame from it, and used that frame as the Ref2V reference for later shots. That kept the same three people, clothes, room and lighting.
- **Describe characters the same way every time.** We repeated identical character descriptions in every prompt, alongside the reference image.
- **Don't let the model draw your UI.** Video models garble text and interfaces. Our prompts explicitly asked for phone screens to face away or show only a soft glow, "no readable interface, no text, no logos." Every real app screen in the film is a genuine screen recording composited in during editing.

## 8. Mistakes and lessons

- **Download models to Drive once.** Re-fetching tens of GB after every runtime reset wastes time and compute units.
- **Save outputs to Drive as you go**, not at the end.
- **Keep one workflow JSON per shot.** "Which settings made that good one?" is a question you'll ask.
- **Check audio sync by analysis, not just by feel.** Our first soundtrack take had the app's beat starting about 170 ms off our edit grid. We caught it with onset analysis and recorded a second take.
- **Write down what's real and what's generated** while you work. It makes honest captions and disclaimers easy later. Our film's end card says: "Dramatization. App footage: real Marooned Local Jam screen recordings."
- **Watch your compute units.** Disconnect the runtime when a batch finishes.

### How the Marooned example was put together (for transparency)

- People, apartment and the glowing neon waves: **AI-generated dramatization** (MiniMax H3). The waves are a visual metaphor, not an app feature.
- App interfaces: **real screen recordings** of marooned.app.
- Soundtrack: recorded from Marooned's **Local Jam** feature itself, with no AI music generator and no added sound effects. To be precise: it was a scripted performance played through three separate browser sessions on one Colab machine, not three people on three physical phones.

## 9. Licensing: read this before you post anything

MiniMax H3 open weights are released under the **MiniMax H3 Community License Agreement** (dated August 2, 2026). The [official license text](https://huggingface.co/MiniMaxAI/MiniMax-H3/blob/main/LICENSE) is the authority; this is my plain-language summary of what it says as I read it on 2026-11-01:

- **Territory is the big one.** The license applies worldwide *except* the "Excluded Territories": the **European Union, the United Kingdom, the Republic of Korea and the United States**. It also says you may not use, distribute or **display the model or its Outputs** outside that territory. Licenses for those regions can be requested from MiniMax. Think about what this means for posting generated videos on global platforms, and get proper advice if it matters to you.
- **Commercial use is allowed within the territory, with conditions.** The license is royalty-free. You need **separate prior written authorization** from MiniMax (api@minimax.io) only if your commercial products and services make **more than US$20 million in yearly revenue**.
- **Product UI credit.** A commercial product or service that *uses MiniMax H3* must prominently display "MiniMax H3" in its user interface.
- **Redistributing the weights** (or derivatives) requires including the license and a NOTICE file. If you host H3 as a service, you need safeguards and a way for people to report misuse.
- **Outputs:** MiniMax claims no rights over what you generate, but you're responsible for it, and you can't use outputs to train or improve other AI models.
- **Acceptable Use Policy** (Exhibit A of the license) lists prohibited uses, such as impersonating real people without consent. MiniMax can update it.

Separately, ComfyUI's H3 page has its own note on commercial use of locally generated outputs. Read both, since the two pages are worded differently. Terms can change, so always check the current text. I'm not a lawyer, and this isn't legal advice.

## 10. Official links

- ComfyUI docs, MiniMax H3: https://docs.comfy.org/tutorials/video/minimax/minimax-h3
- Comfy-Org repackaged weights: https://huggingface.co/Comfy-Org/MiniMax-H3
- MiniMax original weights: https://huggingface.co/MiniMaxAI/MiniMax-H3
- MiniMax local deployment guide: https://platform.minimax.io/docs/guides/local-deploy-h3
- Ref2V Turbo LoRA (lightx2v): https://huggingface.co/lightx2v/Minimax-h3-Turbo
- Colab FAQ (limits, runtimes, compute units): https://research.google.com/colaboratory/faq.html
- Colab in Google AI plans: https://developers.googleblog.com/en/colab-is-now-part-of-your-google-ai-plan/
- ComfyUI: https://github.com/Comfy-Org/ComfyUI

---

Made while building [Marooned](https://marooned.app). If something here is wrong or out of date, open an issue. Model and Colab details change fast.
