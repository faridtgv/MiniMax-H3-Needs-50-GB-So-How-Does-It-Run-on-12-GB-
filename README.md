# Twelve Japanese Bytes — MiniMax H3 on a 12 GB card (ComfyUI workflows)

These are the ComfyUI workflows from the Beyond the Frame video **"MiniMax H3 Needs 50 GB… So How Does It Run on 12 GB?"**. Each teaching from the video is a coloured group in the graph; the red nodes are the changes.

| File | What it is |
|---|---|
| `TJB_1_H3_base_12GB.json` | Teachings **1, 3, 5** in one graph: squeezed model files, ComfyUI's own offloading, and unloading every model before the VAE decode. Also the **baseline** for the speed test. |
| `TJB_4_Pika_H3_Flex_6step.json` | Teaching **4, School of Six**: the same graph plus Pika's H3 Flex adapter with the official ComfyUI settings (stock loader, lcm, 6-step sigmas). |
| `TJB_4b_Pika_H3_Flex_low_vram.json` | The same, with the adapter loaded by a LoRA node that has the **low_vram switch**, set ON. |
| HyperFlow H3 Flash | Teaching **2**: use Video Rebirth's official templates (see below). |
| HyperFlow 8-step LoRA | Teaching **4, School of Eight**: community ComfyUI port only (see below). |
| `make_tjb_workflows.py` | Regenerates the three JSON files. |

All workflows use the **same prompt, size (608×352), frame count (124 ≈ 5 s) and seed**, so the times you measure are comparable.

## The five teachings and where they live

| # | Teaching | In ComfyUI |
|---|---|---|
| 壱 | **Squeeze the steel** (quantization) | Pruned INT8 model + NVFP4 text encoder in the loaders. 8-bit is the sweet spot. |
| 弐 | **A shorter blade** (HyperFlow H3 Flash, 13.7B) | Official HyperFlow Flash templates, GGUF Q4_K_M ≈ 9.1 GB. |
| 参 | **The attendant** (offloading) | Automatic in ComfyUI. On small cards start with `--lowvram` and have plenty of system RAM. |
| 肆 | **Two schools** (distillation) | HyperFlow: 49 → 8 steps (LoRA, community port). Pika H3 Flex: 6 steps, one adapter on the stock loader. |
| 伍 | **Clear the floor** (VAE decode) | `VRAM-Cleanup` unloads every model, then a plain `VAE Decode`. Tiled decode does **not** help with H3. Update ComfyUI: the H3 decode got about 2× faster. |

## Install

1. **ComfyUI** with native MiniMax H3 support (0.43.0 or newer). Update it.
2. **Custom nodes** (ComfyUI Manager or `git clone` into `ComfyUI/custom_nodes`):
   - **ComfyUI-VideoHelperSuite** — saves the mp4.
   - **Comfyui-Memory_Cleanup** (LAOGOU-666) — the `VRAM-Cleanup` node: https://github.com/LAOGOU-666/Comfyui-Memory_Cleanup
3. **Model files** from https://huggingface.co/Comfy-Org/MiniMax-H3 :

```
ComfyUI/models/
├── diffusion_models/  minimax_h3_fl2va_pruned_int8_convrot.safetensors
├── text_encoders/     qwen3vl_32b_minimax_h3_nvfp4_awq.safetensors   (15.7 GB)
└── vae/               minimax_h3_video_vae_int8_convrot.safetensors
                       minimax_h3_audio_vae_fp32.safetensors
```

4. In every loader, **re-select your own files** after opening the workflow.

**Launch flags for small cards** (add to the line that starts ComfyUI):
- `--lowvram` — offload more aggressively (teaching 3).
- `--disable-dynamic-vram` — if you run out of memory while memory is still free. `--lowvram` has no effect while Dynamic VRAM is on.

## Teaching 2 — HyperFlow H3 Flash (official)

Video Rebirth ships ComfyUI nodes and ready workflows. Use those instead of a copy.

```bash
cd ComfyUI/custom_nodes
git clone https://github.com/city96/ComfyUI-GGUF
git clone https://github.com/Video-Rebirth/ComfyUI-HyperFlow-Flash
pip install -r ComfyUI-HyperFlow-Flash/requirements.txt
```

- Weights: https://huggingface.co/VideoRebirthAI/hyperflow-h3-flash → `fl2va_gguf/minimax_h3_fl2va_hyperflow_flash_14b-Q4_K_M.gguf` (9.1 GB) into `models/diffusion_models/`. Smaller: Q3_K_M 7.2 GB; closest to full quality: Q8_0 14.9 GB.
- Text encoder and VAEs: the same Comfy-Org files as above (they recommend the fp16 video VAE).
- Open **Workflow → Browse Templates → ComfyUI-HyperFlow-Flash** → `hyperflow_t2v`, `hyperflow_i2v`, `hyperflow_flf2v` or `hyperflow_ref2v`.
- 4, 6 (default) or 8 steps, CFG 1.
- On PCs with 16–32 GB of system RAM, start ComfyUI with `python main.py --fast-disk`.
- Their own numbers (5.2 s clip, 6 steps, Q4_K_M): RTX 4090 ≈ 157 s, RTX 3090 ≈ 315 s. They measured 24 GB and 32 GB cards; **16 GB and 12 GB cards are not measured yet**.

## Teaching 4 — the two schools

**School of Eight · HyperFlow 8-step LoRA** (for the full-size H3)
- LoRA and loader: https://github.com/Video-Rebirth/hyperflow (official, diffusers).
- In ComfyUI it runs through a **community port** (unofficial at the time of the video): `Adudeguyman/ComfyUI-HyperFlow-H3`. Follow its README.
- Official claim: about 175 s → 60 s on their server (≈3×). Not 6×, because reading the prompt and decoding the picture take as long as before.

**School of Six · Pika H3 Flex** → `TJB_4_Pika_H3_Flex_6step.json`
- Model page: https://huggingface.co/Pika-Labs/Pika-H3-Flex · ComfyUI guide: https://docs.comfy.org/tutorials/video/pika/pika-h3-flex
- Pika ships **two adapters, used one at a time**, not stacked:

| Adapter (→ `models/loras/`) | Base model | Use for |
|---|---|---|
| [`pika_h3_flex_fl2va_v1_bf16.safetensors`](https://huggingface.co/Comfy-Org/MiniMax-H3/blob/main/loras/pika_h3_flex_fl2va_v1_bf16.safetensors) | `minimax_h3_fl2va_pruned_int8_convrot` | text-to-video, first/last frame (this workflow) |
| [`pika_h3_flex_ref2va_v1_bf16.safetensors`](https://huggingface.co/Comfy-Org/MiniMax-H3/blob/main/loras/pika_h3_flex_ref2va_v1_bf16.safetensors) | `minimax_h3_ref2va_pruned_int8_convrot` | reference-to-video (up to 12 references) |

- **Official ComfyUI settings** (already set in the workflow): `LoraLoaderModelOnly` at strength **1.0**, sampler **lcm**, `ManualSigmas` **`1.0, 0.972973, 0.923077, 0.8, 0.571429, 0.333333, 0.0`** (six steps = seven values), CFG 1 via `BasicGuider`.
- Don't change the sigmas or the strength, and **don't stack a turbo or Lightning LoRA** on top — it breaks the output.
- Comfy-Org also ships ready templates: **Workflow → Browse Templates → Pika H3 Flex** (T2V, FL2VA, R2V).
- Pika's own speed figure: **~1.8× faster** DiT denoising than fal's hosted H3 Max (≈2× for 10–15 s clips). "3×" figures on blogs are not Pika's claim.

**Small-card tip · the low_vram switch** → `TJB_4b_Pika_H3_Flex_low_vram.json`
- ComfyUI's stock `LoraLoaderModelOnly` has **no** low_vram switch. The switch is on the **MiniMax-H3 Turbo LoRA** node: ComfyUI Manager → search "MiniMax-H3 Turbo" (Larryvrh/ComfyUI-MiniMax-H3-Turbo).
- **ON:** the LoRA is merged into the model instead of carried next to it → lower memory peak, room for a longer clip.
- **OFF (default):** applied at run time → slightly sharper, more VRAM.
- Merging into an INT8 / FP8 base can look a little softer, so only switch it on if you run out of memory.
- That node was made for the Turbo LoRA; using it with Pika's adapter is not officially tested. If it errors, use `TJB_4_Pika_H3_Flex_6step.json`.

**Fewer strokes can make faces a little plastic and the sound a little thin.** If that happens, add two or three steps back.

## Run the duel yourself

1. Run `TJB_1_H3_base_12GB.json` once to warm up, then once more and note the time (`Prompt executed in … seconds` in the console).
2. Run the HyperFlow setup and `TJB_4_Pika_H3_Flex_6step.json` with the **same prompt, size, frames and seed**.
3. Compare the three mp4s side by side: faces, motion, sound.

Tell me in the comments which school won on your card.

## Licence

The MiniMax H3 and HyperFlow H3 Flash weights are released under the **MiniMax H3 Community License**, which is **not licensed in the EU, the UK, South Korea or the US without MiniMax's prior authorization**, and includes an Acceptable Use Policy. Read the licence before you download. This is not legal advice.

---
Speed numbers marked "official" or "their server" are the makers' own claims. ComfyUI changes fast; these notes reflect November 2026.
