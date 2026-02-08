# Weather Shaders Reference

Complete shader implementations for all weather effects: rain, snow, dust, ground fog,
aurora, wet lens, frost overlay, and WGSL compute particles.

## Shared: Fullscreen Vertex Shader

```glsl
varying vec2 vUv;
void main() {
  vUv = uv;
  gl_Position = vec4(position.xy, 0.0, 1.0);
}
```

## Rain Shaders

### Vertex Shader

Animates raindrop positions with gravity + wind, wrapping around the spawn box.

```glsl
// rain.vert
attribute vec2 aRandom; // per-drop seed, phase

uniform float time;
uniform float intensity;
uniform vec3  windForce;
uniform float gravity;
uniform float spawnHeight;
uniform float spawnRadius;
uniform float dropLength;
uniform vec3  cameraPos;

varying float vAlpha;

float hash(float n) { return fract(sin(n) * 43758.5453); }

void main() {
  float seed = aRandom.x;
  float phase = aRandom.y;

  // Determine if this is top (even) or bottom (odd) vertex of drop
  float vertIdx = mod(float(gl_VertexID), 2.0);

  vec3 pos = position;

  // Fall animation: wrap within spawn box
  float fallSpeed = gravity * (0.8 + seed * 0.4);
  float t = time + phase * 10.0;
  pos.y = mod(pos.y + fallSpeed * t, spawnHeight);

  // Wind displacement — increases with fall time
  float fallProgress = 1.0 - pos.y / spawnHeight;
  pos.x += windForce.x * fallProgress * 0.5;
  pos.z += windForce.z * fallProgress * 0.5;

  // Wrap horizontal
  pos.x = mod(pos.x + spawnRadius, spawnRadius * 2.0) - spawnRadius;
  pos.z = mod(pos.z + spawnRadius, spawnRadius * 2.0) - spawnRadius;

  // Bottom vertex offset (stretched by fall speed for motion blur)
  if (vertIdx > 0.5) {
    float stretch = dropLength * (1.0 + length(windForce) * 0.05);
    pos.y -= stretch;
    pos.x += windForce.x * 0.01;
  }

  // Fade out drops not active at current intensity
  vAlpha = step(seed, intensity) * 0.35;

  // Distance fade
  float dist = length(pos.xz);
  vAlpha *= smoothstep(spawnRadius, spawnRadius * 0.6, dist);

  gl_Position = projectionMatrix * modelViewMatrix * vec4(pos, 1.0);
}
```

### Fragment Shader

```glsl
// rain.frag
precision highp float;
uniform float opacity;
varying float vAlpha;

void main() {
  if (vAlpha < 0.01) discard;
  // Slight blue-white color
  gl_FragColor = vec4(0.7, 0.75, 0.9, vAlpha * opacity);
}
```

## Snow Shaders

### Vertex Shader

Flutter, tumble, and drift with sinusoidal paths.

```glsl
// snow.vert
attribute vec3 aSeed; // seed, phase, size

uniform float time;
uniform float intensity;
uniform vec3  windForce;
uniform float gravity;
uniform float spawnHeight;
uniform float spawnRadius;
uniform float flutterAmp;
uniform float pointSize;
uniform vec3  cameraPos;

varying float vAlpha;
varying float vSize;

void main() {
  float seed = aSeed.x;
  float phase = aSeed.y;
  float sizeVar = aSeed.z;

  vec3 pos = position;

  // Slow fall
  float fallSpeed = gravity * (0.6 + seed * 0.8);
  float t = time + phase * 8.0;
  pos.y = mod(pos.y + fallSpeed * t, spawnHeight);

  // Flutter: sinusoidal lateral drift
  float flutter = sin(t * (1.5 + seed * 2.0) + phase) * flutterAmp;
  float flutter2 = cos(t * (1.0 + seed * 1.5) + phase * 2.0) * flutterAmp * 0.7;
  pos.x += flutter + windForce.x * (1.0 - pos.y / spawnHeight) * 0.8;
  pos.z += flutter2 + windForce.z * (1.0 - pos.y / spawnHeight) * 0.8;

  // Wrap
  pos.x = mod(pos.x + spawnRadius, spawnRadius * 2.0) - spawnRadius;
  pos.z = mod(pos.z + spawnRadius, spawnRadius * 2.0) - spawnRadius;

  // Intensity gating
  vAlpha = step(seed, intensity) * 0.85;

  // Distance fade
  float dist = length(pos.xz);
  vAlpha *= smoothstep(spawnRadius, spawnRadius * 0.5, dist);

  vSize = sizeVar;

  vec4 mvPos = modelViewMatrix * vec4(pos, 1.0);
  gl_Position = projectionMatrix * mvPos;
  gl_PointSize = pointSize * sizeVar * (200.0 / -mvPos.z);
}
```

### Fragment Shader

Soft circular snowflake with slight sparkle.

```glsl
// snow.frag
precision highp float;
uniform float opacity;
varying float vAlpha;
varying float vSize;

void main() {
  if (vAlpha < 0.01) discard;

  // Circular soft point
  vec2 center = gl_PointCoord - 0.5;
  float dist = length(center);
  float alpha = smoothstep(0.5, 0.2, dist);

  // Subtle sparkle on larger flakes
  float sparkle = max(sin(dist * 20.0 + vSize * 10.0) * 0.15, 0.0);

  gl_FragColor = vec4(0.95 + sparkle, 0.97 + sparkle, 1.0, alpha * vAlpha * opacity);
}
```

## Dust / Sandstorm Shaders

### Vertex Shader

```glsl
// dust.vert
attribute float aSeed;

uniform float time;
uniform vec3  windForce;
uniform float pointSize;

varying float vAlpha;

void main() {
  vec3 pos = position;

  // Swirling motion
  float t = time + aSeed * 20.0;
  pos.x += windForce.x * t * 0.3 + sin(t * 2.0 + aSeed * 6.28) * 2.0;
  pos.z += windForce.z * t * 0.3 + cos(t * 1.5 + aSeed * 6.28) * 2.0;
  pos.y += sin(t * 3.0 + aSeed * 10.0) * 1.5;

  // Wrap
  float r = 40.0;
  pos.x = mod(pos.x + r, r * 2.0) - r;
  pos.z = mod(pos.z + r, r * 2.0) - r;
  pos.y = mod(pos.y, 15.0);

  vAlpha = 0.3 + aSeed * 0.3;

  vec4 mvPos = modelViewMatrix * vec4(pos, 1.0);
  gl_Position = projectionMatrix * mvPos;
  gl_PointSize = pointSize * (100.0 / -mvPos.z);
}
```

### Fragment Shader

```glsl
// dust.frag
precision highp float;
uniform vec3  dustColor;
uniform float opacity;
varying float vAlpha;

void main() {
  vec2 c = gl_PointCoord - 0.5;
  float d = length(c);
  float alpha = smoothstep(0.5, 0.15, d);
  gl_FragColor = vec4(dustColor, alpha * vAlpha * opacity);
}
```

## Ground Fog Fragment Shader

Height-attenuated volumetric fog using 2D noise for wisps.

```glsl
// ground_fog.frag
precision highp float;

uniform float time;
uniform vec3  fogColor;
uniform float maxHeight;
uniform float density;
uniform vec2  windDir;
uniform float windSpeed;

varying vec3 vWorldPos;
varying vec2 vUv;

float hash(vec2 p) { return fract(sin(dot(p, vec2(127.1, 311.7))) * 43758.5453); }

float noise(vec2 p) {
  vec2 i = floor(p);
  vec2 f = fract(p);
  f = f * f * (3.0 - 2.0 * f);
  return mix(mix(hash(i), hash(i + vec2(1,0)), f.x),
             mix(hash(i + vec2(0,1)), hash(i + vec2(1,1)), f.x), f.y);
}

float fbm(vec2 p) {
  float sum = 0.0, amp = 1.0;
  for (int i = 0; i < 4; i++) {
    sum += noise(p) * amp;
    p *= 2.0; amp *= 0.5;
  }
  return sum / 1.875;
}

void main() {
  vec2 windOffset = windDir * windSpeed * time * 0.01;

  // Wispy noise pattern
  float n = fbm(vUv * 5.0 + windOffset);
  n = smoothstep(0.3, 0.7, n);

  // Height fade — densest at ground, gone at maxHeight
  float heightFade = 1.0 - smoothstep(0.0, maxHeight, vWorldPos.y);
  heightFade *= heightFade; // Quadratic falloff

  float alpha = n * heightFade * density;

  gl_FragColor = vec4(fogColor, alpha);
}
```

## Aurora Shaders

### Vertex Shader

Undulating curtain displacement.

```glsl
// aurora.vert
uniform float time;

varying vec2 vUv;
varying float vIntensity;

float noise(vec2 p) {
  return fract(sin(dot(p, vec2(127.1, 311.7))) * 43758.5453);
}

void main() {
  vUv = uv;
  vec3 pos = position;

  // Curtain wave
  float wave1 = sin(pos.x * 0.02 + time * 0.3) * 8.0;
  float wave2 = sin(pos.x * 0.05 + time * 0.7) * 3.0;
  float wave3 = cos(pos.x * 0.03 + time * 0.15) * 5.0;
  pos.y += wave1 + wave2;
  pos.z += wave3;

  // Ruffle along curtain
  float ruffle = sin(pos.x * 0.1 + time * 1.5) * 2.0 * uv.y;
  pos.z += ruffle;

  // Intensity varies along curtain
  vIntensity = smoothstep(0.0, 0.3, uv.y) * smoothstep(1.0, 0.6, uv.y);
  vIntensity *= 0.5 + 0.5 * sin(pos.x * 0.03 + time * 0.2);

  gl_Position = projectionMatrix * modelViewMatrix * vec4(pos, 1.0);
}
```

### Fragment Shader

```glsl
// aurora.frag
precision highp float;

uniform vec3 color1; // Green
uniform vec3 color2; // Purple
uniform vec3 color3; // Pink

varying vec2 vUv;
varying float vIntensity;

void main() {
  // Color gradient along height
  vec3 col;
  if (vUv.y < 0.4) {
    col = mix(color2, color1, vUv.y / 0.4);
  } else if (vUv.y < 0.7) {
    col = mix(color1, color3, (vUv.y - 0.4) / 0.3);
  } else {
    col = mix(color3, color2, (vUv.y - 0.7) / 0.3);
  }

  // Horizontal variation
  float hVar = sin(vUv.x * 20.0) * 0.1 + 0.9;

  float alpha = vIntensity * hVar;
  alpha *= smoothstep(0.0, 0.1, vUv.y) * smoothstep(1.0, 0.85, vUv.y);

  gl_FragColor = vec4(col * 1.5, alpha * 0.6);
}
```

## Wet Lens Fragment Shader

Simulates rain droplets on the camera lens with refraction distortion.

```glsl
// wet_lens.frag
precision highp float;

uniform float time;
uniform float intensity; // 0-1
uniform sampler2D tScene;

varying vec2 vUv;

float hash(vec2 p) { return fract(sin(dot(p, vec2(127.1, 311.7))) * 43758.5453); }

void main() {
  vec2 uv = vUv;
  vec4 sceneColor = texture2D(tScene, uv);

  if (intensity < 0.01) {
    gl_FragColor = sceneColor;
    return;
  }

  // Generate droplet pattern
  vec2 grid = floor(uv * 30.0);
  float droplet = 0.0;
  vec2 refractOffset = vec2(0.0);

  for (float dy = -1.0; dy <= 1.0; dy++) {
    for (float dx = -1.0; dx <= 1.0; dx++) {
      vec2 cell = grid + vec2(dx, dy);
      float h = hash(cell + floor(time * 0.5)); // Drops change over time
      if (h > 1.0 - intensity * 0.4) {
        vec2 dropPos = (cell + 0.5 + (hash(cell * 1.3) - 0.5) * 0.8) / 30.0;
        float dist = length(uv - dropPos);
        float radius = 0.005 + hash(cell * 2.7) * 0.01;
        float drop = smoothstep(radius, radius * 0.3, dist);
        droplet = max(droplet, drop);
        // Refraction: offset based on distance from drop center
        vec2 dir = normalize(uv - dropPos);
        refractOffset += dir * drop * 0.003;
      }
    }
  }

  // Streaks (running drops)
  float streak = hash(vec2(floor(uv.x * 60.0), 0.0));
  if (streak > 0.92 && intensity > 0.5) {
    float streakY = fract(uv.y * 3.0 - time * 0.2 + streak);
    float streakAlpha = smoothstep(0.0, 0.02, streakY) * smoothstep(0.15, 0.05, streakY);
    refractOffset.y += streakAlpha * 0.005;
    droplet = max(droplet, streakAlpha * 0.5);
  }

  vec4 refracted = texture2D(tScene, uv + refractOffset);
  vec4 result = mix(sceneColor, refracted, droplet);

  // Slight darkening from water film
  result.rgb *= 1.0 - intensity * 0.05;

  gl_FragColor = result;
}
```

## Frost Overlay Fragment Shader

Progressive frost creep from edges using noise.

```glsl
// frost.frag
precision highp float;

uniform float intensity; // 0=clear, 1=fully frosted
uniform float time;

varying vec2 vUv;

float hash(vec2 p) { return fract(sin(dot(p, vec2(127.1, 311.7))) * 43758.5453); }

float noise(vec2 p) {
  vec2 i = floor(p);
  vec2 f = fract(p);
  f = f * f * (3.0 - 2.0 * f);
  return mix(mix(hash(i), hash(i+vec2(1,0)), f.x),
             mix(hash(i+vec2(0,1)), hash(i+vec2(1,1)), f.x), f.y);
}

float fbm(vec2 p) {
  float s = 0.0, a = 1.0;
  for (int i = 0; i < 5; i++) { s += noise(p) * a; p *= 2.0; a *= 0.5; }
  return s / 1.9375;
}

void main() {
  // Distance from screen edge
  vec2 centered = vUv * 2.0 - 1.0;
  float edgeDist = max(abs(centered.x), abs(centered.y));

  // Frost creeps from edges inward based on intensity
  float frostLine = mix(1.2, -0.2, intensity);
  float n = fbm(vUv * 15.0 + time * 0.02);
  float frost = smoothstep(frostLine, frostLine + 0.3, edgeDist + n * 0.3);

  // Crystal pattern
  float crystal = fbm(vUv * 40.0) * frost;
  float sparkle = max(sin(n * 50.0) * frost * 0.3, 0.0);

  vec3 frostColor = vec3(0.85, 0.9, 1.0) + sparkle;
  float alpha = frost * 0.7 + crystal * 0.2;

  gl_FragColor = vec4(frostColor, alpha);
}
```

## WGSL Compute: GPU Particle Update

Update all precipitation particles on the GPU each frame instead of relying on
vertex shader animation. Enables true collision detection and accumulation.

```wgsl
// particle_update.wgsl
struct Particle {
  pos: vec4<f32>,    // xyz + life
  vel: vec4<f32>,    // xyz + size
}

@group(0) @binding(0) var<storage, read_write> particles: array<Particle>;

struct Params {
  time: f32,
  dt: f32,
  gravity: f32,
  windX: f32,
  windZ: f32,
  spawnHeight: f32,
  spawnRadius: f32,
  groundY: f32,
  count: u32,
  intensity: f32,
}
@group(0) @binding(1) var<uniform> params: Params;

fn hash(n: f32) -> f32 {
  return fract(sin(n) * 43758.5453);
}

@compute @workgroup_size(256)
fn main(@builtin(global_invocation_id) gid: vec3<u32>) {
  if (gid.x >= params.count) { return; }

  var p = particles[gid.x];

  // Apply forces
  p.vel.y += params.gravity * params.dt;
  p.vel.x += (params.windX - p.vel.x) * params.dt * 0.5;
  p.vel.z += (params.windZ - p.vel.z) * params.dt * 0.5;

  // Integrate position
  p.pos.x += p.vel.x * params.dt;
  p.pos.y += p.vel.y * params.dt;
  p.pos.z += p.vel.z * params.dt;

  // Ground collision → respawn at top
  if (p.pos.y < params.groundY) {
    let seed = hash(f32(gid.x) + params.time);
    p.pos.x = (hash(seed * 127.1) - 0.5) * params.spawnRadius * 2.0;
    p.pos.y = params.spawnHeight + hash(seed * 311.7) * 5.0;
    p.pos.z = (hash(seed * 74.7) - 0.5) * params.spawnRadius * 2.0;
    p.vel = vec4<f32>(params.windX * 0.3, params.gravity * 0.5, params.windZ * 0.3, p.vel.w);
  }

  // Wrap horizontal
  let r = params.spawnRadius;
  p.pos.x = ((p.pos.x + r) % (r * 2.0)) - r;
  p.pos.z = ((p.pos.z + r) % (r * 2.0)) - r;

  // Intensity: disable particles by pushing off-screen
  if (hash(f32(gid.x)) > params.intensity) {
    p.pos.y = -1000.0;
  }

  particles[gid.x] = p;
}
```

## Performance Notes

- **Vertex shader animation** (rain/snow vert shaders): Zero CPU cost per particle — all via `time` + `mod()` wrapping. Preferred for WebGL path.
- **Compute particle update** (WGSL): Enables collision with heightmap and true accumulation. Use for WebGPU path when terrain interaction matters.
- **Additive blending** on rain avoids transparent sort cost but loses ability to darken. Use `NormalBlending` for snow/dust.
- **Point size attenuation**: `gl_PointSize = baseSize * (constant / -mvPosition.z)` — near particles larger, far particles smaller. Essential for depth.
- **Shader-defined strings**: Embed shaders as JS template literals or use import-map-compatible `.glsl` file loading.
