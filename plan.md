
# Conversation Summary: What You Want to Build (Detailed Version)

## 1. Goal: Give Agents/LLMs “External Capabilities” Instead of Building an All-in-One Proxy

At first, you wanted to build a **Claude/Anthropic API-compatible proxy** that would allow clients such as Claude Code to *think* they were using Claude, while actually running an open-source model underneath. The proxy would also intercept capability gaps such as image understanding and web search, then provide those capabilities through additional models/services.

Later, you narrowed the direction into something lighter and more focused:

> **Do not build a proxy or replace the model.**
> Instead, build an **“Agent Skill” system** that allows any LLM/Agent to gain capabilities such as vision, search, and web fetching through CLI tools.

The core motivation is that this approach is more general, has lower coupling, is easier to evolve, and avoids being locked into a specific client protocol such as Claude Code or the Anthropic API.

---

## 2. Core Principle: One Skill Does One Thing

You established an important product principle:

* **One skill should do only one thing.**
* Do not build an “everything-in-one” tool.
* Do not put multiple responsibilities into a single CLI.
* Let the Agent or higher-level orchestrator combine the tools.
* Keep the underlying tools simple and single-purpose.

Therefore, the capabilities are split into three independent external tools:

1. **modlens** — vision only
2. **modsearch** — search only
3. **modfetch** — web-page fetching only

They should work together as a **pipeline**, rather than having one tool internally perform the entire workflow.

---

## 3. The Actual Capability Gaps You Want to Solve

### 3.1 Vision Extension — `modlens`

Motivation: many open-source or text-only models do not have multimodal capabilities. You want an external tool to fill that gap.

Workflow:

* **Input:** An image provided by the user, such as a screenshot, document, photograph, chart, etc.
* **Processing:** OCR or image captioning, with VQA potentially added later.
* **Output:** Structured **textual evidence** describing the image.
* **Usage:** The higher-level Agent injects this output into the model's context so that a text-only LLM can reason about the image.

Essentially, you are building:

> **image → text evidence adapter**

This allows a non-multimodal model to consume image information through text.

---

### 3.2 Search Extension — `modsearch`

Motivation: many Agents/models do not have reliable web-search capabilities, or different clients implement search differently.

The external tool provides a standardized interface:

* **Input:** Search query
* **Output:** A list of results containing things such as:

  * title
  * URL
  * snippet
  * etc.
* **Usage:** The Agent searches first, then decides which URLs are worth fetching.

The important point is that `modsearch` **does not fetch and summarize the pages**. Its job is simply to search.

---

### 3.3 Fetch Extension — `modfetch`

Motivation: search only gives you links. To allow the model to actually read a webpage, you need a separate fetching capability.

Workflow:

* **Input:** URL
* **Output:** The page's main textual content.
* For HTML pages, extract the visible/main text.
* For non-HTML content, perform best-effort extraction.
* **Usage:** The Agent takes selected URLs from `modsearch`, calls `modfetch`, and then gives the extracted content to the LLM for summarization or citation.

So the separation becomes:

> `modsearch` → find URLs
> `modfetch` → retrieve page content
> LLM → understand/summarize the content

---

## 4. Interaction/Calling Experience: Minimal, Flat, Agent-Friendly

You emphasized that these tools are primarily **for Agents to call**, rather than humans repeatedly typing commands.

Therefore, you want:

* **No nested subcommands**

  * Avoid something like `modlens extract ...`
* Use flat arguments instead:

  * `modlens -i ./img.png`
* Keep commands short.
* Keep parameters short.
* Make behavior predictable.

The underlying goal is:

> **Minimize the cost of an Agent calling the tool and minimize failure points.**

Fewer parsing layers and fewer state branches should mean fewer opportunities for tool-call errors.

---

## 5. Distribution Strategy: Pure JS/TS + npm

You considered the trade-off between Go and JavaScript/TypeScript, particularly around **distribution and installation friction**.

### 5.1 Why You Rejected/Deprioritized Go

You considered building the CLI in Go and then:

* Cross-compiling for different platforms
* Publishing binaries through GitHub Releases
* Having the Agent detect the platform and download the appropriate binary

However, you realized that this introduces many potential friction points:

* Maintaining multiple OS/architecture combinations
* macOS execution permissions, Gatekeeper, and quarantine issues
* Windows path and permission issues
* Potential antivirus/security-software false positives
* Asking users to install Go and compile the project themselves is inconvenient

This becomes especially awkward if the target users are simply friends or people who want the tool to work immediately.

---

### 5.2 Why You Chose JS/TS + npm

You concluded that **JS/TS with npm** is more practical.

Your assumptions are:

* People using AI coding tools such as Codex or Claude Code are likely to already have Node.js installed.
* npm makes it easier for an Agent to automatically install, update, and run the tool.
* You avoid maintaining cross-platform binaries.
* You avoid creating a complicated release/build pipeline.

Therefore, your current decision is:

> **Pure JS/TS, distributed through npm and executed directly.**

The objective is to make the installation path as short and reliable as possible.

---

## 6. The Final Experience You Want From the Agent's Perspective

Your external capability system should enable an Agent to work like this:

### When the user provides an image

```text
User
 ↓
Agent
 ↓
modlens
 ↓
Textual image evidence
 ↓
Agent context
 ↓
LLM reasoning/summarization
```

### When the user asks for information requiring the web

```text
User
 ↓
Agent
 ↓
modsearch
 ↓
Candidate URLs
 ↓
Agent selects relevant URLs
 ↓
modfetch
 ↓
Webpage content
 ↓
LLM reasoning/summarization/citation
```

The important part is that **the Agent controls the workflow**.

The individual tools do not need to understand the entire workflow.

---

## 7. Naming and Product Positioning

The names you have settled on are:

* **Vision extension:** `modlens`
* **Search extension:** `modsearch`
* **Fetch extension:** `modfetch`

The naming convention is consistent:

* `mod*` represents an external/module capability.
* The suffix represents the specific capability:

  * `lens` → vision
  * `search` → search
  * `fetch` → fetching

This naming scheme also leaves room for future extensions, such as:

* `modpdf`
* `modaudio`
* `modcache`
* `modtranslate`
* etc.

But the same principle should remain:

> **Each module should have one clear responsibility.**

### One-sentence product definition

> **A modular CLI skill system that gives any LLM Agent external capabilities such as vision, web search, and web fetching without replacing the underlying model or depending on a specific Agent/client protocol.**
