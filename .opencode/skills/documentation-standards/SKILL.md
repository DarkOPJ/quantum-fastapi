---
name: documentation-standards
description: Create and maintain industry-standard project documentation in Markdown across frontend, backend, and mobile codebases. Trigger automatically whenever code, architecture, APIs, configuration, or dependencies change in a way that makes existing documentation stale or incomplete, and whenever the user explicitly asks for documentation to be written, reviewed, or updated. Covers README structure, CHANGELOG maintenance, Architecture Decision Records (ADRs), API reference documentation, code-level docstring/comment standards, component documentation, environment/configuration documentation, and general Markdown formatting conventions. Does not cover the actual code architecture, security, or deployment content itself — see the project's other skills for that; this skill only governs how that work gets documented.
---

## Skill Identity

- **Purpose:** Keep project documentation accurate, complete, and industry-standard at all times, across any stack (frontend, backend, mobile), so that both humans and future agent sessions can understand the project without reading all the code.
- **Scope:** README files, changelogs, architecture decision records, API reference docs, code-level documentation (docstrings/comments), component/module documentation, environment and configuration documentation, and general documentation hygiene.
- **This skill owns:** The _existence, structure, accuracy, and freshness_ of documentation.
- **This skill does NOT own:** The underlying technical decisions being documented (architecture, security posture, deployment setup) — those come from the project's other skills. This skill only ensures they are correctly and clearly written down.

## When This Skill Activates

Activate this skill in two situations:

1. **Reactively, after a change.** Whenever a change is made to any of the following, check whether documentation needs to be created or updated in the same session/commit:
   - A new feature, endpoint, screen, or component is added
   - An existing API contract, function signature, or component prop changes
   - A dependency is added, removed, or upgraded in a way that affects setup instructions
   - A configuration or environment variable is added, removed, or renamed
   - An architectural or technology decision is made (e.g. choosing a library, changing a pattern)
   - A bug is fixed whose root cause is non-obvious (worth a note so it isn't reintroduced)
   - A breaking change is introduced
2. **Proactively, on request.** Whenever the user explicitly asks to "document this," "write docs," "update the README," "add a changelog entry," or similar, regardless of whether code changed in this session.

If neither condition applies, do not create documentation unprompted — avoid generating docs for trivial, self-explanatory, or purely internal refactors that don't change any observable behavior or interface.

## Documentation Philosophy

- **Docs-as-code:** Documentation lives in the repository, in Markdown, version-controlled alongside the code it describes — never in an external wiki or disconnected document as the primary source of truth.
- **Update in the same change, not later:** Documentation updates belong in the same commit/PR as the code change that necessitates them. Stale documentation is treated as a bug.
- **Single source of truth, linked not duplicated:** Never restate the same information in multiple files. If two docs need the same fact (e.g. an environment variable list), one owns it and the other links to it.
- **Write for the reader, not the author:** Assume the reader has zero prior context on this specific change. State the "why," not just the "what" — a comment or doc that only restates the code adds no value.
- **Accuracy over completeness:** A short, accurate doc beats a long, aspirational one describing behavior that doesn't actually exist yet.
- **Use the right document type for the right purpose** (see Documentation Types below) rather than cramming everything into one README.

## Documentation Types & When to Use Each

Use this framework (a simplified Diataxis-style split) to decide which document a piece of information belongs in:

| Type             | Purpose                                                      | Example                                               |
| ---------------- | ------------------------------------------------------------ | ----------------------------------------------------- |
| **Tutorial**     | Step-by-step, for a newcomer to get something running        | "Getting Started" section in README                   |
| **How-to guide** | Task-oriented steps for someone who already knows the basics | `docs/how-to/deploy-staging.md`                       |
| **Reference**    | Precise, factual lookup material                             | API reference, config variable tables, CLI flag lists |
| **Explanation**  | Background, reasoning, trade-offs                            | ADRs, architecture overview docs                      |

Do not mix types in one document — a README's "Getting Started" section should not also try to explain _why_ an architectural decision was made; that belongs in an ADR and gets linked to.

## Standard Documentation Structure

Every project (frontend, backend, or mobile) should maintain this structure, creating files as they become needed rather than all upfront:

```
project-root/
├── README.md                  # Entry point: what it is, how to run it
├── CHANGELOG.md                # Version history of user-facing/breaking changes
├── CONTRIBUTING.md             # (if project has multiple contributors) How to contribute
├── docs/
│   ├── adr/                    # Architecture Decision Records
│   │   ├── 0001-use-postgresql.md
│   │   └── 0002-adopt-domain-first-structure.md
│   ├── api/                    # API reference (backend) or SDK reference (frontend/mobile)
│   ├── setup/                  # Platform-specific setup (iOS/Android/web env config)
│   └── architecture.md         # High-level system overview, diagrams, data flow
```

### README.md — required sections, in order

1. **Title & one-line description** — what the project is, in one sentence.
2. **Badges** (optional) — build status, version, license.
3. **Overview** — 2–4 sentences: what problem this solves, who it's for.
4. **Prerequisites** — required runtime versions, tools, accounts/services needed.
5. **Installation** — exact commands, in order, that a new developer runs to get set up.
6. **Configuration** — how to set environment variables/secrets (link to `docs/setup/` for detail, don't duplicate the full variable table here).
7. **Usage / Getting Started** — the smallest possible example that proves the setup worked (e.g. run one command, hit one endpoint, see one screen).
8. **Project Structure** — brief directory overview (link to `docs/architecture.md` for full detail).
9. **Testing** — how to run the test suite.
10. **Contributing** — link to `CONTRIBUTING.md` if it exists.
11. **License**.

### CHANGELOG.md

- Follow the **Keep a Changelog** format: group entries under `## [Unreleased]`, then versioned headers (`## [1.2.0] - 2026-07-26`).
- Group changes under `Added`, `Changed`, `Deprecated`, `Removed`, `Fixed`, `Security`.
- Every breaking change **must** get an entry, written from the consumer's point of view (what they need to do differently), not the implementer's.
- Follow **Semantic Versioning** (`MAJOR.MINOR.PATCH`) for the version headers: breaking change → MAJOR, new backward-compatible feature → MINOR, backward-compatible fix → PATCH.

### Architecture Decision Records (ADRs)

- Create one whenever a non-trivial, hard-to-reverse technical decision is made (choice of database, framework, auth strategy, major pattern adoption/change).
- One file per decision: `docs/adr/NNNN-short-title.md`, numbered sequentially, never renumbered or deleted (superseded decisions get a new ADR that references the old one as superseded, the old file stays for history).
- Standard ADR format:

  ```markdown
  # NNNN. Short title of the decision

  ## Status

  Accepted | Proposed | Superseded by NNNN

  ## Context

  What problem or forces led to this decision needing to be made.

  ## Decision

  What was decided, stated plainly.

  ## Consequences

  What becomes easier or harder as a result. Include trade-offs honestly.
  ```

### API Reference Documentation (backend)

- For REST APIs, keep the OpenAPI/Swagger schema (auto-generated where the framework supports it, e.g. FastAPI) as the primary reference — do not hand-write a separate endpoint list that can drift out of sync.
- For anything the auto-generated schema doesn't capture (rate limits, auth flow overview, webhook behavior, pagination conventions), document it in `docs/api/overview.md` and link to the live schema/docs URL.
- Every endpoint's docstring/description should state: purpose, auth requirements, and any non-obvious side effects — not just repeat the path and method.

### Component / Module Documentation (frontend & mobile)

- Framework-agnostic — applies equally to Vue, React, Svelte, Angular, or any other component-based frontend framework. For UI component libraries, document each component's purpose, props/parameters, and usage example directly alongside the component (e.g. via Storybook stories, or a co-located `.md`/docstring) rather than a separate master document that will drift.
- For mobile, document platform-specific setup (iOS provisioning/signing, Android SDK/build variants) in `docs/setup/ios.md` and `docs/setup/android.md` respectively — don't merge these into one generic "setup" doc that hides platform differences.
- Document any non-obvious state management pattern, navigation structure, or shared design-system convention in `docs/architecture.md`.

### Environment & Configuration Documentation

- Maintain a single table of all environment variables (name, purpose, required/optional, example value — never a real secret) in one location (`docs/setup/environment.md` or README's Configuration section for small projects). All other docs link to it rather than repeating it.

## Code-Level Documentation Standards

- **Python:** Docstrings on all public modules, classes, and functions, in a consistent style (Google or NumPy) — state purpose, parameters, return value, and any raised exceptions.
- **JavaScript/TypeScript:** JSDoc/TSDoc comments on exported functions, components, and complex types — state purpose and describe non-obvious parameters; let TypeScript types speak for themselves where the type alone is self-explanatory.
- **Swift/Kotlin (mobile):** Use the platform-standard doc-comment format (`///` for Swift, KDoc for Kotlin) on public APIs, especially anything crossing a module boundary.
- **Inline comments:** Reserve for explaining _why_, not _what_ — if a comment just restates the next line of code, delete it. Comment on non-obvious business rules, workarounds, or references to an ADR/ticket that explains the reasoning.

## Markdown Formatting Standards

- One `#` H1 per document (the title); use `##`/`###` for hierarchy — never skip a level.
- Fenced code blocks always specify a language for syntax highlighting (` ```python `, ` ```bash `, etc.).
- Use tables for structured, scannable data (config variables, CLI flags, comparison of options) rather than long bullet lists.
- Use relative links between docs in the same repo (`../adr/0001-use-postgresql.md`) so they survive renames of the repo/org.
- Images and diagrams always get descriptive alt text.
- Long documents (roughly 100+ lines) get a table of contents near the top.
- Keep line-level prose reasonably short; prefer lists and tables over dense paragraphs for reference material (paragraphs are fine for Explanation-type docs).

## Agent Rules

- After any code change, explicitly check: does this change an interface, contract, config, dependency, or architectural decision? If yes, update or create the relevant doc in the same turn — do not defer it to "later" or wait to be asked twice.
- Never let a README's Installation/Usage section describe steps that no longer match the actual codebase — verify against current code before writing, don't assume prior docs were correct.
- When a breaking change is made, a CHANGELOG entry is mandatory, not optional.
- When a significant technical decision is made without being asked to write an ADR, proactively suggest writing one rather than only documenting the resulting code.
- Never duplicate content across files. If information already exists in one doc, link to it from others instead of copy-pasting.
- Before restructuring or rewriting a large existing doc, briefly confirm with the user rather than silently overhauling it — small updates and additions don't need confirmation, full rewrites do.
- If documentation is requested for something with no clear existing pattern in the repo, default to the structures defined in this skill rather than inventing a new one-off format.

## Anti-Patterns to Avoid

- Documentation describing aspirational behavior that isn't actually implemented yet.
- A README that has grown to contain the full API reference, full architecture explanation, and full contribution guide instead of linking out to dedicated docs.
- Comments/docstrings that just restate the code with no added context.
- Copy-pasted configuration or API details that exist in more than one file and will drift out of sync.
- Silently leaving documentation stale after a breaking change instead of flagging it.
- Skipping the CHANGELOG for "small" breaking changes — from the consumer's side, there's no such thing.
