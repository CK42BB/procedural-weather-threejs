---
name: procedural-weather
description: >
  Generate dynamic procedural weather effects in Three.js using WebGPU compute with
  WebGL2 fallback. Covers rain (drizzle to monsoon), snow (flurries to blizzard), hail,
  fog and mist (ground fog, volumetric), lightning (bolt, sheet, cloud-internal),
  dust/sandstorms, wind-driven debris, rainbows, aurora borealis, and a weather state
  machine for smooth transitions. All precipitation uses GPU particle systems with
  collision, splashes, and accumulation. Provides per-effect lighting changes, ambient
  sound cue hints, and camera post-processing (wet lens, frost). Use when building
  weather systems, atmospheric effects, environmental ambience, climate simulation, or
  any scene needing dynamic weather. Triggers: "procedural weather", "rain effect",
  "snow particles", "lightning", "fog shader", "weather system", "storm", "blizzard",
  "sandstorm", "aurora", "rainbow", "precipitation", "weather transition", "wet lens".
---

# Procedural Weather

Dynamic, layered weather effects in Three.js — GPU particle precipitation, volumetric
fog, lightning, and smooth state transitions.

## Architecture Overview

```
┌──────────────────────────────────────────────────────┐
│                Weather System                         │
│                                                      │
│  WeatherController (state machine)                   │
│    ├── current state + target state                  │
│    ├── transition progress (0→1)                     │
│    └── drives all subsystems:                        │
│                                                      │
│  ┌─ Precipitation ──────────────────────────────┐    │
│  │  GPU particles: rain, snow, hail             │    │
│  │  Ground splashes, accumulation               │    │
│  └──────────────────────────────────────────────┘    │
│  ┌─ Atmosphere ─────────────────────────────────┐    │
│  │  Fog, mist, dust, haze                       │    │
│  │  Volumetric (WebGPU) or exponential (WebGL)  │    │
│  └──────────────────────────────────────────────┘    │
│  ┌─ Electrical ─────────────────────────────────┐    │
│  │  Lightning bolts, sheet flashes              │    │
│  │  Cloud-internal illumination                 │    │
│  └──────────────────────────────────────────────┘    │
│  ┌─ Optical ────────────────────────────────────┐    │
│  │  Rainbows, aurora, god rays                  │    │
│  │  Wet lens, frost overlay                     │    │
│  └──────────────────────────────────────────────┘    │
│  ┌─ Environment ────────────────────────────────┐    │
│  │  Sky darkening, light color shift            │    │
│  │  Wind direction (shared across systems)      │    │
│  └──────────────────────────────────────────────┘    │
└──────────────────────────────────────────────────────┘
```

## Renderer Setup

```javascript
import * as THREE from 'three';

async function createRenderer(canvas) {
  let renderer, gpuAvailable = false;
  try {
    const WebGPU = (await import('three/addons/capabilities/WebGPU.js')).default;
    if (WebGPU.isAvailable()) {
      const { default: WebGPURenderer } = await import(
        'three/addons/renderers/webgpu/WebGPURenderer.js'
      );
      renderer = new WebGPURenderer({ canvas, antialias: true });
      await renderer.init();
      gpuAvailable = true;
    }
  } catch (e) {}
  if (!renderer) {
    renderer = new THREE.WebGLRenderer({ canvas, antialias: true });
    renderer.toneMapping = THREE.ACESFilmicToneMapping;
  }
  renderer.setSize(innerWidth, innerHeight);
  renderer.setPixelRatio(Math.min(devicePixelRatio, 2));
  return { renderer, gpuAvailable };
}
```

## Shared Wind System

Wind drives all weather subsystems — precipitation angle, fog drift, debris direction.

```javascript
class WindSystem {
  constructor() {
    this.direction = new THREE.Vector3(1, 0, 0.3).normalize();
    this.baseSpeed = 5;       // m/s
    this.gustSpeed = 0;
    this.gustFrequency = 0.5;
    this.turbulence = 0.1;
    this._time = 0;
  }

  update(dt) {
    this._time += dt;
    // Gusts: low-frequency intensity variation
    const gustEnvelope = Math.sin(this._time * this.gustFrequency) * 0.5 + 0.5;
    this.gustSpeed = gustEnvelope * this.baseSpeed * 0.6;
  }

  get speed() { return this.baseSpeed + this.gustSpeed; }

  // Wind force vector for particle displacement
  get force() {
    const s = this.speed;
    const turb = new THREE.Vector3(
      Math.sin(this._time * 2.3) * this.turbulence,
      0,
      Math.cos(this._time * 1.7) * this.turbulence
    );
    return this.direction.clone().multiplyScalar(s).add(turb);
  }
}
```

## Precipitation System

GPU particle system for rain, snow, and hail. Particles spawn in a box above the
camera, fall with gravity + wind, and recycle when below ground.

### Particle Buffer Layout

```javascript
function createPrecipitationBuffers(maxParticles) {
  // Each particle: vec4(x, y, z, life) + vec4(vx, vy, vz, size)
  const positions = new Float32Array(maxParticles * 4);
  const velocities = new Float32Array(maxParticles * 4);
  return { positions, velocities, count: maxParticles };
}
```

### Rain System

```javascript
class RainSystem {
  constructor(scene, options = {}) {
    this.scene = scene;
    this.count = options.count ?? 50000;
    this.spawnRadius = options.spawnRadius ?? 40;
    this.spawnHeight = options.spawnHeight ?? 30;
    this.dropLength = options.dropLength ?? 0.3;
    this.intensity = options.intensity ?? 1.0; // 0-1

    this._buildGeometry();
  }

  _buildGeometry() {
    // Each raindrop: a short line segment (2 vertices)
    const positions = new Float32Array(this.count * 6); // 2 verts × xyz
    const randoms = new Float32Array(this.count * 2);   // per-drop seed + phase

    for (let i = 0; i < this.count; i++) {
      const x = (Math.random() - 0.5) * this.spawnRadius * 2;
      const y = Math.random() * this.spawnHeight;
      const z = (Math.random() - 0.5) * this.spawnRadius * 2;

      // Top vertex
      positions[i * 6]     = x;
      positions[i * 6 + 1] = y;
      positions[i * 6 + 2] = z;
      // Bottom vertex (offset by drop length)
      positions[i * 6 + 3] = x;
      positions[i * 6 + 4] = y - this.dropLength;
      positions[i * 6 + 5] = z;

      randoms[i * 2]     = Math.random();     // seed
      randoms[i * 2 + 1] = Math.random();     // phase offset
    }

    const geometry = new THREE.BufferGeometry();
    geometry.setAttribute('position', new THREE.BufferAttribute(positions, 3));
    geometry.setAttribute('aRandom', new THREE.BufferAttribute(randoms, 2, false, this.count));

    this.material = new THREE.ShaderMaterial({
      uniforms: {
        time:       { value: 0 },
        intensity:  { value: this.intensity },
        windForce:  { value: new THREE.Vector3() },
        gravity:    { value: -15.0 },
        spawnHeight: { value: this.spawnHeight },
        spawnRadius: { value: this.spawnRadius },
        dropLength: { value: this.dropLength },
        opacity:    { value: 0.35 },
        cameraPos:  { value: new THREE.Vector3() },
      },
      vertexShader: RAIN_VERT,     // See references/weather-shaders.md
      fragmentShader: RAIN_FRAG,   // See references/weather-shaders.md
      transparent: true,
      depthWrite: false,
      blending: THREE.AdditiveBlending,
    });

    this.mesh = new THREE.LineSegments(geometry, this.material);
    this.mesh.frustumCulled = false;
    this.scene.add(this.mesh);
  }

  update(time, wind, cameraPos) {
    this.material.uniforms.time.value = time;
    this.material.uniforms.windForce.value.copy(wind.force);
    this.material.uniforms.cameraPos.value.copy(cameraPos);
    this.material.uniforms.intensity.value = this.intensity;
    // Recenter spawn box around camera
    this.mesh.position.x = cameraPos.x;
    this.mesh.position.z = cameraPos.z;
  }

  dispose() {
    this.scene.remove(this.mesh);
    this.mesh.geometry.dispose();
    this.material.dispose();
  }
}
```

### Snow System

Snow uses `Points` with per-particle flutter and tumble. Larger particles, slower fall,
more wind influence.

```javascript
class SnowSystem {
  constructor(scene, options = {}) {
    this.scene = scene;
    this.count = options.count ?? 20000;
    this.spawnRadius = options.spawnRadius ?? 50;
    this.spawnHeight = options.spawnHeight ?? 25;
    this.intensity = options.intensity ?? 1.0;

    this._build();
  }

  _build() {
    const positions = new Float32Array(this.count * 3);
    const seeds = new Float32Array(this.count * 3); // seed, phase, size

    for (let i = 0; i < this.count; i++) {
      positions[i * 3]     = (Math.random() - 0.5) * this.spawnRadius * 2;
      positions[i * 3 + 1] = Math.random() * this.spawnHeight;
      positions[i * 3 + 2] = (Math.random() - 0.5) * this.spawnRadius * 2;
      seeds[i * 3]     = Math.random();
      seeds[i * 3 + 1] = Math.random() * Math.PI * 2;
      seeds[i * 3 + 2] = 0.5 + Math.random() * 1.5; // size variation
    }

    const geometry = new THREE.BufferGeometry();
    geometry.setAttribute('position', new THREE.BufferAttribute(positions, 3));
    geometry.setAttribute('aSeed', new THREE.BufferAttribute(seeds, 3));

    this.material = new THREE.ShaderMaterial({
      uniforms: {
        time:        { value: 0 },
        intensity:   { value: this.intensity },
        windForce:   { value: new THREE.Vector3() },
        gravity:     { value: -1.5 },
        spawnHeight: { value: this.spawnHeight },
        spawnRadius: { value: this.spawnRadius },
        flutterAmp:  { value: 2.0 },
        opacity:     { value: 0.85 },
        pointSize:   { value: 3.0 },
        cameraPos:   { value: new THREE.Vector3() },
      },
      vertexShader: SNOW_VERT,     // See references/weather-shaders.md
      fragmentShader: SNOW_FRAG,   // See references/weather-shaders.md
      transparent: true,
      depthWrite: false,
    });

    this.mesh = new THREE.Points(geometry, this.material);
    this.mesh.frustumCulled = false;
    this.scene.add(this.mesh);
  }

  update(time, wind, cameraPos) {
    this.material.uniforms.time.value = time;
    this.material.uniforms.windForce.value.copy(wind.force);
    this.material.uniforms.cameraPos.value.copy(cameraPos);
    this.material.uniforms.intensity.value = this.intensity;
    this.mesh.position.x = cameraPos.x;
    this.mesh.position.z = cameraPos.z;
  }

  dispose() {
    this.scene.remove(this.mesh);
    this.mesh.geometry.dispose();
    this.material.dispose();
  }
}
```

### Splash / Impact System

Ground splashes when rain hits surfaces — instanced ring sprites at impact points.

```javascript
class SplashSystem {
  constructor(scene, maxSplashes = 500) {
    this.scene = scene;
    this.max = maxSplashes;
    const geo = new THREE.PlaneGeometry(0.15, 0.15);
    geo.rotateX(-Math.PI / 2);

    this.material = new THREE.ShaderMaterial({
      uniforms: { time: { value: 0 } },
      vertexShader: SPLASH_VERT,
      fragmentShader: SPLASH_FRAG,
      transparent: true,
      depthWrite: false,
    });

    this.mesh = new THREE.InstancedMesh(geo, this.material, maxSplashes);
    this.mesh.frustumCulled = false;
    this.lifetimes = new Float32Array(maxSplashes);
    this.nextIdx = 0;
    this.scene.add(this.mesh);
  }

  spawn(position) {
    const dummy = new THREE.Object3D();
    dummy.position.copy(position);
    dummy.position.y += 0.01;
    dummy.scale.setScalar(0.5 + Math.random() * 0.5);
    dummy.updateMatrix();
    this.mesh.setMatrixAt(this.nextIdx, dummy.matrix);
    this.lifetimes[this.nextIdx] = 1.0;
    this.mesh.instanceMatrix.needsUpdate = true;
    this.nextIdx = (this.nextIdx + 1) % this.max;
  }

  update(dt) {
    for (let i = 0; i < this.max; i++) {
      if (this.lifetimes[i] > 0) {
        this.lifetimes[i] -= dt * 4; // ~0.25s lifetime
      }
    }
    this.material.uniforms.time.value += dt;
  }
}
```

## Fog & Atmosphere

### Exponential Fog (WebGL — built-in)

```javascript
function applyFog(scene, density = 0.01, color = 0xcccccc) {
  scene.fog = new THREE.FogExp2(color, density);
  scene.background = new THREE.Color(color);
}
```

### Ground Fog (Shader-based)

Height-attenuated fog that pools in valleys. Applied as a post-process or scene material.

```javascript
function createGroundFog(size = 200, maxHeight = 8) {
  const geo = new THREE.PlaneGeometry(size, size, 1, 1);
  geo.rotateX(-Math.PI / 2);

  const material = new THREE.ShaderMaterial({
    uniforms: {
      time:      { value: 0 },
      fogColor:  { value: new THREE.Color(0xdddddd) },
      maxHeight: { value: maxHeight },
      density:   { value: 0.8 },
      windDir:   { value: new THREE.Vector2(1, 0) },
      windSpeed: { value: 2.0 },
    },
    vertexShader: `
      varying vec3 vWorldPos;
      varying vec2 vUv;
      void main() {
        vWorldPos = (modelMatrix * vec4(position, 1.0)).xyz;
        vUv = uv;
        gl_Position = projectionMatrix * modelViewMatrix * vec4(position, 1.0);
      }
    `,
    fragmentShader: GROUND_FOG_FRAG, // See references/weather-shaders.md
    transparent: true,
    depthWrite: false,
    side: THREE.DoubleSide,
  });

  const mesh = new THREE.Mesh(geo, material);
  mesh.position.y = maxHeight * 0.5;
  mesh.renderOrder = 999;
  return mesh;
}
```

### Dust / Sandstorm

Dense particle field with color tinting and reduced visibility.

```javascript
function createDustStorm(scene, options = {}) {
  const count = options.count ?? 30000;
  const color = options.color ?? new THREE.Color(0xc4a060);
  const positions = new Float32Array(count * 3);
  const seeds = new Float32Array(count);

  for (let i = 0; i < count; i++) {
    positions[i * 3]     = (Math.random() - 0.5) * 80;
    positions[i * 3 + 1] = Math.random() * 15;
    positions[i * 3 + 2] = (Math.random() - 0.5) * 80;
    seeds[i] = Math.random();
  }

  const geo = new THREE.BufferGeometry();
  geo.setAttribute('position', new THREE.BufferAttribute(positions, 3));
  geo.setAttribute('aSeed', new THREE.BufferAttribute(seeds, 1));

  const material = new THREE.ShaderMaterial({
    uniforms: {
      time:      { value: 0 },
      dustColor: { value: color },
      windForce: { value: new THREE.Vector3(8, 0, 2) },
      opacity:   { value: 0.4 },
      pointSize: { value: 4.0 },
    },
    vertexShader: DUST_VERT,
    fragmentShader: DUST_FRAG,
    transparent: true,
    depthWrite: false,
    blending: THREE.NormalBlending,
  });

  const mesh = new THREE.Points(geo, material);
  mesh.frustumCulled = false;
  scene.add(mesh);
  return { mesh, material };
}
```

## Lightning System

Procedural branching lightning bolts with flash illumination.

```javascript
class LightningSystem {
  constructor(scene) {
    this.scene = scene;
    this.bolts = [];
    this.flashLight = new THREE.PointLight(0xccccff, 0, 500);
    this.flashLight.position.set(0, 80, 0);
    this.scene.add(this.flashLight);
    this._nextStrike = 2 + Math.random() * 8;
    this._flashDecay = 0;
  }

  // Generate a branching bolt path
  generateBolt(start, end, generations = 5, jitter = 15) {
    if (generations <= 0) return [start, end];

    const mid = start.clone().lerp(end, 0.4 + Math.random() * 0.2);
    const perpX = (Math.random() - 0.5) * jitter;
    const perpZ = (Math.random() - 0.5) * jitter;
    mid.x += perpX;
    mid.z += perpZ;

    const left = this.generateBolt(start, mid, generations - 1, jitter * 0.6);
    const right = this.generateBolt(mid, end, generations - 1, jitter * 0.6);

    // Branch: 30% chance at each midpoint
    let branch = [];
    if (Math.random() < 0.3 && generations > 2) {
      const branchEnd = mid.clone().add(
        new THREE.Vector3((Math.random() - 0.5) * jitter * 2, -jitter, (Math.random() - 0.5) * jitter * 2)
      );
      branch = this.generateBolt(mid, branchEnd, generations - 2, jitter * 0.4);
    }

    return [...left, ...right.slice(1), ...branch];
  }

  _createBoltMesh(points) {
    const positions = new Float32Array(points.length * 3);
    for (let i = 0; i < points.length; i++) {
      positions[i * 3]     = points[i].x;
      positions[i * 3 + 1] = points[i].y;
      positions[i * 3 + 2] = points[i].z;
    }
    const geo = new THREE.BufferGeometry();
    geo.setAttribute('position', new THREE.BufferAttribute(positions, 3));

    const material = new THREE.LineBasicMaterial({
      color: 0xeeeeff,
      transparent: true,
      opacity: 1.0,
      linewidth: 2,
    });

    // Core bolt
    const line = new THREE.Line(geo, material);
    // Glow (wider, transparent)
    const glowMat = new THREE.LineBasicMaterial({
      color: 0x8888ff,
      transparent: true,
      opacity: 0.4,
      linewidth: 1,
    });
    const glow = new THREE.Line(geo.clone(), glowMat);
    glow.scale.setScalar(1.02);

    const group = new THREE.Group();
    group.add(line);
    group.add(glow);
    return { group, material, glowMat, life: 0.3 };
  }

  strike(origin, groundY = 0) {
    const start = origin ?? new THREE.Vector3(
      (Math.random() - 0.5) * 100, 70 + Math.random() * 30, (Math.random() - 0.5) * 100
    );
    const end = new THREE.Vector3(start.x + (Math.random() - 0.5) * 20, groundY, start.z + (Math.random() - 0.5) * 20);

    const points = this.generateBolt(start, end, 6, 12);
    const bolt = this._createBoltMesh(points);
    this.scene.add(bolt.group);
    this.bolts.push(bolt);

    // Flash
    this.flashLight.position.copy(start);
    this.flashLight.intensity = 15;
    this._flashDecay = 0.15;
  }

  update(dt, stormIntensity = 0.5) {
    // Auto-strike timing
    this._nextStrike -= dt;
    if (this._nextStrike <= 0 && stormIntensity > 0.3) {
      this.strike();
      this._nextStrike = (1 / stormIntensity) * (3 + Math.random() * 7);
    }

    // Fade bolts
    for (let i = this.bolts.length - 1; i >= 0; i--) {
      const bolt = this.bolts[i];
      bolt.life -= dt;
      const alpha = Math.max(bolt.life / 0.3, 0);
      bolt.material.opacity = alpha;
      bolt.glowMat.opacity = alpha * 0.4;
      if (bolt.life <= 0) {
        this.scene.remove(bolt.group);
        bolt.group.traverse(c => { if (c.geometry) c.geometry.dispose(); });
        this.bolts.splice(i, 1);
      }
    }

    // Flash decay
    if (this._flashDecay > 0) {
      this._flashDecay -= dt;
      this.flashLight.intensity *= 0.85;
    } else {
      this.flashLight.intensity = 0;
    }
  }

  dispose() {
    for (const bolt of this.bolts) {
      this.scene.remove(bolt.group);
    }
    this.scene.remove(this.flashLight);
  }
}
```

## Optical Effects

### Rainbow

Arc rendered as a screen-space shader overlay or a mesh arc in world space.

```javascript
function createRainbow(scene, sunDir) {
  const geo = new THREE.TorusGeometry(120, 3, 16, 64, Math.PI);
  const material = new THREE.ShaderMaterial({
    uniforms: {
      opacity: { value: 0.25 },
    },
    vertexShader: `
      varying vec2 vUv;
      void main() {
        vUv = uv;
        gl_Position = projectionMatrix * modelViewMatrix * vec4(position, 1.0);
      }
    `,
    fragmentShader: `
      varying vec2 vUv;
      uniform float opacity;
      void main() {
        float t = vUv.x;
        // ROYGBIV spectrum across the arc width
        vec3 col;
        if (t < 0.14)      col = vec3(1.0, 0.0, 0.0);
        else if (t < 0.28) col = vec3(1.0, 0.5, 0.0);
        else if (t < 0.42) col = vec3(1.0, 1.0, 0.0);
        else if (t < 0.57) col = vec3(0.0, 0.8, 0.0);
        else if (t < 0.71) col = vec3(0.0, 0.4, 1.0);
        else if (t < 0.85) col = vec3(0.3, 0.0, 0.8);
        else               col = vec3(0.5, 0.0, 0.5);
        // Soft edges
        float edge = smoothstep(0.0, 0.05, t) * smoothstep(1.0, 0.95, t);
        gl_FragColor = vec4(col, opacity * edge);
      }
    `,
    transparent: true,
    depthWrite: false,
    side: THREE.DoubleSide,
  });

  const rainbow = new THREE.Mesh(geo, material);
  // Position opposite to sun direction
  rainbow.position.set(-sunDir.x * 80, 20, -sunDir.z * 80);
  rainbow.rotation.z = Math.PI;
  rainbow.rotation.y = Math.atan2(-sunDir.x, -sunDir.z);
  scene.add(rainbow);
  return rainbow;
}
```

### Aurora Borealis

Undulating curtain of colored light using a displaced vertical plane.

```javascript
function createAurora(scene) {
  const geo = new THREE.PlaneGeometry(300, 40, 128, 16);
  const material = new THREE.ShaderMaterial({
    uniforms: {
      time: { value: 0 },
      color1: { value: new THREE.Color(0x00ff88) },
      color2: { value: new THREE.Color(0x4400ff) },
      color3: { value: new THREE.Color(0xff0066) },
    },
    vertexShader: AURORA_VERT,     // See references/weather-shaders.md
    fragmentShader: AURORA_FRAG,   // See references/weather-shaders.md
    transparent: true,
    depthWrite: false,
    side: THREE.DoubleSide,
    blending: THREE.AdditiveBlending,
  });

  const mesh = new THREE.Mesh(geo, material);
  mesh.position.set(0, 60, -100);
  mesh.rotation.x = -0.3;
  scene.add(mesh);
  return { mesh, material };
}
```

## Weather State Machine

Smooth transitions between weather states by interpolating all subsystem parameters.

```javascript
class WeatherController {
  constructor(scene, camera, wind) {
    this.scene = scene;
    this.camera = camera;
    this.wind = wind;
    this.rain = null;
    this.snow = null;
    this.lightning = null;
    this.fog = null;
    this.currentState = 'clear';
    this.targetState = 'clear';
    this.transition = 1.0;     // 1 = fully arrived at target
    this.transitionSpeed = 0.3; // 0→1 over ~3 seconds
  }

  setState(stateName) {
    if (stateName === this.currentState && this.transition >= 1) return;
    this.currentState = this.targetState;
    this.targetState = stateName;
    this.transition = 0;
  }

  update(dt) {
    this.wind.update(dt);

    if (this.transition < 1) {
      this.transition = Math.min(this.transition + dt * this.transitionSpeed, 1);
    }

    const t = this.transition;
    const from = WEATHER_STATES[this.currentState];
    const to = WEATHER_STATES[this.targetState];

    // Interpolate parameters
    const rainIntensity = lerp(from.rain, to.rain, t);
    const snowIntensity = lerp(from.snow, to.snow, t);
    const fogDensity = lerp(from.fogDensity, to.fogDensity, t);
    const stormIntensity = lerp(from.lightning, to.lightning, t);
    const skyDarkness = lerp(from.skyDarkness, to.skyDarkness, t);
    const windMult = lerp(from.windMultiplier, to.windMultiplier, t);

    this.wind.baseSpeed = 5 * windMult;

    // Update subsystems
    if (this.rain) {
      this.rain.intensity = rainIntensity;
      this.rain.update(performance.now() * 0.001, this.wind, this.camera.position);
    }
    if (this.snow) {
      this.snow.intensity = snowIntensity;
      this.snow.update(performance.now() * 0.001, this.wind, this.camera.position);
    }
    if (this.lightning) {
      this.lightning.update(dt, stormIntensity);
    }

    // Fog
    if (this.scene.fog) {
      this.scene.fog.density = fogDensity;
    }

    // Sky darkening
    const skyColor = new THREE.Color(0x87ceeb).lerp(new THREE.Color(0x333340), skyDarkness);
    if (this.scene.background instanceof THREE.Color) {
      this.scene.background.copy(skyColor);
    }
  }
}

function lerp(a, b, t) { return a + (b - a) * t; }

const WEATHER_STATES = {
  clear:      { rain: 0, snow: 0, fogDensity: 0.0005, lightning: 0, skyDarkness: 0, windMultiplier: 0.5 },
  cloudy:     { rain: 0, snow: 0, fogDensity: 0.002, lightning: 0, skyDarkness: 0.2, windMultiplier: 0.8 },
  drizzle:    { rain: 0.3, snow: 0, fogDensity: 0.004, lightning: 0, skyDarkness: 0.3, windMultiplier: 0.7 },
  rain:       { rain: 0.7, snow: 0, fogDensity: 0.006, lightning: 0.1, skyDarkness: 0.5, windMultiplier: 1.2 },
  heavyRain:  { rain: 1.0, snow: 0, fogDensity: 0.01, lightning: 0.3, skyDarkness: 0.7, windMultiplier: 1.8 },
  storm:      { rain: 1.0, snow: 0, fogDensity: 0.015, lightning: 0.8, skyDarkness: 0.85, windMultiplier: 2.5 },
  lightSnow:  { rain: 0, snow: 0.3, fogDensity: 0.003, lightning: 0, skyDarkness: 0.15, windMultiplier: 0.6 },
  snow:       { rain: 0, snow: 0.7, fogDensity: 0.006, lightning: 0, skyDarkness: 0.3, windMultiplier: 1.0 },
  blizzard:   { rain: 0, snow: 1.0, fogDensity: 0.025, lightning: 0, skyDarkness: 0.6, windMultiplier: 3.0 },
  fog:        { rain: 0, snow: 0, fogDensity: 0.03, lightning: 0, skyDarkness: 0.25, windMultiplier: 0.2 },
  sandstorm:  { rain: 0, snow: 0, fogDensity: 0.02, lightning: 0.1, skyDarkness: 0.5, windMultiplier: 3.0 },
};
```

## Camera Post-Processing

### Wet Lens Effect

Droplets on screen when raining, applied as a fullscreen overlay.

```javascript
function createWetLensOverlay(intensity = 0.5) {
  return new THREE.ShaderMaterial({
    uniforms: {
      time:      { value: 0 },
      intensity: { value: intensity },
      tScene:    { value: null },
    },
    vertexShader: FULLSCREEN_VERT,
    fragmentShader: WET_LENS_FRAG, // See references/weather-shaders.md
    transparent: true,
  });
}
```

### Frost Overlay

Progressive frost creep from screen edges during cold/blizzard conditions.

```javascript
function createFrostOverlay(intensity = 0.0) {
  return new THREE.ShaderMaterial({
    uniforms: {
      intensity: { value: intensity }, // 0=clear, 1=fully frosted
      time:      { value: 0 },
    },
    vertexShader: FULLSCREEN_VERT,
    fragmentShader: FROST_FRAG,
    transparent: true,
  });
}
```

## Complete Scene Assembly

```javascript
async function init() {
  const canvas = document.querySelector('#canvas');
  const { renderer } = await createRenderer(canvas);

  const scene = new THREE.Scene();
  scene.background = new THREE.Color(0x87ceeb);
  scene.fog = new THREE.FogExp2(0x87ceeb, 0.001);

  const camera = new THREE.PerspectiveCamera(60, innerWidth / innerHeight, 0.5, 500);
  camera.position.set(0, 5, 20);

  const { OrbitControls } = await import('three/addons/controls/OrbitControls.js');
  const controls = new OrbitControls(camera, renderer.domElement);
  controls.enableDamping = true;

  // Ground
  const ground = new THREE.Mesh(
    new THREE.PlaneGeometry(200, 200),
    new THREE.MeshStandardMaterial({ color: 0x4a7c3f, roughness: 0.9 })
  );
  ground.rotation.x = -Math.PI / 2;
  ground.receiveShadow = true;
  scene.add(ground);

  // Lighting
  const sun = new THREE.DirectionalLight(0xfff4e5, 1.2);
  sun.position.set(30, 40, 20);
  scene.add(sun);
  scene.add(new THREE.HemisphereLight(0x87ceeb, 0x4a7c3f, 0.5));

  // Weather
  const wind = new WindSystem();
  const weather = new WeatherController(scene, camera, wind);
  weather.rain = new RainSystem(scene);
  weather.snow = new SnowSystem(scene);
  weather.lightning = new LightningSystem(scene);

  // Start with rain
  weather.setState('rain');

  const clock = new THREE.Clock();
  renderer.setAnimationLoop(() => {
    weather.update(clock.getDelta());
    controls.update();
    renderer.render(scene, camera);
  });

  window.addEventListener('resize', () => {
    camera.aspect = innerWidth / innerHeight;
    camera.updateProjectionMatrix();
    renderer.setSize(innerWidth, innerHeight);
  });
}

init();
```

## Performance Guidelines

| Effect | Particle Count | Draw Calls | Notes |
|--------|---------------|------------|-------|
| Rain | 30K–80K | 1 (LineSegments) | Additive blend, no depth write |
| Snow | 10K–30K | 1 (Points) | Larger point size = fewer needed |
| Splashes | 200–500 | 1 (InstancedMesh) | Short lifetime, ring-buffer |
| Dust | 15K–40K | 1 (Points) | Camera-centered spawn box |
| Lightning | 1–3 bolts | 2–6 (Lines) | Ephemeral, negligible cost |
| Fog plane | 1 quad | 1 | Fragment-heavy, keep simple |
| Aurora | 1 plane | 1 | Vertex displacement only |

**Total budget**: All weather combined stays under 5 draw calls and ~100K particles.

**Key rules**:
- All animation in vertex shaders via uniforms (time, wind). Zero per-frame JS particle loops.
- `frustumCulled = false` on all particle meshes — spawn box follows camera.
- Recenter spawn box on camera each frame so particles always surround the viewer.
- `depthWrite: false` on all transparent weather effects to avoid sorting artifacts.

## Common Pitfalls

1. **Rain falls through roof**: Precipitation has no collision. For indoor scenes, cull particles below a Y threshold or use a height map mask uniform.
2. **Snow accumulation**: Not simulated per-particle. Use a ground plane shader that blends white coverage based on `snowIntensity * time`, or a decal system.
3. **Lightning too regular**: Randomize interval per `stormIntensity`. Real lightning is clustered — bursts of 2–3 strikes then silence.
4. **Fog and sky mismatch**: Always set `scene.fog.color` = `scene.background`. Mismatched colors create visible hard edges at the distance limit.
5. **Weather pops on/off**: Always use the state machine transition. Interpolating intensity from 0→target over 2–3 seconds looks natural.

## References

- `references/weather-shaders.md` — Complete GLSL vertex/fragment shaders for rain, snow, dust, ground fog, aurora, wet lens, frost, and WGSL compute particles.
- `references/weather-types.md` — Detailed profiles for 12 weather states with parameter tables, artistic direction, environmental effects, and combination rules.
