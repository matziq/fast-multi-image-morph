# Skill: fast-multi-image-morph

**Name:** `fast-multi-image-morph`  
**Version:** 1.1  
**Owner:** matziq / Grok Imagine workflow  
**Purpose:** Turn N images of the **same subject** (person or consistent character) in different outfits / hairstyles / poses / scenes into a single high-energy **portrait video** of extremely fast clean seamless morphing wardrobe changes.

## Hard output requirements (always — do not override)

| Requirement | Value | Notes |
|-------------|-------|-------|
| **Duration** | **≥ 10 seconds** | Minimum 10s every run. Default 15s. Never deliver shorter than 10s. |
| **Resolution** | **Full 1080p** | Always deliver true 1080p. Prefer `resolution_name="1080p"` on generate; always finish with `imagine_upscale_video(target_resolution="1080p")` on each ≤10s clip before stitch. Never leave final at 720p. |
| **Aspect ratio** | **2:3** | Always. Not optional. Target frame **1080×1620** (2:3 @ 1080p height). |

Default output: **15-second (min 10), 2:3 (1080×1620), full 1080p, 24 fps, with playing-card-flip SFX on every transition, zero magic FX, modest clothing.**

---

## When to invoke this skill

User says something like:
- “Make a fast multi-image morph video of these”
- “Wardrobe change morphing montage”
- “Extremely fast clean morph through all these looks”
- “@fast-multi-image-morph” or “use the fast-multi-image-morph skill”
- Any request that matches the Anna-Luna style rapid wardrobe morph

---

## Inputs (required / optional)

| Parameter | Required | Default | Description |
|-----------|----------|---------|-------------|
| `images` | **Yes** | — | Ordered list of Imagine asset IDs (or local paths that will be registered). Same subject required. |
| `duration` | No | `15` | Final video length in seconds. **Hard floor: 10.** If user asks for less than 10, still produce 10. Cap ~60. |
| `aspect_ratio` | — | **`2:3` fixed** | Always `2:3`. Ignore requests for other ratios unless user explicitly forks a different skill. |
| `resolution` | — | **`1080p` fixed** | Always full 1080p. Never ship 720p as final. |
| `modest` | No | `true` | Force higher necklines / no cleavage on every source image first. |
| `sfx` | No | `card_flip` | `card_flip` (default), `none`, or path to custom short SFX. |
| `magic` | No | `false` | If true, allow sparkles/energy (usually leave false). |
| `identity_name` | No | `"the subject"` | e.g. “Anna Luna” for stronger face lock. |

---

## Full procedure (execute exactly)

### 0. Prep
```bash
mkdir -p /workspace/artifacts/tmp
```

### 1. Register any local files as assets
For every local image path → `imagine_create_asset`.

### 2. (Recommended) Modest-clothing pass
For **every** source image (in parallel batches of 4–8):

```
imagine_image_to_image(
  asset_id = <src>,
  prompt = "Make the clothing more modest with a higher neckline that fully covers the chest and cleavage. Keep the exact same face, pose, hairstyle, expression, background, lighting, and overall style. No other changes."
)
```

Collect the new modest asset IDs in the **same order**.  
If `modest=false`, skip and use originals.

### 3. Group into sets of 6
```
groups = [
  images[0:6], images[6:12], images[12:18], images[18:24], ...
]
```
Pad the last group by repeating the final image if needed so every group has exactly 6.

### 4. Generate 6-second morph clips (one per group) — prefer 1080p
For each group of 6 assets:

```
imagine_reference_to_video(
  prompt = """Clean seamless morphing wardrobe change of the same woman/subject {identity_name}.
Extremely fast pure morphing transformations between different outfits, hairstyles, poses and scenes.
No magic effects, no sparkles, no light flashes, no energy bursts, no special effects, no particles —
just clean seamless morphing. Keep facial identity perfectly consistent.
All clothing is modest with high necklines and no cleavage.
High quality cinematic portrait video, natural lighting. 2:3 aspect.""",
  aspect_ratio = "2:3",          # ALWAYS 2:3
  first_frame_asset_id = group[0],
  last_frame_asset_id  = group[5],
  keyframes = [
    {"asset_id": group[1], "timestamp_s": 1.0},
    {"asset_id": group[2], "timestamp_s": 2.0},
    {"asset_id": group[3], "timestamp_s": 3.0},
    {"asset_id": group[4], "timestamp_s": 4.0}
  ],
  duration = "6",
  resolution_name = "1080p"     # ALWAYS request full 1080p; fall back to 720p only if 1080p is rejected, then upscale in step 4b
)
```

If `resolution_name="1080p"` fails (non-Pro / rate / capability), regenerate that clip with `resolution_name="720p"` and still run step 4b upscale.

Save the resulting video asset IDs.

### 4b. Upscale every clip to full 1080p (mandatory)
Each clip is 6s (under the 10s 1080p upscale limit). For **every** clip asset:

```
imagine_upscale_video(
  asset_id = <clip_asset_id>,
  target_resolution = "1080p"
)
```

**Never skip this.** Never use ffmpeg `scale` to upscale. Collect the upscaled asset IDs / paths.

### 5. Normalize each upscaled clip to 2:3 @ 1080×1620 (silent)
Only re-encode for fps/format/crop if probe shows wrong size; do **not** use scale as an upscaler (upscale already done).

```bash
# Target: full 1080p 2:3 = 1080×1620
for i in 1 2 3 4; do
  ffmpeg -i clip${i}_1080.mp4 \
    -vf "scale=1080:1620:force_original_aspect_ratio=increase,crop=1080:1620,setsar=1,fps=24,format=yuv420p" \
    -an -c:v libx264 -crf 18 -preset medium \
    /workspace/artifacts/tmp/c$i.mp4 -y
done
```

(scale here is only for aspect crop / normalize after true super-res upscale — source is already 1080-class.)

### 6. Concatenate → long silent video
```bash
ffmpeg -i c1.mp4 -i c2.mp4 -i c3.mp4 -i c4.mp4 \
  -filter_complex "[0:v][1:v][2:v][3:v]concat=n=4:v=1:a=0[outv]" \
  -map "[outv]" -c:v libx264 -crf 18 -preset medium \
  /workspace/artifacts/tmp/long_silent.mp4 -y
```

(Adjust `n=` and inputs to the actual number of groups.)

### 7. Speed map to target duration (floor 10s)
```bash
# raw_duration = sum of clip lengths (e.g. 6 * num_groups)
# target = max(requested_duration, 10)
# factor = target / raw_duration
ffmpeg -i long_silent.mp4 \
  -filter:v "setpts=PTS*(TARGET/RAW)" \
  -an -c:v libx264 -crf 18 -preset medium \
  /workspace/artifacts/tmp/final_silent.mp4 -y
```

If raw duration is already ≥ target and user did not ask to compress, you may skip speed-up and keep natural length (still ≥10s).

### 8. Generate card-flip SFX track
```python
import wave, array, math, struct
# Create a short ~0.30 s card-flip style noise burst (or load existing card_flip.wav)
# Then overlay  (num_images - 1)  copies at intervals = duration / (num_images - 1)
# Write stereo 48 kHz flips_{duration}s.wav matching final length (≥10s)
```

(Use the exact Python wave overlay that was proven in the Anna Luna run.)

### 9. Mux video + audio
```bash
ffmpeg -i final_silent.mp4 -i flips_Ns.wav \
  -c:v copy -c:a aac -b:a 128k -shortest \
  /workspace/artifacts/tmp/final_wardrobe_morph.mp4 -y
```

### 10. Register & deliver
```
imagine_create_asset(file_path="/workspace/artifacts/tmp/final_wardrobe_morph.mp4")
→ render_imagine_media(asset_id=...)
```

Verify with ffprobe: duration ≥ 10, height ≥ 1080 (or 1620 for 2:3 full-1080 height), aspect ~2:3.

### 11. Cleanup
```bash
rm -rf /workspace/artifacts/tmp/*
```

---

## Critical rules (do not break)

1. **Always ≥10s, full 1080p, 2:3** — these three are non-negotiable on every delivery.
2. **Never use pure `imagine_text_to_image` or `imagine_text_to_video`** for named real people / consistent characters — always pin real image references via first/last/keyframes.
3. **Moderation safety**: if any create_asset fails, re-run the modest pass with even stronger language (“fully covered high turtleneck, zero skin showing on chest”) and regenerate only the offending clips.
4. **Identity lock**: always include “Keep facial identity perfectly consistent” and the subject’s name when known.
5. **No magic by default**: the prompt must explicitly forbid sparkles, energy bursts, light flashes, particles.
6. **Aspect ratio**: always 2:3; normalize with force_original_aspect_ratio=increase + crop (never letterbox).
7. **1080p path**: request `resolution_name="1080p"` on generate; **always** `imagine_upscale_video(..., "1080p")` per clip before stitch. Never use ffmpeg alone as the upscaler.
8. **Duration math**: final length must be ≥10s exact to request when request ≥10; use setpts, never trim below 10s.
9. **Audio**: SFX must be precisely timed to the morph transitions (interval ≈ final_duration / (N−1)).

---

## Example invocation prompt (for future use)

```
Use the fast-multi-image-morph skill.
Images: @asset1 @asset2 … @asset24
Duration: 15s   # will be floored to 10 if lower
Aspect: 2:3     # always
Resolution: 1080p  # always full 1080p
Modest: true
SFX: card_flip
Subject: Anna Luna
```

---

## Changelog
- **v1.1 (2026-10-12)**: Hard requirements — always at least **10 seconds**, always **full 1080p** (`resolution_name=1080p` + mandatory `imagine_upscale_video` to 1080p per clip), always **2:3** (target 1080×1620). Duration floor enforced. Removed optional 720p final path.
- v1.0 (2026-10-10): Extracted from Anna Luna 24-look wardrobe morph session. Proven pipeline that survived moderation after modest-clothing pre-pass. Default was 15s 2:3 720p.

---

**End of skill definition.**  
Copy this entire file into any future conversation or save as a Grok project instruction / automation prompt to re-use instantly.
