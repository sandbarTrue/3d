# 3D 效果测试优化方案

> 日期: 2026-04-11 | 编写: researcher | 关联: Task #4

---

## 一、性能测试方案

### 1.1 监控工具集成

#### stats.js — FPS / 渲染时间 / 内存

```javascript
import Stats from 'stats.js';

const stats = new Stats();
stats.showPanel(0);  // 0=FPS, 1=MS(渲染耗时), 2=MB(内存)
document.body.appendChild(stats.dom);

// 定位到右上角，不遮挡场景
stats.dom.style.position = 'fixed';
stats.dom.style.top = '0px';
stats.dom.style.right = '0px';
stats.dom.style.left = 'auto';

function animate() {
  stats.begin();
  // ... 渲染逻辑 ...
  composer.render();  // 或 renderer.render(scene, camera)
  stats.end();
  requestAnimationFrame(animate);
}
```

**面板切换：** 点击 stats 面板可在 FPS / MS / MB 之间切换。

#### renderer.info — Draw Call 和几何体统计

```javascript
// 每 60 帧输出一次渲染统计
let frameCount = 0;
function logRenderInfo() {
  frameCount++;
  if (frameCount % 60 === 0) {
    const info = renderer.info;
    console.table({
      'Draw Calls': info.render.calls,
      'Triangles': info.render.triangles,
      'Points': info.render.points,
      'Textures': info.memory.textures,
      'Geometries': info.memory.geometries,
      'Programs': info.programs?.length || 'N/A'
    });
  }
}
```

#### Chrome DevTools 性能分析

| 工具 | 用途 | 操作路径 |
|---|---|---|
| Performance Tab | CPU 火焰图、帧率时间线 | F12 → Performance → Record |
| Memory Tab | 内存快照、泄漏检测 | F12 → Memory → Heap Snapshot |
| Layers Panel | GPU 合成层可视化 | F12 → More Tools → Layers |
| `chrome://gpu` | GPU 加速状态、WebGL 版本 | 地址栏直接输入 |

#### Spector.js — WebGL 帧调试

```html
<!-- 开发环境嵌入 -->
<script src="https://spectorcdn.babylonjs.com/spector.bundle.js"></script>
<script>
  const spector = new SPECTOR.Spector();
  spector.displayUI();  // 显示浮动按钮，点击捕获当前帧
</script>
```

**用途：** 逐 draw call 查看 shader 输入/输出、texture 状态、blend 模式。适合排查材质渲染异常。

### 1.2 性能指标与目标

| 指标 | 目标值 (Desktop) | 目标值 (Mobile) | 红线 |
|---|---|---|---|
| FPS | >= 60 | >= 30 | < 24 则需优化 |
| Draw Calls | < 50 | < 30 | > 100 严重瓶颈 |
| 三角面数 | < 100K | < 50K | > 500K 需 LOD |
| 内存 (JS Heap) | < 100MB | < 60MB | > 200MB 泄漏风险 |
| 首次渲染时间 | < 2s | < 3s | > 5s 体验差 |
| 纹理内存 | < 50MB VRAM | < 20MB VRAM | 注意 PNG→VRAM 膨胀 |

### 1.3 性能测试流程

```
1. 打开页面 → stats.js 开始记录
2. 静置 10s → 记录稳态 FPS
3. 触发鼠标交互（快速移动、连续 hover）→ 记录交互 FPS
4. 滚动页面（如有 scroll 动画）→ 记录滚动 FPS
5. 持续运行 5min → 观察内存趋势（是否持续增长 = 泄漏）
6. 切换到后台再切回 → 验证 requestAnimationFrame 暂停/恢复
7. 输出 renderer.info 快照 → 记录 draw calls / triangles
```

---

## 二、兼容性检查清单

### 2.1 浏览器矩阵

| 浏览器 | 版本 | 引擎 | 优先级 | 预期支持 |
|---|---|---|---|---|
| Chrome | 120+ | Blink/WebGPU | P0 (必须) | WebGPU + WebGL2 |
| Edge | 120+ | Blink/WebGPU | P0 | 同 Chrome |
| Safari | 17+ | WebKit | P1 (重要) | WebGL2 (WebGPU 自 Safari 26+) |
| Firefox | 120+ | Gecko | P1 | WebGL2 (WebGPU 实验性) |
| Safari iOS | 17+ | WebKit | P1 | WebGL2, 注意内存限制 |
| Chrome Android | 120+ | Blink | P1 | WebGL2 |
| Samsung Internet | Latest | Blink | P2 (次要) | WebGL2 |

### 2.2 设备分级

| 级别 | 设备特征 | 预期效果 | 降级策略 |
|---|---|---|---|
| **高端** | GPU >= RTX 3060 / M1+ / Adreno 740+ | 全特效 60fps | 无需降级 |
| **中端** | GPU = GTX 1060 / Intel Iris / Adreno 640 | 全特效 30-60fps | 降低 bloom resolution |
| **低端** | 集成显卡 / 老旧手机 (Adreno 505) | 基础几何 + 简化材质 | 关闭后处理、减少粒子数、降低几何细分 |

### 2.3 兼容性检查项

- [ ] **WebGL2 检测**: `renderer.capabilities.isWebGL2` — 若 false 则降级
- [ ] **最大纹理尺寸**: `gl.getParameter(gl.MAX_TEXTURE_SIZE)` — 通常 4096~16384
- [ ] **浮点纹理支持**: `renderer.capabilities.floatFragmentTextures` — 影响 GPGPU 粒子
- [ ] **设备像素比**: `window.devicePixelRatio` — Retina 屏需限制 `renderer.setPixelRatio(Math.min(dpr, 2))`
- [ ] **触摸事件**: `'ontouchstart' in window` — 移动端交互切换
- [ ] **HDR 环境贴图加载**: 验证 `RGBELoader` 在各浏览器正常工作
- [ ] **后处理兼容性**: `EffectComposer` + `UnrealBloomPass` 在 Safari 上的表现（Safari WebGL 实现偶有差异）
- [ ] **resize 响应**: 窗口缩放 / 旋转屏幕后场景比例正确

### 2.4 自动检测与降级代码

```javascript
function detectCapabilities(renderer) {
  const gl = renderer.getContext();
  const caps = {
    webgl2: renderer.capabilities.isWebGL2,
    floatTextures: renderer.capabilities.floatFragmentTextures,
    maxTextureSize: gl.getParameter(gl.MAX_TEXTURE_SIZE),
    maxVertexUniforms: gl.getParameter(gl.MAX_VERTEX_UNIFORM_VECTORS),
    isMobile: /Android|iPhone|iPad|iPod/i.test(navigator.userAgent),
    dpr: Math.min(window.devicePixelRatio, 2),
    gpu: renderer.info.render // renderer.debug.checkShaderErrors 用于 shader 调试
  };

  // 性能分级
  if (caps.isMobile || caps.maxTextureSize <= 4096) {
    return { tier: 'low', ...caps };
  } else if (caps.maxTextureSize >= 16384 && !caps.isMobile) {
    return { tier: 'high', ...caps };
  }
  return { tier: 'mid', ...caps };
}

function applyQualityPreset(tier, config) {
  switch (tier) {
    case 'high':
      config.particleCount = 5000;
      config.bloomEnabled = true;
      config.geometryDetail = 64;
      break;
    case 'mid':
      config.particleCount = 2000;
      config.bloomEnabled = true;
      config.bloomResolution = 0.5;  // 半分辨率 bloom
      config.geometryDetail = 32;
      break;
    case 'low':
      config.particleCount = 500;
      config.bloomEnabled = false;
      config.geometryDetail = 16;
      break;
  }
  return config;
}
```

---

## 三、交互测试用例

### 3.1 鼠标跟随测试

| 编号 | 测试场景 | 预期结果 | 验证方法 |
|---|---|---|---|
| M-01 | 鼠标缓慢移动 | 物体平滑跟随旋转，无抖动 | 目测 + FPS 稳定 |
| M-02 | 鼠标快速划过 | 物体追随有平滑延迟(lerp)，不跳变 | 目测过渡曲线 |
| M-03 | 鼠标移出窗口 | 物体缓慢回归中心位置 | 目测回弹动画 |
| M-04 | 鼠标静止不动 30s | 场景保持 idle 动画，FPS 不下降 | stats.js 监控 |
| M-05 | 鼠标在物体上 hover | 有视觉反馈（放大/发光/材质变化） | 目测效果触发 |

### 3.2 滚动动画测试（如有 GSAP ScrollTrigger）

| 编号 | 测试场景 | 预期结果 | 验证方法 |
|---|---|---|---|
| S-01 | 缓慢滚动 | 动画与滚动进度严格同步 | 目测同步性 |
| S-02 | 快速滚动 | 动画平滑跟随，不跳帧 | 录屏慢放检查 |
| S-03 | 回滚（向上滚动）| 动画正确反转 | 目测反向播放 |
| S-04 | 滚动到底再滚回顶 | 全流程无内存增长 | Memory Tab 监控 |
| S-05 | 触控板惯性滚动 | 动画跟随惯性，结束后静止 | macOS 触控板测试 |

### 3.3 移动端触摸测试

| 编号 | 测试场景 | 预期结果 | 验证方法 |
|---|---|---|---|
| T-01 | 单指滑动 | 等同鼠标移动，物体跟随旋转 | 真机测试 |
| T-02 | 双指缩放 | 如有缩放功能则正常响应，否则不触发 | 真机测试 |
| T-03 | 横竖屏切换 | canvas resize 正确，场景比例不变形 | 真机旋转 |
| T-04 | 长按 | 不弹出系统菜单（需 `touch-action: none`） | 真机长按 |
| T-05 | 低端机触摸交互 | 响应延迟 < 100ms，不卡死 | 低端安卓机测试 |

### 3.4 边界场景测试

| 编号 | 测试场景 | 预期结果 |
|---|---|---|
| E-01 | 浏览器标签切到后台再切回 | `requestAnimationFrame` 暂停/恢复正常，无动画跳变 |
| E-02 | 窗口从全屏拖到小窗 | canvas 自适应，无拉伸变形 |
| E-03 | 同时打开多个标签页 | 不可见标签的 rAF 暂停，不抢 GPU 资源 |
| E-04 | 系统休眠唤醒 | 场景正常恢复渲染 |
| E-05 | 页面刷新/前进后退 | 无 WebGL context lost 报错；资源正确释放 |

---

## 四、视觉质量评估标准

### 4.1 对标维度 — 与 Spline 效果逐项对比

| 评估维度 | 评分标准 (1-5) | Spline 参考 | 我们的实现 | 备注 |
|---|---|---|---|---|
| **玻璃透射感** | 5=完美透射+色散，1=不透明 | 透明、有折射、边缘色散 | 待评估 | 重点看 ior + thickness 效果 |
| **玻璃反射** | 5=清晰环境反射，1=无反射 | HDR 环境反射 | 待评估 | 依赖 envMap 质量 |
| **渐变自然度** | 5=有机流动感，1=生硬线性 | 多色平滑过渡 | 待评估 | noise 扰动是否足够 |
| **几何体精度** | 5=光滑无锯齿，1=明显多边形 | 高细分光滑表面 | 待评估 | segments 够高即可 |
| **光照真实感** | 5=物理准确 PBR，1=平面着色 | 多光源 + 环境光 | 待评估 | HDR envMap 是关键 |
| **粒子氛围感** | 5=梦幻星尘效果，1=随机噪点 | 缓慢漂浮、大小渐变 | 待评估 | additive blending + sizeAttenuation |
| **形变流畅度** | 5=液态流动感，1=卡顿跳变 | 有机 blob 持续形变 | 待评估 | noise frequency + animation speed |
| **Bloom 辉光** | 5=柔和自然光晕，1=过曝/无 | 亮部柔和扩散 | 待评估 | threshold + strength 调参 |
| **交互响应** | 5=即时且平滑，1=延迟明显 | 鼠标跟随无感延迟 | 待评估 | lerp factor 调节 |
| **整体氛围** | 5=可直接用于产品页，1=demo感 | 专业级视觉 | 待评估 | 综合打分 |

### 4.2 评估方法

1. **截图对比**: 将我们的效果截图与 Spline 官方 Gallery 截图并排对比
2. **录屏对比**: 录制 10s 动画循环，与 Spline 对应效果的录屏对比流畅度
3. **盲测打分**: 找 2-3 人，不告知来源，分别看两个效果打分（1-5）
4. **参数微调记录**: 每次调参后截图存档，形成调参→效果的映射表

---

## 五、常见性能瓶颈及优化建议

### 5.1 瓶颈识别速查表

| 症状 | 可能原因 | 诊断方法 | 优化方案 |
|---|---|---|---|
| FPS 低但 GPU 空闲 | CPU 瓶颈（JS 计算过重） | Chrome Performance → 看 JS 火焰图 | 将计算移到 Worker 或 GPU（shader） |
| FPS 低且 GPU 满载 | 片元着色器过重 / 过度绘制 | 降低分辨率看 FPS 是否提升 | 简化 shader / 降低 pixelRatio |
| Draw calls 过高 | 每个物体独立材质/几何体 | `renderer.info.render.calls` | `InstancedMesh` / `mergeGeometries` |
| 内存持续增长 | 资源未 dispose | Memory Tab → Heap Snapshot 对比 | 严格 `dispose()` 所有材质/几何体/纹理 |
| 首帧加载慢 | 资源体积大 / 同步加载 | Network Tab → 检查资源大小 | Draco 压缩 / KTX2 纹理 / 异步加载 |
| 移动端卡顿 | pixelRatio 过高 / 粒子过多 | 限制 `setPixelRatio(Math.min(dpr, 2))` | 设备分级降级（见 2.4 节） |
| Bloom 模糊/拖影 | bloom 分辨率过低或过高 | 调整 UnrealBloomPass resolution | 使用 `window.innerWidth/2` 降低 bloom 渲染分辨率 |

### 5.2 核心优化手段

#### (1) 几何体优化

```javascript
// 合并静态几何体 — 减少 draw calls
import { mergeGeometries } from 'three/addons/utils/BufferGeometryUtils.js';

const geometries = meshes.map(m => m.geometry.clone().applyMatrix4(m.matrixWorld));
const mergedGeometry = mergeGeometries(geometries);
const mergedMesh = new THREE.Mesh(mergedGeometry, sharedMaterial);

// InstancedMesh — 相同几何体的重复绘制
const instancedMesh = new THREE.InstancedMesh(geometry, material, count);
const dummy = new THREE.Object3D();
for (let i = 0; i < count; i++) {
  dummy.position.set(Math.random() * 10, Math.random() * 10, Math.random() * 10);
  dummy.updateMatrix();
  instancedMesh.setMatrixAt(i, dummy.matrix);
}
```

#### (2) 纹理优化

```javascript
// 限制纹理尺寸 — 大多数场景 1024x1024 足够
texture.minFilter = THREE.LinearMipmapLinearFilter;
texture.generateMipmaps = true;

// 使用 KTX2 压缩纹理（GPU 友好，VRAM 占用小）
import { KTX2Loader } from 'three/addons/loaders/KTX2Loader.js';
const ktx2Loader = new KTX2Loader()
  .setTranscoderPath('/basis/')
  .detectSupport(renderer);
```

#### (3) 渲染优化

```javascript
// 限制 pixel ratio
renderer.setPixelRatio(Math.min(window.devicePixelRatio, 2));

// 按需渲染 — 无交互时不渲染（如果不需要持续动画）
let needsRender = true;
function animate() {
  if (needsRender) {
    composer.render();
    // needsRender = false;  // 如果场景静止
  }
  requestAnimationFrame(animate);
}

// 后处理降分辨率
const bloomPass = new UnrealBloomPass(
  new THREE.Vector2(window.innerWidth * 0.5, window.innerHeight * 0.5),  // 半分辨率
  0.8, 0.3, 0.85
);
```

#### (4) 资源释放（防内存泄漏）

```javascript
function disposeObject(obj) {
  if (obj.geometry) obj.geometry.dispose();
  if (obj.material) {
    if (Array.isArray(obj.material)) {
      obj.material.forEach(mat => disposeMaterial(mat));
    } else {
      disposeMaterial(obj.material);
    }
  }
}

function disposeMaterial(material) {
  for (const key of Object.keys(material)) {
    const value = material[key];
    if (value && typeof value.dispose === 'function') {
      value.dispose();  // 释放纹理
    }
  }
  material.dispose();
}

// 页面卸载时清理
window.addEventListener('beforeunload', () => {
  scene.traverse(disposeObject);
  renderer.dispose();
  renderer.forceContextLoss();
});
```

#### (5) 粒子系统优化

```javascript
// 大量粒子用 shader 在 GPU 端做动画，避免 CPU 逐粒子更新
// 顶点着色器内计算位移
const particleShader = {
  vertexShader: `
    attribute float aOffset;
    uniform float u_time;
    void main() {
      vec3 pos = position;
      pos.y += sin(u_time * 0.5 + aOffset * 6.28) * 0.3;
      pos.x += cos(u_time * 0.3 + aOffset * 3.14) * 0.2;
      vec4 mvPosition = modelViewMatrix * vec4(pos, 1.0);
      gl_PointSize = (5.0 / -mvPosition.z);
      gl_Position = projectionMatrix * mvPosition;
    }
  `,
  fragmentShader: `
    void main() {
      float d = length(gl_PointCoord - 0.5);
      if (d > 0.5) discard;
      float alpha = 1.0 - smoothstep(0.3, 0.5, d);
      gl_FragColor = vec4(0.5, 0.8, 1.0, alpha * 0.6);
    }
  `
};
```

### 5.3 优化检查清单（上线前）

- [ ] `renderer.info.render.calls` < 50
- [ ] `renderer.info.render.triangles` < 100K
- [ ] `renderer.setPixelRatio(Math.min(dpr, 2))` 已设置
- [ ] 所有纹理 <= 2048x2048
- [ ] 无 `console.log` 在渲染循环中
- [ ] `dispose()` 在组件卸载/页面离开时调用
- [ ] `requestAnimationFrame` 在不可见时自动暂停
- [ ] 移动端已测试触摸交互
- [ ] 低端设备降级策略已实现
- [ ] Bloom 渲染分辨率根据设备分级调整

---

## 附录：测试工具安装

```bash
# stats.js
npm install stats.js

# 或在 HTML 中 CDN 引入（开发调试用）
# <script src="https://cdnjs.cloudflare.com/ajax/libs/stats.js/r17/Stats.min.js"></script>

# Spector.js Chrome 扩展
# 在 Chrome Web Store 搜索 "Spector.js" 安装
```

---

## 参考资源

- [Building Efficient Three.js Scenes (Codrops 2025)](https://tympanus.net/codrops/2025/02/11/building-efficient-three-js-scenes-optimize-performance-while-maintaining-quality/)
- [100 Three.js Tips That Actually Improve Performance](https://www.utsubo.com/blog/threejs-best-practices-100-tips)
- [Draw Calls: The Silent Killer](https://threejsroadmap.com/blog/draw-calls-the-silent-killer)
- [Three.js Performance Tips (Three.js Journey)](https://threejs-journey.com/lessons/performance-tips)
- [WebGL Browser Support (Can I Use)](https://caniuse.com/webgl)
- [r3f-perf (React Three Fiber Performance Monitor)](https://github.com/utsuboco/r3f-perf)
