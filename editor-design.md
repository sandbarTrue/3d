# Shape Editor Design Document
> For: html-3d-lab | Date: 2026-04-12 | Author: UX Designer Agent

---

## 1. Shape Catalog (15 Shapes)

All shapes use Three.js r164 built-in geometries. Each has default constructor params, a suggested cosine palette phase (`vec3 d` param), and an aesthetic rating.

| # | Name | Constructor | Default Params | Palette Phase (d) | Aesthetic |
|---|------|-----------|----------------|-------------------|-----------|
| 1 | **Sphere** | `SphereGeometry` | `(1, 64, 64)` | `vec3(0.00, 0.15, 0.20)` | A - classic, smooth gradients |
| 2 | **Torus** | `TorusGeometry` | `(0.7, 0.25, 64, 128)` | `vec3(0.25, 0.05, 0.10)` | A - great rim/fresnel surface |
| 3 | **Torus Knot** | `TorusKnotGeometry` | `(0.55, 0.18, 200, 32, 2, 3)` | `vec3(0.00, 0.15, 0.20)` | S - best shape for shader, complex surface catches light beautifully |
| 4 | **Icosahedron** | `IcosahedronGeometry` | `(0.8, 0)` | `vec3(0.10, 0.20, 0.30)` | A - faceted gem look |
| 5 | **Icosahedron (smooth)** | `IcosahedronGeometry` | `(0.8, 3)` | `vec3(0.10, 0.20, 0.30)` | A - organic blob |
| 6 | **Octahedron** | `OctahedronGeometry` | `(0.8, 0)` | `vec3(0.30, 0.10, 0.00)` | B+ - diamond/crystal |
| 7 | **Dodecahedron** | `DodecahedronGeometry` | `(0.7, 0)` | `vec3(0.05, 0.25, 0.15)` | A - unique pentagonal facets |
| 8 | **Capsule** | `CapsuleGeometry` | `(0.5, 0.8, 16, 32)` | `vec3(0.15, 0.00, 0.10)` | A - pill shape, smooth gradients |
| 9 | **Cylinder** | `CylinderGeometry` | `(0.6, 0.6, 1.2, 64)` | `vec3(0.20, 0.10, 0.05)` | B+ - clean geometric |
| 10 | **Cone** | `ConeGeometry` | `(0.7, 1.2, 64)` | `vec3(0.35, 0.15, 0.05)` | B - sharp apex, good gradient flow |
| 11 | **Box (rounded)** | `BoxGeometry` + bevel* | `(1, 1, 1, 4, 4, 4)` | `vec3(0.05, 0.10, 0.25)` | B+ - use higher segments for softer normals |
| 12 | **Tetrahedron** | `TetrahedronGeometry` | `(0.8, 0)` | `vec3(0.30, 0.00, 0.15)` | B - minimal facets, dramatic lighting |
| 13 | **Ring** | `RingGeometry` | `(0.4, 0.8, 64)` | `vec3(0.10, 0.05, 0.30)` | B - flat disc, works as accent |
| 14 | **Torus Knot (trefoil)** | `TorusKnotGeometry` | `(0.55, 0.18, 200, 32, 3, 2)` | `vec3(0.15, 0.25, 0.10)` | S - variant knot, mesmerizing |
| 15 | **Lathe (vase)** | `LatheGeometry` | custom profile points, 64 segments | `vec3(0.20, 0.00, 0.20)` | A - organic vase/goblet shape |

*Box "rounding" is achieved by using subdivided BoxGeometry segments -- the shader displacement naturally softens it.

**Top aesthetic picks**: Torus Knot (#3, #14), Sphere (#1), Torus (#2), Dodecahedron (#7), Capsule (#8). These shapes have the most surface area variation, producing the richest cosine palette gradients and Fresnel effects.

---

## 2. UI Layout

### Design Philosophy
Dark glassmorphism theme consistent with the existing deep-space background (#08070f). All UI overlays use `backdrop-filter: blur` with semi-transparent backgrounds. No external UI library -- pure HTML/CSS for maximum control and zero dependencies.

### Layout Structure

```
+------------------------------------------------------------------+
|  [logo]  3D Shape Lab               [Undo] [Redo] [Clear] [?]   |  <- Top bar (40px)
+------------------------------------------------------------------+
|                                                                    |
|                                                                    |
|                                                                    |
|                         VIEWPORT (3D canvas)                       |
|                                                                    |
|                                                                    |
|                                                   +--------------+ |
|                                                   | Properties   | |
|                                                   | Panel        | |
|                                                   | (right side) | |
|                                                   | 280px wide   | |
|                                                   |              | |
|                                                   | [appears on  | |
|                                                   |  selection]  | |
|                                                   +--------------+ |
|                                                                    |
+------------------------------------------------------------------+
|  [Sphere] [Torus] [Knot] [Ico] [Oct] [Dod] [Cap] [>>]          |  <- Shape bar (80px)
+------------------------------------------------------------------+
```

### Panel Details

#### A. Top Bar (fixed, 40px height)
- Left: App title "3D Shape Lab" in light text
- Right: Icon buttons -- Undo, Redo, Clear Scene, Help toggle
- Background: `rgba(8, 7, 15, 0.7)` + `backdrop-filter: blur(12px)`
- Border-bottom: `1px solid rgba(167, 139, 250, 0.15)`

#### B. Shape Picker Bar (fixed bottom, 80px height)
- Horizontal scrollable strip of shape thumbnails
- Each thumbnail: 56x56px rounded-rect with mini 3D preview (static rendered icon or SVG silhouette)
- Hover: glow border `box-shadow: 0 0 12px rgba(167, 139, 250, 0.5)`
- Click: adds shape to scene center
- Overflow: horizontal scroll with fade-out edges, or a "..." expand button
- Background: `rgba(8, 7, 15, 0.75)` + `backdrop-filter: blur(16px)`
- Border-top: `1px solid rgba(167, 139, 250, 0.12)`

#### C. Properties Panel (right side, 280px, slides in on selection)
- Appears only when a shape is selected
- Slide-in from right with CSS transition (300ms ease-out)
- Background: `rgba(12, 10, 24, 0.85)` + `backdrop-filter: blur(20px)`
- Border-left: `1px solid rgba(167, 139, 250, 0.15)`
- Sections inside (collapsible with chevron):
  1. **Transform** -- position XYZ, rotation XYZ, scale uniform
  2. **Palette** -- preset selector + manual cosine params
  3. **Material** -- Fresnel strength, specular intensity, rim colors
  4. **Geometry** -- shape-specific params (radius, tube, segments, p/q for knots)
- Close button (X) at top-right of panel
- Delete shape button at bottom (red-tinted)

#### D. Viewport (remaining space)
- Full canvas behind all UI
- UI panels have `pointer-events: none` on container, `pointer-events: auto` on actual panel elements
- This ensures clicks pass through to the 3D scene in empty areas

---

## 3. Control Specification

### 3.1 Transform Controls (per selected shape)

| Control | Type | Range | Default |
|---------|------|-------|---------|
| Position X | Slider | -5 to 5 | 0 |
| Position Y | Slider | -5 to 5 | 0 |
| Position Z | Slider | -5 to 5 | 0 |
| Rotation X | Slider | 0 to 360 deg | 0 |
| Rotation Y | Slider | 0 to 360 deg | 0 |
| Rotation Z | Slider | 0 to 360 deg | 0 |
| Scale | Slider | 0.1 to 3.0 | 1.0 |

### 3.2 Palette Presets (8 presets)

Each preset defines the 4 cosine palette vectors: `a`, `b`, `c`, `d` (each vec3).

| # | Name | a | b | c | d | Vibe |
|---|------|---|---|---|---|------|
| 1 | **Nebula** | (0.55, 0.45, 0.60) | (0.50, 0.50, 0.45) | (1.0, 0.7, 0.4) | (0.00, 0.15, 0.20) | Purple-blue-gold, the current hero look |
| 2 | **Coral Reef** | (0.50, 0.50, 0.50) | (0.50, 0.50, 0.50) | (1.0, 1.0, 1.0) | (0.25, 0.05, 0.10) | Pink-coral-teal warm tones |
| 3 | **Aurora** | (0.50, 0.50, 0.50) | (0.50, 0.50, 0.50) | (1.0, 1.0, 0.5) | (0.10, 0.20, 0.30) | Green-blue-purple northern lights |
| 4 | **Sunset** | (0.50, 0.50, 0.30) | (0.50, 0.50, 0.30) | (1.0, 0.7, 0.4) | (0.00, 0.05, 0.20) | Orange-red-magenta warm gradient |
| 5 | **Mint Ice** | (0.60, 0.70, 0.65) | (0.30, 0.40, 0.35) | (0.8, 0.6, 1.0) | (0.05, 0.25, 0.15) | Cool mint-lavender pastel |
| 6 | **Ember** | (0.55, 0.35, 0.25) | (0.50, 0.45, 0.40) | (1.0, 0.5, 0.2) | (0.35, 0.15, 0.05) | Fire-like amber-red |
| 7 | **Deep Ocean** | (0.40, 0.45, 0.60) | (0.35, 0.40, 0.50) | (0.8, 1.0, 1.0) | (0.00, 0.10, 0.30) | Deep blue-teal-cyan |
| 8 | **Hologram** | (0.50, 0.50, 0.50) | (0.50, 0.50, 0.50) | (1.0, 1.0, 1.0) | (0.00, 0.33, 0.67) | Full rainbow iridescent |

UI: Row of 8 colored circles (CSS gradient previews). Click to apply. Selected one gets a white ring border.

### 3.3 Material Controls

| Control | Type | Range | Default | Notes |
|---------|------|-------|---------|-------|
| Fresnel Strength | Slider | 0.0 - 1.5 | 0.55 | Maps to `rimColor * fresnel * X` |
| Fresnel Power | Slider | 1.0 - 6.0 | 3.0 | Exponent in `pow(1-dot, X)` |
| Specular Intensity | Slider | 0.0 - 1.0 | 0.35 | Multiplier on spec highlights |
| Specular Sharpness | Slider | 8 - 256 | 64 | `pow(dot, X)` exponent |
| Rim Color 1 | Color picker | -- | varies | HSL color input |
| Rim Color 2 | Color picker | -- | varies | HSL color input |
| Min Brightness | Slider | 0.0 - 0.3 | 0.12 | `max(baseColor, vec3(X))` floor |
| Auto-Rotate Speed | Slider | 0.0 - 0.5 | 0.07 | Y-axis rotation per frame |

### 3.4 Geometry-Specific Controls

These appear dynamically based on selected shape type:

**Sphere**: radius (0.2-2.0), widthSegments (16-128), heightSegments (16-128)
**Torus**: radius (0.3-1.5), tube (0.05-0.5), radialSegments (16-128), tubularSegments (32-256)
**TorusKnot**: radius (0.2-1.2), tube (0.05-0.4), p (1-5, integer), q (1-5, integer)
**Icosahedron/Octahedron/Dodecahedron/Tetrahedron**: radius (0.2-1.5), detail (0-4, integer)
**Capsule**: radius (0.2-1.0), length (0.2-2.0)
**Cylinder**: radiusTop (0.1-1.5), radiusBottom (0.1-1.5), height (0.3-3.0)
**Cone**: radius (0.2-1.5), height (0.5-3.0)
**Box**: width (0.2-2.0), height (0.2-2.0), depth (0.2-2.0)

Changing geometry params recreates the geometry (dispose old, create new, assign to mesh).

---

## 4. Interaction Model

### 4.1 Adding Shapes
1. User clicks a shape thumbnail in the bottom bar
2. New mesh is created with `createCompanionMaterial()` using default palette
3. Shape placed at scene center `(0, 0, 0)` with scale `(0, 0, 0)`
4. GSAP elastic scale-in animation to `(1, 1, 1)` over 0.6s
5. Shape is automatically selected after adding

### 4.2 Selecting Shapes
- **Click** on a shape in the viewport -> select it
- Visual feedback:
  - Selected shape gets a subtle wireframe overlay (clone mesh with `MeshBasicMaterial({ wireframe: true, color: 0xa78bfa, opacity: 0.3, transparent: true })`)
  - GSAP pulse: scale to 1.05 then back to 1.0 (200ms)
  - Properties panel slides in from right
- **Double-click background** (no shape hit) -> deselect all, properties panel slides out
- **Click a different shape** -> deselect old, select new (smooth transition)

### 4.3 Moving Shapes
- **Primary method**: Three.js `TransformControls` attached to selected shape
  - Mode toggle buttons in properties panel: Translate / Rotate / Scale
  - Keyboard shortcuts: G (grab/translate), R (rotate), S (scale)
  - TransformControls provides gizmo handles with axis constraints
- **Secondary method**: Direct drag on shape (without gizmo)
  - Hold Shift + drag -> freeform XY-plane drag via raycaster
  - This is simpler and faster for casual use

### 4.4 Resizing Shapes
- **Mouse wheel** on selected shape -> uniform scale up/down
  - Scroll up = bigger (+ 0.05 per tick, capped at 3.0)
  - Scroll down = smaller (- 0.05 per tick, floored at 0.1)
  - GSAP micro-animation on each tick for smooth feel
- **Scale slider** in properties panel for precise control
- **TransformControls scale mode** for non-uniform scaling

### 4.5 Deleting Shapes
- **Delete key** or **Backspace** -> remove selected shape with GSAP shrink-out animation (scale to 0 over 0.3s, then dispose)
- **Delete button** in properties panel (red) -> same behavior
- Confirmation not needed -- undo will restore

### 4.6 Deselecting
- **Double-click** on empty background -> deselect
- **Escape key** -> deselect
- Properties panel slides out (300ms)

### 4.7 Undo/Redo
- Simple command stack: each action (add, move, resize, delete, palette change) pushes to undo stack
- Ctrl+Z / Cmd+Z -> undo
- Ctrl+Shift+Z / Cmd+Shift+Z -> redo
- Max 50 undo steps

---

## 5. Technical Approach

### 5.1 UI Framework: Pure HTML/CSS (No External UI Library)
**Decision**: Custom HTML overlays, NOT lil-gui or tweakpane.

**Rationale**:
- lil-gui/tweakpane have a "dev tools" aesthetic that clashes with the polished dark-glass look
- Custom CSS gives full control over glassmorphism, animations, transitions
- All UI is HTML elements positioned over the canvas with `position: fixed`
- Sliders use `<input type="range">` with custom CSS styling
- Color pickers use `<input type="color">` or simple HSL sliders
- This avoids any additional CDN dependency

### 5.2 Shape Interaction: TransformControls + Custom Raycaster

```
Import: three/addons/controls/TransformControls.js
Import: three/addons/controls/OrbitControls.js  (replace current mouse-camera)
```

**TransformControls** for selected shape manipulation:
- Provides translate/rotate/scale gizmos out of the box
- Well-tested, handles edge cases (camera angle, depth)
- Fires `'dragging-changed'` event to disable OrbitControls while dragging

**OrbitControls** to replace current mouse-camera tracking:
- Current camera follows mouse -- this conflicts with shape dragging
- OrbitControls gives proper rotate/zoom/pan of the viewport
- Target set to `(0, 0, 0)`, damping enabled for smooth feel
- Auto-rotate can be an option

**Raycaster** for selection:
- On click, cast ray from camera through mouse position
- Test against all user-added shapes (maintain an array)
- Closest hit -> select that shape

### 5.3 Selection Visualization: Wireframe Overlay

**Approach**: When a shape is selected, create a clone mesh with wireframe material and add as child.

```javascript
const outlineMat = new THREE.MeshBasicMaterial({
  wireframe: true,
  color: 0xa78bfa,       // purple accent matching the UI
  opacity: 0.25,
  transparent: true,
  depthTest: true,
});
// Clone geometry, scale slightly larger (1.02) for outline effect
const outline = new THREE.Mesh(selectedMesh.geometry, outlineMat);
outline.scale.setScalar(1.02);
selectedMesh.add(outline);
```

**Why not outline pass**: Outline post-processing (e.g., `OutlinePass`) requires an additional render pass and is heavier. The wireframe approach is simpler, zero-cost, and visually clear. If we want a glow-outline later, we can add it as a stretch goal.

### 5.4 Architecture

```
State Management (plain JS object):
  shapes[]           - array of { mesh, type, palettePreset, materialParams, geoParams }
  selectedId         - index of selected shape, or null
  undoStack[]        - command objects for undo
  redoStack[]        - command objects for redo

Event Flow:
  User clicks shape bar -> addShape(type) -> push to shapes[], select it
  User clicks viewport -> raycaster -> selectShape(index) or deselectAll()
  User adjusts slider -> updateProperty(key, value) -> push undo command
  User presses Delete -> removeShape(selectedId) -> push undo command
```

### 5.5 Material System Adaptation

The existing `createCompanionMaterial()` factory is the right foundation. We extend it to accept uniform overrides:

```javascript
function createEditorMaterial(config) {
  // config: { paletteA, paletteB, paletteC, paletteD, rimColor1, rimColor2,
  //           fresnelStrength, fresnelPower, specIntensity, specSharpness, minBrightness }
  // Returns ShaderMaterial with all params as uniforms (not baked GLSL strings)
}
```

**Key change**: Current `createCompanionMaterial` bakes palette/rim values as GLSL string literals. The editor version must use **uniforms** for all tunable values so they can be changed at runtime without recompiling the shader.

### 5.6 Performance Considerations
- Geometry disposal on param change (prevent memory leaks)
- Limit max shapes to ~30 (warn user)
- Use `Object3D.visible = false` for shapes far from camera (frustum culling is already on)
- Throttle property panel updates to 60fps max
- Post-processing remains single bloom pass (no additional outline pass)

---

## 6. Mobile Considerations

### 6.1 Touch Gestures

| Gesture | Action |
|---------|--------|
| **Tap** shape in bottom bar | Add shape to scene |
| **Tap** shape in viewport | Select it |
| **Tap** empty space | Deselect |
| **One-finger drag** on viewport | Orbit camera (OrbitControls handles this) |
| **One-finger drag** on selected shape | Move shape (when TransformControls active) |
| **Two-finger pinch** | Zoom camera OR resize selected shape |
| **Two-finger rotate** | Rotate camera |
| **Long-press** (500ms) on shape | Show context menu (Delete, Duplicate, Reset) |
| **Swipe up** on bottom bar | Expand shape picker to full grid view |

### 6.2 Responsive Layout

**Breakpoints**:
- Desktop (>768px): Full layout as described above
- Mobile (<768px):
  - Properties panel becomes a bottom sheet (slides up from bottom, max 50vh)
  - Shape picker bar remains at bottom but with smaller thumbnails (48x48)
  - Top bar collapses to icon-only
  - TransformControls gizmo size increased for touch targets

### 6.3 Touch-Friendly Controls
- All sliders have min height of 44px (iOS touch target)
- Slider thumb enlarged to 24px diameter on mobile
- Preset palette circles are 40px minimum on mobile
- Properties panel sections use accordion with large tap targets
- Delete button is oversized (48px height) with red background for visibility

### 6.4 Performance on Mobile
- Reduce particle count from 350 to 150 on mobile (`navigator.maxTouchPoints > 0`)
- Lower geometry segments for new shapes on mobile (e.g., 32 instead of 64)
- Disable bloom pass on low-end devices (check `renderer.capabilities`)
- Use `devicePixelRatio` cap of 2 (already done)

---

## 7. CSS Design Tokens

```css
:root {
  /* Colors */
  --bg-primary: rgba(8, 7, 15, 0.85);
  --bg-secondary: rgba(12, 10, 24, 0.75);
  --border-subtle: rgba(167, 139, 250, 0.15);
  --border-hover: rgba(167, 139, 250, 0.4);
  --accent-purple: #a78bfa;
  --accent-pink: #f472b6;
  --accent-blue: #60a5fa;
  --text-primary: #e2e0f0;
  --text-secondary: #8b85a8;
  --danger: #ef4444;

  /* Spacing */
  --panel-padding: 12px;
  --gap-sm: 6px;
  --gap-md: 12px;
  --gap-lg: 20px;

  /* Effects */
  --blur-panel: blur(20px);
  --blur-bar: blur(12px);
  --transition-panel: 300ms cubic-bezier(0.4, 0, 0.2, 1);
  --shadow-glow: 0 0 20px rgba(167, 139, 250, 0.15);
}
```

---

## 8. Implementation Priority (for Task #2)

### Phase 1 - Core (Must Have)
1. Refactor camera to use OrbitControls
2. Create `createEditorMaterial()` with uniform-based params
3. Build shape picker bottom bar (HTML/CSS)
4. Implement add-shape flow
5. Implement raycaster selection + wireframe overlay
6. Build properties panel with transform sliders
7. Integrate TransformControls for selected shape

### Phase 2 - Polish (Should Have)
8. Palette presets UI + live switching
9. Material controls (Fresnel, specular sliders)
10. Geometry-specific controls (regenerate on param change)
11. Delete shape (key + button)
12. Mouse wheel resize

### Phase 3 - Nice to Have
13. Undo/Redo system
14. Mobile responsive layout + touch gestures
15. Duplicate shape
16. Export scene as JSON / import
17. Keyboard shortcuts (G/R/S/Delete/Escape)

---

## 9. File Structure

Everything stays in a single HTML file (per project constraint). The code organization within the `<script type="module">` block:

```
1. Imports (Three.js + addons)
2. Constants & Design Tokens
3. Renderer, Scene, Camera setup
4. GLSL shared code
5. Background shader
6. Editor Material Factory (uniform-based)
7. Shape Catalog (registry of constructors + defaults)
8. State Management (shapes array, selection, undo stack)
9. UI: Top Bar
10. UI: Shape Picker Bar
11. UI: Properties Panel
12. Interaction: OrbitControls
13. Interaction: TransformControls
14. Interaction: Raycaster selection
15. Interaction: Mouse wheel resize
16. Interaction: Keyboard shortcuts
17. Particles (existing, scaled down)
18. Post-processing (existing)
19. Animation loop
20. Entrance animation
```

---

## 10. Key Design Decisions Summary

| Decision | Choice | Rationale |
|----------|--------|-----------|
| UI framework | Pure HTML/CSS | Matches dark-glass aesthetic, no dependencies |
| Shape manipulation | TransformControls | Battle-tested, handles all edge cases |
| Camera control | OrbitControls | Replaces mouse-follow, enables proper 3D navigation |
| Selection visual | Wireframe overlay | Lightweight, no extra render pass |
| Material params | Uniforms (not baked GLSL) | Runtime adjustable without shader recompile |
| Panel layout | Bottom bar + right panel | Industry standard (Spline, Blender) |
| Palette system | 8 presets + manual override | Quick to use, deep when needed |
| Mobile strategy | Bottom sheet + enlarged touch targets | Responsive without separate codebase |
