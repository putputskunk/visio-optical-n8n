---
name: comfyui
description: >
  Build, run, and debug ComfyUI workflows for diffusion-based image/video
  generation, and automate ComfyUI via its HTTP/WebSocket API. Use when the user
  mentions ComfyUI, a ComfyUI workflow .json (UI graph or API format), nodes like
  CheckpointLoader / CLIPTextEncode / KSampler / VAEDecode / SaveImage / LoadImage,
  custom nodes, the /prompt queue endpoint, or generating images/video to feed
  into a downstream pipeline (e.g. n8n social publishing).
---

# ComfyUI

ComfyUI is a node-based GUI and backend for Stable Diffusion / diffusion models.
A **workflow** is a graph of nodes; the backend executes it and returns generated
media. It is scriptable over HTTP + WebSocket, which makes it a good "render
engine" behind an automation pipeline (e.g. generate a marketing image/video in
ComfyUI, then hand the file to n8n to publish to social platforms).

## Two JSON formats — know which you have

ComfyUI exports **two different** JSON shapes. Confusing them is the most common
mistake:

1. **UI / graph format** (`Save` button, or the workflow embedded in a PNG's
   metadata). Contains `nodes`, `links`, `groups`, widget layout, positions.
   This is for loading back into the editor — **not** what the API accepts.
2. **API format** (`Save (API Format)`, requires "Enable Dev mode Options" in
   settings). A flat object keyed by node id:

```json
{
  "3": {
    "class_type": "KSampler",
    "inputs": {
      "seed": 12345, "steps": 20, "cfg": 7, "sampler_name": "euler",
      "scheduler": "normal", "denoise": 1,
      "model": ["4", 0], "positive": ["6", 0],
      "negative": ["7", 0], "latent_image": ["5", 0]
    }
  },
  "4": { "class_type": "CheckpointLoaderSimple",
         "inputs": { "ckpt_name": "sd_xl_base_1.0.safetensors" } }
}
```

In API format, an input value of `["4", 0]` means "wired from node id `4`,
output slot `0`". Literal values (seed, steps, text) are inline. **The `/prompt`
endpoint takes the API format**, wrapped as `{ "prompt": { ...the object... } }`.

## A minimal text-to-image graph

The canonical SD1.5/SDXL pipeline, by node:
- **CheckpointLoaderSimple** → outputs `MODEL`, `CLIP`, `VAE`.
- **CLIPTextEncode** ×2 (positive + negative prompts) → `CONDITIONING`.
- **EmptyLatentImage** (width/height/batch) → `LATENT`.
- **KSampler** (model, positive, negative, latent, seed, steps, cfg, sampler,
  scheduler, denoise) → `LATENT`.
- **VAEDecode** (latent + vae) → `IMAGE`.
- **SaveImage** (image, filename_prefix) → writes to `output/`.

Img2img swaps `EmptyLatentImage` for `LoadImage` → `VAEEncode`, and sets KSampler
`denoise` < 1.0. Inpainting adds a mask. Upscaling chains a second KSampler or an
`UpscaleModelLoader` + `ImageUpscaleWithModel`.

## API automation

Base URL defaults to `http://127.0.0.1:8188`.

- **Queue a job**: `POST /prompt` with `{ "prompt": <api-format>, "client_id": "<uuid>" }`.
  Returns `{ "prompt_id": "...", "number": N }`.
- **Track progress**: connect WebSocket `ws://host:8188/ws?clientId=<uuid>`.
  Messages include `status`, `executing` (with `node` = null when the prompt
  finishes), `progress`, and `executed` (carries output refs).
- **Fetch results without WS**: poll `GET /history/{prompt_id}`; the entry's
  `outputs` map node ids → `{ "images": [{ "filename", "subfolder", "type" }] }`.
- **Download a file**: `GET /view?filename=...&subfolder=...&type=output`.
- **Upload an input image**: `POST /upload/image` (multipart) → use the returned
  name in a `LoadImage` node.
- Other endpoints: `GET /object_info` (every node's schema — great for building
  prompts programmatically), `GET /queue`, `POST /interrupt`, `GET /system_stats`.

Typical automation loop: load an API-format template JSON → mutate inputs (prompt
text, seed, `filename_prefix`) → `POST /prompt` → wait on WS for completion →
read `/history/{id}` → `GET /view` the image/video → hand the file off downstream.

### Integrating with n8n (this project's likely use)

In n8n: an HTTP Request node `POST`s the API-format prompt to `/prompt`, then a
**Wait/Loop + IF** polls `/history/{prompt_id}` until outputs appear, then another
HTTP Request `GET`s `/view` to retrieve the binary, which n8n forwards to the
social-publishing nodes (TikTok/Meta/Pinterest). Use a unique `client_id` per run
and a unique `filename_prefix` to avoid collisions, and pin the `seed` if you need
reproducible renders.

## Models & file layout

- `models/checkpoints/` — full checkpoints (`.safetensors` preferred over `.ckpt`).
- `models/loras/`, `models/vae/`, `models/controlnet/`, `models/clip/`,
  `models/upscale_models/`, `models/embeddings/`.
- `input/` — source images for LoadImage; `output/` — results; `temp/` — previews.
- A workflow referencing a model filename that isn't present will fail at the
  loader node — check `models/...` before queuing.

## Custom nodes

- Installed under `custom_nodes/` (usually via **ComfyUI-Manager**). Each is a
  Python package registering new `class_type`s.
- **Portability gotcha**: a workflow using custom nodes won't run on an instance
  that lacks them — you'll get "node type not found". When sharing a workflow,
  list required custom nodes (ComfyUI-Manager can install missing ones from an
  embedded workflow). Popular ones: ControlNet aux, IPAdapter, AnimateDiff /
  video nodes, Impact Pack, WAS Node Suite.

## Video generation

Video via custom nodes/models (AnimateDiff, SVD/Stable Video Diffusion, and newer
video diffusion models) typically outputs a frame batch → a `VideoCombine`-style
node muxes frames to mp4/webm in `output/`. For a social pipeline, render to mp4,
then hand the file to the publisher. Watch VRAM: video graphs are memory-heavy;
reduce frame count/resolution or enable tiled VAE if you hit OOM.

## Debugging

- **"Node type not found"** → missing custom node; install it or remove the node.
- **OOM / CUDA out of memory** → lower resolution/batch/steps, use `--lowvram` or
  `--medvram` launch flags, enable tiled VAE, or use an fp8/quantized model.
- **Black or NaN images** → wrong/missing VAE, or fp16 instability (try the model's
  recommended VAE, or `--no-half-vae`).
- **API job seems stuck** → you likely sent UI-format JSON to `/prompt`; it needs
  API format. Also confirm `client_id` matches your WS connection.
- **`/history` empty after finish** → query by the exact `prompt_id` returned from
  `/prompt`; results are keyed by it.
- Use `GET /object_info/{ClassType}` to confirm exact input names/types when a
  prompt is rejected for a bad input key.

## When asked to build or modify a ComfyUI workflow

1. Determine which format is in play (UI graph vs API). For automation, produce
   **API format**.
2. Keep node-id wiring (`["id", slot]`) consistent when adding/removing nodes.
3. Verify referenced checkpoints/loras/custom nodes exist on the target instance;
   call out any that must be installed.
4. For automation, give the exact `/prompt` payload and the poll-then-`/view`
   retrieval steps.
