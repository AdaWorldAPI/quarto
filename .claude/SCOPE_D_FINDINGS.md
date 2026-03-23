# Scope D Findings: Rendering Pipeline & Pandoc Integration

> Analysis of quarto (TS) and quarto-r repos for Rust transcoding of the publisher.

---

## 1. Rendering Pipeline Stages

The rendering pipeline from notebook to PDF/HTML has these stages:

```
 .qmd / .ipynb / .Rmd
         |
         v
 [1] Cell Detection & Parsing
     (quarto-core/src/markdown/language.ts)
     Identify executable language blocks via regex:
       /^\{=?([a-zA-Z0-9_-]+)(?: *[ ,].*?)?/
         |
         v
 [2] Notebook Parsing (for .ipynb)
     (core/src/jupyter/notebook.ts)
     JSON parse -> JupyterNotebook type, extract kernelspec language
         |
         v
 [3] Cell Execution (knitr for R, jupyter for Python)
     Engine chosen based on language blocks in frontmatter/document
     Produces markdown with embedded outputs
         |
         v
 [4] Markdown Preprocessing
     (editor/src/pandoc/pandoc_converter.ts lines 123-131)
     - Normalize newlines
     - Run preprocessor chain
     - Create block capsules (protect raw content through pandoc)
         |
         v
 [5] Pandoc: Markdown -> JSON AST
     (editor-server/src/server/pandoc.ts lines 82-108)
     CLI call: pandoc --from <format> --to json [--lua-filter ...]
         |
         v
 [6] Quarto Lua Filters (cross-refs, shortcodes, etc.)
     Applied via --lua-filter flags during pandoc invocation
         |
         v
 [7] Pandoc: JSON AST -> Output Format
     (editor-server/src/server/pandoc.ts lines 109-116)
     CLI call: pandoc --from json --to <format> [options]
         |
         v
 [8] Post-processing (markdown post-processors)
         |
         v
 PDF / HTML / DOCX / etc.
```

**Key insight**: The TS repo is the *visual editor and IDE tooling* (Panmirror, VS Code extension, LSP). The actual CLI render pipeline lives in the Deno-based `quarto-cli` repo. This repo handles the editor's bidirectional markdown-to-prosemirror conversion, which shares the same pandoc invocation pattern.

---

## 2. Pandoc AST Format

The JSON AST structure is defined in `packages/editor-types/src/pandoc.ts`:

```typescript
export interface PandocAst {
  blocks: PandocToken[];
  'pandoc-api-version': PandocApiVersion;  // e.g. [1, 23, 1]
  meta: Record<string, unknown>;
  heading_ids?: string[];  // used only for reading, not writing
}

export interface PandocToken {
  t: string;   // token type name
  c?: any;     // token content (varies by type)
}

export interface PandocAttr {
  id: string;
  classes: string[];
  keyvalue: Array<[string, string]>;
}
```

### Key Node Types (PandocTokenType enum)

From `packages/editor/src/api/pandoc.ts`:

| Category | Types |
|----------|-------|
| **Inline** | `Str`, `Space`, `Strong`, `Emph`, `Code`, `Superscript`, `Subscript`, `Strikeout`, `SmallCaps`, `Underline`, `Quoted`, `RawInline`, `Math`, `InlineMath`, `DisplayMath`, `Cite`, `Link`, `Image`, `Note`, `Span`, `LineBreak`, `SoftBreak` |
| **Block** | `Para`, `Plain`, `Header`, `CodeBlock`, `BlockQuote`, `BulletList`, `OrderedList`, `DefinitionList`, `Table`, `Figure`, `RawBlock`, `LineBlock`, `HorizontalRule`, `Div`, `Null` |
| **Table alignment** | `AlignRight`, `AlignLeft`, `AlignDefault`, `AlignCenter`, `ColWidth`, `ColWidthDefault` |

### Example AST JSON

```json
{
  "pandoc-api-version": [1, 23, 1],
  "meta": {},
  "blocks": [
    { "t": "Header", "c": [1, ["my-heading", [], []], [{"t": "Str", "c": "Hello"}]] },
    { "t": "Para", "c": [{"t": "Str", "c": "World"}] },
    { "t": "CodeBlock", "c": [["", ["python"], []], "print('hello')"]}
  ]
}
```

The `c` field structure varies by type:
- **Header**: `[level, [id, classes, keyvals], [inline_content...]]`
- **CodeBlock**: `[[id, classes, keyvals], code_text]`
- **Link**: `[[id, classes, keyvals], [inline_content...], [url, title]]`
- **Image**: same structure as Link

---

## 3. Quarto Additions on Top of Pandoc

### 3a. Cell Execution

From `packages/quarto-core/src/markdown/language.ts`:

```typescript
// Detects executable language blocks: ```{python}, ```{r}, etc.
export function isExecutableLanguageBlock(token: Token): token is TokenMath | TokenCodeBlock {
  if (isDisplayMath(token)) {
    return true;
  } else if (isCodeBlock(token) && token.attr?.[kAttrClasses].length) {
    const clz = token.attr?.[kAttrClasses][0];
    return !!clz.match(/^\{=?([a-zA-Z0-9_-]+)(?: *[ ,].*?)?/);
  }
  return false;
}
```

Language name extraction strips the `{` prefix and `-` engine prefixes:

```typescript
export function languageNameFromBlock(token: Token) {
  // ... match /^\{?=?([a-zA-Z0-9_-]+)/ then split("-").pop()
}
```

### 3b. Notebook Format (.ipynb) Parsing

From `packages/core/src/jupyter/notebook.ts`:

```typescript
export function jupyterFromJSON(nbContents: string): JupyterNotebook {
  const nbJSON = JSON.parse(nbContents);
  const nb = nbJSON as JupyterNotebook;
  // Fallback logic for missing kernelspec:
  // 1. Try language_info.name
  // 2. If name includes "python", default to python
  // 3. Default to python3 kernelspec
  return nb;
}
```

### 3c. Preprocessing & Block Capsules

The converter (`pandoc_converter.ts`) runs a chain:
1. **Preprocessors** - text transforms before pandoc
2. **Block capsule filters** - encapsulate raw content (e.g., YAML, shortcodes) into pandoc-safe wrappers so they survive the markdown-to-AST round-trip
3. **Token filters** - post-parse AST token manipulation
4. **Postprocessors** - transform the resulting prosemirror doc

### 3d. Cross-references

The `crossref.ts` in `editor-types` is specifically for Crossref.org DOI lookup (bibliography), not Quarto's internal cross-reference system (@fig-, @tbl-, @sec-). Quarto's internal cross-ref system is implemented as Lua filters in the CLI, not in this TS codebase.

### 3e. Format Adjustment

Quarto always forces certain pandoc extensions on:
```typescript
// Always enabled for reading:
['raw_html', 'raw_attribute', 'backtick_code_blocks', autoIds,
 'grid_tables', 'pipe_tables', 'multiline_tables', 'simple_tables',
 'tex_math_dollars']

// Always disabled:
['smart']  // for reading
['auto_identifiers', 'gfm_auto_identifiers', 'smart']  // for writing
```

---

## 4. Extension System

### 4a. Editor Extensions (this repo)

The visual editor uses a plugin-based extension system defined in `packages/editor/src/api/extension-types.ts`:

```typescript
export interface Extension {
  marks?: PandocMark[];
  nodes?: PandocNode[];
  baseKeys?: (schema: Schema) => readonly BaseKeyBinding[];
  inputRules?: (schema: Schema, markFilter: MarkInputRuleFilter) => readonly InputRule[];
  commands?: (schema: Schema) => readonly ProsemirrorCommand[];
  plugins?: (schema: Schema) => readonly Plugin[];
  // ... more hooks
}

export type ExtensionFn = (context: ExtensionContext) => Extension | null;
```

Each extension can provide:
- **PandocTokenReaders**: Map pandoc AST tokens to prosemirror nodes
- **PandocNodeWriters**: Map prosemirror nodes back to pandoc AST
- **PandocMarkWriters**: Handle inline marks (bold, italic, etc.)
- **Preprocessors/Postprocessors**: Transform markdown text or doc tree

### 4b. Lua Filters (Quarto CLI)

Pandoc Lua filters are the primary extension mechanism for the render pipeline. They are invoked via `--lua-filter` flags:

```typescript
// heading-ids.lua - extract explicit heading IDs
"--lua-filter", path.join(options.pandoc.resourcesDir, 'heading-ids.lua')

// parser.lua - tokenize for IDE support
"--lua-filter", path.join(resourcePath, 'parser.lua')

// sourcepos.lua - source position tracking
"--lua-filter", path.join(pandoc.resourcesDir, 'sourcepos.lua')

// md-writer.lua - custom markdown writer (pandoc 3.2+ compat)
substituteCustomMarkdownWriter(format)  // replaces "markdown" with md-writer.lua path
```

### 4c. Quarto CLI Extension System (from _quarto-cli_ repo, not this one)

Quarto extensions use `_extensions/<name>/_extension.yml` manifests. They can provide:
- **Shortcode handlers** (Lua)
- **Filters** (Lua filters applied during render)
- **Formats** (custom output formats built on existing ones)
- **Revealjs plugins**

This is all Lua-based and runs inside pandoc's Lua interpreter. **For a Rust transcoding, these Lua filters would still need to be executed by pandoc** -- they are not something the publisher itself implements.

---

## 5. quarto-r Interface

### How R calls the CLI

From `quarto-r/R/quarto.R`:

```r
# Find binary via $QUARTO_PATH or Sys.which("quarto")
quarto_path <- function(normalize = TRUE) {
  path_env <- get_quarto_path_env()
  quarto_path <- if (is.na(path_env)) {
    path <- unname(Sys.which("quarto"))
    if (nzchar(path)) path else return(NULL)
  } else {
    path_env
  }
  xfun::normalize_path(quarto_path)
}

# Execute quarto CLI via processx::run()
quarto_run <- function(args = character(), quarto_bin = find_quarto(), ...) {
  res <- processx::run(
    quarto_bin,
    args = args,
    echo = echo,
    error_on_status = TRUE,
    env = custom_env,
    ...
  )
  invisible(res)
}
```

From `quarto-r/R/render.R`, the render function simply builds CLI arguments and calls `quarto_run`:

```r
quarto_render <- function(input = NULL, output_format = NULL, ...) {
  quarto_bin <- find_quarto()
  args <- c("render", input)
  if (!missing(output_format)) {
    args <- c(args, "--to", paste(output_format, collapse = ","))
  }
  # ... build more args for execute, cache, metadata, etc.
  quarto_run(args, echo = TRUE, quarto_bin = quarto_bin, libpaths = libpaths)
}
```

### What Changes If the CLI Is a Rust Binary?

**Almost certainly nothing.** The R package interacts with quarto exclusively through:

1. **`processx::run(quarto_bin, args)`** - spawns the binary as a subprocess
2. **`system2(quarto_bin, "--version", stdout = TRUE)`** - for version checks
3. **`Sys.which("quarto")` / `$QUARTO_PATH`** - for binary discovery

The interface is purely:
- Find a binary path
- Pass CLI arguments (render, --to, --execute, --metadata-file, etc.)
- Read stdout/stderr

The R package does not:
- Link against any Deno/JS/TS libraries
- Parse any internal quarto data structures
- Communicate via IPC, sockets, or shared memory
- Depend on the binary being Deno-specific in any way

**The only contract is the CLI argument interface and exit codes.** As long as a Rust binary accepts the same `quarto render <input> --to <format> [options]` interface and produces the same stdout/stderr output, `quarto-r` will work unchanged.

Environment variables passed by quarto-r that the Rust binary must honor:
- `QUARTO_R` - path to R binary (for knitr engine)
- `R_LIBS` - R library paths for subprocess

---

## Summary for Rust Transcoding

| Component | Rust Impact |
|-----------|-------------|
| **Pandoc invocation** | Shell out to pandoc binary, pipe stdin/stdout. Same as current `child_process.execFile()`. |
| **JSON AST** | Deserialize with serde. Well-defined schema: `{ blocks, pandoc-api-version, meta }`. |
| **Lua filters** | Passed through to pandoc via `--lua-filter`. No need to reimplement. |
| **Notebook parsing** | Straightforward JSON deserialization of `.ipynb` format. |
| **Cell detection** | Regex-based, trivially portable. |
| **quarto-r** | Zero changes needed. It calls a binary by path. |
| **Extension system** | Extensions are Lua scripts read by pandoc. Rust just needs to pass `--lua-filter` paths. |
| **Block capsules** | String manipulation to protect content through pandoc. Reimplement in Rust. |
| **Editor extensions** | Prosemirror-specific (TS only). Not relevant to CLI render pipeline. |
