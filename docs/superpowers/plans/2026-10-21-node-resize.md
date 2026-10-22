# 画布节点右下角拖柄缩放 Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 为 ToonFlow 无限画布的全部 9 种节点增加右下角拖柄自由缩放能力，不改变任何现有视觉风格。

**Architecture:** 在所有节点的共同骨架 `packages/nodeScaffold/src/nodeSkeleton.vue` 中加入拖柄按钮与拖动逻辑（复用 `groupNode.vue` 已验证的 pointer-capture + `batchCanvasHistory` + `updateNode(style)` 范式）；尺寸写入 `node.style.width/height`（px 字符串）从而天然获得自动保存、撤销、落盘与复制粘贴兼容；用户设置尺寸后骨架切换 `width/height: 100%` 并用 `:deep()` 放开内容层高度封顶；分组父框由 `groupNode.vue` 新增 dimensions watcher 调用 `fitGroupBounds` 贴合。

**Tech Stack:** Vue 3（script setup）、@vue-flow/core 1.48.2、Element Plus、SCSS、bun workspaces。

**Spec:** `docs/superpowers/specs/2026-10-21-node-resize-design.md`

---

## 前置说明（执行前必读）

1. **工作仓库**：`/Users/fans/代码项目/Toonflow-app`（专用克隆，master 浅克隆）。本计划在此仓库内建 feature 分支执行，等效于独立 worktree。
2. **仓库测试现状**：全仓库**没有任何单元测试框架**（无 vitest/playwright/jest 配置与测试文件），AGENTS.md 也未规定测试框架。因此本计划的验证手段为：**`bun run typecheck` + 开发模式手动 QA 清单**（Task 7），不引入新测试框架（YAGNI，且引入测试基建本身是独立子项目）。
3. **Git 纪律（FATAL-005）**：用户全局规则要求所有 commit 必须由用户明确说 "commit" 或调用 /commit 技能后才执行。计划中的 commit 步骤执行到该步骤时**暂停并征求用户确认**，其余步骤照常推进。
4. **代码风格（AGENTS.md，必须遵守）**：
   - 自有变量/函数/计算属性/ref/事件处理函数：小驼峰；DOM 类名与选择器：小驼峰 SCSS 嵌套。
   - 禁止全大写常量、短横线/蛇形文件名；自有组件模板标签禁止大驼峰/短横线（`<nodeSkeleton />` 不是 `<NodeSkeleton />`）。
   - `<template>` → `<script>` → `<style>` 顺序；优先复用已有工具与 store 方法，不新增抽象层。
5. **设计引用的行号**基于当前 master（`30139c9`），执行时以实际代码为准，若上游有漂移按语义定位。

## 文件结构总览

| 文件 | 动作 | 职责 |
|---|---|---|
| `packages/nodeScaffold/src/nodeSkeleton.vue` | 修改 | 拖柄按钮（模板）、缩放交互与显隐逻辑（脚本）、`userSized` 尺寸跟随与拖柄样式（样式）、右键副本/剪贴板复制携带 `style` |
| `apps/web/src/pages/workspace/panels/canvas/components/groupNode.vue` | 修改 | 子节点尺寸变化时贴合分组框（`fitGroupBounds` watcher） |
| `apps/web/src/pages/workspace/panels/canvas/nodeClipboard.ts` | 修改 | `copyNodeToClipboard` 参数类型扩展到 `style`，使单节点剪贴板复制携带尺寸 |
| 无新建文件 | — | 逻辑内聚在上述三处，不新增抽象层 |

> 依赖安装：`bun install` 已在计划执行前完成（如在干净环境执行，先跑根目录 `bun install`）。

---

### Task 1: 建立 feature 分支

**Files:** 无代码改动

- [ ] **Step 1: 确认工作区状态**

Run: `cd /Users/fans/代码项目/Toonflow-app && git status --porcelain`
Expected: 两条记录——`?? docs/superpowers/`（设计/计划文档）与 `M packages/mcp/src/stdio.ts`。后者是克隆/安装过程中产生的**纯文件权限位变化**（100644 → 100755，`git diff` 仅显示 mode 行，无内容改动），**保持原样、不纳入任何提交**；如出现其他意外改动，停下询问用户。

- [ ] **Step 2: 创建分支**

```bash
cd /Users/fans/代码项目/Toonflow-app
git checkout -b feat/node-resize
```

Expected: `Switched to a new branch 'feat/node-resize'`

- [ ] **Step 3: 暂存设计文档并征求用户 commit 确认**

```bash
git add docs/superpowers/specs/2026-10-21-node-resize-design.md docs/superpowers/plans/2026-10-21-node-resize.md
```

**⚠️ 暂停点**：向用户确认是否 commit 文档（FATAL-005）。用户同意后：

```bash
git commit -m "docs: 添加节点缩放功能设计文档与实施计划"
```

---

### Task 2: nodeSkeleton 脚本——缩放状态与交互逻辑

**Files:**
- Modify: `packages/nodeScaffold/src/nodeSkeleton.vue`（script 段：import 区、`useVueFlow` 解构区约 266-300 行、`slotsVisible` 附近约 351 行、`moveHandle` 函数附近约 510 行之前）

- [ ] **Step 1: 补充 import**

在文件顶部现有 import 中补两处（保持既有顺序风格，不新增依赖包）：

```ts
import { computed, inject, nextTick, onBeforeUnmount, onScopeDispose, ref, shallowRef, watch, watchEffect, type Component, type ShallowRef } from "vue";
```

（相对原行 165 仅新增 `onBeforeUnmount`。）

- [ ] **Step 2: 在 `useVueFlow()` 解构列表中补充两个 action**

在 `updateNodeInternals,`（原 286 行附近）旁新增：

```ts
  updateNode,
  updateNodeInternals,
  screenToFlowCoordinate,
```

- [ ] **Step 3: 注入批量历史并声明缩放状态**

在 `const selectionConnection = inject...`（原 263 行）之前加入：

```ts
const batchHistory = inject<((action: () => Promise<void>) => Promise<void>) | undefined>("batchCanvasHistory", undefined);
const resizeState = shallowRef<{
  node: import("@vue-flow/core").GraphNode;
  element: HTMLElement;
  pointerId: number;
  point: { x: number; y: number };
  width: number;
  height: number;
  resolve: () => void;
}>();
```

> 说明：`GraphNode` 类型改为在 import 区统一引入更干净——在原 166 行追加 `import type { GraphNode } from "@vue-flow/core";`，状态类型直接写 `GraphNode`，避免内联 import。

- [ ] **Step 4: 定义显隐与用户尺寸计算属性**

在 `slotsVisible`（原 351 行）定义之后加入：

```ts
// ACT: 右下角缩放拖柄的门控：只读、文档模式、多选（>1）、连线中均不显示；悬停/选中/缩放中的显隐由 CSS 控制（与 groupNode 一致）。
// slotsHiddenByInteraction 门控是本功能额外引入（spec"隐藏场景"已确认）：框选开始或拖动节点期间隐藏，
// 框选后需点击节点（onNodeClick 复位）才会重新显示——与槽位显隐行为一致，QA 时按此预期验证。
const resizeEnabled = computed(
  () => !props.readOnly && !documentTarget.value && !slotsHiddenByInteraction.value && !connectionStartHandle.value && getSelectedNodes.value.length <= 1
);
const userSized = computed(() => {
  const style = node.style;
  return !!style && typeof style === "object" && !!(style.width || style.height);
});
```

- [ ] **Step 5: 实现三个交互函数**

在 `moveHandle` 函数（原 510 行）之前加入（代码取自 `groupNode.vue:89-141`，裁剪为单角、不改 position）：

```ts
function startResize(event: PointerEvent) {
  if (!resizeEnabled.value || event.button !== 0 || resizeState.value) return;
  const target = findNode(nodeId);
  if (!target) return;
  const action = () => new Promise<void>(resolve => {
    const element = event.currentTarget as HTMLElement;
    resizeState.value = {
      node: target, element, pointerId: event.pointerId,
      point: flowScreenToFlow(event),
      width: target.dimensions.width, height: target.dimensions.height, resolve,
    };
    target.resizing = true;
    element.setPointerCapture(event.pointerId);
  });
  void (batchHistory ? batchHistory(action) : action()).catch(error => {
    finishResize();
    ElMessage.error(error instanceof Error ? error.message : "节点缩放失败");
  });
}

function flowScreenToFlow(event: PointerEvent) {
  return screenToFlowCoordinate({ x: event.clientX, y: event.clientY });
}

function moveResize(event: PointerEvent) {
  const state = resizeState.value;
  if (!state || event.pointerId !== state.pointerId) return;
  if (findNode(nodeId) !== state.node) return finishResize();
  const point = screenToFlowCoordinate({ x: event.clientX, y: event.clientY });
  const width = Math.max(120, state.width + (point.x - state.point.x));
  const height = Math.max(80, state.height + (point.y - state.point.y));
  updateNode(nodeId, {
    style: { ...(typeof state.node.style === "object" ? state.node.style : {}), width: `${width}px`, height: `${height}px` },
  });
  state.node.dimensions = { width, height };
}

function finishResize(event?: PointerEvent) {
  const state = resizeState.value;
  if (!state || (event && event.pointerId !== state.pointerId)) return;
  resizeState.value = undefined;
  state.node.resizing = false;
  if (state.element.hasPointerCapture(state.pointerId)) state.element.releasePointerCapture(state.pointerId);
  state.resolve();
}

onBeforeUnmount(() => finishResize());
watch(() => props.readOnly, readOnly => {
  if (readOnly) finishResize();
});
```

> 注意：`flowScreenToFlow` 是临时换算封装，若觉得多余可直接在 `startResize` 内联 `screenToFlowCoordinate(...)`（AGENTS.md：仅当可读性有明确提升才抽取）。二选一，保持与 groupNode 结构对应即可。
> `ElMessage` 已在原 180 行 import，无需新增。

- [ ] **Step 6: 类型检查脚本改动**

Run: `cd /Users/fans/代码项目/Toonflow-app && bun run --filter '@toonflow/nodes-scaffold' typecheck`
Expected: 通过（若该 workspace 无 typecheck 脚本则跳过，在 Task 7 用根级 typecheck 兜底）。
如报错：优先检查 `updateNode`/`screenToFlowCoordinate` 是否在解构列表中拼写正确、`GraphNode` import 是否到位。

- [ ] **Step 7: 右键"创建副本/复制节点"携带尺寸**

同一文件 `handleCommand` 中 `command === "copy" || command === "duplicate"` 分支的 `addNodes({...})`（原 475-482 行）**不含 `style`**，右键副本会丢尺寸。将调用改为：

```ts
    addNodes({
      id,
      zIndex: node.zIndex,
      type: node.type,
      position: copyPosition,
      style: typeof node.style === "object" && node.style ? { ...node.style } : node.style,
      data: { ...data, label: `${props.label} - 副本` },
      parentNode: node.parentNode,
    });
```

- [ ] **Step 8: "复制到剪贴板"命令携带尺寸（scaffold 侧）**

同文件两处修改：

1. `copyNodeToClipboard` 的 inject 类型（原 418 行）参数扩展 `style`：

```ts
const copyNodeToClipboard = inject<((node: { type?: string; data: NodeData; style?: import("@vue-flow/core").Node["style"] }) => Promise<void>) | undefined>(
  "copyNodeToClipboard",
  undefined
);
```

（若同文件已 import type `Node` 则直接用 `Node["style"]`；为与既有 import 风格一致，可在 166 行处追加 `type Node` 一并引入，避免内联 import。）

2. `handleCommand` 中 `command === "clipboard"` 的调用（原 440 行）补 `style`：

```ts
      await copyNodeToClipboard({ type: node.type, data, style: node.style });
```

- [ ] **Step 9: 类型检查**

Run: `cd /Users/fans/代码项目/Toonflow-app && bun run --filter '@toonflow/nodes-scaffold' typecheck`
Expected: PASS。宿主侧（index.vue provide）参数类型取自 `Parameters<typeof copyNodeToClipboard>[0]`，在 Task 6 扩展函数签名后自动跟随；若此时根级 typecheck 报宿主 provide 参数不兼容，先完成 Task 6 再复跑。

---

### Task 3: nodeSkeleton 模板——根元素绑定与拖柄按钮

**Files:**
- Modify: `packages/nodeScaffold/src/nodeSkeleton.vue`（template 段第 4 行根 div、第 159 行 `cardContainer` 闭合后）

- [ ] **Step 1: 根元素加入尺寸类与选中/缩放类，并接管 attrs.style**

将根 div（原第 4 行）：

```html
  <div ref="skeletonElement" v-bind="$attrs" class="nodeSkeleton" :class="{ documentNode: documentTarget }" @wheel.capture="zoomCanvas" @dblclick.stop="focusNode" @contextmenu="documentTarget && openMenu($event)">
```

改为：

```html
  <div ref="skeletonElement" v-bind="boundAttrs" class="nodeSkeleton" :class="{ documentNode: documentTarget, userSized: userSized, nodeSelected: node.selected, nodeResizing: !!resizeState }" @wheel.capture="zoomCanvas" @dblclick.stop="focusNode" @contextmenu="documentTarget && openMenu($event)">
```

并在 script 中加入（`userSized` 计算属性附近）：

```ts
const attrs = useAttrs();
const boundAttrs = computed(() => ({
  ...attrs,
  // ACT: 用户缩放过（style 有尺寸）时，节点自带的内联宽度让位于骨架 100% 填充，避免内联样式压过 wrapper 尺寸。
  style: userSized.value ? { width: "100%", height: "100%" } : attrs.style,
}));
```

`useAttrs` 需要从 vue import（并入 Step 1 的 import 行）。注意 `defineOptions({ inheritAttrs: false })` 已存在（原 188 行），行为不变。

- [ ] **Step 2: 插入拖柄按钮**

在 `cardContainer` 的闭合 `</div>`（原 159 行）与外层根 div 闭合（原 160 行）之间插入：

```html
    <button
      v-if="resizeEnabled"
      class="resizeCorner nodrag nopan"
      type="button"
      aria-label="调整节点大小"
      @pointerdown.stop.prevent="startResize"
      @pointermove.stop="moveResize"
      @pointerup.stop="finishResize"
      @pointercancel="finishResize"
      @lostpointercapture="finishResize"
      @mousedown.stop
      @click.stop
      @dblclick.stop />
```

> 按钮必须保持 `nodrag nopan` + `pointerdown.stop.prevent`，否则按下会拖动节点或画布（设计文档"交互隔离"要求）。单角（右下）仅一个按钮，无需 `v-for`。

- [ ] **Step 3: 再次类型检查**

Run: `cd /Users/fans/代码项目/Toonflow-app && bun run typecheck`
Expected: PASS（根级命令跑全部 workspace，作为兜底）。

---

### Task 4: nodeSkeleton 样式——拖柄外观与 userSized 尺寸跟随

**Files:**
- Modify: `packages/nodeScaffold/src/nodeSkeleton.vue`（style 段第 580 行 `.nodeSkeleton` 块）

- [ ] **Step 1: 根块加入 userSized 与拖柄显隐规则**

在 `.nodeSkeleton { ... }` 内、`&.documentNode` 块（原 586 行）之后加入：

```scss
  &.userSized {
    width: 100%;
    height: 100%;
    display: flex;
    flex-direction: column;

    .cardContainer {
      flex: 1;
      min-height: 0;
    }

    .contentCard {
      height: 100%;

      :deep(.el-card__body) {
        height: 100%;
      }

      // ACT: 放开各组件内容层的高度封顶，让内容跟随用户拖出的高度（含 img/video/canvas 与文本预览）。
      :deep(img),
      :deep(video),
      :deep(canvas),
      :deep(.textPreview) {
        max-height: 100%;
      }
    }
  }

  &.nodeSelected .resizeCorner,
  &:hover .resizeCorner,
  &.nodeResizing .resizeCorner {
    opacity: 1;
  }

  .resizeCorner {
    position: absolute;
    right: -5px;
    bottom: -5px;
    width: 9px;
    height: 9px;
    padding: 0;
    border: 1px solid var(--el-color-primary);
    border-radius: 2px;
    background: var(--el-bg-color);
    opacity: 0;
    touch-action: none;
    cursor: nwse-resize;
    z-index: 3;

    &:focus-visible {
      opacity: 1;
    }
  }
```

> 说明：
> - 拖柄视觉规格（9×9、primary 边框、2px 圆角、hover/selected/resizing 显隐）完全照抄 `groupNode.vue:214-230`，风格不变。
> - `.userSized` 的 `:deep()` 覆盖用于通用媒体/文本封顶；各组件特有的封顶（如 imageGridSplit 140px）若 QA 发现残留，在对应插件内做最小修复（Task 7 QA 驱动，避免预先过度修改）。
> - 拖柄定位在 `.nodeSkeleton`（`position: relative`，原 581 行）之内，无需新增定位上下文。

- [ ] **Step 2: 类型检查 + 样式编译确认**

Run: `cd /Users/fans/代码项目/Toonflow-app && bun run typecheck && bun run --filter '@toonflow/web' build 2>&1 | tail -20`
Expected: typecheck PASS；web build 成功（SCSS 编译无错）。build 慢是预期，仅需确认无编译错误。

---

### Task 5: groupNode——子节点缩放后贴合分组框

**Files:**
- Modify: `apps/web/src/pages/workspace/panels/canvas/components/groupNode.vue`（import 区、children watcher 原 150-158 行）

- [ ] **Step 1: 导入 fitGroupBounds**

```ts
import { fitGroupBounds } from "../selectionNodes";
```

- [ ] **Step 2: 扩展 children watcher，尺寸变化时贴合**

将原 150-158 行的 children watcher 替换为（保留 expandParent 逻辑，追加贴合）：

```ts
watch(
  () => props.readOnly ? [] : flow.getNodes.value.filter(node => node.parentNode === props.id),
  children => {
    if (props.readOnly) return;
    // ACT: 拖动期间保持分组框不变，松手后由画布统一贴合内容。
    children.forEach(node => { node.expandParent = false; });
  },
  { immediate: true },
);

// ACT: 子节点被右下角拖柄缩放后（dimensions 变化，非拖动 position 变化），父分组框及祖先分组框重新贴合。
watch(
  () => {
    if (props.readOnly) return "";
    return flow.getNodes.value
      .filter(node => node.parentNode === props.id)
      .map(node => `${node.dimensions.width}x${node.dimensions.height}`)
      .join(";");
  },
  () => {
    if (props.readOnly) return;
    const affected = new Set<GraphNode>();
    let current: GraphNode | undefined = flow.findNode(props.id);
    const visited = new Set<string>();
    while (current && !visited.has(current.id)) {
      visited.add(current.id);
      if (current.type === "canvasGroup") affected.add(current);
      current = current.parentNode ? flow.findNode(current.parentNode) : undefined;
    }
    if (affected.size) fitGroupBounds(flow.getNodes.value, affected);
  },
);
```

> 说明：watcher 只监听 `dimensions` 字符串指纹——普通拖动只改 `position` 不触发；本功能拖动中每帧更新 dimensions → 分组框实时跟随（fitGroupBounds 改的是 group position/style 与子节点 position，不回写子节点 dimensions，无循环触发风险）。`GraphNode` 类型该文件原 40 行已完整 import，直接使用即可。

- [ ] **Step 3: 类型检查**

Run: `cd /Users/fans/代码项目/Toonflow-app && bun run typecheck`
Expected: PASS。

---

### Task 6: nodeClipboard——单节点剪贴板复制携带尺寸

**Files:**
- Modify: `apps/web/src/pages/workspace/panels/canvas/nodeClipboard.ts`（原 35 行 `copyNodeToClipboard` 签名）

- [ ] **Step 1: 扩展签名**

将：

```ts
export function copyNodeToClipboard(node: Pick<Node, "type" | "data">, directory: string) {
```

改为：

```ts
export function copyNodeToClipboard(node: Pick<Node, "type" | "data" | "style">, directory: string) {
```

函数体内部 `{ ...node, id, position }` 的展开与 `copyNodesToClipboard` 的 JSON 快照（保留 `style`、剔除运行态字段）无需改动；`readClipboardNodes` 的格式校验不涉及 `style`，粘贴路径（`{...node}` 展开）已保留 `style`。

- [ ] **Step 2: 宿主 provide 类型自动跟随确认**

Run: `cd /Users/fans/代码项目/Toonflow-app && bun run typecheck`
Expected: PASS。`index.vue:268` 的 provide 参数类型是 `Parameters<typeof copyNodeToClipboard>[0]`，签名扩展后自动兼容 Task 2 Step 8 传入的 `style`。若报错，检查 provide 实现是否对参数做了二次收敛（有的话删掉收敛，直接透传）。

---

### Task 7: 开发模式手动 QA（核心验证）

**Files:** 无代码改动（除 QA 中发现的残留封顶修复）

- [ ] **Step 1: 启动开发版桌面应用**

Run: `cd /Users/fans/代码项目/Toonflow-app && bun run dev:desktop`
Expected: 构建并拉起 Electrobun 开发窗口（首次需构建插件，可能较慢）。保持运行。

- [ ] **Step 2: 逐项执行 QA 清单**

在开发窗口的创作画布中验证（对照 spec 验证清单）：

- [ ] 9 种节点（text/image/imageGeneration/video/videoGeneration/frameAnalysis/audio/audioGeneration/director3d）各创建一个：悬停右下角出现拖柄；拖动可自由缩放宽高；缩到 120×80 以下被钳制
- [ ] 节点本体拖动、画布平移/缩放（滚轮）不受拖柄影响
- [ ] 连到被缩放节点的连线端点自动跟随；缩放后仍可正常连新线
- [ ] 图片/视频节点拖高时内容真正撑高（无 240px 留白）；文本节点拖高显示更多文本
- [ ] 骨架宽度正确跟随：audio(360)/director3d(320)/text(220) 缩放后无"外框变了内容没变"或溢出
- [ ] 缩放后自动保存；关闭画布重开，尺寸保留
- [ ] 撤销（Cmd+Z）：一次拖拽 = 一步；重做恢复
- [ ] 三条复制路径分别验证携带尺寸：画布选中 Cmd+C/V（走 `toObject` 快照，预期直接通过）、右键"创建副本/复制节点"（Task 2 Step 7 修复点）、右键"复制到剪贴板"→ 粘贴（Task 2 Step 8 + Task 6 修复点）
- [ ] 节点拖入分组再缩放：分组框贴合；分组自身 4 角缩放不受影响、不重复触发
- [ ] 框选多选多个节点时无拖柄；选中态单节点有拖柄
- [ ] 首页"灵感创作"只读画布：悬停节点无拖柄
- [ ] 双击节点进文档面板（document 模式）：占位符无拖柄，返回画布后正常
- [ ] 深色、浅色主题下拖柄均清晰可见（依赖 `--el-color-primary`/`--el-bg-color`，理论自动适配）
- [ ] 连线进行中（从某端口拖出连线）：目标节点拖柄隐藏

- [ ] **Step 3: 修复 QA 发现的问题**

- 内容层残留封顶（spec 已知候选：imageGridSplit 140px、videoPlayer `.videoPreview` 140/240px 包裹层）。注意 `:deep(video)` 只命中 `<video>` 元素本体，若封顶打在包裹 div 的类上（如 `.videoPreview`），通用规则不会生效——在对应插件组件内放开为跟随容器，最小改动。
- 其他问题按根因修复，修完回到 Step 2 对应项复验。

- [ ] **Step 4: 根级类型检查收尾**

Run: `cd /Users/fans/代码项目/Toonflow-app && bun run typecheck`
Expected: PASS。

- [ ] **Step 5: 征求用户 commit 确认（FATAL-005 暂停点）**

```bash
git add -p   # 逐文件确认改动（禁止 git add -A）
git commit -m "feat: 画布节点右下角拖柄自由缩放"
```

仅在用户明确同意后执行。

---

### Task 8: 构建打包桌面版

**Files:** 无源码改动

- [ ] **Step 1: 生产构建**

Run: `cd /Users/fans/代码项目/Toonflow-app && bun run build 2>&1 | tail -20`
Expected: 退出码 0，无构建错误。

- [ ] **Step 2: 打包**

Run: `cd /Users/fans/代码项目/Toonflow-app && bun run package:desktop 2>&1 | tail -20`
Expected: 产出桌面应用包（路径以命令输出为准，通常在 `apps/desktop/dist` 或 `build` 下）。

- [ ] **Step 3: 冒烟验证**

启动打包产物，重复 Task 7 Step 2 QA 清单中的核心 4 项：拖柄缩放、连线跟随、保存重开、撤销。
Expected: 全部通过。

- [ ] **Step 4: 交付用户，说明安装方式**

向用户报告：产物路径；可与官方 `/Applications/toonflow.app` 并存（改名区分）或替换（需用户决定，不擅自覆盖官方 app）。数据目录（`艾玛呀呀小丸子/toonflow`）为应用共用，替换不影响画布数据。
