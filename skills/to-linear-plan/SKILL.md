---
name: to-linear-plan
description: Turn a PRD held as a resource document under a Linear umbrella issue into a multi-phase, tracer-bullet implementation plan saved to ./plans/ — then, after approval, fill the umbrella issue with the plan (single-phase) or create one child issue per phase under it (multi-phase). Invoke as /to-linear-plan <umbrella-issue>, where the argument is a Linear issue link or identifier.
disable-model-invocation: true
---

# PRD to Plan

Break a PRD into a phased implementation plan using vertical slices (tracer bullets). The PRD lives as a **resource document attached to a Linear umbrella issue** (created by `/to-linear-prd`). Outputs: a Markdown file in `./plans/`, and — after user approval — Linear issues anchored on that same umbrella issue: the umbrella itself holds the plan when it's single-phase, or one child issue per phase when it's multi-phase.

## Input

A single Linear reference to the **umbrella issue** — a full `linear.app/.../issue/PHO-123/...` URL or a bare `PHO-123` identifier (extract the identifier). This is the umbrella issue `/to-linear-prd` created, with the PRD attached to it as a resource document. If it's missing, ask for it before doing anything else.

## Process

### 1. Fetch the PRD

`get_issue` the umbrella issue. The PRD itself lives in a **resource document attached to that issue**, not in the issue description — find the attached document in the issue's documents/attachments and `get_document` it to read the full PRD content. (If you can't locate it that way, fall back to `list_documents` with a `query` of the feature name and match on the one parented to this issue.)

Treat the document content (plus any decisions in the umbrella issue's description or comments) as the PRD. If it's thin or contradicts what's already in the conversation, surface that and reconcile with the user before slicing.

### 2. Explore the codebase

If you have not already, explore to understand the current architecture, existing patterns, and integration layers.

### 3. Identify durable architectural decisions

Before slicing, identify high-level decisions unlikely to change throughout implementation:

- Route structures / URL patterns
- Database schema shape
- Key data models
- Authentication / authorization approach
- Third-party service boundaries

These go in the plan header so every phase can reference them.

### 4. Draft vertical slices

Break the PRD into **tracer bullet** phases. Each phase is a thin vertical slice that cuts through ALL integration layers end-to-end, NOT a horizontal slice of one layer.

<vertical-slice-rules>
- Each slice delivers a narrow but COMPLETE path through every layer (schema, API, UI, tests)
- A completed slice is demoable or verifiable on its own
- Prefer many thin slices over few thick ones
- Do NOT include specific file names, function names, or implementation details that are likely to change as later phases are built
- DO include durable decisions: route paths, schema shapes, data model names
</vertical-slice-rules>

### 5. Quiz the user

Present the proposed breakdown as a numbered list. For each phase show:

- **Title**: short descriptive name
- **User stories covered**: which user stories from the PRD this addresses

Ask the user:

- Does the granularity feel right? (too coarse / too fine)
- Should any phases be merged or split further?

Also state where the Linear issues will land: the **umbrella issue** is the anchor (state its identifier). If the plan is single-phase, the umbrella issue itself will hold the plan; if multi-phase, child issues will be created under the umbrella, inheriting its team/project.

Iterate until the user approves the breakdown. Do not write files or touch Linear until they approve — this approval is the gate for both outputs.

### 6. Write the plan file

Create `./plans/` if it doesn't exist. Write the plan as a Markdown file named after the feature (e.g. `./plans/user-onboarding.md`), using the template below. The `Source PRD` line must link the umbrella issue (identifier + URL) and the PRD resource document.

<plan-template>
# Plan: <Feature Name>

> Source PRD: <umbrella issue identifier> — <umbrella issue URL> (PRD document: <document title / URL>)

## Architectural decisions

Durable decisions that apply across all phases:

- **Routes**: ...
- **Schema**: ...
- **Key models**: ...
- (add/remove sections as appropriate)

---

## Phase 1: <Title>

**User stories**: <list from PRD>

### What to build

A concise description of this vertical slice. Describe the end-to-end behavior, not layer-by-layer implementation.

### Acceptance criteria

- [ ] Criterion 1
- [ ] Criterion 2
- [ ] Criterion 3

---

## Phase 2: <Title>

**User stories**: <list from PRD>

### What to build

...

### Acceptance criteria

- [ ] ...

<!-- Repeat for each phase -->
</plan-template>

### 7. Write the Linear issues

This step writes to Linear (hard to undo); the step-5 approval is what authorizes it — never create or modify issues for a breakdown the user hasn't approved.

The **umbrella issue is the anchor** — do NOT create a new parent issue. Branch on phase count:

- **Single-phase plan** — the umbrella issue *is* the plan. `save_issue` with `id`: the umbrella issue identifier, updating its `description` to the full plan (architectural decisions + the one phase's **What to build** + **Acceptance criteria** checklist). Do not create any child issues. The PRD remains attached as its resource document.

- **Multi-phase plan** — keep the umbrella as the parent:
  1. `save_issue` with `id`: the umbrella issue identifier, updating its `description` to the plan overview — the **Architectural decisions**, then a numbered list of the phases.
  2. For each phase **in order**, `save_issue` a child:
     - `title`: `Phase N: <Title>`
     - `description`: the phase's **What to build** + **Acceptance criteria** (as a checklist), plus a link back to the umbrella issue
     - `parentId`: the umbrella issue identifier
     - `team` / `project`: inherited from the umbrella issue (read them from the `get_issue` in step 1)
     - `blockedBy`: the previous phase's child issue (for phases after the first), so the sequential tracer-bullet order is encoded as Linear blocking relations

     Create children sequentially so each phase can reference the previous child's identifier for `blockedBy`.

After writing, annotate the plan file: add the umbrella issue identifier to the top (e.g. `> Tracked in: <umbrella identifier> — <URL>`) and, for a multi-phase plan, each child identifier to its phase heading (e.g. `## Phase 1: <Title> (PHO-456)`). Report back the umbrella issue and any created child issues with their identifiers and URLs.
