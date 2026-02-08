# Weather Types Reference

Detailed profiles for all weather states with parameter tables, artistic direction,
environmental effects, and combination rules.

## State Parameter Table

All values feed into the `WeatherController` state machine and its subsystems.

| State | Rain | Snow | Fog | Lightning | Sky Dark | Wind × | Character |
|-------|------|------|-----|-----------|----------|--------|-----------|
| clear | 0 | 0 | 0.0005 | 0 | 0 | 0.5 | Calm, blue sky |
| cloudy | 0 | 0 | 0.002 | 0 | 0.2 | 0.8 | Overcast, flat light |
| drizzle | 0.3 | 0 | 0.004 | 0 | 0.3 | 0.7 | Gentle, misty |
| rain | 0.7 | 0 | 0.006 | 0.1 | 0.5 | 1.2 | Steady downpour |
| heavyRain | 1.0 | 0 | 0.01 | 0.3 | 0.7 | 1.8 | Torrential, loud |
| storm | 1.0 | 0 | 0.015 | 0.8 | 0.85 | 2.5 | Violent, dramatic |
| lightSnow | 0 | 0.3 | 0.003 | 0 | 0.15 | 0.6 | Peaceful, gentle |
| snow | 0 | 0.7 | 0.006 | 0 | 0.3 | 1.0 | Steady, muffled |
| blizzard | 0 | 1.0 | 0.025 | 0 | 0.6 | 3.0 | Whiteout, fierce |
| fog | 0 | 0 | 0.03 | 0 | 0.25 | 0.2 | Still, mysterious |
| sandstorm | 0 | 0 | 0.02 | 0.1 | 0.5 | 3.0 | Abrasive, orange |
| aurora | 0 | 0 | 0.001 | 0 | 0 | 0.3 | Clear night, polar |

## Weather State Profiles

### Clear

**Character**: Calm blue sky, bright directional sun, minimal atmosphere.

```javascript
const CLEAR = {
  sky: {
    zenithColor: 0x1a6abf,
    horizonColor: 0x87ceeb,
    sunIntensity: 1.5,
    ambientIntensity: 0.4,
  },
  particles: { rain: 0, snow: 0, dust: 0 },
  atmosphere: { fogDensity: 0.0005, fogColor: 0x87ceeb },
  wind: { base: 2, gustFrequency: 0.2, gustAmplitude: 0.3 },
  camera: { wetLens: 0, frost: 0 },
  artistic: `
    The baseline. Maximum visibility, sharp shadows, vibrant colors.
    Atmosphere is barely perceptible — just enough haze at the horizon
    to suggest distance. Wind is a gentle breeze. Grass sways slowly.
    Birds optional. The scene should feel open and inviting.
  `,
};
```

### Drizzle

**Character**: Light misty rain, diffused light, everything slightly softened.
Contemplative, quiet.

```javascript
const DRIZZLE = {
  sky: {
    zenithColor: 0x6688aa,
    horizonColor: 0x99aabb,
    sunIntensity: 0.6,
    ambientIntensity: 0.5,   // Ambient goes up as sun diffuses
  },
  particles: {
    rain: 0.3,
    dropLength: 0.15,        // Shorter drops — smaller droplets
    dropOpacity: 0.2,        // More transparent
    splashChance: 0.1,       // Fewer visible splashes
  },
  atmosphere: {
    fogDensity: 0.004,
    fogColor: 0xaabbcc,
    groundFog: 0.3,          // Slight mist at ground level
  },
  wind: { base: 3, gustFrequency: 0.3, gustAmplitude: 0.2 },
  camera: { wetLens: 0.15, frost: 0 },
  artistic: `
    Soft, gentle, almost romantic. The world feels slightly out of focus.
    Light is flat and diffused — no harsh shadows. Colors are muted but
    not grey. Think "walking in a light mist." The wet lens effect should
    be barely noticeable — just occasional tiny droplets. Sound would be
    a soft patter, barely audible over ambient noise.
  `,
};
```

### Rain

**Character**: Steady moderate rainfall, puddles forming, distinct sound of rain.
The classic "rainy day."

```javascript
const RAIN = {
  sky: {
    zenithColor: 0x445566,
    horizonColor: 0x778899,
    sunIntensity: 0.3,
    ambientIntensity: 0.5,
  },
  particles: {
    rain: 0.7,
    dropLength: 0.3,
    dropOpacity: 0.35,
    splashChance: 0.4,
    splashScale: 1.0,
  },
  atmosphere: {
    fogDensity: 0.006,
    fogColor: 0x889999,
    groundFog: 0.1,
  },
  wind: { base: 6, gustFrequency: 0.5, gustAmplitude: 0.6 },
  camera: { wetLens: 0.4, frost: 0 },
  environment: {
    surfaceWetness: 0.6,     // Darken ground, increase specular
    puddleCoverage: 0.2,    // Reflective patches on ground
  },
  artistic: `
    Clearly raining — you'd grab an umbrella. Distinct angled streaks
    visible against dark surfaces. Ground develops a wet sheen. Occasional
    distant rumble suggests a broader system. Visibility reduced but still
    good. Reflections become important — wet surfaces mirror the sky.
    Wind angles the rain noticeably.
  `,
};
```

### Heavy Rain

**Character**: Intense downpour, reduced visibility, percussive splashes,
approaching storm energy.

```javascript
const HEAVY_RAIN = {
  sky: {
    zenithColor: 0x2a3344,
    horizonColor: 0x556677,
    sunIntensity: 0.15,
    ambientIntensity: 0.4,
  },
  particles: {
    rain: 1.0,
    dropLength: 0.4,
    dropOpacity: 0.4,
    splashChance: 0.7,
    splashScale: 1.3,
  },
  atmosphere: {
    fogDensity: 0.01,
    fogColor: 0x667788,
    groundFog: 0.0,          // Washed away by rain
  },
  wind: { base: 10, gustFrequency: 0.8, gustAmplitude: 1.2 },
  camera: { wetLens: 0.65, frost: 0 },
  environment: {
    surfaceWetness: 0.9,
    puddleCoverage: 0.5,
    occasionalLightning: true,
  },
  artistic: `
    Dramatic and immersive. Rain is visible as a curtain — you can see
    wave-like intensities sweep across the field. Splashes bounce visibly
    off every surface. Visibility drops to maybe 60 meters. Wind drives
    the rain at a strong angle. Occasional distant lightning flickers
    illuminate the clouds briefly. The world feels urgent and powerful.
  `,
};
```

### Storm

**Character**: Full thunderstorm. Dark sky, violent wind, frequent lightning.
Maximum drama.

```javascript
const STORM = {
  sky: {
    zenithColor: 0x1a1a2e,
    horizonColor: 0x333344,
    sunIntensity: 0.05,
    ambientIntensity: 0.25,
  },
  particles: {
    rain: 1.0,
    dropLength: 0.5,
    dropOpacity: 0.4,
    splashChance: 0.9,
    splashScale: 1.5,
  },
  atmosphere: {
    fogDensity: 0.015,
    fogColor: 0x334455,
  },
  lightning: {
    intensity: 0.8,
    minInterval: 2,
    maxInterval: 8,
    boltBranching: 6,
    flashDuration: 0.15,
    flashIntensity: 15,
    clusterBursts: true,     // 2-3 quick strikes then pause
  },
  wind: { base: 15, gustFrequency: 1.2, gustAmplitude: 2.5 },
  camera: { wetLens: 0.8, frost: 0 },
  artistic: `
    Awe-inspiring and slightly terrifying. The sky is near-black between
    lightning flashes. When lightning fires, everything turns blue-white
    for an instant — shadows flip direction. Thunder would shake the
    ground. Rain is almost horizontal in the strongest gusts. Trees (if
    present) bend dramatically. This is the cinematic storm scene — hold
    nothing back. The contrast between darkness and lightning flash is key.
  `,
};
```

### Light Snow

**Character**: Gentle, peaceful snowfall. Each flake visible. Quiet.

```javascript
const LIGHT_SNOW = {
  sky: {
    zenithColor: 0x7788aa,
    horizonColor: 0xaabbcc,
    sunIntensity: 0.5,
    ambientIntensity: 0.6,   // Snow reflects a lot of ambient
  },
  particles: {
    snow: 0.3,
    flakeSize: 2.5,
    flutterAmplitude: 2.0,
    fallSpeed: -1.2,
    sparkle: true,           // Flakes catch light
  },
  atmosphere: {
    fogDensity: 0.003,
    fogColor: 0xccccdd,
  },
  wind: { base: 2, gustFrequency: 0.3, gustAmplitude: 0.4 },
  camera: { wetLens: 0, frost: 0.05 },
  environment: {
    groundWhiteness: 0.2,    // Dusting of snow
  },
  artistic: `
    Magical and serene. Individual flakes drift lazily, catching light
    as they tumble. You can follow a single flake's path. The world
    gets slightly quieter — snow absorbs sound. Colors desaturate slightly.
    A thin dusting of white begins on horizontal surfaces. Think "first
    snow of the season" — wonder and calm.
  `,
};
```

### Snow

**Character**: Steady snowfall, accumulating, the world transforming white.

```javascript
const SNOW = {
  sky: {
    zenithColor: 0x667788,
    horizonColor: 0x99aabb,
    sunIntensity: 0.3,
    ambientIntensity: 0.6,
  },
  particles: {
    snow: 0.7,
    flakeSize: 3.0,
    flutterAmplitude: 1.5,
    fallSpeed: -1.8,
  },
  atmosphere: {
    fogDensity: 0.006,
    fogColor: 0xbbbbcc,
  },
  wind: { base: 5, gustFrequency: 0.5, gustAmplitude: 0.8 },
  camera: { wetLens: 0, frost: 0.15 },
  environment: {
    groundWhiteness: 0.6,
  },
  artistic: `
    Immersive snowfall. Dense enough that the sky disappears into white.
    The ground is clearly covered. Everything feels muffled and insulated.
    Visibility is reduced — distant objects fade into white. The light is
    flat, directionless. Frost starts creeping at the edges of the lens.
  `,
};
```

### Blizzard

**Character**: Whiteout conditions. Nearly zero visibility, fierce wind,
survival weather.

```javascript
const BLIZZARD = {
  sky: {
    zenithColor: 0x778899,
    horizonColor: 0xaabbcc,
    sunIntensity: 0.1,
    ambientIntensity: 0.5,
  },
  particles: {
    snow: 1.0,
    flakeSize: 4.0,
    flutterAmplitude: 0.5,   // Low — wind overpowers flutter
    fallSpeed: -2.5,
  },
  atmosphere: {
    fogDensity: 0.025,
    fogColor: 0xccccdd,
  },
  wind: { base: 20, gustFrequency: 1.5, gustAmplitude: 3.0 },
  camera: { wetLens: 0, frost: 0.6 },
  artistic: `
    Hostile and disorienting. Snow moves nearly horizontally. Visibility
    drops to 5-10 meters. The world is a white void with vague shapes.
    The camera frosts heavily — only the center remains clear. Wind
    noise would dominate everything. This is weather as antagonist.
  `,
};
```

### Fog

**Character**: Still, mysterious, intimate. The world shrinks to a small bubble.

```javascript
const FOG = {
  sky: {
    zenithColor: 0x99aabb,
    horizonColor: 0xbbccdd,
    sunIntensity: 0.2,
    ambientIntensity: 0.6,
  },
  particles: { rain: 0, snow: 0, dust: 0 },
  atmosphere: {
    fogDensity: 0.03,
    fogColor: 0xcccccc,
    groundFog: 0.9,
    groundFogHeight: 6,
  },
  wind: { base: 0.5, gustFrequency: 0.1, gustAmplitude: 0.1 },
  camera: { wetLens: 0.05, frost: 0 },
  artistic: `
    Atmospheric and contemplative. Objects emerge from and disappear
    into the white. Sounds feel close and muffled. Light is completely
    diffused — no shadows, no direction. Ground fog pools in low areas
    and drifts slowly. Trees are silhouettes. The mood is dreamlike
    and slightly eerie. Beautiful for close-up intimate scenes.
  `,
};
```

### Sandstorm

**Character**: Harsh, abrasive, orange-tinted. Desert survival weather.

```javascript
const SANDSTORM = {
  sky: {
    zenithColor: 0x8b6b3a,
    horizonColor: 0xc4a060,
    sunIntensity: 0.3,       // Sun visible as dim disc
    ambientIntensity: 0.4,
  },
  particles: {
    dust: 1.0,
    dustColor: 0xc4a060,
    dustSize: 4.0,
    dustOpacity: 0.45,
  },
  atmosphere: {
    fogDensity: 0.02,
    fogColor: 0xc4a060,      // Match dust color
  },
  wind: { base: 20, gustFrequency: 1.0, gustAmplitude: 3.0 },
  camera: { wetLens: 0, frost: 0 },
  environment: {
    colorTint: 0xc4a060,     // Everything shifts orange/brown
    tintStrength: 0.3,
  },
  artistic: `
    The entire world shifts to a warm orange-brown palette. Dust
    particles swirl in visible eddies. The sun is a dim disc behind
    the haze. Visibility drops to 15-20 meters. Horizontal wind drives
    particles in visible streams close to the ground. Static electricity
    can produce occasional lightning within the dust cloud. Abrasive
    and hostile, but with a stark beauty.
  `,
};
```

### Aurora

**Character**: Clear polar night with northern lights. Serene, magical.

```javascript
const AURORA = {
  sky: {
    zenithColor: 0x050515,
    horizonColor: 0x0a0a2a,
    sunIntensity: 0,         // Night
    ambientIntensity: 0.15,
    starField: true,
  },
  particles: { rain: 0, snow: 0, dust: 0 },
  atmosphere: {
    fogDensity: 0.001,
    fogColor: 0x050515,
  },
  aurora: {
    enabled: true,
    colors: [0x00ff88, 0x4400ff, 0xff0066],
    waveSpeed: 0.3,
    curtainHeight: 40,
    curtainWidth: 300,
    intensity: 0.6,
  },
  wind: { base: 1, gustFrequency: 0.1, gustAmplitude: 0.2 },
  camera: { wetLens: 0, frost: 0.02 },
  artistic: `
    Dark clear sky filled with stars. The aurora is the star — an
    undulating curtain of green, purple, and pink light rippling across
    the sky. The light is reflected faintly on water or snow below.
    Everything is still and quiet. The mood is wonder and smallness
    before cosmic beauty. Use additive blending for the curtain glow.
  `,
};
```

## Transition Rules

Not all weather transitions make physical sense. Use this matrix for valid
direct transitions (✓ = natural, ~ = possible but abrupt):

```
           → clear cloudy drizzle rain heavy storm lSnow snow bliz fog sand aurora
clear      │  ─     ✓      ✓      ~    ✗     ✗     ✓     ~    ✗    ✓   ~    ✓
cloudy     │  ✓     ─      ✓      ✓    ~     ✗     ✓     ✓    ✗    ✓   ✗    ✗
drizzle    │  ✓     ✓      ─      ✓    ✓     ✗     ✗     ✗    ✗    ✓   ✗    ✗
rain       │  ~     ✓      ✓      ─    ✓     ✓     ✗     ✗    ✗    ~   ✗    ✗
heavy      │  ✗     ~      ✓      ✓    ─     ✓     ✗     ✗    ✗    ✗   ✗    ✗
storm      │  ✗     ✗      ✗      ✓    ✓     ─     ✗     ✗    ✗    ✗   ✗    ✗
lSnow      │  ✓     ✓      ✗      ✗    ✗     ✗     ─     ✓    ~    ✓   ✗    ✓
snow       │  ~     ✓      ✗      ✗    ✗     ✗     ✓     ─    ✓    ~   ✗    ✗
blizzard   │  ✗     ✗      ✗      ✗    ✗     ✗     ~     ✓    ─    ✗   ✗    ✗
fog        │  ✓     ✓      ✓      ~    ✗     ✗     ✓     ~    ✗    ─   ✗    ✗
sandstorm  │  ~     ✗      ✗      ✗    ✗     ✗     ✗     ✗    ✗    ✗   ─    ✗
aurora     │  ✓     ✗      ✗      ✗    ✗     ✗     ✓     ✗    ✗    ✗   ✗    ─
```

For transitions marked ✗, route through an intermediate state:
- storm → clear: go storm → rain → cloudy → clear
- blizzard → clear: go blizzard → snow → lightSnow → clear

```javascript
function findTransitionPath(from, to) {
  const ROUTES = {
    'storm→clear':     ['rain', 'cloudy', 'clear'],
    'blizzard→clear':  ['snow', 'lightSnow', 'clear'],
    'clear→storm':     ['cloudy', 'rain', 'heavyRain', 'storm'],
    'clear→blizzard':  ['lightSnow', 'snow', 'blizzard'],
    'sandstorm→rain':  ['sandstorm→clear', 'cloudy', 'drizzle', 'rain'],
  };
  const key = `${from}→${to}`;
  return ROUTES[key] ?? [to]; // Direct if no route defined
}
```

## Biome Presets

Quick weather configurations appropriate for different environments.

```javascript
const BIOME_WEATHER = {
  temperate: {
    defaultState: 'cloudy',
    probabilities: { clear: 0.3, cloudy: 0.25, drizzle: 0.15, rain: 0.15, fog: 0.1, storm: 0.05 },
    transitionMinutes: [15, 45], // Weather changes every 15-45 min
  },
  tropical: {
    defaultState: 'clear',
    probabilities: { clear: 0.35, cloudy: 0.15, rain: 0.2, heavyRain: 0.15, storm: 0.1, fog: 0.05 },
    transitionMinutes: [5, 20],  // Tropical weather changes fast
  },
  desert: {
    defaultState: 'clear',
    probabilities: { clear: 0.7, cloudy: 0.1, sandstorm: 0.15, fog: 0.05 },
    transitionMinutes: [30, 120],
  },
  boreal: {
    defaultState: 'cloudy',
    probabilities: { clear: 0.15, cloudy: 0.2, lightSnow: 0.2, snow: 0.2, blizzard: 0.1, fog: 0.1, aurora: 0.05 },
    transitionMinutes: [20, 60],
  },
  savanna: {
    defaultState: 'clear',
    probabilities: { clear: 0.5, cloudy: 0.2, drizzle: 0.1, heavyRain: 0.1, sandstorm: 0.1 },
    transitionMinutes: [20, 90],
  },
};
```
