# Procedural Weather — Teaching Three.js Skill

A Claude Code skill for generating dynamic procedural weather systems in Three.js, with **GPU particle precipitation**, volumetric fog, lightning, optical effects, and smooth state transitions. **WebGPU compute with WebGL2 fallback**.

Part of the **Teaching Three.js** skill series.

## What This Skill Does

When installed, this skill teaches Claude Code how to build complete weather systems including:

- **Rain** — GPU particle `LineSegments` with wind-angled drops, ground splashes, and wet lens camera overlay (drizzle → monsoon intensity range)
- **Snow** — `Points`-based flakes with sinusoidal flutter, tumble, sparkle, size variation, and frost camera overlay (flurries → blizzard)
- **Fog** — exponential scene fog plus shader-driven ground fog with noise wisps and wind drift
- **Lightning** — procedural branching bolts with recursive midpoint displacement, glow layer, and PointLight flash illumination
- **Dust/Sandstorm** — swirling particle field with atmospheric color tinting
- **Aurora Borealis** — undulating displaced plane with multi-color gradient and additive glow
- **Rainbow** — torus-arc with ROYGBIV spectral gradient
- **Camera post-processing** — wet lens (droplets + streaks + refraction) and frost overlay (edge-creeping ice crystals)
- **Weather state machine** — 12 states with smooth parameter interpolation, transition routing, and biome presets

## Installation

### Claude Code (CLI)

```bash
claude install-skill path/to/procedural-weather
```

Or copy the `procedural-weather/` folder into your Claude Code skills directory.

### Manual Usage

| File | Purpose |
|------|---------|
| `SKILL.md` | Main skill — all subsystems, state machine, scene assembly |
| `references/weather-shaders.md` | Complete GLSL/WGSL shaders for every effect |
| `references/weather-types.md` | 12 weather state profiles, transition matrix, biome presets |

## Quick Start Prompts

> "Create a scene with gentle rain and ground fog"

> "Build a thunderstorm with lightning bolts and dark sky"

> "Make a peaceful snowfall scene with frost creeping on the camera"

> "Create a desert sandstorm with orange atmosphere and swirling dust"

> "Build a weather system that transitions from clear to storm over time"

> "Generate a night scene with aurora borealis and light snow"

> "Add a weather state machine with all 12 states and smooth transitions"

## Requirements

- **Three.js r170+** for WebGPU and TSL support
- WebGPU browser (Chrome 121+, Edge 121+) for compute particle updates
- WebGL2 fallback (vertex-shader animation path) works in all modern browsers

## Skill Architecture

```
procedural-weather/
├── SKILL.md                            # Core skill (read first)
├── README.md                           # This file
└── references/
    ├── weather-shaders.md              # GLSL, WGSL shader code for all effects
    └── weather-types.md                # 12 state profiles, transitions, biome presets
```

## Key Concepts

### Weather State Machine

All weather is driven by interpolating between 12 discrete states. The controller smoothly
blends rain intensity, snow intensity, fog density, lightning frequency, sky darkness, and
wind speed between any two states over 2–3 seconds.

| State | Character |
|-------|-----------|
| clear | Calm blue sky, full visibility |
| cloudy | Overcast, flat diffused light |
| drizzle | Light misty rain, soft atmosphere |
| rain | Steady downpour, wet surfaces |
| heavyRain | Torrential, reduced visibility |
| storm | Full thunderstorm, violent wind, frequent lightning |
| lightSnow | Peaceful drifting flakes |
| snow | Steady accumulating snowfall |
| blizzard | Whiteout, horizontal snow, fierce wind |
| fog | Still, mysterious, world shrinks to a bubble |
| sandstorm | Orange haze, swirling dust, desert hostility |
| aurora | Clear polar night with northern lights |

### Zero Per-Frame JS Loops

All particle animation happens in vertex shaders via `time` + `mod()` wrapping. The CPU
only updates a handful of uniforms each frame — particle positions are never touched in JS.

### Performance Budget

All weather effects combined stay under **5 draw calls** and **~100K particles**:

| Effect | Particles | Draw Calls |
|--------|-----------|------------|
| Rain | 30K–80K | 1 |
| Snow | 10K–30K | 1 |
| Splashes | 200–500 | 1 |
| Dust | 15K–40K | 1 |
| Lightning | 1–3 bolts | 2–6 |
| Fog/Aurora | 1 mesh each | 1 each |

## Series: Teaching Three.js

Independent skills that compose naturally for complete environments:

- [procedural-landscapes](../procedural-landscapes/) — terrain generation
- [procedural-grass](../procedural-grass/) — animated ground cover
- [procedural-clouds](../procedural-clouds/) — sky and cloudscapes
- **procedural-weather** ← this skill — precipitation, fog, lightning, effects

## License

MIT — use freely in your projects.
