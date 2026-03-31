# GSoC Proposal: A Hyperjump-Based Replacement for `vscode-json-languageservice`

---

## Personal Info

- Hrishikesh Sonawane
- thehrishikeshsonawane@gmail.com
- github.com/hrishi2814
- richiekesh.cv --- you can checkout some of my side projects here :)
- Timezone: IST (UTC+5:30)
- I am currently a Computer Engineering senior at Pune University, Pune , India and will graduate at around June 2026
---

## Project View & Implementation Plan

### The Problem

`vscode-json-languageservice` is the backbone of JSON editing across virtually every major editor and IDE. Its reach extends far beyond VS Code itself which makes its stagnation a community-wide bottleneck. The problem isn't just missing keywords: the internal validator is tightly coupled spaghetti that makes adding proper `$dynamicRef`, bundled schema, and vocabulary support extremely difficult. Contributing upstream is not a viable path forward.

### The Reference: `vscode-json-languageservice` Capabilities

To achieve true parity, the replacement must replicate the following core capabilities found in the official Microsoft service:

*   **Smart Validation**: Syntax checking (JSONC support) and recursive schema validation with a "Best Match" scoring algorithm to reduce noise in `oneOf`/`anyOf` branches.
*   **Contextual Completion**: IntelliSense for property keys and values, including support for `defaultSnippets`, `enumDescriptions`, and heuristic suggestions when no schema is present.
*   **Rich Documentation**: Schema-driven Hover support and Completion details, with a strong preference for `markdownDescription` over plain text.
*   **Navigation & Structure**: Document Symbols (outline view), Folding Ranges, and Document Links (making `$ref` values clickable).
*   **Proprietary Extensions**: Full support for VS Code-specific keywords such as `errorMessage`, `deprecationMessage`, `doNotSuggest`, and parser-level overrides like `allowComments`.

---

The right approach is to treat `@hyperjump/json-schema` as the authoritative validation and annotation engine, it already handles draft-04 through 2020-12 correctly and build a clean LSP feature layer on top of it. This project is about that feature layer.

As prior exploration, I implemented a language server for the JSON Reference (JRef) format, covering syntax validation, semantic `$ref` resolution, and go-to-definition with JSON Pointer deep linking. That work informed the architecture described below.

---

### Drop-in Replacement: LSP vs API Parity

I believe the primary goal is **LSP feature parity** , ensuring the server behaves identically to the official service from an editor's perspective. While programmatic API parity (matching the `LanguageService` TypeScript interface) is a logical follow-on, I will focus on the LSP wire behavior first to provide immediate value to the broader ecosystem.

---

### Architecture

The service is built as three cooperating layers:

```
┌─────────────────────────────────────────┐
│           LSP Transport Layer           │  server.ts — connection, lifecycle
├─────────────────────────────────────────┤
│           Feature Providers             │  validation, completion, hover,
│                                         │  symbols, folding, links
├──────────────────┬──────────────────────┤
│  Document Store  │   Schema Service     │  jsonc-parser AST + hyperjump
│  (jsonc-parser)  │   (@hyperjump)       │  schema registry + cache
└──────────────────┴──────────────────────┘
```

**Document Store** — wraps `jsonc-parser` to produce a comment-aware, error-tolerant AST. Maintains an in-memory map of open documents. Critical for "healing" broken JSON during mid-edit completion.

**Schema Service** — manages schema registration, fetching, and caching. Uses `@hyperjump/json-schema`'s `registerSchema` and `getSchema` APIs. Handles schema association and `folderUri` workspace scoping.

**Feature Providers** — standalone modules consuming the AST and Hyperjump annotations.

---

### Schema Association Logic

The governing schema is selected via a strict hierarchy:
1. **Inline `$schema`** in the document root.
2. **Workspace configuration** (`json.schemas`) with glob and `folderUri` matching.
3. **Built-in associations** (e.g., `package.json`, `tsconfig.json`).
4. **SchemaStore.org** lookup via filename.
5. **Heuristic Fallback**: If no schema is found, suggestions are derived from document-wide sibling keys/values (Statistician pattern).

---

### Completion Engine: The "Context-Inferrer" Strategy

The core challenge is resolving schema annotations for **invalid JSON** (e.g., `{"foo": | }`).

1. **Path-Healing**: When `jsonc-parser` detects a syntax error at the cursor, I will implement a "Context-Inferrer" that walks the partial AST to construct a valid JSON Pointer path.
2. **Annotation Traversal**: Using Hyperjump's annotation API, I will query the schema at that inferred path.
3. **Draft-Agnosticism**: By using annotations, I abstract away the keyword differences between Draft 07 (`definitions`) and 2020-12 (`$defs`).
4. **Branch Merging**: Active `anyOf`/`oneOf` branches are merged to union their `properties`, `required`, and `enum` suggestions.

---

### Best-Match Scoring & Discriminators

To eliminate diagnostic noise in complex schemas, I will implement a scoring utility that wraps Hyperjump's `DETAILED` output:

| Signal | Weight |
| :--- | :--- |
| **Discriminator Match** | High (Matches a property like `kind` or `type` to an enum) |
| **Const/Enum Match** | High (Exact value match) |
| **Required Property** | Medium (Property key presence) |
| **Valid Child** | Low (Sub-node validates against schema) |

Only the highest-scoring branch's errors are reported to the user.

---

### Custom VS Code Keywords

VS Code's proprietary extensions will be implemented using Hyperjump's `addKeyword` API to integrate seamlessly into the annotation pipeline.

| Keyword | Impact |
| :--- | :--- |
| `defaultSnippets` | Completion items injected with snippet syntax and tab stops. |
| `markdownDescription`| Preferred over `description` for Hover/Completion UI. |
| `errorMessage` | Overrides generic validation messages at specific nodes. |
| `allowComments` | Configures the **Document Store** parser to skip syntax errors for comments. |
| `allowTrailingCommas`| Configures the parser to ignore trailing commas in objects/arrays. |
| `deprecationMessage` | Emits warnings with strikethrough hints in the editor. |

---

### Testing Strategy: The "Oracle" Approach

**Ground Truth**: `vscode-json-languageservice` is the oracle. The test suite will execute both implementations against the same inputs, asserting identical outputs for completions, diagnostics, and hover markdown.

**Matrix Coverage**:
*   **Drafts**: 04, 07, 2019-09, 2020-12.
*   **Features**: Validation, Completion (Key/Value/Snippet), Hover, Document Symbols, Folding Ranges, `$ref` Links.
*   **Advanced**: `$dynamicRef`, Bundled Schemas, `unevaluatedProperties`.

---

### Phase Plan

**Phase 1 — Core (Weeks 1–4)**
*   LSP scaffold + Document Store (jsonc-parser integration).
*   Schema Service with `$schema` detection and glob association.
*   Basic validation via Hyperjump → LSP Diagnostics.
*   Document Symbols & Folding Ranges.

**Phase 2 — Features (Weeks 5–9)**
*   Hover Provider (Title/Description extraction).
*   Completion Engine with "Path-Healing" for broken JSON.
*   Best-Match Scoring algorithm with Discriminator support.
*   Implementation of proprietary VS Code keywords via `addKeyword`.

**Phase 3 — Polish & Parity (Weeks 10–13)**
*   Oracle test suite pass (Parity validation against Microsoft service).
*   Support for `$dynamicRef` and Bundled Schemas.
*   Performance optimization: Debounced validation and annotation caching.
*   Final Parity Report.

---

### What I'm Uncertain About

1. **Annotation Granularity**: Verifying if the current Hyperjump annotation API can handle the high-frequency, partial-path lookups required for <50ms completion responses without full document re-validation.
2. **Extensibility Bridge**: Whether to replicate `JSONWorkerContribution` exactly or provide a modern alternative with a compatibility shim for existing ecosystem contributions.

---

## Final Deliverables:

- A working project that gets the core features right, with proper test suites for comprehensive testing.

## Why me

- I think I am a good fit for this project as I am decently proficient with typescript, I have solid fullstack development experience with typescript, with my recent full time contract developer work at a web dev agency , and as well as the fact, that I just possess massive interest in the project itself, this is a level of work that I think I can learn from extremely well and with the added guidance from the mentors I will be able to take my skills to a new level. 

- I would like to see this project through till the end even if it means we will be working on this for a year after the gsoc period.

## Extras

- I have intentionally kept this proposal focused on high-level architecture and LSP wire behavior. I want to remain agile and avoid over-prescribing granular API details until I can collaborate directly with the mentors to align with their long-term vision for the @hyperjump ecosystem

## Scheduling

- In the initial period i.e, May , I might only be able to give about 15-20hrs/week as my End Semester Exams will most probably be starting then.
- I can then consistently work more than 30+hrs/week till the end of the gsoc period.

- I am available throught 10 am to 8pm IST for any and all meetings throughout the week.
