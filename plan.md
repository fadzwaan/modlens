# Project Plan

## 1. Project Vision

Build a collection of small, composable CLI tools that provide external capabilities to LLM agents.

The project does **not** replace the underlying LLM and does **not** act as a Claude/Anthropic/OpenAI API proxy.

Instead, each `mod*` tool provides one narrowly defined capability that an Agent can invoke when needed.

Initial ecosystem:

```text
modlens    → vision / image understanding
modsearch  → web search
modfetch   → web-page fetching
```

The Agent remains responsible for reasoning, decision-making, and composing these capabilities.

```text
                         LLM / Agent
                              │
              ┌───────────────┼───────────────┐
              │               │               │
              ▼               ▼               ▼
          modlens         modsearch       modfetch
           Vision           Search          Fetch
              │               │               │
              ▼               ▼               ▼
        Image → Text     Query → URLs     URL → Text
```

---

# 2. Core Principles

## 2.1 One Skill, One Responsibility

Every module must have one clear responsibility.

* `modlens` handles visual parsing.
* `modsearch` handles web search.
* `modfetch` handles web-page retrieval.

Do not turn individual modules into autonomous research agents or all-in-one utilities.

For example:

```text
Good:

Agent
 ├── modsearch
 ├── modfetch
 └── modlens
```

Instead of:

```text
Avoid:

modresearch
 ├── search
 ├── fetch
 ├── OCR
 ├── vision
 └── summarization
```

Composition belongs to the Agent or higher-level orchestrator.

---

## 2.2 Low Coupling

The tools should not depend on a specific Agent or client protocol.

They should be usable by:

* Claude Code
* Codex
* other coding agents
* custom LLM agents
* future Agent frameworks

The CLI interface should remain independent from any specific model provider or Agent implementation.

---

## 2.3 Agent-First Interface

The tools are primarily designed to be invoked by Agents.

Therefore:

* commands should be short;
* parameters should be predictable;
* output should be machine-readable;
* failures should be explicit;
* interactive prompts should be avoided;
* unnecessary state should not be maintained.

The goal is to minimize Agent invocation complexity and the tool's failure surface.

---

# 3. Current Project: `modlens`

## 3.1 Purpose

`modlens` converts image sources into structured textual evidence for non-vision LLM workflows.

Supported image sources:

* local image paths;
* remote image URLs.

Conceptually:

```text
Image Source
     │
     ▼
  modlens
     │
     ▼
Vision Provider
     │
     ▼
Structured Evidence
     │
     ▼
Agent / LLM Context
```

The output is intended to give a text-only LLM enough information to reason about the original image.

---

# 4. `modlens` Architecture

## 4.1 Provider Abstraction

`modlens` uses a pluggable vision-provider architecture.

The CLI and orchestration layer should not be tightly coupled to a specific vision engine.

Current structure:

```text
src/
├── main.ts
├── analyzer.ts
├── prompt.ts
├── schema.ts
└── providers/
    ├── index.ts
    └── antigravity.ts
```

Responsibilities:

### `main.ts`

CLI entry point.

Responsible for:

* argument parsing;
* command execution;
* exit codes;
* user-facing errors.

### `analyzer.ts`

Orchestrates the complete analysis flow:

```text
input resolution
      ↓
provider invocation
      ↓
output parsing
      ↓
result envelope
```

It should not contain provider-specific implementation details.

### `prompt.ts`

Contains the vision extraction prompt.

The prompt should focus on producing useful evidence rather than conversational responses.

### `schema.ts`

Defines the structured output contract.

The schema is passed to the provider where supported so that the provider returns structured data instead of requiring markdown scraping.

### `providers/index.ts`

Defines the provider interface and provider registry.

The provider abstraction should isolate provider-specific behavior.

### `providers/antigravity.ts`

Implements the current Antigravity CLI (`agy`) provider.

It is responsible for:

* building the `agy` invocation;
* passing the required prompt/schema;
* executing the provider;
* parsing provider output.

---

# 5. Current Vision Provider

## 5.1 Antigravity CLI

The current default provider is:

```text
Antigravity CLI
agy
```

The provider should be treated as an implementation detail behind the `modlens` provider interface.

The architecture is:

```text
                 modlens
                    │
                    ▼
            Provider Interface
                    │
                    ▼
          Antigravity Provider
                    │
                    ▼
                   agy
                    │
                    ▼
             Vision Model
```

The default provider is therefore **Antigravity**, not Gemini CLI directly.

---

## 5.2 Model Selection

The provider can invoke a selected vision model.

Example:

```bash
modlens -i screenshot.png -m gemini-3.1-pro-high
```

This distinction should remain explicit:

```text
Provider = Antigravity CLI
Model    = selected vision model
```

The architecture should not assume that Gemini is permanently the only available model.

---

# 6. Structured Output

Structured output is a core requirement of `modlens`.

The provider should use schema enforcement where supported:

```text
Image
  ↓
Vision model
  ↓
JSON schema enforcement
  ↓
Structured evidence
```

The system should avoid relying on free-form markdown scraping to determine the result structure.

This gives Agents a predictable contract and makes the output easier to consume programmatically.

The schema should be versioned carefully because changes to the output contract can affect downstream Agent integrations.

---

# 7. CLI Interface

The CLI should remain flat and minimal.

Examples:

```bash
modlens -i screenshot.png
```

```bash
modlens -i screenshot.png -o lens.json
```

```bash
modlens -i screenshot.png \
  -o lens.json \
  -m gemini-3.1-pro-high \
  --prompt "focus on the table"
```

The CLI should not introduce unnecessary subcommands such as:

```bash
modlens image extract ...
```

The exact set of supported flags should remain small and stable.

---

# 8. Input Handling

`modlens` should support:

```text
Local path
    ↓
modlens

Remote URL
    ↓
modlens
```

Input resolution should happen before the provider is invoked.

The provider should receive a normalized input rather than needing to understand every possible input source.

---

# 9. Output Contract

The preferred output is structured JSON.

Example conceptual result:

```json
{
  "description": "...",
  "elements": [],
  "text": [],
  "observations": []
}
```

The exact fields are defined by `schema.ts` and the corresponding skill documentation.

The output should be:

* deterministic in structure;
* easy for Agents to parse;
* suitable for direct context injection;
* explicit about failures.

Human-readable output may be supported, but machine-readable output is the primary design target.

---

# 10. Agent Skill Definition

The repository contains:

```text
skills/
└── modlens/
    ├── SKILL.md
    └── references/
        └── output-schema.md
```

`SKILL.md` should explain to an Agent:

* when to use `modlens`;
* how to invoke it;
* what inputs it accepts;
* what output it produces;
* how to interpret failures.

`references/output-schema.md` should document the structured result.

The skill documentation should remain aligned with the actual CLI behavior and schema.

---

# 11. Verification

## 11.1 Unit-Level Verification

The standard verification command is:

```bash
pnpm typecheck && pnpm test
```

Tests should cover at least:

* CLI argument handling where appropriate;
* provider invocation construction;
* schema handling;
* provider output parsing;
* error conditions;
* normalized result behavior.

---

## 11.2 End-to-End Verification

Real vision-provider executions consume the user's `agy` quota and may take approximately 15–40 seconds.

Therefore, end-to-end testing should not be performed in bulk without approval.

A typical manual test is:

```bash
modlens -i screenshot.png
```

The test should confirm:

```text
input
 ↓
modlens
 ↓
agy
 ↓
vision model
 ↓
schema-valid result
```

---

# 12. Operational Documentation

Operational documentation lives under:

```text
docs/
```

Documentation uses front-matter metadata such as:

```yaml
summary:
read_when:
```

Before creating a new operational document:

```bash
pnpm docs:list
```

Review the existing documentation index first.

Before implementing a change, inspect the relevant `read_when` guidance and read the applicable documentation.

Current documentation includes:

* `commit`
* `testing`
* `research-gemini-claude-skills`

The `research-gemini-claude-skills` document is historical and describes the earlier Gemini CLI era. It should not override the current Antigravity-based implementation.

---

# 13. Dependencies and Distribution

The project uses the Node.js ecosystem.

Primary technologies:

* TypeScript
* Node.js
* pnpm
* npm-compatible package distribution

Installation:

```bash
pnpm install
```

The CLI is exposed through:

```text
dist/main.js
```

Build and package configuration should keep the executable entry point stable.

---

# 14. Repository Hygiene

`.gitignore` must include at minimum:

```text
node_modules/
dist/
skills/**/outputs/
```

It should also ignore common:

* logs;
* caches;
* temporary files;
* operating-system files.

Generated Agent outputs should not be accidentally committed.

---

# 15. Future Provider Architecture

The provider abstraction exists so that the vision engine can change without redesigning `modlens`.

Potential future providers include:

```text
Antigravity
     │
     ├── Gemini
     └── other supported models

PaddleOCR

DeepSeek OCR

Local vision models

Other multimodal providers
```

Conceptually:

```text
                    modlens
                       │
                       ▼
              Provider Interface
                       │
       ┌───────────────┼────────────────┐
       │               │                │
       ▼               ▼                ▼
 Antigravity       PaddleOCR       Future Provider
       │
       ▼
 Vision Model
```

Adding a provider should ideally require changes primarily within the provider implementation rather than the CLI, analyzer, or output contract.

---

# 16. `modsearch`

`modsearch` is a separate capability and should remain independent from `modlens`.

Its responsibility is:

> Query the web and return candidate search results.

Conceptual interface:

```text
Query
 ↓
modsearch
 ↓
Search results
 ├── title
 ├── URL
 └── snippet
```

It should not:

* fetch pages;
* summarize pages;
* perform autonomous research;
* replace the Agent's reasoning.

The exact search backend remains a separate implementation decision.

---

# 17. `modfetch`

`modfetch` is another independent capability.

Its responsibility is:

> Retrieve readable content from a URL.

Conceptual interface:

```text
URL
 ↓
modfetch
 ↓
Page retrieval
 ↓
Content extraction
 ↓
Text
```

For HTML:

```text
HTML
 ↓
Visible/main-content extraction
 ↓
Readable text
```

For other supported content:

```text
Resource
 ↓
Best-effort extraction
 ↓
Text
```

It should not become a search engine or summarization system.

---

# 18. Combined Agent Workflow

The three tools should be composable without being coupled together.

## Image workflow

```text
User
 ↓
Agent
 ↓
modlens
 ↓
structured evidence
 ↓
LLM reasoning
```

## Web workflow

```text
User
 ↓
Agent
 ↓
modsearch
 ↓
candidate URLs
 ↓
Agent selects URLs
 ↓
modfetch
 ↓
page text
 ↓
LLM reasoning
```

## Combined workflow

```text
                       Agent
                         │
          ┌──────────────┼──────────────┐
          ▼              ▼              ▼
      modlens        modsearch       modfetch
          │              │              │
          ▼              ▼              ▼
      image text      search URLs     page text
          │              │              │
          └──────────────┴──────────────┘
                         │
                         ▼
                       LLM
```

The Agent remains the orchestration layer.

---

# 19. Security and Reliability

Future implementations must account for external input and external execution.

## `modlens`

Consider:

* untrusted image inputs;
* provider command execution;
* provider failures;
* credential handling;
* quota exhaustion;
* malformed provider output.

## `modfetch`

Consider:

* malicious URLs;
* redirects;
* internal/localhost addresses;
* oversized responses;
* timeouts;
* unsupported protocols;
* resource exhaustion.

## All modules

Tools should fail explicitly and return meaningful exit codes.

The Agent must be able to distinguish:

```text
successful result
```

from:

```text
tool execution failure
```

---

# 20. Development Roadmap

## Phase 1 — Stabilize `modlens`

Current priority.

Tasks:

* stabilize CLI interface;
* stabilize provider abstraction;
* stabilize Antigravity provider;
* finalize structured output schema;
* improve input resolution;
* improve error handling;
* maintain unit tests;
* maintain Agent skill documentation;
* verify real end-to-end behavior.

Success condition:

```text
image
 ↓
modlens
 ↓
agy
 ↓
schema-valid structured evidence
```

---

## Phase 2 — Improve Provider Extensibility

Add another provider only when there is a practical need.

The goal is to prove that the provider abstraction actually isolates engine-specific code.

A provider should not require significant changes to:

* `main.ts`;
* `analyzer.ts`;
* the CLI interface;
* unrelated documentation.

---

## Phase 3 — Build `modsearch`

Create the independent search capability.

Priorities:

* simple query interface;
* structured result output;
* predictable errors;
* Agent-friendly invocation.

Do not integrate search into `modlens`.

---

## Phase 4 — Build `modfetch`

Create the independent URL-fetching capability.

Priorities:

* URL input;
* reliable retrieval;
* readable text extraction;
* structured output;
* timeout and size limits;
* predictable errors.

Do not integrate fetching into `modsearch`.

---

## Phase 5 — Agent Integration

Test the tools from real Agent workflows.

Examples:

```text
Agent → modlens
Agent → modsearch
Agent → modsearch → modfetch
```

The purpose is to validate the ecosystem from the Agent's perspective rather than merely testing each CLI in isolation.

---

# 21. Explicit Non-Goals

The project is not intended to become:

* a Claude proxy;
* an Anthropic API replacement;
* an LLM hosting platform;
* a general Agent framework;
* an autonomous research agent;
* an all-in-one AI CLI;
* a replacement for the underlying model.

The project provides **capabilities**, not intelligence.

---

# 22. Current Status

| Component                        | Status                        |
| -------------------------------- | ----------------------------- |
| Modular Agent-skill architecture | Established                   |
| Single-responsibility principle  | Established                   |
| `modlens`                        | In active development         |
| Local image input                | Supported                     |
| Remote image URL input           | Supported                     |
| Provider abstraction             | Implemented                   |
| Antigravity CLI (`agy`) provider | Current default               |
| Gemini model through provider    | Supported                     |
| Schema-enforced output           | Implemented                   |
| Agent skill definition           | Present                       |
| Unit verification                | `pnpm typecheck && pnpm test` |
| `modsearch`                      | Planned                       |
| `modfetch`                       | Planned                       |
| Multi-provider vision support    | Future                        |
| Autonomous orchestration         | Not planned                   |

---

# 23. Final Architecture

The long-term system is intentionally simple:

```text
                         LLM / Agent
                              │
               ┌──────────────┼──────────────┐
               │              │              │
               ▼              ▼              ▼
           modlens        modsearch       modfetch
               │              │              │
               ▼              ▼              ▼
          Vision          Web Search      Web Fetch
               │              │              │
               └──────────────┴──────────────┘
                              │
                              ▼
                         Agent Context
                              │
                              ▼
                              LLM Reasoning
```

The division of responsibility is:

```text
LLM
→ intelligence and reasoning

Agent
→ decisions and orchestration

modlens
→ visual evidence

modsearch
→ search results

modfetch
→ web content
```

The central design principle remains:

> **The model provides intelligence.
> The Agent provides orchestration.
> The `mod*` tools provide capabilities.**
