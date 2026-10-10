# Skill: fast-multi-image-morph

**Name:** `fast-multi-image-morph`  
**Version:** 1.0  
**Owner:** matziq / Grok Imagine workflow  
**Purpose:** Turn N images of the **same subject** (person or consistent character) in different outfits / hairstyles / poses / scenes into a single high-energy **portrait video** of extremely fast clean seamless morphing wardrobe changes.

Default output: **15-second, 2:3 (720×1080), 24 fps, with playing-card-flip SFX on every transition, zero magic FX, modest clothing.**

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
| `duration` | No | `15` | Final video length in seconds (6–60). |
| `aspect_ratio` | No | `2:3` | `2:3`, `9:16`, `1:1`, `16:9`, etc. |
| `resolution` | No | `720p` | Target height for portrait (`720` → 720×1080). Use `imagine_upscale_video` later if higher res needed. |
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

### 4. Generate 6-second morph clips (one per group)
For each group of 6 assets:

```
imagine_reference_to_video(
  prompt = """Clean seamless morphing wardrobe change of the same woman/subject {identity_name}.
Extremely fast pure morphing transformations between different outfits, hairstyles, poses and scenes.
No magic effects, no sparkles, no light flashes, no energy bursts, no special effects, no particles —
just clean seamless morphing. Keep facial identity perfectly consistent.
All clothing is modest with high necklines and no cleavage.
High quality cinematic portrait video, natural lighting. {aspect_ratio} aspect.""",
  aspect_ratio = "2:3",          # or requested
  first_frame_asset_id = group[0],
  last_frame_asset_id  = group[5],
  keyframes = [
    {"asset_id": group[1], "timestamp_s": 1.0},
    {"asset_id": group[2], "timestamp_s": 2.0},
    {"asset_id": group[3], "timestamp_s": 3.0},
    {"asset_id": group[4], "timestamp_s": 4.0}
  ],
  duration = "6",
  resolution_name = "720p"
)
```

Save the 4 (or N) resulting video asset IDs / file paths.

### 5. Normalize each clip to target resolution (silent)
```bash
# Example for 2:3 720×1080
for i in 1 2 3 4; do
  ffmpeg -i clip$i.mp4 \
    -vf "scale=720:1080:force_original_aspect_ratio=increase,crop=720:1080,setsar=1,fps=24,format=yuv420p" \
    -an -c:v libx264 -crf 18 -preset medium \
    /workspace/artifacts/tmp/c$i.mp4 -y
done
```

### 6. Concatenate → long silent video
```bash
ffmpeg -i c1.mp4 -i c2.mp4 -i c3.mp4 -i c4.mp4 \
  -filter_complex "[0:v][1:v][2:v][3:v]concat=n=4:v=1:a=0[outv]" \
  -map "[outv]" -c:v libx264 -crf 18 -preset medium \
  /workspace/artifacts/tmp/long_silent.mp4 -y
```

### 7. Speed-up to exact target duration
```bash
# e.g. 24.17 s → 15 s  →  factor = 15 / 24.17
ffmpeg -i long_silent.mp4 \
  -filter:v "setpts=PTS*(15/24.17)" \
  -an -c:v libx264 -crf 18 -preset medium \
  /workspace/artifacts/tmp/final_silent.mp4 -y
```

### 8. Generate card-flip SFX track
```python
import wave, array, math, struct
# Create a short ~0.30 s card-flip style noise burst (or load existing card_flip.wav)
# Then overlay  (num_images - 1)  copies at intervals = duration / (num_images - 1)
# Write stereo 48 kHz flips_15s.wav
```

(Use the exact Python wave overlay that was proven in the Anna Luna run.)

### 9. Mux video + audio
```bash
ffmpeg -i final_silent.mp4 -i flips_15s.wav \
  -c:v copy -c:a aac -b:a 128k -shortest \
  /workspace/artifacts/tmp/final_wardrobe_morph.mp4 -y
```

### 10. Register & deliver
```
imagine_create_asset(file_path="/workspace/artifacts/tmp/final_wardrobe_morph.mp4")
→ render_imagine_media(asset_id=...)
```

### 11. Cleanup
```bash
rm -rf /workspace/artifacts/tmp/*
```

---

## Critical rules (do not break)

1. **Never use pure `imagine_text_to_image` or `imagine_text_to_video`** for named real people / consistent characters — always pin real image references via first/last/keyframes.
2. **Moderation safety**: if any create_asset fails, re-run the modest pass with even stronger language (“fully covered high turtleneck, zero skin showing on chest”) and regenerate only the offending clips.
3. **Identity lock**: always include “Keep facial identity perfectly consistent” and the subject’s name when known.
4. **No magic by default**: the prompt must explicitly forbid sparkles, energy bursts, light flashes, particles.
5. **Aspect ratio**: always normalize with force_original_aspect_ratio=increase + crop (never letterbox).
6. **Duration math**: final length must be exact; use setpts, never trim after the fact unless user asks.
7. **Audio**: SFX must be precisely timed to the morph transitions (interval ≈ final_duration / (N−1)).

---

## Example invocation prompt (for future use)

```
Use the fast-multi-image-morph skill.
Images: @asset1 @asset2 … @asset24
Duration: 15s
Aspect: 2:3
Modest: true
SFX: card_flip
Subject: Anna Luna
```

---

## Changelog
- v1.0 (2026-10-10): Extracted from Anna Luna 24-look wardrobe morph session. Proven pipeline that survived moderation after modest-clothing pre-pass.

---

**End of skill definition.**  
Copy this entire file into any future conversation or save as a Grok project instruction / automation prompt to re-use instantly.
