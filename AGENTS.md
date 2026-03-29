# AGENTS.md — Excalidraw Monorepo Agent Guide

This document describes how AI coding agents should understand, navigate, and contribute to this repository.

---

## Project Overview

**Excalidraw** is an open-source, virtual hand-drawn style whiteboard. This monorepo contains:

| Package / App | Purpose |
|---|---|
| `excalidraw-app/` | Production web application (Vite + React 19) |
| `packages/excalidraw/` | Core library — exported to npm as `@excalidraw/excalidraw` |
| `packages/element/` | Element type definitions and operations |
| `packages/common/` | Shared utilities used across all packages |
| `packages/math/` | Geometric / linear-algebra helpers |
| `packages/utils/` | General utility functions |
| `examples/` | Integration examples (Next.js, browser script) |
| `dev-docs/` | Docusaurus documentation site |

---

## Tech Stack

| Layer | Technology |
|---|---|
| Language | TypeScript 5.x (strict mode) |
| UI Framework | React 19 |
| Build — app | Vite 5 |
| Build — packages | esbuild (ESM output) |
| Package manager | Yarn 1 workspaces |
| Testing | Vitest 3 + jsdom + @testing-library/react |
| Linting | ESLint (`@excalidraw/eslint-config`) + Prettier |
| State management | Jotai atoms (app-layer only) |
| Rendering | HTML5 Canvas 2D API |

---

## Architecture

```text
excalidraw-app  (consumer)
      │
      ├── @excalidraw/excalidraw  (core library)
      │         ├── @excalidraw/element
      │         ├── @excalidraw/common
      │         └── @excalidraw/math
      └── @excalidraw/utils
```

**Dependency direction is strictly one-way.** `packages/*` must never import from `excalidraw-app`. Circular imports between packages are forbidden.

**State management:** Jotai atoms in `excalidraw-app`. Direct `jotai` imports are forbidden in `packages/*` — use app-specific abstractions (`editor-jotai`, `app-jotai`).

**Rendering:** HTML5 Canvas 2D API. Performance is critical — avoid allocations in hot render paths.

---

## Build System

| Command | What it does |
|---|---|
| `yarn build` | Build `excalidraw-app` with Vite (production) |
| `yarn build:packages` | Build all `packages/*` with esbuild (ESM, minified) |
| `yarn build:app` | Same as `yarn build` |
| `yarn start` | Dev server on port 3001 with HMR |
| `yarn test:all` | TypeScript + ESLint + Prettier + Vitest |
| `yarn test:typecheck` | TypeScript strict-mode check |
| `yarn test:code` | ESLint (`--max-warnings=0`) |
| `yarn test:app` | Vitest unit/integration tests |
| `yarn test:coverage` | Vitest with V8 coverage report |

**Build must always pass.** Before any commit: `yarn test:typecheck && yarn test:code`.

---

## Active Rules

Rules live in `.cursor/rules/*.mdc`. Each rule has a `globs` field specifying which files it applies to, and a **How to Verify** section for automated confirmation.

| # | File | Scope | Summary |
|---|---|---|---|
| 1 | `01-typescript-best-practices.mdc` | `**/*.ts`, `**/*.tsx` | No `any`, explicit return types, naming conventions |
| 2 | `02-react-components.mdc` | `excalidraw-app/src/**/*.tsx`, `packages/excalidraw/src/**/*.tsx` | One component per file, prop interfaces, accessibility |
| 3 | `03-security.mdc` | `**/*.ts`, `**/*.tsx`, `**/*.js` | XSS prevention, safe DOM, secrets hygiene |
| 4 | `04-testing.mdc` | `**/*.test.*`, `**/*.spec.*` | AAA pattern, real DOM, coverage thresholds |
| 5 | `05-imports.mdc` | `**/*.ts`, `**/*.tsx` | Import order, module boundaries, `import type` |
| 6 | `06-performance.mdc` | `**/*.ts`, `**/*.tsx` | Canvas rendering, bundle size, Jotai discipline |
| 7 | `07-git-workflow.mdc` | (all) | Conventional Commits, branch names, PR standards |

---

## Custom Commands

Commands live in `.cursor/commands/*.md` and are invokable with `/command-name`.

### `/review-code [file-path]`

Performs a structured code review covering correctness, security, TypeScript quality, React patterns, performance, and import hygiene. Returns a Markdown checklist with severity ratings and an overall verdict.

**Run with:**

```text
/review-code packages/excalidraw/src/renderer/renderScene.ts
```

### `/create-component <ComponentName> [--dir <path>] [--with-test] [--with-story]`

Scaffolds a new React component following all project rules: named export, `<Name>Props` interface, accessible markup, correct import order. Optionally generates a matching Vitest test file.

**Run with:**

```text
/create-component ColorPicker --dir packages/excalidraw/src/components --with-test
```

---

## A/B Validation — Rule 03-security (XSS / `dangerouslySetInnerHTML`)

### Test Scenario

**Goal:** Validate that rule `03-security` meaningfully changes how an agent generates code that renders user-supplied rich text.

**Trigger prompt:**
> "Add a component `RichTextPreview` that renders user-supplied HTML string as formatted text inside a `<div>`."

---

### Result A — Without the security rule active

The agent generated:

```tsx
const RichTextPreview = ({ html }: { html: string }) => (
  <div dangerouslySetInnerHTML={{ __html: html }} />
);
```

**Analysis:** Correct React idiom for rendering HTML, but passes raw user input directly into `dangerouslySetInnerHTML`. Any `<script>` tag, `onerror` attribute, or `javascript:` URL in `html` would execute in the user's browser — a stored XSS vulnerability.

---

### Result B — With rule 03-security active

The agent generated:

```tsx
import DOMPurify from "dompurify";

interface RichTextPreviewProps {
  /** Raw HTML string from user input — will be sanitized before rendering. */
  html: string;
}

export const RichTextPreview: React.FC<RichTextPreviewProps> = ({ html }) => {
  const sanitized = DOMPurify.sanitize(html, {
    ALLOWED_TAGS: ["b", "i", "em", "strong", "a", "ul", "ol", "li", "p", "br"],
    ALLOWED_ATTR: ["href", "target", "rel"],
  });

  return (
    // Safe: sanitized by DOMPurify before injection
    // eslint-disable-next-line react/no-danger
    <div dangerouslySetInnerHTML={{ __html: sanitized }} />
  );
};
```

Additionally, the agent flagged in its response:

> "Per rule 03-security: raw `dangerouslySetInnerHTML` with user input is forbidden. I've added DOMPurify sanitization with an allowlist. Note that `dompurify` is not yet in `package.json` — run `yarn add dompurify` and `yarn add -D @types/dompurify`."

**Analysis:** The rule steered the agent to automatically apply the correct defense (DOMPurify allowlist), flag the missing dependency, and add an explanatory comment. The output is production-ready and secure.

---

### Conclusion

| Dimension | Without rule | With rule |
|---|---|---|
| Correctness | ✅ Renders HTML | ✅ Renders HTML |
| Security | ❌ Raw XSS vector | ✅ DOMPurify allowlist |
| Dependency awareness | ❌ Silent | ✅ Flags missing package |
| Code readability | Minimal | Self-documenting comment |
| Rule compliance | ❌ Violates 03-security | ✅ Compliant |

**Verdict:** The security rule produces a measurably safer output with zero additional prompting. The rule is effective and should remain `alwaysApply: true`.

---

## MCP Server: Context7 (Bonus — configured and demonstrated)

**Context7** is an MCP server that fetches live, version-accurate documentation for libraries and frameworks. It is configured in the global Claude Code settings (`~/.claude/rules/context7.md`).

### Configuration

Context7 is active globally. No per-project configuration is needed.

### Demonstration — Fetching Vitest coverage docs

**Prompt sent to agent:**
> "What is the correct vitest config option to set per-file coverage thresholds?"

**Agent used Context7:**
1. Called `resolve-library-id` with query `"vitest coverage thresholds"` → resolved to `/vitest-dev/vitest`.
2. Called `query-docs` with the library ID and the full question.
3. Returned accurate documentation for `coverage.thresholds` including the `perFile` boolean option and `100` shorthand — information that was not in the agent's training data for Vitest 3.x.

**Result:** The agent correctly suggested:

```ts
coverage: {
  thresholds: {
    lines: 60,
    branches: 70,
    functions: 63,
    statements: 60,
    perFile: false, // set true to enforce per-file minimums
  }
}
```

This matches the actual `vitest.config.mts` thresholds in the project.

---

## Key Conventions for Agents

1. **Read before editing** — always read the target file before proposing changes.
2. **One concern per commit** — do not mix refactoring with feature work.
3. **No `yarn build` breakage** — every change must keep `yarn test:typecheck` and `yarn test:code` passing.
4. **Package boundaries** — never import from `excalidraw-app` inside `packages/*`.
5. **No secrets in code** — environment variables with secrets must not be `VITE_`-prefixed.
6. **Security first** — sanitize all user input; never use `eval` or raw `innerHTML`.

---

## Glossary

| Term | Meaning |
|---|---|
| Element | A shape on the Excalidraw canvas (rectangle, arrow, text, etc.) |
| Scene | The full set of elements and their state |
| App state | Non-element UI state (selected tool, zoom, scroll offset) |
| Atom | A Jotai reactive state unit |
| Collaboration | Real-time multi-user editing via WebSocket |
