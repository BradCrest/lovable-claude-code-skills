---
name: ai-image-sprite-pipeline
description: "Use when scaling AI-generated UI assets in a React / React Native project that LV or Claude Code can't generate themselves (avatars, sprite catalogs, themed icons), when integrating raw AI-generator outputs into a component enum system, or when asset volume exceeds manual cleanup capacity. Covers the 3-reference-image strategy, Python+Pillow post-processing, and enum-driven asset resolution."
---

# AI Image Generation → Sprite Pipeline

When your app uses many AI-generated illustrations (Pokédex-style sprite
catalogs, custom avatars, themed assets), you need a **repeatable
pipeline**:

1. Prompt the AI → consistent style
2. Strip backgrounds → transparent PNG
3. Resize → asset sizes
4. Integrate → React / React Native component map

This skill captures the pipeline that scales from 10 to 200+ assets.

## Contents

- [The 3-reference-image strategy](#the-3-reference-image-strategy)
- [Prompt template](#prompt-template)
- [Background removal with Python + Pillow](#background-removal-with-python--pillow)
- [File structure pattern](#file-structure-pattern)
- [Component integration](#component-integration)
- [AI failure modes](#ai-failure-modes)

## The 3-reference-image strategy

For style consistency across many sprites, **attach 3 reference images**
per generation:

| Image | Purpose |
|---|---|
| **1: Subject reference** | The specific thing being depicted (e.g. a real-world photo of the cat) |
| **2: Style reference A** | An existing sprite of similar subject in your target style |
| **3: Style reference B** | Another existing sprite (different subject, same style) |

Why three:
- 1 alone → AI guesses the style, doesn't match your existing assets
- 1 + 2 → AI may copy too much from ref 2, ignoring subject differences
- 1 + 2 + 3 → AI averages style across 2-3, applies to 1 — best consistency

If you don't have existing sprites yet (cold start), generate the first 1-2
manually, then use those as ref for the rest.

## Prompt template

Use a **3-section structure**: Subject + Style + Pose/Action.

```markdown
Generate ONE chibi-style PNG sticker.

ATTACHED IMAGES:
- Image 1: real photo of the subject (preserve identity)
- Image 2 & 3: existing chibi sprites (preserve art style)

SUBJECT:
- <breed / type / kind>, <coloring>, <distinctive features>
- <eye color>, <accessories>, <proportions>

ART STYLE:
- Hand-drawn chibi sticker, soft watercolor texture
- Match Image 2 & 3 aesthetic exactly
- Plain solid WHITE background (will be stripped in post)
- Single subject, centered, full body
- Canvas <width>×<height> px

POSE / ACTION:
- <one sentence describing pose>
```

### Tips

- **Use ALL CAPS** for emphasis on critical constraints
- **List requirements explicitly** rather than describing in prose
- **Specify what NOT to do** for things AI typically gets wrong
- **Canvas size**: pick something divisible by 256 (1024×1024, 2048×2048,
  2816×1536). AI generators handle these aspect ratios better.

### Multi-output: generate independently

If you need 14 poses × 14 breeds = 196 sprites: **don't generate them in
one conversation**. Style drifts quickly across turns.

Instead:
- Open separate AI conversation per pose × breed combo
- Re-attach the 3 ref images every single time
- Don't rely on "remember the last 3 I sent"

This is annoying but the only way to maintain consistency at scale.

## Background removal with Python + Pillow

AI outputs often have:
- Gray checkerboard "transparency" placeholder (not actually transparent)
- Off-white backgrounds that need cleaning
- Drop shadows you don't want

Pipeline:

```python
# scripts/process-sprites.py
from PIL import Image, ImageChops
from pathlib import Path

INPUT_DIR = Path("originals-2816")
OUTPUT_DIR = Path("processed-512")
OUTPUT_DIR.mkdir(exist_ok=True)

def strip_bg(img: Image.Image) -> Image.Image:
    """Remove gray/white backgrounds, return RGBA with transparent bg."""
    img = img.convert("RGBA")
    pixels = img.load()
    w, h = img.size
    
    # Method 1: mark gray pixels as transparent
    for x in range(w):
        for y in range(h):
            r, g, b, a = pixels[x, y]
            # Grayscale check: R,G,B within 8 of each other, brightness 140-220
            if abs(r-g) < 8 and abs(g-b) < 8 and 140 <= (r+g+b)//3 <= 220:
                pixels[x, y] = (r, g, b, 0)
    
    # Method 2: flood fill from corners for stubborn backgrounds
    # (use PIL's ImageDraw.floodfill if needed)
    
    # Crop to alpha bounding box
    bbox = img.getbbox()  # finds non-zero alpha bounds
    if bbox:
        img = img.crop(bbox)
    
    # Resize so longest edge is 512px
    max_dim = 512
    if max(img.size) > max_dim:
        img.thumbnail((max_dim, max_dim), Image.LANCZOS)
    
    return img

for src in INPUT_DIR.glob("**/*.png"):
    img = Image.open(src)
    out = strip_bg(img)
    rel = src.relative_to(INPUT_DIR)
    dest = OUTPUT_DIR / rel
    dest.parent.mkdir(parents=True, exist_ok=True)
    out.save(dest, optimize=True)
    print(f"Processed: {rel}")
```

### Typical compression result

| Stage | File size |
|---|---|
| AI output (2816×1536 PNG with gray bg) | 5-8 MB each |
| After bg removal + crop | 1-3 MB |
| After 512px downscale | 50-300 KB |

For 196 sprites: ~1.2 GB raw → ~40 MB processed (97% reduction).

## File structure pattern

For breed × pose matrix (e.g. a Pokédex-style sprite catalog):

```
project-root/
└── public/
    └── sprites/                        # 線上資源 (web-accessible)
        ├── PoseA/
        │   ├── c1-poseA.png
        │   ├── c2-poseA.png
        │   └── ...
        ├── PoseB/
        │   └── ...
        ├── c1-legacy-pose.png          # Legacy poses at root
        └── c2-legacy-pose.png
└── design/
    └── sprites-source/                 # Original 2816px (local-only)
        └── ...                         # gitignored
```

Naming convention:
- `c1`, `c2`, ... = breed/variant id (1-N)
- Each pose gets a folder (`PoseA/`) or root-level filename
- Filename pattern: `c{N}-{pose_slug}.png`

This lets a simple lookup function work:

```ts
function spritePath(breedCid: string, poseSlug: string): string {
  // c1-poseA.png in PoseA/ folder
  return `/sprites/${capitalize(poseSlug)}/${breedCid}-${poseSlug}.png`;
}
```

## Component integration

Map sprites to React components via enum:

```ts
// src/components/icons/SpriteIcons.tsx

export type BreedPalette = 
  | "cream" | "gray" | "silver" | "black"
  | "calico" | "orange" | "siamese" | ...;

export const ALL_BREED_PALETTES: BreedPalette[] = [
  "cream", "gray", "silver", ...
];

export const BREED_INFO: Record<BreedPalette, { name: string; cid: string }> = {
  cream:    { name: "Cream", cid: "c1" },
  gray:     { name: "Gray Tabby", cid: "c2" },
  silver:   { name: "Silver", cid: "c3" },
  // ... etc
};

// Pose definitions in separate file
export const POSE_DEFS = [
  { key: "loaf",      zh: "趴姿", en: "Loaf",    folder: "Loaf",    slug: "loaf"    },
  { key: "stretch",   zh: "伸懶腰", en: "Stretch", folder: "Stretch", slug: "stretch" },
  // ...
];

export function ChibiSprite({ 
  palette, pose, size = 80 
}: { 
  palette: BreedPalette; 
  pose: string; 
  size?: number 
}) {
  const cid = BREED_INFO[palette].cid;
  const poseDef = POSE_DEFS.find(p => p.key === pose);
  if (!poseDef) return null;
  
  return (
    <img 
      src={`/sprites/${poseDef.folder}/${cid}-${poseDef.slug}.png`}
      alt={`${BREED_INFO[palette].name} ${poseDef.en}`}
      width={size}
      height={size}
      style={{ objectFit: "contain" }}
    />
  );
}
```

Add new breed:
1. Generate sprites (14 poses) via AI
2. Process via Python script
3. Add to `BREED_INFO` map + `ALL_BREED_PALETTES`
4. Sprites are automatically used

## AI failure modes

| Failure | Frequency | Workaround |
|---|---|---|
| Style drift across multi-turn | High | New conversation per generation |
| Refuses to generate (safety filter) | Low | Rephrase, remove trigger words |
| Generates wrong subject details | Medium | Re-attach reference, repeat prompt |
| Adds drop shadow you didn't want | Medium | Specify "no shadow underneath" |
| Watercolor texture too heavy | Medium | Specify exact existing sprite as ref |
| Generates 2 subjects when you wanted 1 | Low | "Single subject, centered" in prompt |
| Different art style from existing | Medium | More style refs, explicit "match Image 2 & 3" |
| Special features ignored (e.g. one-eared cat) | High | Accept standard, manually edit in post |

**Last one is important**: if your subject has a feature the AI can't
preserve (asymmetric ears, scars, accessories), **expect it to fail** and
fix manually with Photoshop / Pillow scripting. AI image models have
strong "average" biases.

## Cheat sheet

```bash
# 1. Generate (per sprite)
# Open AI Studio, drag 3 refs + paste prompt, download PNG

# 2. Process (batch)
python3 scripts/process-sprites.py

# 3. Integrate
# Add new breed/pose to enum, sprites auto-resolve
```

## Companion skills

- `lovable-vs-claude-code-allocation` — neither LV nor CC generates these; you do
- `lovable-feature-spec-pattern` — document your sprite catalog as a feature
