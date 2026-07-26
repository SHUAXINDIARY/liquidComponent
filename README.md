# Liquid Component

一个用于收集和研究液态质感 Web UI 的原生前端组件仓库。组件重点展示折射、流动、弹性运动等视觉效果，同时保留键盘操作、触控交互和减少动态效果等基础可用性支持。

仓库中的组件目前均为独立 HTML 文件，CSS、SVG 滤镜、JavaScript 和所需纹理都内嵌在文件中，无需安装依赖或执行构建命令。

## 使用方式

直接使用浏览器打开对应的 HTML 文件即可预览。例如：

```bash
open liquid-glass-switch.html
```

也可以通过任意静态文件服务器运行：

```bash
npx serve -m http.server 8000
```

然后访问 `http://localhost:8000/liquid-glass-switch.html`。

## 已有组件

| 组件 | 文件 | 交互方式 | 说明 |
| --- | --- | --- | --- |
| Liquid Glass Switch | [`liquid-glass-switch.html`](./liquid-glass-switch.html) | 点击、键盘、平滑拖拽、快速甩动 | 带背景折射、材质形变、弹簧回弹及 Safari 毛玻璃降级的液态玻璃开关 |

## Liquid Glass Switch

### 组件说明

Liquid Glass Switch 是一个可交互的双态开关。滑块经过网格背景时会产生可见的折射，按下时会放大并增强透明度与扭曲强度，释放后通过弹簧动画吸附到开启或关闭位置。

组件支持以下交互和状态：

- 点击开关切换状态。
- 水平拖拽滑块，滑块经过平滑跟随后根据释放位置决定最终状态。
- 快速甩动时根据拖拽速度预测落点，不要求指针必须越过中点。
- 使用键盘激活原生 `button`，通过 `role="switch"` 和 `aria-checked` 暴露开关语义。
- 系统启用“减少动态效果”时，跳过位置弹簧动画。
- 多点触控时仅跟踪第一个有效指针，避免状态互相覆盖。

### 实现原理

#### 1. 结构与状态

组件使用一个带 `role="switch"` 的原生 `button` 作为轨道，内部 `span.thumb` 作为可移动滑块。开关状态保存在 `aria-checked` 中，CSS 根据该属性设置开启和关闭时的轨道颜色与滑块位置。

JavaScript 分别维护滑块位移、运动速度、缩放比例、白色遮罩透明度和滤镜位移强度。视觉状态与指针状态分开保存，因此用户在动画尚未结束时再次按下，组件仍可从当前画面平滑继续。

#### 2. 液态玻璃折射

滑块通过 `backdrop-filter: url(#liquid-glass)` 引用内嵌 SVG 滤镜。滤镜按以下顺序处理滑块背后的画面：

1. `feGaussianBlur` 对背景进行轻微模糊，减少位移后的硬边缘。
2. 第一张内嵌 PNG 作为位移贴图，`feDisplacementMap` 读取其红色和绿色通道，分别控制像素在 X、Y 方向的偏移。
3. `feColorMatrix` 提升折射区域的饱和度，让玻璃边缘更明显。
4. 第二张内嵌 PNG 提供高光纹理，经 `feComposite` 和 `feComponentTransfer` 裁切并降低透明度。
5. 两次 `feBlend` 将折射、饱和色彩和高光重新合成最终的玻璃效果。

拖拽或动画期间，代码会持续更新滑块的 `transform` 和 `feDisplacementMap.scale`。这不仅驱动位置与材质变化，也促使 Chromium 重新采样滑块背后的内容，避免浏览器只移动已经缓存的滤镜图层。

#### 3. 材质动画

按下开关时，`animateMaterial(true)` 同时执行三组插值：

- 将滑块由 `0.65` 放大到 `0.9`。
- 降低白色背景的不透明度，使背后的折射更加可见。
- 增大位移滤镜的 `scale`，强化液态扭曲。

释放后，`animateMaterial(false)` 将这些参数恢复。插值采用四次缓出曲线 `1 - (1 - p)^4`，因此变化前段迅速、末段柔和。

在 Safari/WebKit 降级模式下，滑块仍保留相同的缩放和透明度动画，但视觉效果不依赖无法生效的 SVG 折射。交互开始时改为增强普通 `backdrop-filter` 毛玻璃，交互结束后恢复静止样式。

#### 4. 位移动画

滑块吸附到目标位置时没有使用固定的 CSS 位移动画，而是在 `requestAnimationFrame` 中计算近似临界阻尼弹簧：

```text
acceleration = stiffness * (target - position) - damping * velocity
velocity += acceleration * deltaTime
position += velocity * deltaTime
```

弹簧模型让滑块快速启动并自然减速，同时避免明显的反复震荡。每帧时间步长限制为最多 `1/30` 秒，减少页面短暂卡顿后数值突变造成的跳跃。

#### 5. 拖拽与释放判定

组件使用 Pointer Events 统一处理鼠标、触控笔和触摸操作，并通过 Pointer Capture 确保指针移出按钮后仍能继续拖拽。

指针移动时只更新目标位置，`followDragTarget()` 再通过 `requestAnimationFrame` 让滑块以指数插值追向目标：

```text
follow = 1 - exp(-28 * deltaTime)
position += (target - position) * follow
```

这种方式会保留轻微的跟随感，避免滑块逐像素绑定鼠标所产生的生硬移动；插值包含帧间隔，因此在不同刷新率的屏幕上具有接近一致的手感。轨道颜色按照滑块的实际视觉位置同步渐变，而不是直接跟随指针目标。

移动超过 3 像素后才会被识别为拖拽，以区分普通点击。拖拽期间根据滑块进度在关闭色与开启色之间实时插值；释放时将最近的指针速度外推 90 毫秒，以预测滑块落点，再按预测位置是否越过轨道中点决定最终状态。

### 浏览器兼容性

核心交互使用标准 HTML、CSS、Pointer Events 和 `requestAnimationFrame`。完整折射效果依赖浏览器对 `backdrop-filter: url(...)` 与 SVG 滤镜组合的支持。

- Chromium：使用完整 SVG 位移贴图、饱和度和高光合成效果。
- macOS Safari：静止状态与 Chromium 保持一致；按下、拖拽和切换期间使用 `blur(14px) saturate(1.45)` 普通毛玻璃替代 SVG 折射。
- iPhone 和 iPad 上的 WebKit 浏览器：采用与 Safari 相同的毛玻璃降级方案。

降级模式通过 User Agent 判断 WebKit 环境，并排除桌面 Chromium。静止状态使用不透明白色滑块和 `blur(0)`，因此不会改变原始 UI；只有交互期间滑块透明度降低后，背后的普通毛玻璃才会显现。`blur(0)` 与交互态滤镜之间可以连续插值，避免从 `none` 切换滤镜产生突变。

系统启用“减少动态效果”时，轨道颜色和 Safari 毛玻璃的 CSS 过渡会立即完成，位置弹簧动画也会直接跳到目标状态。

## 仓库结构

```text
.
├── README.md
└── liquid-glass-switch.html
```

后续新增组件时，保持每个演示文件可独立运行，并同步更新“已有组件”和对应的“实现原理”章节。
