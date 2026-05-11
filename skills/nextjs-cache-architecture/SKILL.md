---
name: nextjs-cache-architecture
description: >
  Next.js 16+ caching architecture using use cache, cacheLife(), cacheTag(),
  and updateTag(). Applies to any App Router project regardless of domain.
Dmetadata:
  author: mohamed-hossam1
  version: 2.0.0
---

## How to Use This Skill

Read the user input below, then apply every rule and template in this file to their actual project.
Replace all placeholders (`[Entity]`, `[collection]`, etc.) with names from their codebase before writing any code.

```text
$ARGUMENTS
```

---

## Mental Model — Read This First

Caching in Next.js 16 is **architecture**, not optimization. Design it upfront.

```
Layer 1 — Static shell
  Synchronous layout, nav, headers. No data fetching. Prerendered at build time.

Layer 2 — Cached shared content
  Same output for all users. Uses "use cache" + collection tag + cacheLife().
  Revalidated in the background. Wrapped in <Suspense>.

Layer 3 — Auto-scoped entity / filtered content
  Per-item pages or search results. Auto-keyed by arguments and closures.
  Entity tag added only when a mutation needs surgical invalidation.

Layer 4 — Dynamic personalized content
  User-specific. Reads cookies/headers OUTSIDE the cache boundary.
  Passes derived primitives as props into a nested cached component.
  Wrapped in <Suspense>.

Layer 5 — Invalidation
  Mutations call revalidation utilities only.
  Utilities call updateTag() — collection tags for bulk, entity tags for surgical.
```

### The single decision that drives everything else

```
Before adding cacheTag(CACHE_TAGS.[entity](id)):
  "Will a mutation ever call updateTag() on this specific entry individually?"
  YES → add entity tag factory to registry, tag the fetch, wire the revalidation utility
  NO  → auto-keying handles scoping; only the collection tag is needed
```

---

## Core Concepts

### Auto cache key generation

Next.js generates a unique cache key for every `"use cache"` function automatically.
You never construct cache keys manually.

| Component | What it includes |
|---|---|
| Build ID | Changes on every deploy — all caches invalidated automatically |
| Function ID | Hash of the function's file path and position in source |
| Arguments | Every value passed to the function at call time |
| Closure variables | Every outer-scope value captured by the function |

```tsx
// Every unique resourceId produces a separate cache entry — automatically
async function Parent({ resourceId }: { resourceId: string }) {
  const fetchData = async (filter: string) => {
    "use cache";
    // key = [buildId] + [fn hash] + resourceId (closure) + filter (argument)
    return fetch(`/api/resources/${resourceId}?filter=${filter}`);
  };
  return fetchData("active");
}
```

Tags are for **invalidation**. Auto-keying handles **scoping**. These are different concerns.

### "use cache" placement rules

| Placement | When to use |
|---|---|
| Top of an async function body | Single data-fetching function |
| Top of an async Server Component body | Entire component output is cacheable |
| Never in a page component | Page components orchestrate; they do not fetch |

`"use cache"` must be the first statement in the function body — before any `await`.

```ts
// CORRECT
async function getItems() {
  "use cache";
  cacheLife("hours");
  cacheTag(CACHE_TAGS.items);
  return db.items.findMany();
}

// WRONG — directive after await
async function getItem(id: string) {
  const row = await db.items.findUnique({ where: { id } });
  "use cache"; // ignored — too late
  return row;
}
```

### Closure variable rules

Keep closure variables to serializable primitives: `string`, `number`, `boolean`, plain objects, arrays.

```tsx
// BAD — large object serialized into cache key
async function Parent({ item }: { item: Item }) {
  const fetch = async () => {
    "use cache";
    return getItem(item.id); // item is the whole object in closure
  };
}

// GOOD — extract only the primitive needed
async function Parent({ item }: { item: Item }) {
  const id = item.id;
  const fetch = async () => {
    "use cache";
    return getItem(id); // only a string in closure
  };
}
```

Next.js throws at runtime if a closure variable is non-serializable:
class instances, functions, Symbols, circular references — all forbidden.

---

## Step 1 — Enable Cache Components

```ts
// next.config.ts
import type { NextConfig } from "next";

const nextConfig: NextConfig = {
  cacheComponents: true,
};

export default nextConfig;
```

Import these at the top of every file that uses caching:

```ts
import { cacheLife, cacheTag, updateTag } from "next/cache";
```

---

## Step 2 — Build the Cache Tag Registry

**File**: `lib/cache/tags.ts`

Single source of truth for all tag strings. Raw tag strings are never written anywhere else.

### Rules

- Lowercase only
- Entity tags use `domain:id` format
- Match your actual data model — do not invent names
- Do not add entity factories speculatively — only when a mutation requires `updateTag()` on that entry

### Template

```ts
// lib/cache/tags.ts

export const CACHE_TAGS = {
  // COLLECTION TAGS — one per logical data group, always present
  [collection]: "[collection]",
  [anotherCollection]: "[anotherCollection]",

  // ENTITY TAG FACTORIES — only when a mutation targets a single entry via updateTag()
  [entity]: (id: string | number) => `[entity]:${id}`,
} as const;
```

### Example

```ts
// lib/cache/tags.ts

export const CACHE_TAGS = {
  // Collection tags — always present
  products: "products",
  categories: "categories",
  users: "users",

  // Entity tag factories — only where surgical invalidation is needed
  product: (id: string | number) => `product:${id}`,
  // "category" and "user" omitted — no mutation targets a single entry individually
} as const;
```

---

## Step 3 — Build Revalidation Utilities

**File**: `lib/cache/revalidate.ts`

All `updateTag()` calls live here. Mutations import these functions — they never call `updateTag()` directly.

### Template

```ts
// lib/cache/revalidate.ts
"use server";

import { updateTag } from "next/cache";
import { CACHE_TAGS } from "./tags";

function invalidateTags(tags: string[]) {
  for (const tag of tags) updateTag(tag);
}

// Bulk — any entry in the collection changed
export async function revalidate[Collection]Cache() {
  invalidateTags([CACHE_TAGS.[collection]]);
}

// Surgical — one specific entry changed
// Only write this if CACHE_TAGS.[entity] factory exists in the registry
export async function revalidate[Entity]Cache(id: string | number) {
  invalidateTags([
    CACHE_TAGS.[collection], // always invalidate the parent collection too
    CACHE_TAGS.[entity](id),
  ]);
}
```

### Example

```ts
// lib/cache/revalidate.ts
"use server";

import { updateTag } from "next/cache";
import { CACHE_TAGS } from "./tags";

function invalidateTags(tags: string[]) {
  for (const tag of tags) updateTag(tag);
}

export async function revalidateProductsCache() {
  invalidateTags([CACHE_TAGS.products]);
}

export async function revalidateProductCache(id: string | number) {
  invalidateTags([CACHE_TAGS.products, CACHE_TAGS.product(id)]);
}

export async function revalidateCategoriesCache() {
  invalidateTags([CACHE_TAGS.categories]);
}
```

---

## Step 4 — Implement Data Fetching

Place `"use cache"` in data-fetching functions. Never fetch inside page components.

### Collection fetch

```ts
// lib/data/[domain].ts

export async function get[Collection]() {
  "use cache";
  cacheLife("hours");
  cacheTag(CACHE_TAGS.[collection]);

  const res = await fetch(`${BASE_URL}/[endpoint]`);
  return res.json();
}
```

### Entity fetch

```ts
export async function get[Entity](id: string) {
  "use cache";
  cacheLife("hours");
  cacheTag(CACHE_TAGS.[collection]);
  // Add CACHE_TAGS.[entity](id) only if a mutation calls updateTag on this specific entry

  const res = await fetch(`${BASE_URL}/[endpoint]/${id}`);
  return res.json();
}
```

### What not to do

```tsx
// WRONG — fetching in page component bypasses caching and invalidation
export default async function Page() {
  const res = await fetch("/api/items");
  const data = await res.json();
  return <View data={data} />;
}
```

---

## Step 5 — Structure Rendering Boundaries

Every page follows this pattern:

```
Page component (sync, orchestration only — no data fetching)
  ├── Static shell (layout, nav — no data)
  ├── <Suspense> → Cached shared content
  └── <Suspense> → Dynamic personalized content
```

### Standard page

```tsx
// app/[route]/page.tsx
import { Suspense } from "react";

export default function AnyPage() {
  return (
    <>
      <StaticShell />

      <Suspense fallback={<SharedSkeleton />}>
        <SharedContent />
      </Suspense>

      <Suspense fallback={<PersonalizedSkeleton />}>
        <PersonalizedSection />
      </Suspense>
    </>
  );
}

async function SharedContent() {
  "use cache";
  cacheLife("hours");
  cacheTag(CACHE_TAGS.[collection]);

  const data = await get[Collection]();
  return <[Collection]List data={data} />;
}
```

### Dynamic route page

```tsx
// app/[domain]/[id]/page.tsx
import { Suspense } from "react";

export default function EntityPage({
  params,
}: {
  params: Promise<{ id: string }>;
}) {
  return (
    <Suspense fallback={<EntitySkeleton />}>
      <EntityDetail params={params} />
    </Suspense>
  );
}

async function EntityDetail({
  params,
}: {
  params: Promise<{ id: string }>;
}) {
  const { id } = await params;
  return <CachedEntityView id={id} />;
}

async function CachedEntityView({ id }: { id: string }) {
  "use cache";
  cacheLife("hours");
  cacheTag(CACHE_TAGS.[collection]);
  // Add CACHE_TAGS.[entity](id) only if a mutation needs surgical invalidation

  const item = await get[Entity](id);
  return <[Entity]View item={item} />;
}
```

### Filtered / search params page

```tsx
// app/[route]/page.tsx
import { Suspense } from "react";
import SuspenseOnSearchParams from "@/components/SuspenseOnSearchParams";

export default function FilteredPage({
  searchParams,
}: {
  searchParams: Promise<Record<string, string>>;
}) {
  return (
    <SuspenseOnSearchParams fallback={<FilteredListSkeleton />}>
      <FilteredList searchParams={searchParams} />
    </SuspenseOnSearchParams>
  );
}

async function FilteredList({
  searchParams,
}: {
  searchParams: Promise<Record<string, string>>;
}) {
  "use cache";
  cacheLife("minutes");
  cacheTag(CACHE_TAGS.[collection]);
  // searchParams is an argument → auto-keyed per unique param combination

  const { q = "", page = "1" } = await searchParams;
  return await get[Collection]ByFilter(q, page);
}
```

### SuspenseOnSearchParams — required for filtered pages

Standard `<Suspense>` does not re-trigger its fallback on client-side navigation when only `searchParams` changes.
Use this wrapper on every page with search or filter params.

```tsx
// components/SuspenseOnSearchParams.tsx
"use client";

import { useSearchParams } from "next/navigation";
import { Suspense } from "react";

export default function SuspenseOnSearchParams({
  fallback,
  children,
}: {
  fallback: React.ReactNode;
  children: React.ReactNode;
}) {
  const searchParams = useSearchParams();
  return (
    <Suspense key={searchParams.toString()} fallback={fallback}>
      {children}
    </Suspense>
  );
}
```

---

## Step 6 — Handle Personalized Content

Never call `cookies()`, `headers()`, or `auth()` inside a `"use cache"` function.
Read them outside the cache boundary and pass derived primitives as props.

```tsx
// CORRECT

// 1. Read request-time data outside the cache boundary
async function PersonalizedSection() {
  const cookieStore = await cookies();
  const userId = cookieStore.get("userId")?.value;

  // 2. Pass as prop — auto-included in cache key
  return <CachedPersonalizedView userId={userId} />;
}

// 3. Cache the stable rendering
async function CachedPersonalizedView({
  userId,
}: {
  userId: string | undefined;
}) {
  "use cache";
  cacheLife("minutes");
  // userId is an argument → auto-keyed per user

  const data = await getPersonalizedData(userId);
  return <div>{/* render */}</div>;
}
```

```tsx
// WRONG — dynamic API inside cached function
async function CachedView() {
  "use cache";
  const cookieStore = await cookies(); // throws or produces incorrect behavior
  return <div />;
}
```

### Exception: `"use cache: private"`

Use only when compliance requirements prevent refactoring to the pattern above:

```tsx
async function getData() {
  "use cache: private";
  const session = (await cookies()).get("session")?.value; // allowed
  return fetchData(session);
}
```

---

## Step 7 — Wire Mutations to Invalidation

Mutations call revalidation utilities. They never call `updateTag()` directly.

```ts
// app/actions/[domain].ts
"use server";

import {
  revalidate[Collection]Cache,
  revalidate[Entity]Cache,
} from "@/lib/cache/revalidate";

export async function create[Entity](payload: unknown) {
  await db.[entity].create(payload);
  await revalidate[Collection]Cache();
}

export async function update[Entity](id: string | number, payload: unknown) {
  await db.[entity].update(id, payload);
  await revalidate[Entity]Cache(id);
}

export async function delete[Entity](id: string | number) {
  await db.[entity].delete(id);
  await revalidate[Entity]Cache(id);
}
```

```ts
// WRONG — updateTag scattered in business logic, raw strings
export async function update[Entity](id: string, payload: unknown) {
  await db.[entity].update(id, payload);
  updateTag("[collection]");
  updateTag(`[entity]:${id}`);
}
```

---

## Cache Duration Reference

| Profile | Use when |
|---|---|
| `"seconds"` | Near-real-time data (live feeds, counters) |
| `"minutes"` | Frequently updated content (dashboards, notifications) |
| `"hours"` | Moderately stable content (listings, articles, configs) |
| `"days"` | Rarely updated content (reference data, documentation) |
| `"max"` | Effectively permanent (build-time constants, static assets) |

For fine-grained control:

```ts
cacheLife({
  stale: 3600,      // serve stale for up to 1 hour
  revalidate: 7200, // background revalidation every 2 hours
  expire: 86400,    // hard expiration at 1 day
});
```

---

## Limitations

- Edge runtime is not supported — requires Node.js
- Static export (`output: "export"`) is not supported
- `Math.random()` and `Date.now()` inside `"use cache"` execute once at build time, not per request

For request-time non-determinism:

```tsx
import { connection } from "next/server";

async function DynamicContent() {
  await connection(); // defers execution to request time
  const id = crypto.randomUUID(); // different per request
  return <div>{id}</div>;
}
```

---

## Debugging Order

When cache behavior is wrong — stale data, no invalidation, unexpected freshness:

1. Is `"use cache"` the first statement in the function body, before any `await`?
2. Is a dynamic API (`cookies`, `headers`, `auth`) called inside a cached function?
3. Does the collection tag in `cacheTag()` exactly match the tag string in the registry?
4. If surgical invalidation is needed, does the entity tag factory exist in `lib/cache/tags.ts`?
5. Is the revalidation utility actually called after the mutation completes?
6. Are Suspense boundaries correctly isolating dynamic from cached sections?
7. Run `next build` and inspect static vs dynamic route output.

---

## Post-Implementation Checklist

- [ ] `next.config.ts` has `cacheComponents: true`
- [ ] `lib/cache/tags.ts` has a collection tag for every data domain
- [ ] Entity tag factories added only where mutations require surgical `updateTag()`
- [ ] `lib/cache/revalidate.ts` contains all `updateTag()` calls — nowhere else
- [ ] Every `"use cache"` function has both `cacheTag()` and `cacheLife()`
- [ ] `"use cache"` is the first statement in every function that uses it
- [ ] No dynamic request APIs (`cookies`, `headers`, `auth`) inside cached functions
- [ ] Closure variables are primitives — not whole objects or class instances
- [ ] Page components are synchronous and do not fetch data
- [ ] `SuspenseOnSearchParams` used on every page with search or filter params
- [ ] Mutations call revalidation utilities — never `updateTag()` directly
- [ ] `next build` confirms expected static/dynamic rendering boundaries

---

## Rules Summary

### Always

- Centralize tag strings in `lib/cache/tags.ts`
- Centralize `updateTag()` calls in `lib/cache/revalidate.ts`
- Add a collection tag to every `"use cache"` function
- Put `"use cache"` first — before any `await`
- Include both `cacheTag()` and `cacheLife()` in every cached function
- Read `cookies()` / `headers()` outside cached functions, pass results as props
- Use `SuspenseOnSearchParams` on any page with search or filter params
- Run `next build` to verify boundaries

### Never

- Write raw tag strings outside `lib/cache/tags.ts`
- Call `updateTag()` directly in server actions, route handlers, or components
- Add entity tag factories without a confirmed mutation that requires them
- Call dynamic request APIs inside `"use cache"` functions
- Capture large objects or non-serializable values in closures
- Fetch data inside `page.tsx`
- Place `"use cache"` after an `await` — it will be ignored
