# Project Plan — Modular Agent Capability Skills

## 1. Project Goal

Build a collection of small, composable CLI tools that give LLM agents capabilities they may not natively have.

The system should **not** replace the underlying LLM and should **not** act as a proxy for Claude, Anthropic, OpenAI, or any other model provider.

Instead, an Agent calls these tools when it needs additional capabilities.

The initial capability set is:

* `modlens` — vision
* `modsearch` — web search
* `modfetch` — web-page fetching

The core architecture is:

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

The Agent is responsible for deciding **when and how to compose these capabilities**.

---

# 2. Core Design Principles

## 2.1 One Skill, One Responsibility

Every module should have one clearly defined responsibility.

* `modlens` should only handle vision.
* `modsearch` should only handle search.
* `modfetch` should only handle fetching.

Do not turn one CLI into an all-in-one research assistant.

For example:

```text
Bad:

modresearch
 ├── search
 ├── fetch
 ├── summarize
 ├── OCR
 └── citation generation
```

Instead:

```text
Good:

modsearch → find information
modfetch  → retrieve information
modlens   → understand images
LLM       → reason, summarize, decide
```

Composition belongs to the Agent or higher-level orchestrator.

---

## 2.2 Low Coupling

The tools should not depend on a particular Agent implementation.

They should work with:

* Claude Code
* Codex
* other coding agents
* custom LLM agents
* future agent frameworks

Avoid designing around a single client's proprietary tool protocol.

The CLI should provide a simple, stable interface that any Agent capable of executing commands can use.

---

## 2.3 Agent-Oriented Design

These tools are primarily designed to be invoked by Agents rather than manually operated by humans.

Therefore:

* commands should be short;
* arguments should be predictable;
* output should be machine-readable where appropriate;
* failures should be explicit;
* unnecessary interaction should be avoided;
* avoid interactive prompts whenever possible.

The goal is to minimize the Agent's tool-call complexity.

---

# 3. Package Structure

The initial ecosystem consists of three independent packages.

## 3.1 `modlens`

Purpose:

> Convert image information into textual evidence that an LLM can consume.

Example:

```bash
modlens -i ./image.png
```

Basic pipeline:

```text
Image
  ↓
modlens
  ↓
Vision backend
  ↓
Structured text evidence
  ↓
Agent context
```

### V1 Backend

The initial implementation will use **Gemini CLI** as the vision backend.

`modlens` should act as an adapter around the backend rather than tightly coupling the entire project to Gemini.

Conceptually:

```text
                 modlens
                    │
                    ▼
            Vision Backend API
                    │
             ┌──────┴──────┐
             ▼             ▼
        Gemini CLI      Future engines
             │
             ▼
       Textual Evidence
```

### Future Backends

Potential future backends include:

* PaddleOCR
* DeepSeek OCR
* other multimodal models
* other OCR engines
* local vision models

The backend should therefore be replaceable without changing the fundamental `modlens` interface.

---

## 3.2 `modsearch`

Purpose:

> Search the web and return candidate results.

Example:

```bash
modsearch "latest NVIDIA earnings"
```

Expected conceptual output:

```text
[
  {
    "title": "...",
    "url": "...",
    "snippet": "..."
  }
]
```

`modsearch` should **not**:

* fetch the pages;
* summarize the pages;
* decide which result is relevant;
* perform the entire research workflow.

Its responsibility ends at providing search results.

The Agent decides which URLs should be fetched.

---

## 3.3 `modfetch`

Purpose:

> Retrieve readable content from a URL.

Example:

```bash
modfetch "https://example.com/article"
```

Basic pipeline:

```text
URL
 ↓
modfetch
 ↓
HTTP request
 ↓
Content extraction
 ↓
Readable text
```

For HTML:

```text
HTML
 ↓
Extract visible/main content
 ↓
Text
```

For non-HTML content:

```text
Resource
 ↓
Best-effort extraction
 ↓
Text
```

`modfetch` should not become a search engine or summarization tool.

---

# 4. Intended Agent Workflows

## 4.1 Image Workflow

When a user provides an image:

```text
User
 ↓
Agent detects image
 ↓
Agent calls modlens
 ↓
modlens extracts textual evidence
 ↓
Agent adds evidence to context
 ↓
LLM reasons about the image
```

Example:

```text
User:
"What does this screenshot mean?"

Agent:
→ modlens -i screenshot.png

modlens:
→ textual description/evidence

Agent:
→ provides evidence to LLM

LLM:
→ explains screenshot
```

The LLM remains responsible for reasoning.

`modlens` provides the evidence.

---

## 4.2 Web Research Workflow

When a user asks for information requiring the web:

```text
User
 ↓
Agent
 ↓
modsearch
 ↓
Search results
 ↓
Agent selects relevant URLs
 ↓
modfetch
 ↓
Page text
 ↓
LLM
 ↓
Answer / summary / citations
```

Example:

```text
modsearch "latest Malaysia inflation rate"
        ↓
    search results
        ↓
Agent selects relevant sources
        ↓
modfetch <selected URL>
        ↓
    article text
        ↓
LLM analyzes information
```

The tools remain independent.

---

# 5. CLI Design

## 5.1 Flat Commands

Avoid nested commands.

Preferred:

```bash
modlens -i image.png
```

Not:

```bash
modlens image extract --input image.png
```

Similarly:

```bash
modsearch "query"
```

and:

```bash
modfetch "https://example.com"
```

The interface should remain shallow.

---

## 5.2 Short and Predictable Flags

Prefer simple arguments such as:

```text
-i    input image
-o    output
-f    output format
```

Exact flags should be finalized during implementation.

Avoid exposing unnecessary configuration unless it is required.

---

## 5.3 Machine-Friendly Output

Because Agents are the primary consumers, output should be easy for an LLM or wrapper process to parse.

Where appropriate, support structured output such as JSON.

For example:

```json
{
  "title": "Example Article",
  "url": "https://example.com",
  "content": "..."
}
```

Human-readable output may still be supported, but machine-readable output should be treated as an important design requirement.

---

# 6. Distribution

## 6.1 Technology Choice

The project will use:

* JavaScript / TypeScript
* Node.js
* npm

The initial goal is to distribute the tools through npm rather than compiled platform-specific binaries.

Example conceptual installation:

```bash
npm install -g modlens
```

or execution through npm tooling where appropriate.

---

## 6.2 Why Not Go Binaries?

Go was considered for the CLI, particularly because it produces standalone executables.

However, cross-platform binary distribution introduces additional complexity:

* OS/architecture combinations
* release artifacts
* executable permissions
* macOS Gatekeeper/quarantine
* Windows security software
* installation and update handling

Using Node.js/npm reduces much of this distribution complexity for the intended audience.

This decision can be revisited if real-world usage demonstrates that native binaries provide a significant advantage.

---

# 7. Repository Strategy

The modules should remain conceptually independent.

Possible structure:

```text
modlens/
modsearch/
modfetch/
```

Each package should have:

* its own CLI;
* its own README;
* its own dependencies;
* its own tests;
* its own release lifecycle.

If a monorepo becomes useful later, the packages can still remain independently usable and publishable.

The important requirement is **logical independence**, not whether the Git repositories are physically separate.

---

# 8. `modlens` Backend Architecture

`modlens` should not hard-code the assumption that Gemini is the only possible vision engine.

The internal architecture should conceptually separate:

```text
CLI
 │
 ▼
Input handling
 │
 ▼
Vision abstraction
 │
 ├── Gemini CLI
 ├── PaddleOCR
 ├── DeepSeek OCR
 └── Future backend
 │
 ▼
Normalized evidence
 │
 ▼
CLI output
```

This allows future engines to be added without redesigning the public CLI.

For example:

```bash
modlens -i screenshot.png
```

should remain valid even if the underlying backend changes.

---

# 9. Error Handling

Tools should fail clearly and predictably.

Errors should distinguish between cases such as:

* invalid arguments;
* missing input;
* invalid URL;
* network failure;
* backend failure;
* authentication/configuration failure;
* unsupported content;
* extraction failure.

Avoid silently returning misleading output.

An Agent should be able to determine whether:

```text
success
```

or:

```text
tool failed
```

occurred.

Exit codes should therefore be considered part of the CLI contract.

---

# 10. Security Considerations

Because `modfetch` accepts arbitrary URLs, security must be considered from the beginning.

Potential concerns include:

* malicious URLs;
* redirects;
* very large responses;
* request timeouts;
* unsupported protocols;
* localhost/internal network access;
* resource exhaustion.

The implementation should establish reasonable limits rather than blindly fetching anything indefinitely.

Similarly, `modlens` should avoid unnecessarily exposing credentials or sensitive environment information to external vision backends.

---

# 11. What the Tools Should NOT Do

The project should deliberately avoid scope expansion during the initial implementation.

### `modlens` should NOT become:

* a general chatbot;
* a search engine;
* a web scraper;
* a document-management system;
* an autonomous research agent.

### `modsearch` should NOT become:

* a webpage reader;
* a summarizer;
* an autonomous researcher.

### `modfetch` should NOT become:

* a search engine;
* a summarizer;
* an LLM agent.

The LLM/Agent remains the reasoning and orchestration layer.

---

# 12. Initial Development Priority

Development should proceed in small, independently testable stages.

## Phase 1 — `modlens`

Build the first usable capability:

```text
image
 ↓
modlens
 ↓
Gemini CLI
 ↓
structured text evidence
```

Goals:

* establish CLI interface;
* establish input/output contract;
* implement Gemini backend;
* normalize output;
* implement basic error handling;
* test with screenshots, documents, photos, and charts.

---

## Phase 2 — `modsearch`

Build a standalone search CLI.

Goals:

* accept a query;
* perform search;
* return structured results;
* handle search failures;
* keep output predictable.

The exact search backend can be selected separately from the CLI design.

---

## Phase 3 — `modfetch`

Build the URL-to-text capability.

Goals:

* accept URLs;
* retrieve pages;
* extract readable content;
* handle redirects;
* handle failures/timeouts;
* support structured output.

---

## Phase 4 — Agent Integration

After the individual tools work independently, test them from actual Agents.

Test workflows such as:

```text
Agent → modlens
Agent → modsearch
Agent → modsearch → modfetch
```

The goal is to verify that the tools are genuinely useful when called by an Agent rather than merely working as standalone CLIs.

---

# 13. Success Criteria

The project is successful if an Agent can acquire capabilities it does not natively possess without needing a specialized integration.

### Vision

```text
Agent + modlens
=
text-only Agent capable of consuming image evidence
```

### Search

```text
Agent + modsearch
=
Agent capable of discovering web sources
```

### Fetch

```text
Agent + modfetch
=
Agent capable of reading web content
```

### Composition

```text
Agent + modsearch + modfetch
=
Agent capable of performing web-based research
```

The critical requirement is that the Agent remains in control.

---

# 14. Future Extensions

The `mod*` naming convention allows additional capabilities to be introduced later.

Potential examples:

```text
modpdf
modaudio
modcache
modtranslate
modvideo
```

However, every new module must follow the same rule:

> **One module = one capability.**

A new module should only be created when the capability is sufficiently distinct to justify an independent interface.

---

# 15. Current Decisions

The following decisions are currently considered established:

| Decision                                | Status            |
| --------------------------------------- | ----------------- |
| Build an Anthropic/Claude proxy         | Rejected          |
| Replace the underlying LLM              | Rejected          |
| Build standalone Agent capability tools | **Yes**           |
| One skill = one responsibility          | **Yes**           |
| `modlens`                               | **Yes**           |
| `modsearch`                             | **Yes**           |
| `modfetch`                              | **Yes**           |
| Flat CLI interface                      | **Yes**           |
| Agent handles composition               | **Yes**           |
| JS/TS                                   | **Yes**           |
| npm distribution                        | **Yes**           |
| `modlens` V1 backend                    | **Gemini CLI**    |
| Pluggable vision backends               | **Yes**           |
| PaddleOCR / DeepSeek OCR                | Future candidates |
| All-in-one research CLI                 | **No**            |

---

# 16. Current Scope

The immediate objective is **not** to build a complete Agent framework.

The immediate objective is to prove that a small set of independent CLI capabilities can effectively extend an existing Agent.

The first milestone is therefore:

```text
                 Existing Agent
                       │
                       ▼
                    modlens
                       │
                       ▼
                Image → Evidence
```

Once this works reliably, expand the ecosystem with:

```text
modsearch
    +
modfetch
```

The long-term vision is a collection of small, composable capability modules that Agents can discover and invoke as needed.

> **The model provides intelligence.
> The Agent provides orchestration.
> The `mod*` tools provide capabilities.**
