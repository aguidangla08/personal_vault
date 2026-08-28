# Documentation Practices for AI-Assisted Code Development

## Context

A colleague explained that even when using AI coding agents (like Claude Code, Copilot, Cursor, etc.), he still produces the same three core documents he would write if he were coding entirely by hand:

1. **Specs Document**
2. **Requirement List**
3. **Design Document**
4. **Tasks Phases Document**

The idea is that AI agents don't replace planning — they replace _typing_. Good upfront documentation actually becomes **more** important with AI agents, because it's the main way you control what the AI builds, and it gives the agent (and future readers) a clear source of truth instead of vague prompts.

Below is what each document typically contains.

---

## 1. Specs Document (Specification Document)

**Purpose:** Describes _what_ the software should do, from a functional/behavioral standpoint — the contract between "what's needed" and "what will be built."

**Typically contains:**

- **Overview / Purpose** — why this feature or system exists, the problem it solves
- **Scope** — what's included and explicitly what's _not_ included
- **Functional behavior** — expected inputs, outputs, and behavior for each feature or endpoint
- **User stories / use cases** — "As a user, I want to..." scenarios
- **Edge cases and error handling** — what should happen when things go wrong
- **Acceptance criteria** — how you know the feature is "done" and correct
- **Dependencies** — other systems, APIs, or services it relies on

**Why it matters with AI agents:** This is the document you feed (or summarize) to the AI agent so it knows _exactly_ what "correct" looks like, reducing hallucinated features or missed edge cases.

---

## 2. Requirement List

**Purpose:** A more granular, often checklist-style breakdown of _specific, verifiable_ things the system must satisfy — functional and non-functional.

**Typically contains:**

- **Functional requirements** — concrete capabilities (e.g., "System must allow CSV export of results")
- **Non-functional requirements** — performance, security, scalability, availability, compliance (e.g., "API response time < 200ms", "Must comply with GDPR")
- **Constraints** — technology stack limitations, budget, timeline, platform restrictions
- **Priorities** — must-have vs. nice-to-have (often using MoSCoW: Must/Should/Could/Won't)
- **Traceability IDs** — unique identifiers (REQ-001, REQ-002...) so each requirement can be tracked to code, tests, and specs

**Why it matters with AI agents:** Requirements are easy to turn into **test cases** and **prompts**. A numbered requirement list lets you (or the AI) verify coverage — nothing silently gets dropped when the agent generates code.

---

## 3. Design Document (Technical Design / Architecture Document)

**Purpose:** Describes _how_ the system will be built — the technical plan that turns specs and requirements into an actual architecture.

**Typically contains:**

- **Architecture overview** — components, services, and how they interact (often with diagrams)
- **Data model** — database schema, data structures, entity relationships
- **API design** — endpoints, request/response formats, contracts
- **Technology choices** — languages, frameworks, libraries, and justification for choosing them
- **Sequence/flow diagrams** — how data or control flows through the system for key operations
- **Trade-offs and alternatives considered** — why this approach was chosen over others
- - **AI-generated alternative approaches** — asking the AI agent itself to propose multiple possible solutions/architectures before committing to one, surfacing approaches the designer might not have considered
- **Security and performance considerations**
- **Rollout / migration plan** — if replacing or extending an existing system

**Why it matters with AI agents:** This document constrains the _implementation_ — it prevents the AI from inventing its own architecture that conflicts with existing systems, and gives it a blueprint to follow file-by-file or module-by-module.
A useful practice is to explicitly ask the AI agent to generate **multiple candidate solutions** during the design phase rather than just one. Since AI models are exposed to a wide range of patterns and approaches, this can surface options the designer wouldn't have thought of. The key is having **clear selection criteria** beforehand (e.g., maintainability, performance, consistency with existing codebase, complexity, cost) so the designer — not the AI — makes the final, deliberate choice between the alternatives.

It also **reduces ambiguity for the AI**. A given spec or requirement can usually be implemented in several valid ways (different patterns, data structures, libraries, or levels of abstraction). If you don't decide the path, the AI will — and it may pick one that's inconsistent with your codebase, harder to maintain, more  complex than expected, or simply not what you had in mind. The design document is where *you* make that decision instead of leaving it to the model's default choice.

---

## 4. Tasks Phases Document

**Purpose:** Breaks the work down into concrete tasks and groups them into working phases — turning the design into an executable, ordered plan.

**Typically contains:**
- **Task breakdown** — the design and requirements decomposed into discrete, actionable tasks
- **Phases / groups** — tasks clustered into logical stages (e.g., "Phase 1: Data layer", "Phase 2: API endpoints", "Phase 3: UI integration")
- **Dependencies between tasks** — which tasks must be completed before others can start
- **Sequencing rationale** — why this order (e.g., foundational pieces first, risky/uncertain parts early, quick wins for validation)
- **Scope per phase** — what's delivered/testable at the end of each phase
- **Status tracking** — often a checklist or table (not started / in progress / done)

**Why it matters with AI agents:** AI agents work best with focused, well-scoped chunks of work rather than an entire system at once. This document prevents the AI from tackling everything simultaneously (which increases errors and context loss) and lets you review and validate each phase before moving to the next — keeping the AI's output aligned and easier to verify incrementally.

---

## How These Four Documents Work Together


```
Specs Document        →  WHAT problem are we solving, and for whom?
Requirement List       →  WHAT specific, testable things must be true?
Design Document        →  HOW will it technically be built?
Tasks Phases Document  →  IN WHAT ORDER will it be built?
```

A common workflow when using AI agents:

1. Write the **specs** to frame the problem.
2. Break specs into a **requirement list** (testable, trackable).
3. Write the **design document** describing the technical approach.
4. Break the design into a **tasks phases document**, sequencing the work.
5. Feed relevant sections of these documents to the AI agent as context/prompts, phase by phase.
6. Use the requirement list as an acceptance checklist to verify the AI's output at each phase.

This mirrors traditional software engineering discipline — the AI agent changes _how fast_ code gets written, not _whether_ you still need to think clearly about what you're building and why.