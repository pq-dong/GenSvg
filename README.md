# GenSVG

An AI-powered SVG generator that streams JSONL and renders it incrementally, inspired by [vercel-labs/json-render](https://github.com/vercel-labs/json-render).

[中文文档](README.zh-CN.md)

## Overview

**GenSVG** provides a robust, type-safe, and guardrailed system for generating SVGs using AI. Instead of letting the AI output raw, unpredictable SVG strings, this project forces the AI to output a sequence of JSON patches (JSONL) that conform to a strictly defined schema (Catalog).

### Core Values

- 🔒 **Guardrailed**: AI can only use SVG elements defined in your Catalog.
- ⚡ **Predictable**: JSON output always matches the predefined Zod schema.
- 🚀 **Fast**: Supports streaming rendering, updating the UI progressively as the AI generates the response.
- 🛡️ **Type-Safe**: Full TypeScript and Zod validation.
- 🧩 **IoC (Inversion of Control)**: The Catalog dictates how elements are rendered, keeping the core renderer agnostic.

## Architecture

1. **Schema Layer**: Defines the structure of the SVG specification.
2. **Catalog Layer**: Defines the available SVG elements, their properties (via Zod), how they render, and descriptions to prompt the AI.
3. **Streaming Layer**: Parses JSON Patch (RFC 6902) streams incrementally.
4. **Renderer Layer**: Converts the JSON specification into an actual SVG string or React component.

## Installation

```bash
npm install
npm run dev
```

## Usage

### 1. Define a Catalog

Create a catalog of allowed SVG elements using Zod schemas. The Catalog dictates the `tag` name or custom `render` function.

```typescript
import { z } from "zod";
import { defineSVGSchema, defineSVGCatalog } from "./core/schema";

const schema = defineSVGSchema((s) => ({
  // ... schema definition
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
      description: "A rectangle",
      hasChildren: false,
    },
    // You can even define custom renderers!
    Avatar: {
      props: z.object({ src: z.string(), size: z.number() }),
      description: "User Avatar",
      render: (props) =>
        `<g><circle r="${props.size / 2}"/><image href="${props.src}"/></g>`,
    },
  },
});
```

### 2. Stream AI Responses with `useSVGStream`

We provide a React hook to manage the streaming state easily.

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
      <button onClick={() => generate("Draw a red circle")}>Generate</button>
      <SVGRenderer spec={spec} catalog={myCatalog} />
    </div>
  );
}
```

## Advanced Features

### JSON Patch (RFC 6902)

The AI outputs standard JSON Patch operations (`add`, `replace`, `remove`), allowing it to not only build the SVG but also modify existing elements on the fly.

### `<defs>` Support & ID Prefixing

Elements marked with `isDef: true` in the Catalog (like `LinearGradient`, `Filter`) are automatically collected and rendered inside a `<defs>` block at the top of the SVG. The `SVGRenderer` automatically prefixes IDs to prevent collisions when multiple SVGs are rendered on the same page.

## Supported AI Providers

GenSVG supports multiple AI providers using the [Vercel AI SDK](https://sdk.vercel.ai/docs). You can configure them via environment variables:

- **Gemini** (Default): Uses `@ai-sdk/google`
- **Doubao / OpenAI**: Uses `@ai-sdk/openai` compatible endpoint

Set `VITE_AI_PROVIDER="doubao"` or `VITE_AI_PROVIDER="openai"` in your `.env` to switch providers.

## Supported Elements

The default catalog supports a wide range of SVG elements:

- **Basic Shapes**: `Rect`, `Circle`, `Ellipse`, `Line`
- **Paths & Polygons**: `Path`, `Polyline`, `Polygon`
- **Text**: `Text`
- **Containers**: `Group`
- **Definitions**: `LinearGradient`, `RadialGradient`, `Stop`, `Filter`, `FeGaussianBlur`, `FeDropShadow`

## Next Steps (Roadmap: The "To-Agent" Vision)

Based on the philosophy that the future of software is **"To-Agent"** rather than "To-Human", the evolution of this project focuses on autonomous generation, self-healing, and programmatic integration. Manual human operations (like visual layer tweaking or manual code export) are deprioritized in favor of empowering AI agents to autonomously generate, iterate, and maintain visual components.

- **Phase 1: Agentic Iteration (Incremental Modification)**
  - **Targeted JSONL Patches**: Move beyond one-shot generation. Agents can output targeted JSON patches to modify existing states (e.g., an agent deciding to "change the left eye to blue" without regenerating the whole SVG).
  - **State Awareness**: Feed the current JSON state back to the agent so it can reason about spatial relationships and make precise adjustments.

- **Phase 2: Dynamic Data Binding & Animation**
  - **Props Injection**: Allow the SVG spec to accept external data (e.g., `data={{ progress: 75 }}`) and bind it to SVG attributes, turning the output into reusable, data-driven components.
  - **Motion Integration**: Support `<animate>`, `<animateTransform>`, or seamless integration with Framer Motion for AI-choreographed animations.

- **Phase 3: Autonomous Self-Healing**
  - **LLM Feedback Loop**: If Zod validation fails during streaming, automatically catch the error, feed it back to the agent, and prompt it to fix the specific attribute in the background without human intervention.
  - **Robust Partial Rendering**: Enhanced AST recovery for broken JSON patches during network interruptions.

- **Phase 4: Domain-Specific Catalogs (Agent Toolkits)**
  - **`@svg-render/charts`**: Pre-built catalog for data visualization. Agents output raw data, and the Catalog handles the D3/SVG math.
  - **`@svg-render/diagrams`**: Nodes, edges, and flowcharts optimized for agentic architecture generation.

- **Phase 5: Headless & CLI Integration (CI/CD for Agents)**
  - **CLI Tool**: Generate SVGs at build time via terminal (`npx svg-render "a red circle"`). This allows agents to generate assets directly within CI/CD pipelines, local file systems, or automated scripts.
  - **Agent API**: Expose the rendering engine as a headless service that other autonomous agents can call to generate visual assets on the fly.

## Contributing

We welcome all forms of contributions! Here's how to get involved:

### Development Workflow

1. **Fork this repository** to your GitHub account
2. **Clone your fork** locally:
   ```bash
   git clone https://github.com/your-username/gensvg.git
   cd gensvg
   ```
3. **Create a feature branch** (based on `dev`):
   ```bash
   git checkout -b feature/your-feature-name
   ```
4. **Install dependencies and start developing**:
   ```bash
   npm install
   npm run dev
   ```
5. **Commit your changes** (semantic commit messages recommended):
   ```bash
   git commit -m "feat: add some feature"
   ```
6. **Push to your fork**:
   ```bash
   git push origin feature/your-feature-name
   ```
7. **Create a Pull Request** to the `dev` branch of this repository

### Branch Naming

| Branch | Description |
|--------|-------------|
| `main` | Main branch, stable production code |
| `dev` | Development branch, work-in-progress code |
| `feature/*` | Feature development branches |
| `fix/*` | Bug fix branches |

Thank you for contributing!
