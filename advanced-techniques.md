# Advanced Three.js Techniques for Spline-Surpassing Visuals

> Research compiled for V5 implementation. All techniques verified for single-HTML CDN workflow (Three.js r164, GSAP).

---

## 1. MeshPhysicalMaterial Mastery

### Current V4 State
Hero blob uses `MeshPhysicalMaterial` with `onBeforeCompile` for noise displacement + animated gradient. Settings: metalness 0.28, roughness 0.12, clearcoat 1.0, iridescence 0.9, sheen 0.8.

### Recommended V5 Upgrades

#### A. "Liquid Chrome" Hero Blob Material
The V4 blob looks good but can be pushed significantly further. The key insight: **high metalness + low roughness + rich environment = liquid metal**. Combined with iridescence, this creates the "alien chrome" look Spline is famous for.

```javascript
const blobMat = new THREE.MeshPhysicalMaterial({
  // BASE: liquid chrome foundation
  color: '#b4a0ff',           // lighter base to catch more env reflections
  metalness: 0.65,            // UP from 0.28 - more metallic = richer reflections
  roughness: 0.08,            // DOWN from 0.12 - sharper reflections

  // CLEARCOAT: adds a second reflection layer (wet look)
  clearcoat: 1.0,
  clearcoatRoughness: 0.05,   // DOWN from 0.08 - crisper top layer

  // IRIDESCENCE: the rainbow oil-slick effect
  iridescence: 1.0,           // MAX - full effect
  iridescenceIOR: 1.8,        // UP from 1.5 - stronger color shift
  iridescenceThicknessRange: [200, 600], // tighter range = more coherent colors

  // SHEEN: soft glow at edges (velvet-like)
  sheen: 1.0,                 // MAX
  sheenRoughness: 0.2,
  sheenColor: new THREE.Color('#ff6eb4'), // warm pink sheen

  // SPECULAR TINT: colored reflections
  specularIntensity: 1.0,
  specularColor: new THREE.Color('#e0d0ff'), // violet-tinted specular

  // EMISSIVE: subtle inner glow (picked up by bloom)
  emissive: '#3b1a7e',
  emissiveIntensity: 0.25,

  // ENVIRONMENT: crank it up
  envMapIntensity: 2.8,       // UP from 2.0 - more environment reflection
});
```

**Why these values work together:**
- `metalness: 0.65` + `roughness: 0.08` = sharp, colored reflections from environment
- `iridescenceIOR: 1.8` = stronger angle-dependent color shift (soap bubble effect)
- `sheenColor: pink` contrasts with purple base = visible edge glow
- `specularColor: violet` = specular highlights are tinted, not plain white
- `envMapIntensity: 2.8` = the environment map is the star of the show

#### B. Glass Companion (Transmission Material)
For one companion, use transmission to create a glass/crystal effect. This is the "Spline glass blob" look.

```javascript
const glassMat = new THREE.MeshPhysicalMaterial({
  color: '#ffffff',
  metalness: 0.0,
  roughness: 0.05,            // nearly smooth glass
  transmission: 1.0,          // fully transparent
  thickness: 1.5,             // refraction magnification
  ior: 1.45,                  // glass-like refraction
  clearcoat: 1.0,
  clearcoatRoughness: 0.03,
  iridescence: 0.4,           // subtle rainbow on glass
  iridescenceIOR: 1.3,
  envMapIntensity: 2.0,
  specularIntensity: 1.0,
  specularColor: new THREE.Color('#c0d8ff'),
  // IMPORTANT: set opacity to 1 when using transmission
  opacity: 1.0,
  transparent: false,         // transmission handles transparency
});
```

**Warning:** Transmission renders the scene again per-object. Use sparingly - ONE glass object max.

#### C. Anisotropic "Brushed Silk" Companion
For another companion, anisotropy creates elongated reflections like brushed metal or silk.

```javascript
const silkMat = new THREE.MeshPhysicalMaterial({
  color: '#ff7eb3',
  metalness: 0.45,
  roughness: 0.25,
  anisotropy: 0.8,            // strong directional reflection
  anisotropyRotation: Math.PI * 0.25, // 45-degree brush direction
  clearcoat: 0.6,
  clearcoatRoughness: 0.15,
  sheen: 0.9,
  sheenColor: new THREE.Color('#fbbf24'),
  sheenRoughness: 0.3,
  envMapIntensity: 2.0,
});
```

### Key Material Combinations That Pop

| Look | metalness | roughness | clearcoat | iridescence | sheen | transmission |
|------|-----------|-----------|-----------|-------------|-------|-------------|
| Liquid Chrome | 0.6-0.8 | 0.05-0.1 | 1.0 | 0.8-1.0 | 0.3 | 0 |
| Glass Crystal | 0.0 | 0.0-0.1 | 1.0 | 0.3-0.5 | 0 | 1.0 |
| Pearlescent | 0.2-0.4 | 0.15-0.25 | 0.8 | 1.0 | 0.8-1.0 | 0 |
| Brushed Silk | 0.4-0.5 | 0.2-0.3 | 0.5 | 0 | 1.0 | 0 |
| Alien Organic | 0.5-0.7 | 0.1-0.15 | 1.0 | 1.0 | 0.5 | 0 |

---

## 2. Post-Processing Stack (The "Spline Look")

### Current V4 State
UnrealBloomPass (strength 0.6, radius 0.4, threshold 0.3) + OutputPass. Effective but basic.

### Recommended V5 Stack

The "Spline look" is: **subtle bloom + vignette + slight chromatic aberration + color grading**. The key is SUBTLETY - these effects should enhance, not dominate.

#### A. Optimized Bloom Settings
```javascript
// REFINED bloom - less aggressive, more cinematic
const bloom = new UnrealBloomPass(
  new THREE.Vector2(innerWidth, innerHeight),
  0.45,   // strength: DOWN from 0.6 - softer, more refined
  0.5,    // radius: UP from 0.4 - wider, more dreamy spread
  0.25    // threshold: DOWN from 0.3 - more elements contribute to glow
);
```

**Why:** Lower strength + wider radius = "airy glow" vs "nuclear glow". The lower threshold lets more surface detail bloom subtly.

#### B. Combined Cinematic Post-Processing Pass (Single ShaderPass)
Combine vignette + chromatic aberration + film grain + color grading into ONE pass for performance.

```javascript
const CinematicShader = {
  uniforms: {
    tDiffuse: { value: null },
    uTime: { value: 0 },
    // Vignette
    uVignetteOffset: { value: 0.95 },
    uVignetteDarkness: { value: 1.1 },
    // Chromatic aberration
    uChromaticStrength: { value: 0.003 },
    // Film grain
    uGrainIntensity: { value: 0.04 },
    // Color grading
    uSaturation: { value: 1.12 },
    uContrast: { value: 1.05 },
    uBrightness: { value: 1.02 },
    // Tint
    uShadowTint: { value: new THREE.Color('#1a0a2e') }, // cool purple shadows
    uHighlightTint: { value: new THREE.Color('#fff5e6') }, // warm highlights
  },
  vertexShader: `
    varying vec2 vUv;
    void main() {
      vUv = uv;
      gl_Position = projectionMatrix * modelViewMatrix * vec4(position, 1.0);
    }
  `,
  fragmentShader: `
    uniform sampler2D tDiffuse;
    uniform float uTime;
    uniform float uVignetteOffset, uVignetteDarkness;
    uniform float uChromaticStrength;
    uniform float uGrainIntensity;
    uniform float uSaturation, uContrast, uBrightness;
    uniform vec3 uShadowTint, uHighlightTint;
    varying vec2 vUv;

    // Simple hash for grain
    float hash(vec2 p) {
      vec3 p3 = fract(vec3(p.xyx) * 0.1031);
      p3 += dot(p3, p3.yzx + 33.33);
      return fract((p3.x + p3.y) * p3.z);
    }

    void main() {
      vec2 uv = vUv;

      // --- CHROMATIC ABERRATION ---
      // Radial: stronger at edges
      vec2 center = uv - 0.5;
      float dist = length(center);
      float caAmount = uChromaticStrength * dist;
      vec2 caDir = normalize(center) * caAmount;

      float r = texture2D(tDiffuse, uv + caDir).r;
      float g = texture2D(tDiffuse, uv).g;
      float b = texture2D(tDiffuse, uv - caDir).b;
      vec3 color = vec3(r, g, b);

      // --- COLOR GRADING ---
      // Brightness
      color *= uBrightness;

      // Contrast (pivot at 0.5)
      color = (color - 0.5) * uContrast + 0.5;

      // Saturation
      float luma = dot(color, vec3(0.2126, 0.7152, 0.0722));
      color = mix(vec3(luma), color, uSaturation);

      // Split toning: tint shadows and highlights separately
      float shadowMask = 1.0 - smoothstep(0.0, 0.5, luma);
      float highlightMask = smoothstep(0.5, 1.0, luma);
      color = mix(color, color * (uShadowTint * 2.0 + 0.5), shadowMask * 0.15);
      color = mix(color, color * (uHighlightTint * 0.5 + 0.75), highlightMask * 0.1);

      // --- VIGNETTE ---
      float vignette = smoothstep(uVignetteOffset, uVignetteOffset - 0.45, dist * (uVignetteDarkness + uVignetteOffset));
      color *= vignette;

      // --- FILM GRAIN ---
      float grain = hash(uv * 1000.0 + uTime * 100.0) - 0.5;
      color += grain * uGrainIntensity;

      gl_FragColor = vec4(color, 1.0);
    }
  `
};

// Usage:
import { ShaderPass } from 'three/addons/postprocessing/ShaderPass.js';
const cinematicPass = new ShaderPass(CinematicShader);
// In render loop: cinematicPass.uniforms.uTime.value = t;
```

#### C. Full Post-Processing Pipeline Order
```javascript
const composer = new EffectComposer(renderer);
composer.addPass(new RenderPass(scene, camera));
composer.addPass(bloomPass);         // Bloom FIRST (operates on HDR data)
composer.addPass(cinematicPass);     // Combined cinematic effects
composer.addPass(new OutputPass());  // Tone mapping + color space LAST
```

### Critical Parameter Values

| Effect | Parameter | Subtle | Medium | Aggressive |
|--------|-----------|--------|--------|------------|
| Bloom strength | `strength` | 0.3 | 0.5 | 0.8 |
| Bloom radius | `radius` | 0.3 | 0.5 | 0.8 |
| Bloom threshold | `threshold` | 0.4 | 0.25 | 0.1 |
| Chromatic aberration | `strength` | 0.002 | 0.004 | 0.008 |
| Vignette darkness | `darkness` | 0.8 | 1.1 | 1.6 |
| Film grain | `intensity` | 0.02 | 0.04 | 0.08 |
| Saturation boost | `saturation` | 1.05 | 1.12 | 1.25 |

**Recommendation for Spline look:** Use "Subtle" to "Medium" values. Over-processing kills the premium feel.

---

## 3. Shader Techniques

### A. onBeforeCompile: Injecting Gradient Colors into PBR

The V4 approach is solid. Key improvements for V5:

#### Smoother Multi-Stop Gradient
Replace the V4 2-axis mix with a smoother 4-color radial gradient:

```glsl
// In fragment shader via onBeforeCompile, replacing #include <color_fragment>
#include <color_fragment>

// Position-based gradient with time animation
vec3 norm01 = vNormal * 0.5 + 0.5;
float angle = atan(norm01.z - 0.5, norm01.x - 0.5);
float t = (angle / 6.28318) + 0.5; // 0..1 around the surface
t = fract(t + uTimeFrag * 0.06);   // slow rotation

// 4-stop smooth gradient (purple -> blue -> cyan -> pink -> purple)
vec3 c0 = vec3(0.486, 0.227, 0.929); // #7c3aed deep purple
vec3 c1 = vec3(0.231, 0.510, 0.965); // #3b82f6 blue
vec3 c2 = vec3(0.024, 0.714, 0.831); // #06b6d4 cyan
vec3 c3 = vec3(0.925, 0.286, 0.600); // #ec4899 pink

vec3 grad;
if (t < 0.25) grad = mix(c0, c1, t * 4.0);
else if (t < 0.5) grad = mix(c1, c2, (t - 0.25) * 4.0);
else if (t < 0.75) grad = mix(c2, c3, (t - 0.5) * 4.0);
else grad = mix(c3, c0, (t - 0.75) * 4.0);

// Blend with height for variation
float heightMix = norm01.y * 0.3 + 0.35;
diffuseColor.rgb = mix(diffuseColor.rgb, grad, heightMix);
```

#### Improved Normal Recalculation
The V4 uses finite-difference normals but recalculates noise 9 times in the vertex shader (3 octaves x 3 positions). Optimization:

```glsl
// Use a smaller epsilon for smoother normals
float eps_n = 0.015; // DOWN from 0.02 - smoother
// Consider using only 2 octaves for normal calc (skip the smallest)
float dT = snoise(pT*1.2+uTime*0.3)*0.35 + snoise(pT*2.5+uTime*0.2+10.0)*0.15;
float dB = snoise(pB*1.2+uTime*0.3)*0.35 + snoise(pB*2.5+uTime*0.2+10.0)*0.15;
float dC = snoise(position*1.2+uTime*0.3)*0.35 + snoise(position*2.5+uTime*0.2+10.0)*0.15;
// Skip the 3rd octave (5.0 freq) - it's too fine for normal contribution
```

### B. Fresnel Rim Emission (For Companions)
Add a glowing edge effect that interacts beautifully with bloom:

```glsl
// In fragment shader
vec3 viewDir = normalize(cameraPosition - vWorldPos);
float fresnel = pow(1.0 - max(dot(vNormal, viewDir), 0.0), 3.0);
vec3 rimColor = vec3(0.6, 0.3, 1.0); // purple rim
vec3 emission = rimColor * fresnel * 1.5; // intensity above 1.0 triggers bloom
color += emission;
```

### C. Animated Displacement Improvements
Add a 4th "breathing" octave and mouse-reactive deformation:

```glsl
// Add a slow, large-scale breathing motion
float breathe = sin(uTime * 0.5) * 0.06;
float n1 = snoise(position * 1.0 + uTime * 0.25) * 0.30;
float n2 = snoise(position * 2.2 + uTime * 0.18 + 10.0) * 0.15;
float n3 = snoise(position * 4.5 + uTime * 0.12 + 20.0) * 0.06;
float disp = n1 + n2 + n3 + breathe;

// Improved mouse reactivity - directional bulge
vec3 mouseDir = normalize(vec3(uMouse, 0.3));
float mouseDot = max(dot(normalize(position), mouseDir), 0.0);
disp += mouseDot * mouseDot * 0.12; // squared for sharper bulge
```

---

## 4. Environment & Lighting

### Current V4 State
Procedural PMREM environment with 6 colored light spots on a sky sphere. 5 directional/point lights.

### Recommended V5 Upgrades

#### A. Enhanced Procedural Environment Map
The environment map is THE most important element for PBR material quality. Make it richer:

```glsl
// Enhanced environment fragment shader
varying vec3 vPos;
void main(){
  vec3 d = normalize(vPos);
  float t = d.y * 0.5 + 0.5;

  // Richer base gradient: deep indigo -> purple -> dark blue
  vec3 col = mix(
    vec3(0.05, 0.01, 0.12),  // deep indigo bottom
    vec3(0.03, 0.04, 0.18),  // dark blue top
    smoothstep(0.0, 1.0, t)
  );

  // KEY LIGHT: large warm white-purple softbox (top-right)
  // WIDER spread (lower exponent) = softer, more realistic
  col += vec3(0.7, 0.5, 0.9) * pow(max(dot(d, normalize(vec3(0.5, 0.6, 0.4))), 0.0), 6.0) * 3.0;

  // FILL: cool blue softbox (left)
  col += vec3(0.3, 0.5, 1.0) * pow(max(dot(d, normalize(vec3(-0.6, 0.3, 0.5))), 0.0), 5.0) * 2.5;

  // RIM: hot pink from behind-below
  col += vec3(1.0, 0.3, 0.6) * pow(max(dot(d, normalize(vec3(0.0, -0.4, -0.7))), 0.0), 4.0) * 2.0;

  // ACCENT: cyan highlight (small, sharp)
  col += vec3(0.1, 0.8, 0.9) * pow(max(dot(d, normalize(vec3(-0.3, 0.5, -0.5))), 0.0), 20.0) * 2.5;

  // ACCENT: gold from top
  col += vec3(0.6, 0.4, 0.1) * pow(max(dot(d, normalize(vec3(0.1, 0.95, -0.1))), 0.0), 25.0) * 1.5;

  // HORIZON GLOW: warm band
  float horizon = exp(-15.0 * d.y * d.y);
  col += vec3(0.15, 0.06, 0.22) * horizon;

  // FLOOR BOUNCE: subtle warm reflection from below
  float floor_bounce = max(-d.y, 0.0);
  col += vec3(0.08, 0.04, 0.12) * floor_bounce;

  gl_FragColor = vec4(col, 1.0);
}
```

**Key improvements over V4:**
- Lower exponents on key/fill lights = wider, softer light sources = more realistic reflections
- Added floor bounce = reflections aren't black underneath
- Higher intensity multipliers = brighter reflections = objects pop more
- Added gold accent for warm highlight contrast

#### B. Three-Point Lighting Refinement
```javascript
// KEY: warm purple, high and right (strongest)
const keyLight = new THREE.DirectionalLight(0xc4a0ff, 3.0); // UP from 2.5
keyLight.position.set(4, 6, 4);

// FILL: cool blue, left and lower (half key intensity)
const fillLight = new THREE.DirectionalLight(0x6ba3ff, 1.5);
fillLight.position.set(-5, 0.5, 3);

// RIM/BACK: saturated pink from behind (creates edge separation)
const rimLight = new THREE.DirectionalLight(0xff5e9e, 2.5); // UP from 2.0, more saturated
rimLight.position.set(0, -1, -6);

// TOP FILL: very subtle warm white (lifts overall brightness)
const topLight = new THREE.DirectionalLight(0xfff0e0, 0.6);
topLight.position.set(0, 10, 0);

// AMBIENT: very dark purple (never black - keeps shadow detail)
scene.add(new THREE.AmbientLight(0x1a1028, 0.8));
```

#### C. RoomEnvironment Alternative (Simpler Setup)
Three.js includes a built-in `RoomEnvironment` that generates a procedural studio without any external files:

```javascript
import { RoomEnvironment } from 'three/addons/environments/RoomEnvironment.js';

const pmremGenerator = new THREE.PMREMGenerator(renderer);
const envMap = pmremGenerator.fromScene(new RoomEnvironment()).texture;
scene.environment = envMap;
pmremGenerator.dispose();
```

**Recommendation:** Custom procedural env (approach A) gives more artistic control. Use `RoomEnvironment` only as a fallback or for testing.

---

## 5. Animation & Interaction

### Current V4 State
`requestAnimationFrame` loop, smooth mouse lerp (0.04), camera follows mouse, blob rotates + bobs, companions rotate.

### Recommended V5 Upgrades

#### A. GSAP-Powered Entrance Animations
Instead of just fading in, choreograph a dramatic entrance:

```javascript
// Import GSAP from CDN
// <script src="https://cdn.jsdelivr.net/npm/gsap@3.12.5/dist/gsap.min.js"></script>

// Initial state: everything off-screen/invisible
blob.scale.set(0, 0, 0);
blob.material.opacity = 0;
torusKnot.position.y = 5; // above viewport
torus.position.y = -5;    // below viewport

// Entrance timeline
const tl = gsap.timeline({ defaults: { ease: 'elastic.out(1, 0.5)' } });
tl.to(blob.scale, { x: 1, y: 1, z: 1, duration: 1.8 }, 0)
  .to(torusKnot.position, { y: 0.5, duration: 1.4 }, 0.3)
  .to(torus.position, { y: -0.6, duration: 1.4 }, 0.5);
```

#### B. Enhanced Mouse Interaction
The V4 mouse lerp of 0.04 is too slow to feel responsive. Improve with GSAP-like easing:

```javascript
// In animation loop:
const mouseSmoothing = 0.06; // UP from 0.04 - slightly more responsive
mouse.x += (mouse.tx - mouse.x) * mouseSmoothing;
mouse.y += (mouse.ty - mouse.y) * mouseSmoothing;

// Camera: subtle parallax (not full rotation, just offset)
camera.position.x = mouse.x * 0.8;
camera.position.y = mouse.y * 0.5;
camera.lookAt(0, 0, 0);

// Blob: direct tilt response
blob.rotation.z = mouse.x * 0.15;  // subtle tilt toward mouse
blob.rotation.x = -mouse.y * 0.1;  // subtle nod toward mouse

// Companions: move OPPOSITE to mouse (parallax depth)
torusKnot.position.x = -2.6 - mouse.x * 0.15;
torus.position.x = 2.4 + mouse.x * 0.15;
```

#### C. Scroll-Reactive Scene
If the page scrolls, smoothly transition the scene:

```javascript
// For scroll-driven animations with GSAP ScrollTrigger:
gsap.registerPlugin(ScrollTrigger);

gsap.to(camera.position, {
  z: 4,  // zoom in as user scrolls
  scrollTrigger: {
    trigger: 'body',
    start: 'top top',
    end: 'bottom bottom',
    scrub: 1.5,
  }
});
```

#### D. Micro-Interactions (Premium Feel)
```javascript
// Mouse proximity glow - objects brighten as cursor approaches
const raycaster = new THREE.Raycaster();
const pointer = new THREE.Vector2();

addEventListener('mousemove', e => {
  pointer.x = (e.clientX / innerWidth) * 2 - 1;
  pointer.y = -(e.clientY / innerHeight) * 2 + 1;
  raycaster.setFromCamera(pointer, camera);

  const intersects = raycaster.intersectObject(blob);
  if (intersects.length > 0) {
    // Blob reacts: increase emissive on hover
    gsap.to(blobMat, { emissiveIntensity: 0.5, duration: 0.3 });
  } else {
    gsap.to(blobMat, { emissiveIntensity: 0.25, duration: 0.5 });
  }
});
```

---

## 6. Color Science

### Premium 3D Color Palettes

#### Primary Palette: "Cosmic Amethyst" (Recommended for V5)
```
Deep Purple:   #7c3aed (primary)
Electric Blue:  #3b82f6 (secondary)
Hot Pink:       #ec4899 (accent)
Cyan:           #06b6d4 (highlight)
Warm Gold:      #f59e0b (warm accent)
```

#### Supporting Palette: "Dark Luxe"
```
Background:     #0a0a12 (near-black with blue tint)
Shadow:         #1a0a2e (purple-black)
Mid:            #2e1065 (deep purple)
Highlight:      #fff5e6 (warm white)
```

### Why These Colors Work for 3D
1. **Purple + Blue gradient** = depth perception (receding colors)
2. **Pink + Cyan accents** = complementary contrast on the color wheel
3. **Gold warm accent** = breaks monotony, adds "luxury" feel
4. **Dark blue-tinted background** (not pure black) = objects have edge separation

### Gradient Color Flows (for onBeforeCompile injection)
```
Hero Blob:     Purple #7c3aed -> Blue #3b82f6 -> Cyan #06b6d4 -> Pink #ec4899
Torus Knot:    Blue #3b82f6 -> Cyan #06b6d4 (cool, complementary to blob)
Torus:         Pink #ec4899 -> Gold #f59e0b (warm, contrasts with blob)
Small shapes:  Individual accent colors from palette
```

### Color Temperature Strategy
- **Hero blob:** warm-to-cool gradient (draws eye)
- **Left companion:** cool-dominant (recedes slightly)
- **Right companion:** warm-dominant (advances slightly)
- **Background:** neutral dark (no competition)
- **Particles:** accent colors at low opacity (atmospheric)

---

## 7. Additional Techniques Worth Implementing

### A. Subtle Background Animation
The V4 background is static. Add slow, barely-perceptible movement:

```glsl
// In background fragment shader
uniform float uTime;
// Add subtle flowing nebula
float nebula = snoise(vec3(uv * 3.0, uTime * 0.02));
col += vec3(0.04, 0.02, 0.06) * nebula * 0.5;
```

### B. Particle Depth of Field
Make far particles blurrier/larger and near particles sharper/smaller:

```glsl
// In particle vertex shader
float depth = -mv.z;
float blur = smoothstep(3.0, 12.0, depth);
gl_PointSize = mix(size * 0.5, size * 2.0, blur) * uPR * (100.0 / depth);
vA *= mix(0.6, 0.2, blur); // far particles more transparent
```

### C. Emissive Hotspots on Blob
Add bright spots that catch the bloom pass:

```glsl
// In blob fragment shader via onBeforeCompile
// After setting diffuseColor.rgb:
float hotspot = pow(snoise(vNormal * 4.0 + uTimeFrag * 0.3), 3.0);
hotspot = max(hotspot, 0.0);
// Boost emissive in bright spots -> triggers bloom
diffuseColor.rgb += vec3(0.3, 0.15, 0.5) * hotspot * 0.8;
```

### D. Smooth LOD for Icosahedron
Use subdivision level 5 instead of 6 if performance is an issue. The visual difference is minimal but vertex count drops from ~40k to ~10k.

### E. Renderer Optimizations
```javascript
renderer.toneMapping = THREE.ACESFilmicToneMapping;
renderer.toneMappingExposure = 1.3;  // slightly DOWN from 1.4 for more contrast
renderer.outputColorSpace = THREE.SRGBColorSpace; // ensure correct color output
```

---

## 8. Implementation Priority for V5

1. **Enhanced procedural environment map** (biggest visual impact for least code change)
2. **Upgraded blob material** (metalness/iridescence/specular tuning)
3. **Combined cinematic post-processing pass** (vignette + chromatic aberration + grain)
4. **Companions as PBR materials** (replace custom ShaderMaterial with MeshPhysicalMaterial variants)
5. **GSAP entrance animation** (choreographed reveal)
6. **Improved mouse interaction** (blob tilt, parallax depth)
7. **Animated background** (subtle noise)
8. **Emissive hotspots** on blob for bloom interaction

### What NOT to Do
- Don't add too many transmission objects (performance killer)
- Don't over-bloom (strength > 0.7 looks cheap)
- Don't make chromatic aberration visible at center (only edges)
- Don't animate too fast (Spline's secret is SLOW, deliberate motion)
- Don't use pure black background (always tint slightly)
- Don't add too many post-processing passes (combine into one)

---

## References

- [Three.js MeshPhysicalMaterial docs](https://threejs.org/docs/pages/MeshPhysicalMaterial.html)
- [Three.js PMREMGenerator docs](https://threejs.org/docs/pages/PMREMGenerator.html)
- [Three.js RoomEnvironment](https://threejs.org/docs/pages/RoomEnvironment.html)
- [Three.js UnrealBloomPass](https://threejs.org/docs/pages/UnrealBloomPass.html)
- [Codrops: Glass and Plastic in Three.js](https://tympanus.net/codrops/2021/10/27/creating-the-effect-of-transparent-glass-and-plastic-in-three-js/)
- [Matt DesLauriers: Filmic Effects in WebGL](https://medium.com/@mattdesl/filmic-effects-for-webgl-9dab4bc899dc)
- [Vertex Displacement with Noise](https://www.clicktorelease.com/blog/vertex-displacement-noise-3d-webgl-glsl-three-js/)
- [Three.js Selective Bloom Example](https://threejs.org/examples/webgl_postprocessing_unreal_bloom_selective.html)
- [Codrops: Light and Refraction in Three.js](https://tympanus.net/codrops/2025/03/13/warping-3d-text-inside-a-glass-torus/)
- [GSAP + Three.js Resources](https://threejsresources.com/tool/gsap)
- [pmndrs/postprocessing library](https://github.com/pmndrs/postprocessing)
- [Color Gradient Trends 2025](https://enveos.com/top-creative-color-gradient-trends-for-2025-a-bold-shift-in-design/)
- [Spline 3D Design](https://spline.design/)
