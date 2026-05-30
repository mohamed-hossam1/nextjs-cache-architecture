# nextjs-skills

A collection of agent skills for Next.js workflows. The repository
currently includes two skills: `nextjs-cache-architecture` and
`next-action-handler`.

[![skills.sh](https://skills.sh/b/mohamed-hossam1/nextjs-cache-architecture)](https://skills.sh/mohamed-hossam1/nextjs-cache-architecture)
<!-- [![skills.sh](https://skills.sh/b/mohamed-hossam1/next-action-handler)](https://skills.sh/mohamed-hossam1/next-action-handler) -->

## Skills

- `nextjs-cache-architecture` helps design and implement caching in
  Next.js 16+ App Router projects, covering tag registries, cache
  boundaries, revalidation utilities, and debugging stale data.
- `next-action-handler` standardizes server actions with input/output
  validation, error normalization, auth context, and pino logging on
  top of `next-safe-action` and `zod`.

## Repository layout

```
skills/
├── nextjs-cache-architecture/
│   ├── SKILL.md                                # Always-loaded core: architecture + steps + common mistakes
│   ├── overlay.yaml                            # Routing manifest (prompt signals, validators)
│   ├── references/                             # Loaded on demand
│   ├── assets/                                 # Drop-in templates the agent copies into your repo
│   ├── scripts/                                # Verifiers and generators
│   └── evals/                                  # Test cases for the skill
├── next-action-handler/
│   ├── SKILL.md
│   └── references/                             # Loaded on demand
src/                                            # Shared runtime code used by skills
```

## Usage

Install a skill from this repository with `npx skills add`:

```bash
npx skills add https://github.com/mohamed-hossam1/nextjs-skills --skill nextjs-cache-architecture
npx skills add https://github.com/mohamed-hossam1/nextjs-skills --skill next-action-handler
```

## Verifying an implementation

`nextjs-cache-architecture` includes a static audit script at
`skills/nextjs-cache-architecture/scripts/audit.mjs`. Run it after
applying the skill to confirm the checklist items are in place.

## Requirements

### `nextjs-cache-architecture`

| Requirement                                 | Status |
| ------------------------------------------- | ------ |
| Next.js 16+                                 | Yes    |
| `cacheComponents: true` in `next.config.ts` | Yes    |
| App Router                                  | Yes    |
| Pages Router                                | No     |
| Edge runtime                                | No     |
| Static export (`output: "export"`)          | No     |

### `next-action-handler`

- Next.js App Router with server actions
- `next-safe-action`, `zod`, `better-auth`, and `pino` (installed by the skill)

## Contributing

See [AGENTS.md](./AGENTS.md) for style, naming, and structure rules.
Each skill prefers fixing the architecture over working around it —
every new pattern must justify its place by a real mutation or
rendering case.

## License

MIT

## Author

Mohamed Hossam (`mohamed-hossam1`).
