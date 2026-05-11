# Agent Instructions

Rules for contributing to this repository.

## Style Guidelines

- No emojis in any markdown files or code comments
- Use `Yes/No` in tables instead of checkmarks
- Use `// Good:` and `// Bad:` comments in code examples
- Keep examples focused on one concept at a time
- Placeholders use `[PascalCase]` for types and components, `[camelCase]` for functions and variables

## File Structure

Each skill topic should have:
- A clear heading with a one-line description
- The concept explained in plain language before any code
- Code examples showing bad vs good patterns
- A rules summary or reference table at the end

## Code Examples

```tsx
// Bad: description of the problem
<problematic code>

// Good: description of the solution
<correct code>
```

## Naming Conventions

- Skill directories: `kebab-case` prefixed with `nextjs-`
- Tag strings: lowercase, `domain:id` format for entity tags
- Revalidation functions: `revalidate[Collection]Cache` / `revalidate[Entity]Cache`
- File paths: always relative to project root, e.g. `lib/cache/tags.ts`

## What not to add

- Do not add patterns that work around the architecture — fix the architecture
- Do not add entity tag factories without a confirmed mutation use case
- Do not introduce new file conventions without updating the checklist in SKILL.md