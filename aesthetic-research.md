# Aesthetic Research: Why V3-V5 Failed & How V6 Should Look

## Executive Summary

**The core problem across V3-V5 is a material strategy failure, not a shape failure.**

V5 went all-in on `MeshPhysicalMaterial` with high metalness (0.65). This was a mistake because PBR multiplies your albedo color by realistic light calculations, which **darkens saturated colors**. The more "physically correct" you make it, the muddier vibrant gradients become. The V3 companions looked decent precisely because they used custom `ShaderMaterial` with direct color output -- bypassing PBR's color-darkening pipeline.

**V6 recommendation: Hybrid approach -- custom ShaderMaterial for vibrant color + Fresnel rim + fake environment reflections. NOT MeshPhysicalMaterial.**

---

## 1. WHY PBR (MeshPhysicalMaterial) KILLS VIBRANT COLORS

### The Technical Problem

```
MeshPhysicalMaterial pipeline:
  albedo color --> * lighting (often < 1.0) --> * metalness factor --> * roughness attenuation --> tone mapping
  Result: Your #b4a0ff becomes dark muddy purple

Custom ShaderMaterial pipeline:
  You define gl_FragColor directly --> tone mapping
  Result: Your colors come out exactly as you designed them
```

PBR materials are designed to simulate reality. In reality, surfaces absorb most light -- that's why a bright orange ball looks darker in shadows. But for **stylized 3D web design** (Spline aesthetic), we don't want reality. We want **candy-like vibrancy**.

### Evidence
- V3 companions (ShaderMaterial with gradients + Fresnel) = "looked decent"
- V5 everything (MeshPhysicalMaterial, metalness 0.65) = "getting uglier"
- ShaderGradient library (shadergradient.co) = gorgeous results, uses pure custom shaders
- Codrops' "Twisted Colorful Spheres" = custom shaders with cosine palettes = beautiful
- Codrops' "Magical Marbles" = onBeforeCompile with HDR colors = beautiful, but ONLY because they used emissive values > 1.0 to fight PBR darkening

### The Verdict

**For Spline-like aesthetics: custom ShaderMaterial wins.** Period. PBR is for product visualization and photorealism, not for dreamy gradient blobs.

---

## 2. COLOR STRATEGY: Specific Palettes

### Palette A: "Aurora" (RECOMMENDED for hero)
Primary gradient flow -- cool-to-warm with high saturation:
```
#7B61FF  (electric violet)     -- anchor
#C084FC  (soft lavender)       -- transition
#F472B6  (hot pink)            -- energy
#60A5FA  (sky blue)            -- contrast accent
#34D399  (mint/cyan)           -- highlight spark
```
Use as: cosine palette or 3-stop gradient mapped to surface normal Y + Fresnel

### Palette B: "Sunset Glass"
Warm-dominant, premium feel:
```
#FF6B6B  (coral red)
#FFA07A  (light salmon)
#FFD93D  (golden yellow)
#FF8C94  (rose)
#C084FC  (lavender accent)
```

### Palette C: "Deep Ocean"
Cool-dominant, mysterious:
```
#667EEA  (indigo)
#764BA2  (deep purple)
#6B8DD6  (periwinkle)
#8E54E9  (vivid purple)
#00D2FF  (electric cyan)
```

### Palette D: "Holographic Candy"
The Spline signature look:
```
#E879F9  (fuchsia)
#A78BFA  (violet)
#67E8F9  (cyan)
#FDE68A  (warm yellow)
#F9A8D4  (pink)
```

### How to use these in GLSL

The IQ cosine palette formula is the gold standard for smooth procedural gradients:

```glsl
vec3 palette(float t, vec3 a, vec3 b, vec3 c, vec3 d) {
    return a + b * cos(6.283185 * (c * t + d));
}
```

**Best parameters for our use case (purple-pink-blue flow):**
```
a = vec3(0.5, 0.5, 0.5)    // brightness center
b = vec3(0.5, 0.5, 0.5)    // amplitude
c = vec3(1.0, 0.7, 0.4)    // frequency per channel
d = vec3(0.00, 0.15, 0.20) // phase offset -- creates purple->pink->blue
```

Alternative warm variant:
```
a = vec3(0.8, 0.5, 0.4)
b = vec3(0.2, 0.4, 0.2)
c = vec3(2.0, 1.0, 1.0)
d = vec3(0.00, 0.25, 0.25)
```

---

## 3. HERO SHAPE: Why "Blob" Fails & What To Do Instead

### The "Noise Potato" Problem

V3-V5 all use the same approach: `IcosahedronGeometry` + 3-octave simplex noise displacement. This creates a shape that is:
- **Random** -- no intentional form language
- **Potato-like** -- no clear silhouette or visual rhythm
- **Directionless** -- no dominant axis, no visual flow

### What Beautiful Organic Shapes Actually Look Like

From Spline examples and Codrops tutorials, beautiful abstract shapes share these traits:

1. **Dominant axis** -- the shape has a clear vertical or diagonal orientation, not uniform blobbing
2. **Smooth undulation, not chaos** -- low-frequency waves with very subtle high-frequency detail
3. **Intentional negative space** -- concavities and convexities create visual interest
4. **Silhouette readability** -- you should be able to draw the outline and it looks "designed"

### Recommended Hero Approaches (Pick ONE)

#### Option A: "Smooth Pebble" (RECOMMENDED)
- Start with sphere geometry (NOT icosahedron -- smoother base)
- Use SINGLE low-frequency noise (scale ~0.6-0.8) with amplitude ~0.15-0.25
- Add subtle twist/rotation deformation along Y axis (like the Codrops "Twisted Spheres")
- Result: smooth, organic form with clear silhouette -- like a river stone

```glsl
// Vertex displacement concept
float twist = sin(position.y * 2.0 + uTime * 0.3) * 0.15;
float n = snoise(position * 0.7 + uTime * 0.15) * 0.2;
transformed += normal * n;
transformed.xz *= mat2(cos(twist), -sin(twist), sin(twist), cos(twist));
```

#### Option B: "Torus Knot Morph"
- Use `TorusKnotGeometry(1, 0.35, 128, 32, 2, 3)` as base instead of sphere
- Very subtle noise displacement (amplitude < 0.1)
- Creates an inherently interesting, designed shape
- Much harder to look like a "noise potato"

#### Option C: "Superellipsoid"
- Use a custom parametric geometry (squished sphere with flat zones)
- Less noise needed because base shape is already interesting
- Gives architectural/designed feel

### Key Displacement Rules
1. **LESS noise, not more** -- amplitude should be 0.15-0.25 max (V5 uses 0.30 + 0.15 + 0.06 = 0.51 total -- WAY too much)
2. **Lower frequency** -- noise scale 0.5-1.0, not 1.0-4.5 as in V5
3. **Single octave is fine** -- FBM with 3 octaves creates visual noise/chaos
4. **Slow animation** -- uTime * 0.1 to 0.2, not 0.25

---

## 4. MATERIAL APPROACH FOR V6

### RECOMMENDED: Custom ShaderMaterial with Premium Tricks

```
Layer 1: Base gradient color (cosine palette or 3-color gradient)
          - Map to: mix of normal.y, Fresnel, and displacement value
          - This gives you FULL control over color vibrancy

Layer 2: Fresnel rim glow
          - Classic Fresnel: pow(1.0 - dot(viewDir, normal), 3.0)
          - Color: brighter/different hue than base (e.g., cyan or pink rim)
          - This replaces what PBR metalness was trying to achieve

Layer 3: Fake environment reflection
          - Sample a gradient based on reflect(viewDir, normal)
          - Mix in at 10-20% opacity
          - Gives depth without PBR's color darkening

Layer 4: Specular highlight (optional)
          - Blinn-Phong specular: pow(dot(halfVector, normal), 64.0)
          - Adds the "glass candy" sparkle point
          - Keep subtle: multiply by 0.3-0.5

Layer 5: Subtle noise pattern on surface (optional)
          - Very fine noise in fragment shader for surface interest
          - Think: soap bubble micro-shimmer
```

### Fragment Shader Skeleton

```glsl
uniform float uTime;
uniform vec2 uMouse;

varying vec3 vNormal;
varying vec3 vPosition;
varying vec3 vViewPosition;
varying float vDisplacement;

// IQ cosine palette
vec3 palette(float t) {
    vec3 a = vec3(0.5, 0.5, 0.5);
    vec3 b = vec3(0.5, 0.5, 0.5);
    vec3 c = vec3(1.0, 0.7, 0.4);
    vec3 d = vec3(0.00, 0.15, 0.20);
    return a + b * cos(6.283185 * (c * t + d));
}

void main() {
    vec3 normal = normalize(vNormal);
    vec3 viewDir = normalize(vViewPosition);

    // Layer 1: Gradient base from displacement + normal direction
    float gradientT = vDisplacement * 0.5 + 0.5 + normal.y * 0.3;
    vec3 baseColor = palette(gradientT);

    // Layer 2: Fresnel rim
    float fresnel = pow(1.0 - max(dot(viewDir, normal), 0.0), 3.0);
    vec3 rimColor = vec3(0.4, 0.9, 1.0); // cyan rim
    baseColor += rimColor * fresnel * 0.6;

    // Layer 3: Fake environment reflection
    vec3 reflectDir = reflect(-viewDir, normal);
    float envGradient = reflectDir.y * 0.5 + 0.5;
    vec3 envColor = mix(vec3(0.8, 0.4, 1.0), vec3(0.2, 0.6, 1.0), envGradient);
    baseColor = mix(baseColor, envColor, 0.15);

    // Layer 4: Specular highlight
    vec3 lightDir = normalize(vec3(0.5, 0.8, 0.6));
    vec3 halfVec = normalize(viewDir + lightDir);
    float spec = pow(max(dot(normal, halfVec), 0.0), 64.0);
    baseColor += vec3(1.0, 0.95, 0.9) * spec * 0.4;

    gl_FragColor = vec4(baseColor, 1.0);
}
```

### Why NOT Matcap

Matcap (`MeshMatcapMaterial`) is tempting because it bakes lighting into a texture, guaranteeing "good" results. However:
- **Static lighting** -- rotation looks fake because reflections don't change with environment
- **No animation** -- can't animate color over time
- **Limited** -- one look per texture, no custom effects
- **Better for sculpting previews**, not for premium web hero elements

Matcap is a fallback option if shader complexity is too high, but for our use case, custom ShaderMaterial is strictly better.

---

## 5. COMPANION OBJECTS: What Worked in V3 & How to Keep It

V3 companions used ShaderMaterial with:
- Bright gradient base colors
- Fresnel rim glow
- Specular highlights

**This was the right approach.** V5 broke them by switching to MeshPhysicalMaterial.

### Companion Design Rules
1. **Keep ShaderMaterial** -- same approach as hero, just different palette parameters
2. **Simple geometries** -- sphere, torus, icosahedron, octahedron, rounded box
3. **Each companion gets a unique color phase** -- shift the `d` parameter in cosine palette
4. **Size ratio**: hero = 1.0, large companion = 0.3-0.4, small companion = 0.15-0.25
5. **Count**: 4-6 companions total, not more. Spline scenes are clean, not cluttered
6. **Placement**: orbital arrangement, varied distances (2.0-4.0 units from center)

---

## 6. COMPOSITION GUIDELINES

### The Spline Layout Formula

```
Background: Deep dark gradient (NOT black, use #0A0A14 to #0D0520)
             Subtle animated noise/nebula at 5-10% opacity

Hero:        Center-slightly-left, occupies ~40% of viewport height
             Slow rotation (0.1-0.2 rad/sec)
             Subtle breathing animation

Companions:  Scattered in a rough oval around hero
             Different sizes (0.15 - 0.4 of hero size)
             Slow independent rotations
             Float/bob at different speeds

Lighting:    Handled IN the shader (not scene lights for ShaderMaterial)
             But keep 1-2 subtle scene lights for any PBR elements

Post-processing:
             UnrealBloomPass: threshold 0.6, strength 0.3-0.5, radius 0.4
             Subtle -- bloom should ENHANCE, not wash out
             Current V5 bloom may be too strong
```

### Size & Position Reference
```
Hero blob:    radius ~1.4, position (0, 0, 0)
Companion 1:  radius ~0.45, position (2.5, 1.2, -1)
Companion 2:  radius ~0.35, position (-2.2, -0.8, 0.5)
Companion 3:  radius ~0.55, position (1.8, -1.5, -0.5)
Companion 4:  radius ~0.25, position (-1.5, 1.8, -1.5)
Companion 5:  radius ~0.20, position (3.0, 0.0, -2.0)
```

---

## 7. SPECIFIC V6 IMPLEMENTATION PRIORITIES

### Must Do (Critical for beauty)
1. **Switch hero to custom ShaderMaterial** with cosine palette gradient + Fresnel
2. **Reduce displacement** to single-octave, low-frequency, amplitude 0.15-0.25
3. **Switch companions back to ShaderMaterial** (like V3, but refined)
4. **Reduce bloom** -- threshold 0.6+, strength < 0.5

### Should Do (Polish)
5. Add twist deformation to hero shape along Y axis for visual interest
6. Give each companion a unique cosine palette phase
7. Animate gradient slowly (shift palette `t` over time)
8. Add subtle glass-like specular highlight to hero

### Nice to Have
9. Mouse-reactive Fresnel intensity (glow follows cursor)
10. Subtle surface shimmer in fragment shader
11. Very faint floating particles (tiny dots, not distracting)

---

## 8. REFERENCE EXAMPLES & URLS

- [ShaderGradient](https://shadergradient.co/customize) -- the gold standard for gradient 3D on web
- [Codrops: Twisted Colorful Spheres](https://tympanus.net/codrops/2021/01/26/twisted-colorful-spheres-with-three-js/) -- cosine palette + noise displacement
- [Codrops: Animated Displaced Sphere](https://tympanus.net/codrops/2024/07/09/creating-an-animated-displaced-sphere-with-a-custom-three-js-material/) -- CustomShaderMaterial + smooth bands
- [Codrops: Magical Marbles](https://tympanus.net/codrops/2021/08/02/magical-marbles-in-three-js/) -- volumetric fake glass
- [IQ Cosine Palettes](https://iquilezles.org/articles/palettes/) -- the math behind beautiful gradients
- [Fresnel Shader Material](https://github.com/otanodesignco/Fresnel-Shader-Material) -- reusable Fresnel implementation
- [Three.js Iridescent Shader](https://github.com/pzsprog/threejs-iridescent-shader) -- holographic effect reference
- [Matcap Library](https://github.com/nidorx/matcaps) -- 641 matcap textures (fallback option)
- [Lamina Layer Material](https://github.com/pmndrs/lamina) -- layer-based shader composition
- [Maxime Heckel Shader Study](https://blog.maximeheckel.com/posts/the-study-of-shaders-with-react-three-fiber/) -- gradient fundamentals
- [Spline Community Gradients](https://community.spline.design/tag/gradient) -- real Spline inspiration
- [Spline Examples Gallery](https://spline.design/examples) -- official examples

---

## 9. SUMMARY: THE V6 FORMULA

```
V6 = Custom ShaderMaterial (NOT PBR)
   + IQ Cosine Palette (vibrant, controllable colors)
   + Fresnel Rim Glow (depth without PBR)
   + Fake Environment Reflection (subtle, 15% mix)
   + Low-Frequency Smooth Displacement (NOT noise potato)
   + Optional Twist Deformation (designed shape)
   + Subtle Bloom Post-Processing (enhance, not wash)
```

**The single most important change: DROP MeshPhysicalMaterial for the hero and companions. Use custom ShaderMaterial with direct color output.**

This is what makes Spline scenes, ShaderGradient, and Codrops demos look beautiful -- they all bypass PBR and paint colors directly.
