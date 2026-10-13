# 视频画面放大缩小 实现方案

> 目标：在播放页实现「双指捏合缩放 + 放大后单指平移」的视频画面缩放能力。
> 弹幕 / 字幕 / 控制 UI 不参与缩放；只对视频画面本身做变换。
> 本文档仅为设计方案，不含任何代码改动。

---

## 0. 需求与交互定义

| 交互 | 行为 |
|---|---|
| 双指捏合 | 以两指焦点为锚点缩放，范围 `1.0x ~ maxScale`（默认 4.0x，可配置） |
| 单指拖动 | **仅在已放大（scale > 1）时**平移画面；未放大时保持原有手势（左亮度 / 右音量 / 横向进度） |
| 松手回弹 | 缩放越界或平移越界时，动画回弹到合法范围 |
| 双击 | `scale > 1` → 复位到 1.0x；`scale == 1` → 播放/暂停（保持原逻辑） |
| 缩放提示 | 可选：缩放/平移时显示倍率或锚点提示气泡（复用现有 gesture 提示层） |
| 排除项 | 弹幕层、字幕层、顶部/底部控制栏、Loading、锁屏按钮均不缩放 |

---

## 1. 现状分析（已确认）

### 1.1 渲染层级（运行时）
```
player_container                (FrameLayout, match_parent)  activity_player.xml
└── DanDanVideoPlayer           (FrameLayout, match_parent, Gravity.CENTER)  动态 addView
    ├── [index 0] 渲染 View      RenderSurfaceView / RenderTextureView / VLCVideoLayout
    └── [index 1] VideoController(FrameLayout, match_parent)   ← 覆盖层
        ├── DanmuView（弹幕）
        ├── 字幕层 / PlayerGestureView / PlayerTopView / PlayerBottomView / ...
```

**关键点**：视频画面与弹幕/字幕/UI 是**兄弟节点覆盖**关系，不是父子。因此「只变换渲染 View」天然就满足「弹幕字幕不缩放」的需求，无需额外处理。

### 1.2 画面尺寸的来源
- 渲染 View 的**基准尺寸**由 `RenderMeasureHelper.doMeasure()` 按 `VideoScreenScale`（原始 / 16:9 / 4:3 / 填充 / CENTER_CROP）计算，经 `onMeasure → setMeasuredDimension` 落地。
- **本方案不修改测量逻辑**，缩放是在基准尺寸之上叠加的「视图变换层」，二者正交。

### 1.3 现有手势（`GestureVideoController`）
基于 `GestureDetector`，当前占用：
- 单击 → 控制栏显隐
- 双击 → 播放/暂停 ← **与双击复位存在冲突，需条件分发**
- 长按 → 倍速
- 垂直滑动 → 左亮度 / 右音量
- 水平滑动 → 进度 ← **与单指平移存在冲突，需状态机**

**当前无任何缩放手势**（无 `ScaleGestureDetector`），项目也无 PhotoView / subsampling-scale-image-view 等缩放库。

### 1.4 关键环境参数
| 参数 | 值 |
|---|---|
| `minSdkVersion` | **21**（Android 5.0） |
| `compileSdkVersion` | 33 |
| `targetSdkVersion` | 29 |
| 播放内核 | ExoPlayer 2.18.1 / IJKPlayer / VLC（可切换） |
| 渲染类型 | SurfaceView / TextureView（可切换）；VLC 用 VLCVideoLayout |
| UI 技术栈 | 传统 View + XML + DataBinding（**无 Compose**） |
| DI | 无框架（`by lazy` + ARouter） |

---

## 2. 方案总览

采用 **「手势驱动 + 视图变换层」** 方案，不侵入测量/内核逻辑：

```
用户双指
   │
   ▼
GestureVideoController ── 新增 ScaleGestureDetector，做手势仲裁
   │  (缩放 / 平移 / 回弹)
   ▼
VideoZoomHelper  ◄── 新增类：状态 + 变换计算 + 边界约束
   │  scaleX/scaleY/translationX/translationY/pivot
   ▼
渲染 View = getRenderView()?.getView()
   (SurfaceView / TextureView / VLCVideoLayout)
```

**为什么用视图变换而不是改测量**：
- 改测量只能实现「比例」类缩放，无法做「自由锚点缩放 + 平移」，且每次都要 `requestLayout`，代价高。
- 视图变换走硬件加速合成，性能好（TextureView 场景），且与现有比例逻辑解耦。

---

## 3. 关键技术设计

### 3.1 变换的施加方式

对**渲染 View** 直接设置原生属性：

```kotlin
view.scaleX = scale
view.scaleY = scale
view.translationX = tx
view.translationY = ty
// pivot 固定为 view 中心（默认），用 translation 补偿锚点
```

选择 pivot = View 中心（默认即可），锚点补偿通过 translation 计算，避免频繁改 pivot 带来的困惑。

**以手指锚点缩放的核心公式**（本地坐标以 View 中心为原点）：

设 `S0/T0` = 手势开始时的缩放与平移，`a` = 手指焦点相对 View 中心的坐标（本地、未变换坐标系），`f` = 本次捏合因子：

```
S1 = clamp(S0 * f, minScale, maxScale)
T1 = T0 + a * (S0 - S1)          // 保证锚点屏幕位置不动
```

焦点换算：把屏幕坐标的焦点转成「相对 View 中心的本地坐标」时，需要除以当前 scale（因为 View 已被变换）。

### 3.2 平移边界约束

变换后画面视觉尺寸 = `w * S`（以中心为基准），可平移半径：

```
maxTx = max(0, (w * S - w) / 2)
maxTy = max(0, (h * S - h) / 2)
tx = tx.coerceIn(-maxTx, maxTx)
ty = ty.coerceIn(-maxTy, maxTy)
```

> 注意：当比例模式为 letterbox（画面小于容器）时，`w*S` 需用**画面实际显示尺寸**而非 View 尺寸，否则边界会偏松。

### 3.3 回弹动画
- 松手后：`S < 1` → 动画到 1.0；`S > maxScale` → 动画到 maxScale；平移越界 → 动画到边界。
- 用 `ValueAnimator`（或 `ViewPropertyAnimator`）驱动，动画期间禁用新手势。

### 3.4 手势仲裁（**本方案最容易踩坑的地方**）

在 `GestureVideoController.onTouchEvent()` 中插入仲裁逻辑：

```
1) 先喂给 ScaleGestureDetector
2) 若 scaleDetector.isInProgress（≥2 指）：
     取消 GestureDetector 当前滑动状态（stopSlide、清 mSeekPosition）
     只处理缩放，直接 return true
3) 若处于「已放大」状态 且 单指：
     onScroll 优先做平移，不走亮度/音量/进度分支
4) 双击 onDoubleTap 条件分发：
     scale > 1 → resetZoom() ；否则 → togglePlay()
```

同时需要处理：
- 两指变一指（`pointerCount` 从 2 降到 1）时，决定是继续平移还是交还原手势，建议以「当前是否放大」为判据。
- 缩放过程中抑制单击/双击识别（`isInProgress` 时给手势检测器置忙）。

### 3.5 缩放能力探测（兼容性核心）

不同内核/表面类型的变换能力差异较大：

| 内核 | 渲染 View | API ≥ 24 | API 21~23 | 说明 |
|---|---|---|---|---|
| ExoPlayer | SurfaceView | ✅ 可变换 | ❌ 变换不生效 | SurfaceView 是独立 layer，旧版本不支持变换 |
| ExoPlayer | TextureView | ✅ | ✅ | 全版本可用，推荐路径 |
| IJKPlayer | SurfaceView | ✅ | ❌ | 同上 |
| IJKPlayer | TextureView | ✅ | ✅ | 可用 |
| VLC | VLCVideoLayout | ⚠️ 需实测 | ⚠️ 需实测 | VLC 自管表面层，自由变换受限于 ScaleType，建议降级或禁用 |

**对策**：
1. 在 `InterSurfaceView` 增加能力探测（如 `fun supportFreeZoom(): Boolean`），或由 `VideoZoomHelper` 按 View 实际类型 + `Build.VERSION.SDK_INT` 判定。
2. 不支持时**禁用双指缩放**（静默降级，不报错）。
3. 可选增强：当 `surfaceType == SURFACE_VIEW` 且 `SDK_INT < 24` 时，提示用户「切换为 TextureView 以获得缩放体验」（不强制重建渲染 View）。

---

## 4. 代码改动点清单（仅列出，不实施）

| 类型 | 文件 | 改动内容 |
|---|---|---|
| 🆕 新增 | `player/utils/VideoZoomHelper.kt` | 缩放状态、变换计算、边界约束、回弹、reset、能力探测 |
| ✏️ 修改 | `controller/base/GestureVideoController.kt` | 新增 `ScaleGestureDetector`；改造 `onTouchEvent` / `onScroll` / `onDoubleTap` 做手势仲裁 |
| ✏️ 修改 | `DanDanVideoPlayer.kt` | 暴露 `getZoomTargetView()`；`initPlayer()` 重建渲染 View 后重置缩放；`setScreenScale()` 时 resetZoom |
| ✏️ 可选 | `surface/InterSurfaceView.kt` + 各实现 | 增加 `supportFreeZoom()` 能力标记 |
| ✏️ 可选 | `wrapper/InterVideoPlayer.kt` / `ControlWrapper.kt` | 透出 `resetZoom()`，供设置/按钮调用 |
| ✏️ 可选 | `controller/video/PlayerGestureView.kt` + `res/layout/layout_player_gesture.xml` | 缩放倍率提示气泡 |
| ✏️ 可选 | 设置面板（新增 `SettingVideoZoomView`）、`PlayerInitializer.kt`、`PlayerConfig.kt` | 缩放开关 / 最大倍率，并做持久化（当前 `screenScale` 也未持久化，可一并补） |
| — | 布局 `activity_player.xml` | **无需改动**（渲染 View 与 UI 已是兄弟节点） |

---

## 5. 需要同时处理的联动点

| 场景 | 处理 |
|---|---|
| 切换画面比例（`setScreenScale`） | 先 `resetZoom()`，再进行比例切换 |
| 锁屏状态 | 锁屏时禁用缩放与平移 |
| 切换分辨率 / 换集（`onVideoSizeChange`） | `resetZoom()` |
| 屏幕方向变化 | `resetZoom()` |
| 悬浮窗模式（popup） | 可选支持；至少保证不破坏现有手势（popup 下已有独立 `mPopupGestureHandler`） |
| 截图（`PlayRecorder.generateRenderImage`） | 注意 TextureView 变换后抓取会带上变换，需评估是否要抓原始帧 |
| 弹幕/字幕对齐 | 无需处理（兄弟节点，不跟随） |

---

## 6. 分阶段实施建议

| 阶段 | 内容 | 产出 |
|---|---|---|
| Phase 1 (MVP) | TextureView 路径：双指缩放 + 边界约束 | 可用的核心缩放 |
| Phase 2 | 单指平移 + 回弹动画 + 双击复位 + 手势仲裁打磨 | 完整体验 |
| Phase 3 | SurfaceView / API 兼容判定与降级 | 全设备不崩 |
| Phase 4 | VLC 适配验证 + 设置开关/倍率 + 提示 UI | 可配置、可发现 |
| Phase 5 | 多内核 × 多表面类型 × 多 API 的回归测试 | 稳定 |

---

## 7. 风险与验证要点

1. **SurfaceView 变换兼容性（最高风险）**：API < 24 无效。必须做能力探测 + 降级，否则会出现「手势响应了但画面不动」的诡异体验。
2. **手势冲突**：单指平移与亮度/音量/进度、双击与播放/暂停。需明确状态机，建议「放大后才让单指平移接管」。
3. **锚点换算精度**：焦点屏幕坐标 → 本地坐标需除以当前 scale，比例错会让缩放「漂移」（手指移动但锚点跑偏）。
4. **letterbox 边界**：画面小于容器时平移约束要用真实显示尺寸。
5. **TextureView 性能/功耗**：相比 SurfaceView 略高，长时间播放需观察发热与掉帧。
6. **VLC 路径**：`VLCVideoLayout` 内部表面由 VLC 托管，自由变换可能无效或有撕裂，需真机实测后再决定「支持/降级/禁用」。
7. **截图功能**：变换后抓图内容会随变换改变，需确认是否符合预期。

---

## 8. 结论

- **推荐方案**：新增 `VideoZoomHelper` + 在 `GestureVideoController` 接入 `ScaleGestureDetector`，对渲染 View 施加 `scale/translation` 视图变换。
- **不需要改动**布局层级与测量逻辑（`RenderMeasureHelper`），弹幕字幕天然不跟随。
- **最大不确定项**是 SurfaceView 在 API 21~23 上的变换能力与 VLC 路径，建议先做 TextureView 的 MVP 打通，再逐步兼容。
