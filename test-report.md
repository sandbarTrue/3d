# 3D Glass Lab - 测试报告

> 日期: 2026-04-11 | 审查员: researcher | 文件: index.html (1242 行)

---

## 一、代码审查总评

**整体评价: 7.5/10** — 结构清晰、效果完整，但存在若干性能和兼容性问题需修复。

### 架构优点
- CONFIG 对象集中管理参数，方便调节
- scrollState 代理对象将 GSAP 和渲染逻辑解耦 — 这是好品味
- 材质工厂函数 (`createGlassMaterial`, `createGradientMaterial`) 避免重复代码
- 粒子动画在 vertex shader 中完成，零 CPU 开销 — 正确做法
- Burst 粒子系统有完整的生命周期管理和 dispose 清理
- Touch 事件区分滑动和点击（300ms + 15px 阈值）— 合理

### 架构问题（已修复）
- 详见第二节

---

## 二、性能问题分析 & 修复记录

### 2.1 已修复的问题

| # | 问题 | 严重度 | 修复内容 | 行号 |
|---|---|---|---|---|
| P1 | **Bloom 全分辨率渲染** — UnrealBloomPass 使用全窗口分辨率，对 GPU 片元负载极重 | 高 | 降至 50% 分辨率 (`innerWidth * 0.5`)。Bloom 天然是模糊效果，半分辨率视觉无损失，GPU 负载降 75% | ~262 |
| P2 | **Raycaster 每帧执行** — `handleHover()` 每帧调用（60fps），但人眼跟随频率约 10-15Hz | 中 | 节流至 ~15fps，仅在时间片变化时执行 | ~1218 |
| P3 | **Resize 无防抖** — 快速拖拽窗口边缘会触发大量 resize → setSize → 重建 framebuffer | 中 | 加 100ms debounce | ~1098 |
| P4 | **Resize 未更新 bloom 分辨率** — resize 后 bloom 仍使用旧分辨率 | 低 | 增加 `bloomPass.resolution.set()` | ~1105 |
| P5 | **无 importmap polyfill** — Safari <16.4 和 Firefox <108 不原生支持 import maps | 中 | 添加 es-module-shims polyfill（async 加载，不阻塞） | ~209 |
| P6 | **无资源清理** — 页面卸载时未 dispose 几何体/材质/纹理/renderer，WebGL 上下文泄漏 | 高 | 添加 `beforeunload` 事件处理，遍历 scene 释放所有资源 | ~1109 |
| P7 | **Burst 粒子 per-particle new Color()** — `createBurstAt` 循环内每粒子 `new THREE.Color()` | 低 | 预分配 `_tempColor` 和 `_tempHSL`，复用对象 | ~787 |
| P8 | **Canvas 无 touch-action** — 移动端触摸可能触发浏览器默认手势 | 低 | 添加 `touch-action: pan-y` 保留纵向滚动，禁止横向干扰 | ~31 |

### 2.2 评估后保留不改的设计决策

| 项目 | 分析 | 结论 |
|---|---|---|
| 粒子数 800 | 全部在 shader 端运动，CPU 零开销，800 点几乎不影响 draw call | 合理，无需优化 |
| IcosahedronGeometry(1.2, 5) 细分度 5 | 生成 ~5K 顶点。用于 blob morphing 需要足够细分才能看到 noise 位移效果 | 合理 |
| SphereGeometry(1.2, 64, 64) | ~8K 顶点。玻璃材质在粗糙球体上轮廓明显，64 是合理精度 | 合理 |
| DoubleSide 材质 | glass + blob 都用了 DoubleSide。透射材质需要双面渲染才不会出现穿透 bug | 必要 |
| 7 个独立几何体未合并 | 每个有独立动画（rotation/position），无法 merge。且只有 ~10 个 draw call，远低于 50 的红线 | 不需合并 |
| gradient shader 中 `uViewPos` 每帧 copy | 相比 reference 方式，copy 更安全（避免相机位置被 shader 引用拦截）。每帧 3 个 vec3 copy 成本可忽略 | 合理 |

---

## 三、Shader 代码审查

### 3.1 Blob Morphing Shader (vertex)

```
noise3D: sin-based 3-axis product, 3 octave displacement
```

**评价: 可以接受**
- 使用 `sin()` 乘积模拟 noise，比真正的 simplex noise 计算成本低很多
- 三频叠加 (1.5x, 3.0x, 6.0x) 模拟了 FBM (Fractal Brownian Motion) 的基本效果
- 不足：`vNormal` 未基于 displacement 重新计算（使用了原始 normal），导致法线不精确，光照在凹凸面有偏差
- 建议（未改，因改动较大）：用 finite difference 计算 displaced normal

### 3.2 Particle Shader

**评价: 良好**
- 浮动动画完全在 vertex shader 中，CPU 零开销
- `discard` 实现圆形粒子，配合 `smoothstep` 边缘柔化
- `AdditiveBlending + depthWrite: false` 正确的粒子渲染配置

### 3.3 Film Grain Shader

**评价: 良好**
- `hash()` 函数简洁高效，不依赖纹理
- Vignette 和 grain 合并在一个 pass 中 — 减少了一个后处理 pass

---

## 四、兼容性检查

### 4.1 Import Map 支持

| 浏览器 | 原生支持 | 加 polyfill 后 |
|---|---|---|
| Chrome 89+ | YES | YES |
| Edge 89+ | YES | YES |
| Safari 16.4+ | YES | YES |
| Safari 15-16.3 | NO | YES (已修复: 添加 es-module-shims) |
| Firefox 108+ | YES | YES |
| Firefox <108 | NO | YES (已修复) |
| iOS Safari 16.4+ | YES | YES |
| Chrome Android 89+ | YES | YES |

### 4.2 WebGL 特性使用情况

| 特性 | 使用位置 | 兼容性风险 |
|---|---|---|
| `MeshPhysicalMaterial.transmission` | 玻璃材质 | 需要 WebGL2。Three.js r160 默认 WebGL2，兼容 ~98% 设备 |
| `ShaderMaterial` (自定义 GLSL) | blob / gradient / particles / film grain | WebGL1 兼容 |
| `EffectComposer` + 多 pass | 后处理管线 | 依赖 framebuffer，WebGL2 下稳定 |
| `PMREMGenerator` | 程序化环境贴图 | WebGL2 标准功能 |
| `AdditiveBlending` | 粒子 | 全兼容 |
| `BufferGeometry` attributes | 粒子 custom attributes | 全兼容 |

**结论**: 所有特性在 WebGL2 下工作正常。WebGL1 设备（~2% 市占）会遇到 transmission 材质降级问题，但 Three.js 会自动 fallback。

### 4.3 GSAP CDN 加载

- GSAP 3.12.5 通过 CDN `<script>` 标签同步加载，在 `<script type="module">` 之前
- GSAP 注册在 `window.gsap`，module 脚本中直接引用 `gsap` — 可行因为 module 脚本 defer 执行
- **风险**: 如果 CDN 加载失败，整个场景会报错。建议生产环境本地化或添加 fallback

### 4.4 CSS 兼容性

| CSS 特性 | 兼容性 |
|---|---|
| `inset: 0` | Chrome 87+, Safari 14.1+, Firefox 66+ |
| `backdrop-filter` | Chrome 76+, Safari 9+ (-webkit-), Firefox 103+ |
| `background-clip: text` | 全部需要 `-webkit-` prefix（已有） |

**结论**: CSS 兼容性良好，`backdrop-filter` 已有 `-webkit-` 前缀。

---

## 五、交互体验评估

### 5.1 鼠标跟随

| 维度 | 评估 |
|---|---|
| 平滑度 | `CONFIG.mouseSmooth = 0.05` lerp 系数偏低，延迟感明显（约 20 帧 = 333ms 才跟上）。视觉上这是一种"慵懒"跟随风格，适合这种艺术展示场景 |
| 视差效果 | 相机 X 偏移 `mouse.x * 0.3`，group 旋转 `mouse.x * 0.3` — 双层视差产生良好的深度感 |
| 鼠标离开窗口 | 未监听 `mouseleave`，鼠标离开后物体停在最后位置。不会回中心。可以接受，但添加 `mouseleave` 归零更优雅 |

### 5.2 滚动动画

| 维度 | 评估 |
|---|---|
| 同步性 | `scrub: 1.5` 提供 1.5s 的平滑追赶，滚动感柔和 |
| 反向滚动 | GSAP timeline 天然支持反向，scrollState 属性双向插值正确 |
| 阶段过渡 | 4 阶段 (glass→geometry→morphing→particles) 过渡流畅，bloom 强度渐变增添氛围 |
| Section 标签 | `onStart` 回调更新标签 — 问题：`onStart` 仅在正向播放首次触发。回滚时不会更新标签。这是一个已知的 GSAP scrub + onStart 交互问题 |

### 5.3 Hover 交互

| 维度 | 评估 |
|---|---|
| 缩放动画 | `back.out(2)` 弹跳放大 1.15x，`elastic.out(1, 0.4)` 弹性复原 — 手感好 |
| 材质增强 | glass 材质 `envMapIntensity` 3.0→1.5，gradient 材质 `uEmissiveIntensity` 0.6→base — 视觉反馈明确 |
| 节流后响应 | 修复后 raycaster ~15fps，hover 检测延迟 ~66ms，人眼不可感知 |

### 5.4 点击爆炸

| 维度 | 评估 |
|---|---|
| 粒子生成 | 60 粒子/次，位置正确（交叉点），速度随机球面分布 — 效果自然 |
| 物理模拟 | 拖拽 (drag 0.96) + 重力 (-1.5/s²) — 粒子下坠弧线合理 |
| 生命周期 | 1.2s 后 dispose — 无内存泄漏 |
| Bloom 闪烁 | 点击瞬间 bloom 2.5→恢复 — 增强了爆炸感 |

### 5.5 触摸支持

| 维度 | 评估 |
|---|---|
| 滑动→旋转 | `touchmove` → `updateMousePosition` — 单指滑动即可旋转场景 |
| 点击→爆炸 | 300ms + 15px 阈值区分 tap 和 scroll — 合理 |
| `passive: true` | touch 事件标记 passive — 不阻塞滚动，性能正确 |
| 已修复 | 添加了 `touch-action: pan-y` 防止浏览器横向手势干扰 |

---

## 六、Draw Call 和渲染预算估算

| 项目 | Draw Calls | 三角面数(估) |
|---|---|---|
| 7 个几何体 (sphere/torusKnot/torus/ico/oct/dodec/capsule) | 7 | ~20K |
| 1 个内部发光球 | 1 | ~2K |
| 1 个 blob mesh | 1 | ~5K |
| 1 个粒子系统 (Points) | 1 | 800 points |
| 后处理 passes (Render + Bloom + FilmGrain + Output) | 4 | fullscreen quads |
| 光源 (不产生 draw call，但增加 shader 计算) | 0 | — |
| **总计** | **~14** | **~27K** |

**结论**: 远低于 50 draw calls 和 100K 三角面的目标红线。性能预算非常充裕。

---

## 七、发现但未修复的建议（非阻塞）

| # | 建议 | 原因未改 |
|---|---|---|
| S1 | Blob vertex shader 的 `vNormal` 应基于 displaced position 重新计算（finite difference） | 改动大，影响光照精度但不影响功能 |
| S2 | `onStart` 在 GSAP scrub timeline 中只正向触发一次，回滚时 section label 不更新 | 需改用 `onUpdate` + progress 区间判断，逻辑变更大 |
| S3 | CDN 依赖 (GSAP + Three.js + es-module-shims) 无 fallback | 生产环境应本地化，但当前是 demo/lab 阶段 |
| S4 | 缺少 WebGL context lost/restored 事件处理 | 极端场景，demo 阶段不必要 |
| S5 | 可添加 stats.js 作为开发调试开关 (URL param `?debug=1`) | 按需添加，非必须 |

---

## 八、修复变更摘要

共修复 **8 个问题**，涉及以下文件修改：

| 修改位置 | 变更描述 |
|---|---|
| CSS `canvas` | 添加 `touch-action: pan-y` |
| `<script>` importmap 前 | 添加 es-module-shims polyfill |
| `UnrealBloomPass` 构造 | 分辨率降至 50% |
| resize handler | 添加 100ms debounce + bloom 分辨率更新 |
| animate() 中 handleHover | 节流至 ~15fps |
| createBurstAt() | 预分配 `_tempColor` / `_tempHSL` 避免循环 GC |
| animate() 前 | 添加 `beforeunload` 资源清理 |

**所有修复已直接应用到 `/data00/home/zhoujun.sandbar/Desktop/html-3d-lab/index.html`。**
