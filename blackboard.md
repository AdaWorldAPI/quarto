# Blackboard — quarto

> Single-binary architecture: Quarto's publishing pipeline transcoded to Rust.

## What Exists

This is the **editor/tooling layer** for Quarto (not the main CLI). TypeScript codebase providing:
- ProseMirror-based visual editor
- Pandoc bidirectional conversion (Markdown ↔ ProseMirror AST)
- Language Server Protocol implementation
- VS Code extension

## Rendering Pipeline

```
Markdown → Pandoc (external process) → JSON AST → ProseMirror Document → Output
```

### Key Conversion Files

- `packages/editor/src/pandoc/pandoc_converter.ts` — orchestrator
  - `toProsemirror()`: Markdown → ProseMirror
  - `fromProsemirror()`: ProseMirror → Markdown
- `packages/editor-server/src/core/pandoc.ts` — spawns pandoc via `child_process.execFile()`
- `packages/editor-server/src/server/pandoc.ts` — JSON-RPC: `markdownToAst()`, `astToMarkdown()`

### Cell/Chunk Detection

- `packages/quarto-core/src/markdown/language.ts` — `isExecutableLanguageBlock()`
- Pattern: `` ```{language options} ... ``` ``
- Languages: Python, R, Julia, Bash, SQL, Mermaid, GraphViz, D3, Stan

### Jupyter Notebook Support

- `packages/core/src/jupyter/notebook.ts` — parses `.ipynb` JSON
- `packages/core/src/jupyter/types.ts` — `JupyterNotebook`, `JupyterCell`, `JupyterOutput`

## Extension System

ProseMirror-based plugin architecture:
- `packages/editor/src/api/extension.ts` — Extension type with nodes, marks, commands, plugins
- `packages/editor/src/editor/editor-extensions.ts` — registry (400+ lines)
- Each extension defines Pandoc readers/writers for roundtrip fidelity

## What Gets Transcoded to Rust

| TS Component | Rust Replacement |
|---|---|
| Pandoc process spawning | Embedded pandoc or `tectonic` for PDF |
| JSON AST manipulation | `pandoc_ast.rs` Rust types |
| Markdown parser | `pulldown-cmark` or Pandoc subprocess |
| Cell detection | Pattern matching on code fences |
| Extension system | Compile-time extension registration |

## Key Files

| File | Purpose |
|---|---|
| `packages/editor/src/pandoc/pandoc_converter.ts` | Main pipeline |
| `packages/editor-server/src/core/pandoc.ts` | Pandoc process exec |
| `packages/editor-server/src/server/pandoc.ts` | JSON-RPC interface |
| `packages/quarto-core/src/markdown/language.ts` | Cell detection |
| `packages/core/src/jupyter/notebook.ts` | .ipynb parsing |
| `packages/core/src/jupyter/types.ts` | Notebook types |
