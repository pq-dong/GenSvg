# GenSVG

一个 AI 驱动的 SVG 生成器，支持流式 JSONL 增量渲染，灵感来自 [vercel-labs/json-render](https://github.com/vercel-labs/json-render)。

## 概述

**GenSVG** 提供了一个健壮、类型安全且受约束的系统，用于使用 AI 生成 SVG。区别与让 AI 直接输出原始、不可预测的 SVG 字符串不同，该项目强制 AI 输出符合严格定义模式（Catalog）的 JSONL 序列。

### 核心价值

- 🔒 **受约束**：AI 只能使用 Catalog 中定义的 SVG 元素。
- ⚡ **可预测**：JSON 输出始终符合预定义的 Zod 模式。
- 🚀 **快速**：支持流式渲染，随着 AI 生成响应逐步更新 UI。
- 🛡️ **类型安全**：完整的 TypeScript 和 Zod 验证。
- 🧩 **控制反转（IoC）**：Catalog 决定元素的渲染方式，保持核心渲染器无关性。

## 架构

1. **Schema 层**：定义 SVG 规范的结构。
2. **Catalog 层**：定义可用的 SVG 元素、其属性（通过 Zod）、渲染方式，以及用于提示 AI 的描述。
3. **Streaming 层**：增量解析 JSON Patch（RFC 6902）流。
4. **Renderer 层**：将 JSON 规范转换为实际的 SVG 字符串或 React 组件。

## 安装

```bash
npm install
npm run dev
```

## 使用方法

### 1. 定义 Catalog

使用 Zod 模式创建允许的 SVG 元素目录。Catalog 决定 `tag` 名称或自定义 `render` 函数。

```typescript
import { z } from "zod";
import { defineSVGSchema, defineSVGCatalog } from "./core/schema";

const schema = defineSVGSchema((s) => ({
  // ... schema 定义
}));

export const myCatalog = defineSVGCatalog(schema, {
  elements: {
    Rect: {
      tag: "rect",
      props: z.object({
        x: z.number(),
        y: z.number(),
        width: z.number(),
        height: z.number(),
        fill: z.string().optional(),
      }),
      description: "矩形",
      hasChildren: false,
    },
    // 你甚至可以定义自定义渲染器！
    Avatar: {
      props: z.object({ src: z.string(), size: z.number() }),
      description: "用户头像",
      render: (props) =>
        `<g><circle r="${props.size / 2}"/><image href="${props.src}"/></g>`,
    },
  },
});
```

### 2. 使用 `useSVGStream` 流式处理 AI 响应

我们提供了一个 React Hook 来轻松管理流式状态。

```tsx
import { useSVGStream } from "./react/useSVGStream";
import { SVGRenderer } from "./react/SVGRenderer";
import { myCatalog } from "./catalog";

function MyComponent() {
  const { spec, isGenerating, generate, stop } = useSVGStream({
    catalog: myCatalog,
    model: "gemini-3.1-pro-preview",
  });

  return (
    <div>
      <button onClick={() => generate("画一个红色的圆")}>生成</button>
      <SVGRenderer spec={spec} catalog={myCatalog} />
    </div>
  );
}
```

## 高级功能

### JSON Patch（RFC 6902）

AI 输出标准的 JSON Patch 操作（`add`、`replace`、`remove`），使其不仅能构建 SVG，还能即时修改现有元素。

### `<defs>` 支持与 ID 前缀

在 Catalog 中标记为 `isDef: true` 的元素（如 `LinearGradient`、`Filter`）会自动收集并渲染到 SVG 顶部的 `<defs>` 块中。`SVGRenderer` 会自动为 ID 添加前缀，以防止在同一页面上渲染多个 SVG 时发生 ID 冲突。

## 支持的 AI 提供商

GenSVG 使用 [Vercel AI SDK](https://sdk.vercel.ai/docs) 支持多个 AI 提供商。你可以通过环境变量进行配置：

- **Gemini**（默认）：使用 `@ai-sdk/google`
- **Doubao / OpenAI**：使用 `@ai-sdk/openai` 兼容端点

在 `.env` 中设置 `VITE_AI_PROVIDER="doubao"` 或 `VITE_AI_PROVIDER="openai"` 来切换提供商。

## 支持的元素

默认 Catalog 支持多种 SVG 元素：

- **基本形状**：`Rect`、`Circle`、`Ellipse`、`Line`
- **路径与多边形**：`Path`、`Polyline`、`Polygon`
- **文本**：`Text`
- **容器**：`Group`
- **定义**：`LinearGradient`、`RadialGradient`、`Stop`、`Filter`、`FeGaussianBlur`、`FeDropShadow`

## 下一步计划（路线图）

基于 AI 驱动 UI 生成的生产经验，该项目的未来演进重点关注交互性、动态数据和自愈机制：

- **阶段 1：交互性与动画**
  - **事件绑定**：允许 Catalog 定义交互事件（`onClick`、`onHover`），使 AI 生成的 SVG 能够触发 React 状态变更。
  - **动画集成**：支持 `<animate>`、`<animateTransform>`，或与 Framer Motion 无缝集成，实现 AI 编排的动画。

- **阶段 2：动态数据绑定（模板）**
  - **Props 注入**：超越静态生成。允许 SVG 规范接受外部数据（如 `data={{ progress: 75 }}`）并将其绑定到 SVG 属性，将输出转变为可复用的数据驱动组件。

- **阶段 3：自愈与自动修正**
  - **LLM 反馈循环**：如果流式处理期间 Zod 验证失败，自动捕获错误并在后台提示 LLM 修正特定属性。
  - **健壮的部分渲染**：增强网络中断时损坏 JSON 补丁的 AST 恢复能力。

- **阶段 4：生态与领域特定 Catalog**
  - `@svg-render/charts`：数据可视化的预构建 Catalog（AI 输出数据，Catalog 处理 D3/SVG 数学）。
  - `@svg-render/diagrams`：节点、边和流程图。
  - **React 组件包装**：支持 `<foreignObject>` 在 AI 生成的 SVG 画布中嵌入复杂的 React 组件。

- **阶段 5：开发者体验（DX）**
  - **可视化检查器**：类似 DevTools 的 UI，点击 SVG 元素可高亮生成它的确切 JSON Patch。
  - **CLI 工具**：通过终端在构建时生成 SVG（`npx svg-render "a red circle"`）。

## 贡献指南

欢迎所有形式的贡献！以下是参与项目开发的流程：

### 开发流程

1. **Fork 本仓库** 到你的 GitHub 账号
2. **Clone 你的 Fork** 到本地：
   ```bash
   git clone https://github.com/your-username/gensvg.git
   cd gensvg
   ```
3. **创建功能分支**（基于 `dev` 分支）：
   ```bash
   git checkout -b feature/your-feature-name
   ```
4. **安装依赖并开发**：
   ```bash
   npm install
   npm run dev
   ```
5**提交代码**（建议使用语义化提交信息）：
   ```bash
   git commit -m "feat: 添加 xxx 功能"
   ```
6**推送到你的 Fork**：
   ```bash
   git push origin feature/your-feature-name
   ```
7**创建 Pull Request** 到本仓库的 `dev` 分支

### 分支说明

| 分支 | 说明           |
|------|--------------|
| `main` | 主分支，稳定的生产代码  |
| `main` | 主分支，稳定的开发中代码 |
| `feature/*` | 功能开发分支       |
| `fix/*` | Bug 修复分支     |

感谢你的贡献！
