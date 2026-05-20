# nextjs-cache-architecture

An agent skill for designing and implementing caching in Next.js 16+
App Router projects. It teaches the full mental model — tag registry,
revalidation utilities, Suspense boundaries, mutation wiring — not
just where to drop a `"use cache"` directive.

[![skills.sh](https://skills.sh/b/mohamed-hossam1/nextjs-cache-architecture)](https://skills.sh/mohamed-hossam1/nextjs-cache-architecture)

## What this skill teaches

- When and where to place `"use cache"`, and where it must never go
- Building a centralized tag registry in `lib/cache/tags.ts`
- Centralizing every `updateTag()` call in `lib/cache/revalidate.ts`
- Structuring pages as a static shell plus Suspense-isolated cached
  and dynamic sections
- Handling personalized content (cookies, headers, auth) without
  breaking cache boundaries
- Wiring mutations to invalidation utilities so cache and data stay
  in sync
- `SuspenseOnSearchParams` for pages with search or filter parameters
- Migrating from `unstable_cache` and the deprecated single-argument
  `revalidateTag()` form
- Debugging stale data or unexpectedly fresh data

## Repository layout

```
skills/nextjs-cache-architecture/
├── SKILL.md                                # Always-loaded core: architecture + 7 steps + common mistakes
├── overlay.yaml                            # Routing manifest (prompt signals, validators)
├── references/                             # Loaded on demand
│   ├── core-concepts.md                    # Auto cache keys, placement rules, cacheLife, limitations
│   ├── personalized-content.md             # cookies/headers patterns and "use cache: private"
│   ├── debugging-and-checklist.md          # Debug order and post-implementation checklist
│   └── migration-from-unstable-cache.md    # Mechanical rewrites and gotchas
├── assets/                                 # Drop-in templates the agent copies into your repo
│   ├── tags.ts                             # -> lib/cache/tags.ts
│   ├── revalidate.ts                       # -> lib/cache/revalidate.ts
│   └── SuspenseOnSearchParams.tsx          # -> components/SuspenseOnSearchParams.tsx
├── scripts/
│   └── audit.mjs                           # Static checklist verifier — run against your project
└── evals/
    └── evals.json                          # Starter test cases for the skill
```

The agent loads `SKILL.md` on every invocation, then pulls in
references only when the task calls for them, and copies templates
from `assets/` into your project — renaming placeholders such as
`[Entity]` and `[collection]` to match your real names.

## Usage

Describe your domain and the skill triggers on phrases like "set up
caching", "add `use cache`", "invalidate when a post updates", "why is
my data stale", or even an open-ended "I have a `posts` table — how
should I cache it?". The agent applies the architecture to your actual
codebase, replacing every placeholder with your real entity and
collection names.

## Verifying an implementation

After the skill applies the architecture to your project, run the
audit script to check the static parts of the post-implementation
checklist:

```bash
npx skills add https://github.com/mohamed-hossam1/nextjs-cache-architecture --skill nextjs-cache-architecture
```

It verifies that `cacheComponents: true` is set, that
`lib/cache/tags.ts` and `lib/cache/revalidate.ts` exist, that no raw
`updateTag()` calls leak outside the revalidation utility, that no
deprecated single-argument `revalidateTag()` calls remain, and that
`"use cache"` files do not also call `cookies()`, `headers()`, or
`auth()`.

## Requirements

| Requirement                                 | Status |
| ------------------------------------------- | ------ |
| Next.js 16+                                 | Yes    |
| `cacheComponents: true` in `next.config.ts` | Yes    |
| App Router                                  | Yes    |
| Pages Router                                | No     |
| Edge runtime                                | No     |
| Static export (`output: "export"`)          | No     |

## Contributing

See [AGENTS.md](./AGENTS.md) for style, naming, and structure rules.
The skill prefers fixing the architecture over working around it —
every new pattern must justify its place by a real mutation or
rendering case.

## License

MIT

## Author

Mohamed Hossam (`mohamed-hossam1`).
