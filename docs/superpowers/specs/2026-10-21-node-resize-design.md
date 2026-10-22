# 设计文档：画布节点右下角拖柄缩放

- 日期：2026-10-21
- 状态：已确认（用户批准）
- 分支：`feat/node-resize`（基于 master）

## 背景与目标

ToonFlow 的无限画布（Vue Flow）中，节点目前没有任何调整大小的能力。本设计为**全部 9 种节点**（textNode、imageNode、imageGenerationNode、videoNode、videoGenerationNode、frameAnalysisNode、audioNode、audioGenerationNode、director3dNode）增加"右下角拖柄自由缩放"功能。

**非目标**：不改变任何现有视觉风格；不改变节点内容的生成/编辑逻辑；不实现等比缩放（用户明确选择自由缩放）。

## 需求决策（用户已确认）

| 决策点 | 结论 |
|---|---|
| 风格 | 完全保持原样，只加功能 |
| 范围 | 全部 9 种节点都可缩放 |
| 缩放方式 | 自由缩放（宽、高独立拖动） |
| 拖柄位置 | 节点右下角，悬停/选中时显示 |
| 交付方式 | 源码改造 + 本地构建打包桌面版 |

## 交互设计

- **触发**：悬停或选中任意节点时，右下角出现缩放手柄（小方块，沿用 `groupNode.vue` 现有手柄的视觉风格，随深色/浅色主题自适应）。
- **拖动**：按住手柄拖动，节点宽、高各自独立变化（自由缩放）。
- **边界**：最小宽度 120；最小高度为 `max(80, titleBar 高度 + cardHeight)`——`cardHeight` 是骨架既有 computed（≥100，随端口数增长），标题栏高度缩放开始时实测，保证拖矮后标题栏、卡片 chrome 与端口布局都完整（实现期两轮审查细化，比固定 80 更合理）。
- **隐藏场景**：多选节点时隐藏手柄（显示条件统一为 `getSelectedNodes.length <= 1` 且节点处于 hover/selected/resizing 之一，比 groupNode 在"多选中悬停未选中节点"这一边缘场景更严格，取此规则为准）；框选开始或拖动节点期间隐藏（复用 `slotsHiddenByInteraction` 闩锁，框选后需点击节点才重新显示，与槽位显隐行为一致）；文档面板（document）模式下禁用（节点为占位符）；**readOnly 模式（如首页展示画布 `exploreProcess.vue`）下不渲染拖柄**（nodeSkeleton 所有控件均以 `!readOnly` 门控，拖柄沿用）；连接线进行中隐藏。
- **连线**：拖动尺寸时，连到该节点的连线端点自动跟随（Vue Flow `ResizeObserver` → `updateNodeDimensions` 自动重算 `handleBounds`）。

## 技术设计

### 1. 接入点：`packages/nodeScaffold/src/nodeSkeleton.vue`

所有 9 种节点的共同骨架。在 `.cardContainer` 后插入拖柄按钮：

```html
<button class="resizeCorner nodrag nopan" @pointerdown.stop.prevent="startResize" />
```

- 显隐：`:hover` / `.selected` / `.resizing` 时显示，参考 `groupNode.vue:175-179`。
- 仅单选显示：复用 `slotsVisible`（`nodeSkeleton.vue:351`）同款判断（`getSelectedNodes.length <= 1`）。
- document 模式（`nodeDocumentPlaceholder`）下不渲染拖柄。

### 2. 拖动逻辑：复用 `groupNode.vue:89-141` 范式

- `screenToFlowCoordinate` 换算指针坐标（zoom 无关）。
- `setPointerCapture` + `node.resizing = true`。
- 拖动中：`flow.updateNode(id, { style: { ...prev, width, height } })` 并乐观同步 `node.dimensions = { width, height }`（即时反馈）。style 值写 **px 字符串**（``width: `${width}px` ``），与 `groupNode.moveResize` 的已验证实现完全一致，避免 number/string 分歧。
- 结束：释放指针、`node.resizing = false`；如需可调用 `updateNodeInternals([id])` 收尾。
- 一次拖拽通过 `inject("batchCanvasHistory")`（`index.vue:265` 已 provide）合并为**一步撤销**。

### 3. 尺寸存储：`node.style.width/height`

- **不放 `node.data`**：`useCanvasHistory.capture()` 的 key 只保留 `data.label`，data 改动进不了撤销。
- `style` 在 `nodeRuntimeFields`（`index.vue:604-614`）之外，写入后自动触发自动保存 + 历史 record。
- `toObject()` 落盘保留 `style`，画布 JSON 兼容旧文件（分组节点已在用同一机制）。
- 复制/粘贴天然携带尺寸（`nodeClipboard.ts` 保留 style）。

### 4. 关键坑与处理

| 坑 | 处理 |
|---|---|
| 骨架根固定宽度（text 220 / director3d 320 / audio 360 等内联宽度）会与 wrapper 尺寸打架 | 节点存在用户尺寸（`style.width/height`）时，骨架根改为 `width: 100%` **且 `height: 100%`**（只改宽度会导致拖高后卡片仍是 auto 高度、出现空隙），媒体节点内联宽度遵循"用户尺寸优先" |
| 各组件内容层高度封顶，拖高度无效果 | 用户设置高度后，内容层放开封顶并跟随容器高度。封顶不只图片/视频（`img/video max-height:240px`），还包括 textNode `.textPreview max-height:240px`、imageGridSplit 140px 等各组件级封顶——9 种节点 QA 时按**通用规则**逐个核查 |
| 缩放子节点后超出分组框 | 完成后触发分组 `fitGroupBounds` 逻辑（`selectionNodes.ts:39-55`） |
| 分组节点已有 4 角 resize，勿冲突 | 拖柄逻辑只加在 `nodeSkeleton`（普通节点）；分组节点走原有代码，不重复处理 |
| 拖动中 ResizeObserver 延迟校正可能抖动 | 沿用 groupNode 模式：拖动中用乐观 `dimensions`，结束时统一 resolve |

## 交付

1. 克隆仓库：`/Users/fans/代码项目/Toonflow-app`（已完成，depth 1）。
2. 建分支 `feat/node-resize`。
3. `bun install` → `bun run dev:desktop` 开发验证。
4. 验证通过后 `bun run build` + `bun run package:desktop` 打包桌面版。
5. 用户自行决定是否替换 `/Applications/toonflow.app`（官方版与自建版可并存）。

## 验证清单

- [ ] 9 种节点各测：悬停显示拖柄、拖动缩放、最小尺寸限制
- [ ] 连线端点随尺寸自动跟随
- [ ] 保存画布 → 重开，尺寸保留
- [ ] 撤销/重做：一次拖拽 = 一步
- [ ] 复制粘贴节点携带尺寸
- [ ] 分组内节点缩放，分组框表现正常
- [ ] 多选时无拖柄；文档面板无拖柄；readOnly 首页画布无拖柄
- [ ] 各组件内容层封顶（图片/视频 240、textPreview 240、imageGridSplit 140 等）均已放开，拖高有实际效果
- [ ] 深色、浅色主题下手柄样式正常
- [ ] `bun run typecheck` 通过
