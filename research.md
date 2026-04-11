# Spline 核心 3D 效果 & Web 实现技术栈研究报告

> 日期: 2026-04-11 | 研究员: researcher agent

---

## 一、效果-技术映射表

| Spline 效果 | 最佳 Web 实现方式 | 复杂度 | 关键技术点 |
|---|---|---|---|
| **玻璃态材质 (Glassmorphism)** | Three.js `MeshPhysicalMaterial` (transmission + ior + thickness) 或 R3F Drei `MeshTransmissionMaterial` | 中 | `transmission: 1, ior: 1.5, thickness: 0.5, roughness: 0.1`; Drei 版本额外支持色散 (chromaticAberration) 和各向异性模糊 |
| **渐变材质 (Gradient Materials)** | 自定义 GLSL ShaderMaterial（fragment shader 中混合多色） | 中 | `mix()` / `smoothstep()` 插值多颜色；可用 noise 扰动产生有机渐变；也可用 matcap 贴图快速实现 |
| **3D 几何体** | Three.js 内置几何体 (`SphereGeometry`, `TorusGeometry`, `IcosahedronGeometry`) + `BufferGeometry` 自定义 | 低 | 标准几何体参数化控制 segments 精度；不规则形体用 Blender 建模后导入 GLTF |
| **光照系统** | Three.js 光源体系 (`AmbientLight`, `PointLight`, `SpotLight`, `DirectionalLight`) + `EnvironmentMap` | 低-中 | HDR 环境贴图 (`RGBELoader`) 提供真实反射；`PMREMGenerator` 预处理环境贴图 |
| **粒子效果 (Stardust)** | Three.js `Points` + `BufferGeometry` + 自定义 shader / GPGPU | 中-高 | `PointsMaterial` 基础实现；高级可用 `InstancedMesh` 减少 draw call；GPGPU (FBO ping-pong) 实现万级粒子物理 |
| **形变动画 (Blob Morphing)** | Vertex Shader + Perlin/Simplex Noise 位移 | 中 | `IcosahedronGeometry(1, 64)` 高细分 + noise 沿法线位移；`u_time` uniform 驱动动画；GSAP 控制形变幅度 |
| **鼠标跟随交互** | `mousemove` 事件 → lerp 插值 → 相机/物体旋转 | 低 | 归一化鼠标坐标 `[-1, 1]`；`THREE.MathUtils.lerp()` 平滑过渡；Raycaster 实现精确 hover 检测 |
| **后处理 - Bloom 辉光** | Three.js `EffectComposer` + `UnrealBloomPass` | 中 | `threshold` 控制发光阈值、`strength` 强度、`radius` 扩散范围；选择性 Bloom 需要多层渲染 |
| **后处理 - 噪点/景深** | `EffectComposer` + `FilmPass` / `BokehPass` / `ShaderPass` | 中 | Film grain 用 `FilmPass`；景深用 `BokehPass(scene, camera, { focus, aperture, maxblur })` |
| **视差滚动动画** | GSAP ScrollTrigger + Three.js 动画绑定 | 中 | `scrub: true` 将滚动进度映射到 timeline；proxy 对象隔离滚动逻辑和 3D 逻辑 |

---

## 二、技术栈深度分析

### 1. CSS 3D Transforms

**能力边界：**
- 可做：卡片翻转、立方体旋转、简单视差、`perspective` 景深
- 不可做：真实光照、材质反射/折射、粒子系统、自定义着色器
- 限制：3D 变换会导致文本模糊（浏览器像素快照机制）；旋转不可交换（顺序敏感）

**适用场景：** 简单 UI 动效、卡片交互、页面过渡动画。不适合 Spline 级别的 3D 效果。

### 2. Three.js (核心推荐)

**能力：** WebGL 全功能封装，涵盖场景图、几何体、材质、光照、阴影、后处理、物理引擎接入。

**关键材质：**
- `MeshStandardMaterial` — PBR 标准材质，支持 metalness + roughness
- `MeshPhysicalMaterial` — 扩展 PBR，支持 clearcoat、transmission（透射）、ior（折射率）、thickness
- `ShaderMaterial` / `RawShaderMaterial` — 完全自定义 GLSL

**后处理管线：**
```
EffectComposer → RenderPass → UnrealBloomPass → FilmPass → OutputPass
```

**性能关键：**
- `InstancedMesh` 减少 draw call
- `BufferGeometry` 避免 `Geometry` (已废弃)
- `requestAnimationFrame` 驱动渲染循环
- `dispose()` 及时释放 GPU 资源

### 3. React Three Fiber (R3F) + Drei

**优势：** 声明式 Three.js，组件化开发，与 React 生态无缝集成。

**Drei 关键组件：**
- `MeshTransmissionMaterial` — 增强版玻璃材质（色散、各向异性模糊、可见其他透射物体）
- `MeshReflectorMaterial` — 平面反射材质
- `Environment` — HDR 环境贴图加载
- `Float` — 自动浮动动画
- `Stars` — 星空粒子
- `OrbitControls` / `PresentationControls` — 交互控制

**适用场景：** React 项目、需要 UI 与 3D 深度交互的产品页面。

### 4. GLSL 自定义着色器

**核心用途：**
- Noise 位移实现有机形变 (blob morphing)
- 自定义渐变材质
- 程序化纹理（不需要贴图文件）
- Fresnel 效果（边缘发光）

**常用 noise 函数：**
```glsl
// Simplex noise — 性能好，无方向偏差
float noise = snoise(vec3(position * frequency + u_time * speed));
// 位移：沿法线方向
vec3 displaced = position + normal * noise * amplitude;
```

**学习资源：** [The Book of Shaders](https://thebookofshaders.com/)、[Maxime Heckel 的 Shader 教程](https://blog.maximeheckel.com/posts/the-study-of-shaders-with-react-three-fiber/)

### 5. GSAP + ScrollTrigger

**与 Three.js 集成方式：**
```javascript
gsap.to(proxyParams, {
  rotationY: Math.PI * 2,
  scrollTrigger: {
    trigger: section,
    scrub: true,
    start: "top top",
    end: "bottom bottom"
  },
  onUpdate: () => {
    mesh.rotation.y = proxyParams.rotationY;
    renderer.render(scene, camera);
  }
});
```

**优势：** 成熟的缓动库，ScrollTrigger 处理滚动逻辑，timeline 编排复杂动画序列。

### 6. Spline Runtime (对标参考)

**特点：**
- 零代码 3D 设计工具，GUI 操作
- 提供 `@splinetool/runtime` 和 `@splinetool/react-spline` Web 嵌入
- 可导出为 Three.js / React Three Fiber 代码（实验性）
- 包体较大，性能可控性差

**与纯代码方案对比：**
| 维度 | Spline Runtime | 纯 Three.js/R3F |
|---|---|---|
| 开发速度 | 快（GUI 设计） | 慢（纯代码） |
| 定制性 | 有限 | 无限 |
| 性能可控 | 低 | 高 |
| 包体大小 | 大（runtime + scene 文件） | 可按需引入 |
| 学习曲线 | 低 | 高 |

---

## 三、推荐技术组合

### 方案 A：纯 Three.js（推荐用于本项目）

```
Three.js + 自定义 GLSL Shader + GSAP + Vite
```

**理由：**
- 最大控制力，直接操作 WebGL
- 无框架依赖，最小包体
- 适合学习和理解底层原理
- 本项目聚焦效果研究，不需要 UI 框架

**依赖：**
```json
{
  "three": "^0.170.0",
  "gsap": "^3.12.0",
  "vite": "^6.0.0"
}
```

### 方案 B：React Three Fiber 全家桶（适合产品级项目）

```
React + R3F + Drei + GSAP + Vite
```

**理由：**
- 声明式开发效率高
- Drei 提供大量预制材质和效果
- 适合需要 UI + 3D 混合的产品页面

---

## 四、实现路线图（从简单到复杂）

### Phase 1: 基础场景搭建 (Day 1)
1. Vite 项目初始化 + Three.js 引入
2. 场景、相机、渲染器基础搭建
3. 简单几何体 (球体/环面) + 基础材质
4. 灯光设置 (环境光 + 点光源)
5. 鼠标跟随旋转交互

### Phase 2: 玻璃态材质 (Day 1-2)
1. `MeshPhysicalMaterial` 配置 transmission + ior
2. 环境贴图 (HDR) 加载
3. 背景模糊效果
4. 多层玻璃叠加

### Phase 3: 形变动画 (Day 2)
1. 自定义 vertex shader
2. Perlin/Simplex noise 引入
3. 球体 blob morphing
4. u_time 驱动的动画循环
5. 渐变材质 (fragment shader 多色混合)

### Phase 4: 粒子系统 (Day 2-3)
1. `Points` + `BufferGeometry` 基础粒子
2. 随机分布 + 缓慢漂移动画
3. 粒子大小衰减 + 透明度
4. (可选) GPGPU 高性能粒子

### Phase 5: 后处理 + 打磨 (Day 3)
1. `EffectComposer` 管线搭建
2. `UnrealBloomPass` 辉光效果
3. 色调映射 + 噪点
4. 性能优化 (降采样、LOD)
5. 响应式适配

---

## 五、关键效果实现思路与伪代码

### 5.1 玻璃态球体

```javascript
const glassMaterial = new THREE.MeshPhysicalMaterial({
  color: 0xffffff,
  transmission: 1,        // 完全透射
  opacity: 1,
  metalness: 0,
  roughness: 0.1,         // 轻微磨砂
  ior: 1.5,               // 玻璃折射率
  thickness: 0.5,         // 厚度影响折射
  envMapIntensity: 1.5,
  clearcoat: 1,
  clearcoatRoughness: 0.1
});

const sphere = new THREE.Mesh(
  new THREE.SphereGeometry(1, 64, 64),
  glassMaterial
);
```

### 5.2 Blob Morphing (Vertex Shader)

```glsl
// vertex shader
uniform float u_time;
uniform float u_amplitude;
varying vec3 vNormal;
varying vec3 vPosition;

// snoise3D 函数省略（使用 ashima/webgl-noise 库）

void main() {
  float noise = snoise(position * 1.5 + u_time * 0.3) * u_amplitude;
  vec3 displaced = position + normal * noise;

  vNormal = normalMatrix * normal;
  vPosition = displaced;

  gl_Position = projectionMatrix * modelViewMatrix * vec4(displaced, 1.0);
}
```

```glsl
// fragment shader — 基于法线的渐变着色
varying vec3 vNormal;
varying vec3 vPosition;

void main() {
  vec3 color1 = vec3(0.4, 0.2, 0.9);  // 紫色
  vec3 color2 = vec3(0.1, 0.8, 0.7);  // 青色
  vec3 color3 = vec3(0.9, 0.3, 0.5);  // 粉色

  float fresnel = pow(1.0 - abs(dot(vNormal, vec3(0.0, 0.0, 1.0))), 2.0);
  vec3 gradient = mix(color1, color2, vNormal.y * 0.5 + 0.5);
  gradient = mix(gradient, color3, fresnel);

  gl_FragColor = vec4(gradient, 1.0);
}
```

### 5.3 浮动粒子系统

```javascript
const PARTICLE_COUNT = 5000;
const positions = new Float32Array(PARTICLE_COUNT * 3);
const scales = new Float32Array(PARTICLE_COUNT);

for (let i = 0; i < PARTICLE_COUNT; i++) {
  positions[i * 3]     = (Math.random() - 0.5) * 20;  // x
  positions[i * 3 + 1] = (Math.random() - 0.5) * 20;  // y
  positions[i * 3 + 2] = (Math.random() - 0.5) * 20;  // z
  scales[i] = Math.random();
}

const geometry = new THREE.BufferGeometry();
geometry.setAttribute('position', new THREE.BufferAttribute(positions, 3));
geometry.setAttribute('aScale', new THREE.BufferAttribute(scales, 1));

const material = new THREE.PointsMaterial({
  size: 0.05,
  sizeAttenuation: true,
  transparent: true,
  opacity: 0.6,
  blending: THREE.AdditiveBlending,
  depthWrite: false,
  color: 0x88ccff
});

const particles = new THREE.Points(geometry, material);

// 动画循环中缓慢漂移
function animateParticles(time) {
  const pos = particles.geometry.attributes.position.array;
  for (let i = 0; i < PARTICLE_COUNT; i++) {
    pos[i * 3 + 1] += Math.sin(time + i) * 0.001;  // Y轴微小浮动
  }
  particles.geometry.attributes.position.needsUpdate = true;
}
```

### 5.4 鼠标跟随视差

```javascript
const mouse = { x: 0, y: 0 };
const target = { x: 0, y: 0 };

window.addEventListener('mousemove', (e) => {
  mouse.x = (e.clientX / window.innerWidth) * 2 - 1;
  mouse.y = -(e.clientY / window.innerHeight) * 2 + 1;
});

function animate() {
  // lerp 平滑过渡
  target.x += (mouse.x - target.x) * 0.05;
  target.y += (mouse.y - target.y) * 0.05;

  // 物体跟随旋转
  mesh.rotation.y = target.x * 0.5;
  mesh.rotation.x = target.y * 0.3;

  // 相机微移产生视差
  camera.position.x = target.x * 0.2;
  camera.position.y = target.y * 0.2;
  camera.lookAt(scene.position);

  renderer.render(scene, camera);
  requestAnimationFrame(animate);
}
```

### 5.5 Bloom 后处理

```javascript
import { EffectComposer } from 'three/addons/postprocessing/EffectComposer.js';
import { RenderPass } from 'three/addons/postprocessing/RenderPass.js';
import { UnrealBloomPass } from 'three/addons/postprocessing/UnrealBloomPass.js';
import { OutputPass } from 'three/addons/postprocessing/OutputPass.js';

const composer = new EffectComposer(renderer);
composer.addPass(new RenderPass(scene, camera));
composer.addPass(new UnrealBloomPass(
  new THREE.Vector2(window.innerWidth, window.innerHeight),
  0.8,   // strength — 辉光强度
  0.3,   // radius — 扩散半径
  0.85   // threshold — 亮度阈值
));
composer.addPass(new OutputPass());

// 渲染循环中替换 renderer.render()
function animate() {
  composer.render();
  requestAnimationFrame(animate);
}
```

---

## 六、优秀案例参考

### Awwwards 获奖 Three.js 网站
- [Awwwards Three.js Collection](https://www.awwwards.com/websites/three-js/) — 大量获奖 3D 网站合集
- Bruno Simon's Portfolio — 用 Three.js + Cannon.js 制作的 3D 驾车探索网站
- Jordan Breton's Portfolio — 天空浮岛，含草地/瀑布/火焰/风粒子效果

### Codrops 教程（高质量实现参考）
- [Glass Torus with Refraction](https://tympanus.net/codrops/2025/03/13/warping-3d-text-inside-a-glass-torus/) — 玻璃环面内的折射文字
- [Procedural Vortex in Glass Sphere](https://tympanus.net/codrops/2025/03/10/rendering-a-procedural-vortex-inside-a-glass-sphere-with-three-js-and-tsl/) — 玻璃球内的程序化涡旋
- [Dreamy Particle Effect with GPGPU](https://tympanus.net/codrops/2024/12/19/crafting-a-dreamy-particle-effect-with-three-js-and-gpgpu/) — GPGPU 驱动的梦幻粒子
- [Transparent Glass and Plastic](https://tympanus.net/codrops/2021/10/27/creating-the-effect-of-transparent-glass-and-plastic-in-three-js/) — 透明玻璃与塑料材质
- [Twisted Colorful Spheres](https://tympanus.net/codrops/2021/01/26/twisted-colorful-spheres-with-three-js/) — 扭曲变形彩色球体

### 其他资源
- [Three.js Journey](https://threejs-journey.com/) — 最完整的 Three.js 课程
- [Varun.ca — 3 Ways to Create Particles](https://varun.ca/three-js-particles/) — 三种粒子实现方式对比
- [The Book of Shaders](https://thebookofshaders.com/) — GLSL 着色器入门圣经
- [Maxime Heckel's Shader Study](https://blog.maximeheckel.com/posts/the-study-of-shaders-with-react-three-fiber/) — R3F + Shader 实战教程
- [Drei Documentation](https://drei.docs.pmnd.rs/) — React Three Fiber 工具库文档

---

## 七、总结

**核心结论：** Three.js + 自定义 GLSL Shader + GSAP 是复现 Spline 效果的最佳纯代码方案。

**关键洞察：**
1. Spline 的核心视觉冲击力来自 3 个要素：**玻璃材质** + **有机形变** + **粒子氛围**
2. `MeshPhysicalMaterial` 的 `transmission` 系列属性已经能很好地模拟玻璃效果，无需手写折射 shader
3. Blob morphing 的核心是 vertex shader 中的 noise 位移，实现简单但视觉效果出众
4. 后处理的 Bloom 效果是「高级感」的关键，成本低收益高
5. 鼠标跟随 + lerp 平滑是最基础但最有效的交互手段
