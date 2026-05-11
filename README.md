# nextjs-cache-architecture

Agent skill for Next.js 16+ caching architecture using the `"use cache"` directive.

Covers the full mental model from tag registry design to mutation wiring —
not just syntax, but how to structure caching as architecture from day one.

## What this skill teaches

- When and where to place `"use cache"`
- Building a centralized tag registry in `lib/cache/tags.ts`
- Centralizing all `updateTag()` calls in `lib/cache/revalidate.ts`
- The five rendering layers: static, cached, entity, personalized, invalidation
- Handling personalized content without breaking cache boundaries
- Wiring mutations to invalidation utilities correctly
- `SuspenseOnSearchParams` for filtered pages
- Debugging stale or incorrectly fresh data

## Installation

```bash
npx skills add mohamed-hossam1/nextjs-cache-architecture
```

## Usage

Invoke via slash command in your agent:

```
/nextjs-cache-architecture
```

Or describe what you want to cache and the skill applies the architecture
patterns to your actual codebase — replacing all placeholders with your
real entity and collection names.

## Requirements

- Next.js 16+
- `cacheComponents: true` in `next.config.ts`
- App Router

## Author

Mohamed Hossam