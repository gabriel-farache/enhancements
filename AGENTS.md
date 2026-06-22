# Agent Instructions

Read `README.md` first — it covers what qualifies as an enhancement and the
basic contribution workflow. This file supplements it with details agents need
to produce conformant enhancement documents.

## Enhancement Template

Every enhancement lives in `enhancements/<name>/<name>.md`. The canonical
template is `enhancements/template.md`.

### YAML Frontmatter

Each enhancement starts with YAML frontmatter:

```yaml
---
title: your-enhancement-name
authors:
  - "@github-handle"
reviewers:
  - "@github-handle"
approvers:
  - "@github-handle"
creation-date: yyyy-mm-dd
see-also:
  - "/enhancements/related-enhancement.md"
replaces:
  - "/enhancements/old-enhancement.md"
superseded-by:
  - "/enhancements/newer-enhancement.md"
---
```

- `title` must match the enhancement directory name
- `authors`, `reviewers`, and `approvers` use `@github-handle` format
- `creation-date` is a real date in ISO format (`yyyy-mm-dd`), not a placeholder
- `approvers` and `reviewers` must not contain empty strings — use `TBD` if
  unknown at draft time
- `see-also`, `replaces`, and `superseded-by` are optional — omit entirely if
  not applicable, never set to `TBD` or empty strings

### Required Sections

After the frontmatter, the document must include these sections in order:

1. Title (H1)
2. Open Questions
3. Summary
4. Motivation
5. Goals / Non-Goals
6. Proposal
7. User Stories
8. Implementation Details/Notes/Constraints
9. Risks and Mitigations
10. Design Details
11. Upgrade / Downgrade Strategy
12. Implementation History
13. Drawbacks
14. Alternatives (with subsections: Description, Pros, Cons, Status, Rationale)
15. Infrastructure Needed

The template contains HTML comments (`<!-- ... -->`) explaining what goes in
each section. Replace them with real content — do not leave them in the final
document.

## Scaffolding

Create a new enhancement with:

```
make new NAME=your-enhancement-name
```

This creates `enhancements/<NAME>/<NAME>.md` from the template. The name must
match `^[a-z0-9]+(-[a-z0-9]+)*$` (lowercase words separated by hyphens).

## Formatting

Run before every commit:

```
make format
```

Prettier enforces 80-character prose wrap (config in `.prettierrc`). CI runs
`make check-format` on all `*.md` files in PRs targeting `main`.

## Spelling

Run before every commit:

```
make check-spell
```

The custom dictionary is in `cspell.yaml`. Add new technical terms, project
names, or contributor handles there when needed.

## Review Process

- All files are owned by the reviewers listed in `.github/CODEOWNERS`

## Level of Detail

Enhancements describe **what** the system does, not **how** the code implements
it. Stay at the API and data model level:

- **Include:** JSON payloads, endpoint paths, data models, sequence diagrams,
  integration points described by role, mermaid diagrams
- **Exclude:** language-specific constructs — function names, package imports,
  middleware wiring, ORM details, cache implementations, specific HTTP error
  codes per endpoint

Express data structures as JSON or YAML schemas, not language-specific types.
Express behavior as API calls and data flows, not function signatures.

**Wrong:** "The provider implements `ListVMsFromCluster()` using
`kubevirt.io/client-go` and returns a `[]v1.VirtualMachine`. Returns 404 if the
namespace is missing, 409 on conflict, 503 if the API is unavailable."

**Right:** "The provider lists VMs by querying the KubeVirt API
(`GET /apis/kubevirt.io/v1/namespaces/{ns}/virtualmachines`) and returns the
results as a JSON array."

Code-level details and per-endpoint error code tables belong in implementation
documents or OpenAPI specs, not enhancements.

## Writing Guidelines

### Section Completeness

Every section listed in the template must appear in the final document. If a
section does not apply, write "N/A — [one-sentence reason]" rather than silently
omitting the section or leaving an empty heading.

### Summary

The Summary is the first section readers encounter — keep it clear, concise, and
consistent in style across enhancements. State what the enhancement does and why
in two to four sentences. Avoid implementation details, background context, or
motivation (those belong in their own sections). A reader should be able to
decide whether to keep reading based on the Summary alone.

### Template Placeholders

The template contains HTML comments (`<!-- ... -->`) that are authoring
instructions, not content. Remove every HTML comment and replace it with real
content. Never leave `TBD`, `TODO`, or placeholder text in any section. Verify
that all frontmatter fields have been updated from their template defaults — a
`title: neat-enhancement-idea` in the final document is a copy-paste error.

### Diagrams

Use Mermaid for all diagrams — no ASCII art or embedded images. Use sequence
diagrams (`sequenceDiagram`) for component interactions and flowcharts
(`flowchart`) for decision logic or state transitions. Keep diagrams focused:
one diagram per flow, not one diagram for the entire system.

### Risks and Mitigations

Present risks as a table with two columns: Risk and Mitigation. Each risk should
describe a concrete failure mode, not an abstract concern. Each mitigation
should describe a specific countermeasure.

| Risk | Mitigation |
| ---- | ---------- |
| ...  | ...        |

### Drawbacks

The Drawbacks section must contain at least one genuine argument against the
proposal. Every design makes tradeoffs — state what they are and explain why
they are acceptable. "There are no significant drawbacks" is not acceptable.

### Alternatives

Each alternative must follow the template structure with all five subsections:
Description, Pros, Cons, Status, and Rationale. The Rationale must name the
specific tradeoff that drove the decision (e.g., "operational complexity
outweighs the scaling benefit at current scale"), not generic dismissals like
"too complex." Evaluate at least one genuine alternative beyond the proposal.

### Assumptions

When the enhancement has infrastructure, connectivity, RBAC, or dependency
prerequisites, list them in an Assumptions subsection within the Proposal. Not
every enhancement needs this — use judgment.

### Terminology

These documents are "enhancements," not "ADRs," "design docs," or "RFCs." Use
"enhancement" consistently in prose.

## Common Pitfalls

- **Skipping `make format`** — CI will fail the line-length check
- **Wrong date format** — frontmatter `creation-date` must be `yyyy-mm-dd`, not
  any other format
- **Bad enhancement name** — must be lowercase kebab-case; the Makefile enforces
  the regex
- **Leaving template placeholders** — HTML comments are instructions for the
  author, not content for the reader

## Note on CLAUDE.md

`CLAUDE.md` is a symlink to this file. Claude Code does not read `AGENTS.md`
natively, so the symlink ensures both tools share a single source of truth with
zero drift.
