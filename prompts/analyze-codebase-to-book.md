# Prompt: Analyze a Codebase and Produce a Technical Book

Reusable prompt for turning any codebase into a structured technical book using parallel AI agents.

---

## The Prompt

```
Analyze the source code at [path] and produce a comprehensive technical book.

## Phase 1: Exploration
Launch parallel agents to read every file. Each agent covers one major subsystem.
Produce a detailed analysis of: architecture, data flow, key abstractions, design
patterns, and how each module connects to others.

## Phase 2: Structure
Organize findings into a book outline ordered as if building the system from scratch.
Each chapter should solve one clear problem that the next chapter depends on.
Group chapters into parts with a narrative arc.

## Phase 3: Writing
Write each chapter from scratch using the analysis as research notes, not by
restructuring them. Follow these rules:
- Voice: expert peer explaining to another expert. Direct, opinionated, no filler.
- Each chapter: opening (what problem + why it matters), body (how it works + why
  those decisions), "Apply This" closing (5 transferable patterns).
- Code: pseudocode only, 3-5 short blocks per chapter illustrating patterns.
  Never reproduce exact source.
- Diagrams: Mermaid blocks for every architectural concept, data flow, and state machine.
- Layered depth: narrative for leaders, deep-dive sections for implementers.

## Phase 4: Review
Launch review agents that evaluate: opening hooks, flow, content cuts needed,
missing content, diagram opportunities, cross-chapter consistency, and specific
line-level fixes. Compile into an action plan.

## Phase 5: Revision
Apply all review feedback: cut duplication, trim bloat, add diagrams, fix
cross-references, standardize Apply This sections, and verify no exact source
code remains.
```

---

## How It Was Used

This prompt produced "Claude Code from Source" — 18 chapters analyzing Anthropic's AI coding agent. The process used 36 agents across ~6 hours, producing 1.3MB of analysis and a 400-page book.
