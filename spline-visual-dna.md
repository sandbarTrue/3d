# Spline Visual DNA Analysis & Surpass Recipe

## 1. What Spline Does Specifically

### Material System: Layered Compositing
Spline's secret weapon is a **multi-layer material compositing system**. Unlike raw Three.js where you get one material, Spline stacks multiple effect layers with blend modes:

| Layer | Purpose | Blend Mode | Key Params |
|-------|---------|------------|------------|
| **Lighting Layer** | Base PBR shading (Physical mode for premium) | Base | Lambert/Phong/Physical/Toon |
| **Depth Layer (3D Gradient)** | Camera-distance-based color gradient, simulates SSS/fake lighting | Overlay/Screen | Vector or Camera origin, Linear/Smooth blend |
| **Noise Layer** | Organic color variation across surface | Overlay | Simplex Fractal, size ~20, scale ~20, movement ~10-20 |
| **Fresnel Layer** | Rim-light/edge glow based on view angle | Overlay | Color (often darker shade), Bias ~0.14 |
| **Rainbow Layer** | Iridescent color shift | Overlay | Intensity ~20, Movement ~20 |
| **Matcap Layer** | Faked environment reflections, consistent look regardless of lighting | Screen | Iridescent matcap textures common |
| **Glass Layer** | Refraction + blur | Base | Blur ~100, Thickness ~250 |
| **Displace Layer** | Vertex displacement for organic deformation | N/A | Scale ~20, Movement ~10 |

**The "Spline Look" recipe for premium blobs:**
1. Physical lighting layer (base PBR)
2. Simplex Fractal noise layer (overlay) for color richness
3. Fresnel layer (overlay, bias 0.14) for edge shimmer
4. Rainbow layer (overlay, intensity 20) for iridescence
5. Matcap with iridescent texture (screen) for environment capture
6. Near-black background to make materials pop

### Lighting Setup
- Spline supports: directional, point, spot, area lights
- **Premium scenes use 3-point lighting minimum**: key (warm), fill (cool), rim (accent color)
- Built-in HDRI environment presets for reflections
- Area lights are common for soft, wrap-around illumination

### Post-Processing (Spline's 8 Effects)
1. **Bloom** - Soft glow on bright areas (moderate, never blown out)
2. **Depth of Field** - Subtle focus falloff (adds cinematic depth)
3. **Chromatic Aberration** - Subtle RGB split at edges (2-3px max)
4. **Vignette** - Dark corners for cinematic framing
5. **Noise/Grain** - Film grain texture (very subtle)
6. **Hue** - Global color correction
7. **Brightness** - Exposure control
8. **Pixelation** - Stylistic (rarely used for premium)

**Spline's post-processing is conservative by default** - users add 2-3 effects max for performance. The "premium" stack is: Bloom + subtle Vignette + optional DOF.

### Animation Patterns
- **Cycle Ping Pong easing** for blob breathing (5s period)
- Mouse/cursor tracking for parallax and tilt
- Physics engine for collision/gravity
- State-based transitions (expand/contract/morph)
- Slow continuous rotation (0.001-0.005 rad/frame)

---

## 2. What TOP WebGL Sites Do That Spline Doesn't

### Advanced Shader Techniques (Awwwards/FWA Winners)
| Technique | Description | Spline Can't Do This |
|-----------|-------------|---------------------|
| **Custom FBM (Fractal Brownian Motion)** | Multi-octave noise with lacunarity/persistence control for rich organic displacement | Spline only has single-layer displacement |
| **Domain Warping** | Feeding noise into noise for alien/organic distortion | No equivalent in Spline |
| **Thin-Film Iridescence Shader** | Real physics-based thin film interference (Fabry-Perot equations) | Spline's Rainbow layer is a simple approximation |
| **Custom Fresnel with color ramp** | Per-view-angle color mapping, not just edge glow | Spline Fresnel is single-color |
| **Animated Environment Maps** | Dynamic HDRI that responds to scene state | Spline uses static HDRIs |
| **Volumetric rendering** | Ray-marched fog, god rays, subsurface scattering | Not available |
| **Screen-space reflections (SSR)** | Real-time reflections from scene geometry | Not available |
| **Custom tone mapping curves** | ACES Filmic with hand-tuned exposure/gamma | Limited to brightness slider |

### Specific Visual Gaps in Spline
1. **No true subsurface scattering** - only faked with depth gradient
2. **No anisotropic highlights** - can't do brushed metal / hair highlights
3. **No custom GLSL injection** - stuck with built-in layers
4. **No per-vertex color manipulation** - all color is per-material
5. **No real-time cubemap re-rendering** - environment is static
6. **No selective bloom** - bloom applies to entire scene

---

## 3. Our Current V4 vs. What We Should Do

### Side-by-Side Comparison

| Aspect | Our V4 | Target (Spline+) | Gap |
|--------|--------|-------------------|-----|
| **Blob material** | MeshPhysicalMaterial + onBeforeCompile | Good foundation, but color injection is too simplistic | Need layered color compositing with noise-driven gradients |
| **Iridescence** | iridescence: 0.9, IOR: 1.5, thickness [100,800] | Values are good | Could add thin-film shader for more control |
| **Noise displacement** | 3-octave simplex, max amplitude ~0.55 | Decent | Need smoother falloff, maybe domain warping |
| **Normal recomputation** | Finite-difference method | Correct approach | Increase epsilon slightly for smoother normals |
| **Environment map** | Custom procedural cubemap via shader | Good idea, but too dark/subtle | Need brighter hotspots, wider dynamic range |
| **Lighting** | 4 directional + ambient + 3 point orbiting | Decent count | Need area-light-like softness, better color contrast |
| **Post-processing** | UnrealBloomPass only (0.6, 0.4, 0.3) | Bloom is too weak/uniform | Need bloom + vignette + subtle chromatic aberration + film grain |
| **Background** | Procedural gradient (purple/blue) | Good | Could add subtle animated noise/particles |
| **Companion shapes** | Custom ShaderMaterial (Blinn-Phong) | Missing PBR environment integration | Switch to MeshPhysicalMaterial or add env reflection to custom shader |
| **Color palette** | Purple/cyan/pink gradient | Good range | Need smoother transitions, more organic noise-driven variation |
| **Animation** | Continuous noise displacement + mouse tracking | Good | Add breathing rhythm, slow orbit, easing |
| **Particles** | 400 points, additive blending | Fine | Could use textured sprites for softer glow |

### Critical Gaps (Priority Order)
1. **Post-processing stack is too thin** - only bloom, no vignette/chromatic aberration/grain
2. **Environment map is too dark** - reflections aren't visible enough on the blob
3. **Color modulation is basic** - linear sin() mixing vs. noise-driven gradient compositing
4. **No breathing/rhythm** - displacement is constant, no "living" cadence
5. **Companion shapes lack environment reflections** - custom ShaderMaterial bypasses envMap

---

## 4. The "Surpass Spline" Visual Recipe

### A. Hero Blob Material (SPECIFIC VALUES)

```
MeshPhysicalMaterial:
  color: '#6d28d9'          // Deep purple base (slightly darker than current)
  metalness: 0.15           // Lower - less chrome, more organic
  roughness: 0.08           // Very smooth - catches reflections
  clearcoat: 1.0            // Full clearcoat
  clearcoatRoughness: 0.05  // Mirror-sharp clearcoat
  envMapIntensity: 3.0      // INCREASE from 2.0 - reflections must be visible
  iridescence: 1.0          // Full iridescence
  iridescenceIOR: 1.3       // Subtle shift (current 1.5 is too strong)
  iridescenceThicknessRange: [200, 600]  // Narrower range for more coherent shift
  sheen: 0.5                // Reduce from 0.8 - was competing with clearcoat
  sheenRoughness: 0.3
  sheenColor: '#c084fc'     // Lighter purple sheen
  emissive: '#1e1b4b'       // Very dark, just enough for depth
  emissiveIntensity: 0.15   // Subtle
  transmission: 0.0         // Opaque, NOT glass
```

### B. Fragment Shader Color Injection (Spline-Style Layer Compositing)

Replace the simple sin() color mix with a multi-layer system:

```glsl
// Layer 1: Noise-driven base gradient (like Spline's Noise Layer)
float nColor = snoise(vWorldPos * 0.8 + uTime * 0.1);
vec3 baseGrad = mix(
  vec3(0.427, 0.157, 0.851),  // #6d28d9 deep purple
  vec3(0.192, 0.557, 0.929),  // #318eED blue
  nColor * 0.5 + 0.5
);

// Layer 2: Depth gradient (like Spline's Depth Layer)
float depth = dot(normalize(vNormal), normalize(uCamPos - vWorldPos));
vec3 depthColor = mix(
  vec3(0.925, 0.286, 0.600),  // Pink for edges
  baseGrad,                     // Base for facing camera
  smoothstep(0.0, 0.6, depth)
);

// Layer 3: Fresnel rim color (like Spline's Fresnel Layer)
float fresnel = pow(1.0 - depth, 3.0);
vec3 fresnelColor = mix(vec3(0.5, 0.2, 0.8), vec3(0.2, 0.8, 0.9), fresnel);

// Composite with overlay blend mode
diffuseColor.rgb = depthColor + fresnelColor * fresnel * 0.4;
```

### C. Vertex Displacement (Enhanced)

```glsl
// FBM with domain warping for richer organic feel
vec3 warpedPos = position + vec3(
  snoise(position * 0.5 + uTime * 0.1),
  snoise(position * 0.5 + uTime * 0.1 + 100.0),
  snoise(position * 0.5 + uTime * 0.1 + 200.0)
) * 0.3;

// Multi-octave with breathing rhythm
float breathe = sin(uTime * 0.4) * 0.3 + 0.7;  // 0.4-1.0 range
float n1 = snoise(warpedPos * 1.0 + uTime * 0.25) * 0.30 * breathe;
float n2 = snoise(warpedPos * 2.2 + uTime * 0.18 + 10.0) * 0.12;
float n3 = snoise(warpedPos * 4.5 + uTime * 0.12 + 20.0) * 0.04;
float disp = n1 + n2 + n3;
```

### D. Environment Map (Brighter, More Dynamic)

```glsl
// Key hotspot: large, warm, TOP-RIGHT (simulate area light)
col += vec3(0.8, 0.5, 1.0) * pow(max(dot(d, normalize(vec3(0.5, 0.6, 0.4))), 0.0), 6.0) * 4.0;

// Fill: cooler, LEFT (wider spread = lower exponent)
col += vec3(0.3, 0.6, 1.0) * pow(max(dot(d, normalize(vec3(-0.7, 0.2, 0.5))), 0.0), 5.0) * 3.0;

// Rim: warm pink BEHIND (creates silhouette edge)
col += vec3(1.0, 0.4, 0.6) * pow(max(dot(d, normalize(vec3(0.0, -0.3, -0.8))), 0.0), 4.0) * 2.5;

// Secondary accents (these create visible colored reflections on surface)
col += vec3(0.1, 0.9, 0.7) * pow(max(dot(d, normalize(vec3(-0.4, 0.8, -0.3))), 0.0), 10.0) * 2.0;
col += vec3(1.0, 0.8, 0.3) * pow(max(dot(d, normalize(vec3(0.3, -0.6, 0.6))), 0.0), 12.0) * 1.5;

// CRITICAL: Overall brightness 2-3x higher than current
// Lower exponents = wider/softer light spread (like area lights)
// Higher multipliers = brighter reflections visible on PBR surfaces
```

### E. Post-Processing Stack (Complete)

```javascript
// 1. Bloom (slightly stronger than current)
new UnrealBloomPass(resolution, 0.8, 0.5, 0.25)
// strength: 0.8 (up from 0.6)
// radius: 0.5 (up from 0.4 for softer spread)
// threshold: 0.25 (down from 0.3 to catch more highlights)

// 2. Custom Vignette (ShaderPass)
// Darken edges by 30-40%, smooth falloff from center
// innerRadius: 0.4, outerRadius: 1.2, intensity: 0.35

// 3. Chromatic Aberration (ShaderPass)
// RGB channel offset: 1.5-2.0 pixels at edges
// Radial falloff from center (0 at center, max at corners)

// 4. Film Grain (ShaderPass)
// Intensity: 0.03-0.05 (barely visible, adds texture)
// Animated per-frame with random offset

// 5. Color Grading / Tone Adjustment
// Slight warm shift (+0.02 red, +0.01 green, -0.01 blue)
// Contrast boost: 1.1x
```

### F. Lighting Setup (Refined)

```javascript
// Key light: warm purple, STRONG, high position
DirectionalLight(0xc084fc, 3.5)  // position(4, 6, 4)

// Fill: cool blue, moderate, left side
DirectionalLight(0x60a5fa, 2.0)  // position(-6, 2, 4)

// Rim: hot pink, strong, behind and below
DirectionalLight(0xff6b9d, 3.0)  // position(0, -3, -6)

// Top accent: warm white
DirectionalLight(0xfff4e6, 1.0)  // position(0, 10, 0)

// Ambient: very dark purple (just enough to see shadow areas)
AmbientLight(0x120a20, 0.6)

// Orbiting point lights: increase intensity
PointLight(0xc084fc, 8, 12)   // purple, radius 3.5
PointLight(0x60a5fa, 6, 12)   // blue, radius 3.0
PointLight(0xf472b6, 6, 12)   // pink, radius 2.8
```

### G. Animation Principles

```javascript
// BREATHING: All objects should have a slow "alive" rhythm
// Blob: displacement amplitude modulated by sin(t * 0.4)
// Companions: scale oscillation sin(t * 0.3) * 0.02 + 1.0
// Small shapes: gentle float sin(t * 0.5 + offset) * 0.15

// ROTATION: Very slow, barely perceptible
// Blob: rotation.y += 0.001, rotation.x += 0.0005
// Torus knot: rotation.x += 0.002, rotation.y += 0.003
// Torus: rotation.z += 0.003

// MOUSE: Smooth lerp with low alpha
// camera.position lerp alpha: 0.03 (current 0.04 is fine)
// blob tilt toward mouse: 0.05 rad max
// parallax layers: near objects move more than far

// EASING: Use exponential smoothing everywhere
// value += (target - value) * 0.03;  // ~30 frames to settle
```

### H. Companion Shapes Upgrade

**Switch from custom ShaderMaterial to MeshPhysicalMaterial** for environment integration:

```javascript
// Torus Knot
new MeshPhysicalMaterial({
  color: '#7c3aed',
  metalness: 0.6,
  roughness: 0.1,
  clearcoat: 0.8,
  clearcoatRoughness: 0.1,
  envMapIntensity: 2.5,
  iridescence: 0.7,
  iridescenceIOR: 1.4,
  emissive: '#1e1b4b',
  emissiveIntensity: 0.1,
})

// Torus
new MeshPhysicalMaterial({
  color: '#ec4899',
  metalness: 0.5,
  roughness: 0.12,
  clearcoat: 0.9,
  clearcoatRoughness: 0.08,
  envMapIntensity: 2.5,
  sheen: 0.6,
  sheenColor: '#fbbf24',
  sheenRoughness: 0.3,
  emissive: '#831843',
  emissiveIntensity: 0.1,
})
```

---

## 5. Summary: The 10 Commandments to Surpass Spline

1. **Brighter environment map** - 2-3x current brightness, lower exponents for wider spread
2. **Higher envMapIntensity** (3.0+) so reflections are actually visible on surfaces
3. **Noise-driven color compositing** instead of sin() gradients
4. **Domain-warped FBM displacement** for richer organic deformation
5. **Breathing rhythm** - amplitude modulation on all animated elements
6. **Full post-processing stack** - bloom + vignette + chromatic aberration + film grain
7. **MeshPhysicalMaterial for ALL shapes** - companions need environment reflections too
8. **Stronger point lights** (8-10 intensity) with wider orbits
9. **Lower roughness everywhere** (0.05-0.12) so surfaces catch light
10. **Subtle emissive** on everything - prevents pure-black shadow areas

### What This Gives Us Over Spline:
- **Custom FBM + domain warping** (Spline can only do single-layer displacement)
- **Per-vertex color manipulation** via onBeforeCompile (Spline can't inject GLSL)
- **Dynamic environment map** with controllable hotspots (Spline uses static HDRIs)
- **Full post-processing control** with exact parameter tuning (Spline has limited sliders)
- **Thin-film iridescence** with real IOR physics (Spline's Rainbow is a rough approximation)

---

*Research based on: Spline documentation (materials, lighting, post-processing), Spline community showcases, Three.js MeshPhysicalMaterial docs, awwwards Three.js winners, Codrops/pmndrs techniques, The Book of Shaders FBM patterns, and teardown of current V4 implementation.*
