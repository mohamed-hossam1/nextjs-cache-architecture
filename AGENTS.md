# Agent Instructions

Rules for contributing to this repository. The same rules apply to
every AI assistant editing the repo — Claude, Kiro, Cursor, and any
others. `CLAUDE.md` simply points here.

## Repository layout

```
skills/nextjs-cache-architecture/
├── SKILL.md            # Always-loaded core. Keep under ~500 lines.
├── overlay.yaml        # Optional routing manifest. Update when triggering signals change.
├── references/         # Deep-dive markdown the agent loads on demand.
├── assets/             # Drop-in source files copied verbatim into user repos.
├── scripts/            # Executable verifiers / generators with no external deps.
└── evals/              # Test cases for the skill.
```

Top-level docs (`README.md`, `AGENTS.md`, `CLAUDE.md`) describe the
repo and its contribution rules — not the skill's content.

## Style guidelines

- No emojis in any markdown file or code comment.
- Use `Yes/No` in tables instead of checkmarks.
- Use `// Good:` and `// Bad:` comments in code examples — never
  `// WRONG`, `// CORRECT`, or other variants.
- Keep examples focused on one concept at a time.
- Placeholders use `[PascalCase]` for types and components,
  `[camelCase]` for functions, variables, and tag strings.
- Every code snippet should compile if pasted into a real project —
  include the imports it depends on.
- Bad examples come *after* the good pattern they contrast with, never
  alone. The reader should always see the right answer first.

## SKILL.md authoring rules

`SKILL.md` must:

- Open with a YAML frontmatter block that parses cleanly.
  - Required keys: `name`, `description`. Optional: `metadata` (with
    `version`, `author`).
  - Anything else is silently ignored by harnesses — typos become
    invisible bugs.
  - `description` should explicitly list trigger phrases. Agents tend
    to *under*-trigger skills, so be slightly assertive about *when*
    to use it.
  - Bump `metadata.version` when the skill structure or rules change.
- Stay under ~500 lines. If you approach the limit, push detail down a
  layer — into `references/`, `assets/`, or `scripts/` — with clear
  pointers to the new location.
- Explain the *why* before any code. Lean on theory of mind instead of
  blanket `MUST` / `NEVER`. Today's models are smart enough to act on
  reasoning, and reasoning generalizes; rigid rules don't.

## File-structure rules

Each topic in `SKILL.md` should have:

- A clear heading with a one-line description.
- The concept explained in plain language before any code.
- Code examples showing the good pattern, then the failure mode it
  prevents.
- A rules summary or reference table at the end of multi-section
  topics.

## When to add a `references/` file

Add a reference file when content meets all three criteria:

- It is needed only for a subset of invocations (debugging, deep
  theory, rare exceptions, migration paths).
- It is more than ~50 lines.
- Its absence from `SKILL.md` would not confuse a first-time reader.

Reference files must:

- Open with a one-paragraph "load this when..." summary so the agent
  knows whether to read it.
- Include a `## Contents` table of contents if longer than ~80 lines.
- Cross-link back to `SKILL.md` only when strictly necessary —
  references are leaves, not hubs.

## When to add an `assets/` file

Add an asset when the same source file would be re-derived in every
project that uses the skill. The current set:

| Asset                          | Copies to                                |
| ------------------------------ | ---------------------------------------- |
| `tags.ts`                      | `lib/cache/tags.ts`                      |
| `revalidate.ts`                | `lib/cache/revalidate.ts`                |
| `SuspenseOnSearchParams.tsx`   | `components/SuspenseOnSearchParams.tsx`  |

Assets must:

- Carry a header comment explaining what the file is and how to adapt
  it.
- Use the same `[PascalCase]` / `[camelCase]` placeholders as
  `SKILL.md`.
- Be valid TypeScript or TSX after placeholder substitution — no
  missing imports, no shorthand that only works in the surrounding
  doc.

## When to add a `scripts/` file

Add a script when the same verification or transformation would be
repeated manually across every install of the skill. Scripts must:

- Be executable from the command line directly (no compile step).
- Use only the runtime that already ships with the user's environment
  (Node 18+ for `.mjs`, Python 3.10+ for `.py`). No external
  dependencies.
- Print a clear pass/fail summary and exit 0 on success, non-zero on
  failure — so they compose with CI.
- Document usage in their own header comment, not just in `SKILL.md`.

The current set:

| Script        | Purpose                                                  |
| ------------- | -------------------------------------------------------- |
| `audit.mjs`   | Static verification of the post-implementation checklist |

## When to add or change `evals/evals.json`

Eval prompts must read like things a real user would actually type —
domain context, file paths, code snippets, casual phrasing — not
abstract restatements of the skill's features. Each entry needs `id`,
`prompt`, and `expected_output`. Keep `expected_output` describable
in plain text — what *should* the run produce — without dictating the
agent's exact phrasing.

Bump `metadata.version` in `SKILL.md` when adding evals so that
benchmark runs are clearly attributable to the skill version they
tested.

## Code example convention

```tsx
// Bad: description of the problem
<problematic code>

// Good: description of the solution
<correct code>
```

## Naming conventions

- Skill directories: `kebab-case`. Framework-specific skills are
  prefixed with the framework, e.g. `nextjs-`.
- Tag strings: lowercase. Entity tags use `domain:id` format.
- Revalidation functions: `revalidate[Collection]Cache` and
  `revalidate[Entity]Cache`.
- File paths in prose: always relative to project root, e.g.
  `lib/cache/tags.ts` — not `./lib/...`, not `src/lib/...` unless the
  user's project uses `src/`.
- `SKILL.md` frontmatter keys: `name`, `description`, `metadata`.
  Anything else is silently ignored.

## Frontmatter checklist

Every skill's `SKILL.md` frontmatter must:

- Parse as YAML. Verify with
  `python3 -c 'import yaml; yaml.safe_load(open(p).read().split("---")[1])'`.
- Have a `description` that explicitly lists trigger phrases — agents
  tend to under-trigger, so the description should be slightly
  assertive about *when* to use the skill.
- Bump `metadata.version` when the skill structure or rules change.

## What not to add

- Patterns that work around the architecture. If the rules need an
  exception, fix the architecture — do not paper over it.
- Speculative entity tag factories (or analogous "just in case"
  abstractions) without a confirmed mutation that uses them.
  Speculative code drifts away from reality.
- New file conventions in `references/`, `assets/`, `scripts/`, or
  `evals/` without updating this document and the relevant section of
  `SKILL.md`.
- Anything that mutates the user's repo state (deleting files,
  rewriting history) without explicit user instruction.

## Verification before commit

Run these against any change you've made.

- Frontmatter parses:
  ```bash
  python3 -c 'import yaml; yaml.safe_load(open("skills/nextjs-cache-architecture/SKILL.md").read().split("---")[1])'
  ```
- Every `references/` and `assets/` path mentioned in `SKILL.md`
  resolves to a real file.
- No leftover correctness banners:
  ```bash
  grep -E "// (WRONG|CORRECT)" skills/nextjs-cache-architecture/SKILL.md
  ```
- The skill has a title:
  ```bash
  grep -c "^# " skills/nextjs-cache-architecture/SKILL.md
  ```
- Evals JSON parses:
  ```bash
  python3 -c 'import json; json.load(open("skills/nextjs-cache-architecture/evals/evals.json"))'
  ```
- The audit script loads without a syntax error:
  ```bash
  node skills/nextjs-cache-architecture/scripts/audit.mjs --help 2>&1 || true
  ```
