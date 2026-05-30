---
name: next-action-handler
description: "Use when setting up or using next-action-handler, next-safe-action, actionClient/authedActionClient, action metadata/actionName, validationErrors, outputSchema, handleServerError, better-auth errors, or pino logging."
metadata:
  author: mohamed-hossam1
  version: 1.0.1
---

# next-action-handler Skill

## What It Does

`next-action-handler` installs a server action layer built on `next-safe-action`, `better-auth`, `pino`, and `zod`. It standardizes errors, logging, and auth context.

## Installation

If you want a local dev dependency, install it first. Otherwise skip to Usage.

```bash
npm install -D next-action-handler
```

## Usage

From the project root, run the installer:

```bash
npx next-action-handler@latest add
```

`@latest` forces `npx` to use the newest published version. The installer applies
the full handler setup in one pass, with no component selection.

Install path detection order:

1. `lib/` -> `lib/next-action-handler/`
2. `app/lib/` -> `app/lib/next-action-handler/`
3. Otherwise create `lib/next-action-handler/`

Dependencies installed: `better-auth`, `next-safe-action`, `pino`, `pino-pretty`, `server-only`, `zod`.

## Required: auth-helpers.ts

`safe-action.ts` imports `requireUser` from `../auth-helpers`. Create it before using `authedActionClient`.

```ts
// Good: required for authedActionClient
import { headers } from "next/headers";
import { auth } from "./auth";
import { UnauthorizedError } from "./next-action-handler/error/errors";

export async function requireUser() {
  const session = await auth.api.getSession({ headers: await headers() });
  if (!session?.user) throw new UnauthorizedError("You must be logged in");
  return session.user;
}
```

## Action Clients and Metadata

Every action must call `.metadata({ actionName })`. The metadata schema requires it and the logger uses it.

```ts
// Good: metadata actionName is required
import { z } from "zod";
import {
  actionClient,
  authedActionClient,
} from "@/lib/next-action-handler/safe-action";
import { db } from "@/lib/db";

export const submitContactForm = actionClient
  .metadata({ actionName: "submitContactForm" })
  .inputSchema(
    z.object({ email: z.string().email(), message: z.string().min(1) }),
  )
  .action(async ({ parsedInput }) => {
    return { success: true, email: parsedInput.email };
  });

export const updateProfile = authedActionClient
  .metadata({ actionName: "updateProfile" })
  .inputSchema(z.object({ displayName: z.string().min(1) }))
  .action(async ({ parsedInput, ctx }) => {
    await db.users.update({
      id: ctx.user.id,
      displayName: parsedInput.displayName,
    });
    return { updated: true };
  });
```

## Result Shape and Input Validation

`actionClient` returns a `SafeActionResult` union. Only one of `data`, `serverError`, or `validationErrors` is present.

```ts
// Good: result union shape
type SafeActionResult<ServerError, Schema, ShapedErrors, Data> =
  | { data: Data; serverError?: undefined; validationErrors?: undefined }
  | { data?: undefined; serverError: ServerError; validationErrors?: undefined }
  | {
      data?: undefined;
      serverError?: undefined;
      validationErrors: ShapedErrors;
    };
```

Input schema failures land in `validationErrors`, not `serverError`.

```ts
// Good: check validationErrors before serverError
import { z } from "zod";
import { actionClient } from "@/lib/next-action-handler/safe-action";

const schema = z.object({
  email: z.string().email(),
  password: z.string().min(8, "Password must contain at least 8 characters"),
});

export const loginAction = actionClient
  .metadata({ actionName: "loginAction" })
  .inputSchema(schema)
  .action(async ({ parsedInput }) => {
    return { success: true, email: parsedInput.email };
  });

export async function submitLogin() {
  const result = await loginAction({ email: "bad", password: "short" });

  if (result.validationErrors) {
    console.error(result.validationErrors.email?._errors?.[0]);
    return;
  }
  if (result.serverError) {
    console.error(result.serverError.message);
    return;
  }

  console.log(result.data.email);
}
```

## Output Validation

`outputSchema` mismatches become `serverError` (not `validationErrors`). Output validation failures are detected as `ActionOutputDataValidationError` and mapped to a `PublicServerError` with code `INTERNAL_SERVER_ERROR` and message `Unexpected response. Please try again.` The error is normalized and logged using `InternalServerError("Action output validation failed", error)`.

`handleServerError` returns a `PublicServerError` shape `{ code, message }` and uses `DEFAULT_SERVER_ERROR_MESSAGE` when `expose` is false for all other errors.

```ts
// Good: output validation returns serverError
import { z } from "zod";
import { actionClient } from "@/lib/next-action-handler/safe-action";
import { db } from "@/lib/db";

const outputSchema = z.object({
  id: z.string(),
  name: z.string(),
  email: z.string().email(),
});

export const getUser = actionClient
  .metadata({ actionName: "getUser" })
  .inputSchema(z.object({ userId: z.string() }))
  .outputSchema(outputSchema)
  .action(async ({ parsedInput }) => {
    const user = await db.user.findUnique({
      where: { id: parsedInput.userId },
    });
    return { id: user.id, name: user.name, email: user.email };
  });
```

If you customize `handleServerError`, keep the `ActionOutputDataValidationError` branch from `safe-action.ts`. See patterns for a copy-paste block.

## Error Classes

Throw these inside actions. They are normalized and logged; only safe messages reach the client.

| Class                 | Code                    | Expose | Default message             |
| --------------------- | ----------------------- | ------ | --------------------------- |
| `BadRequestError`     | `BAD_REQUEST`           | Yes    | "Bad request"               |
| `ValidationError`     | `VALIDATION_ERROR`      | Yes    | "Invalid input"             |
| `UnauthorizedError`   | `UNAUTHORIZED`          | Yes    | "Unauthorized"              |
| `ForbiddenError`      | `FORBIDDEN`             | Yes    | "Forbidden"                 |
| `NotFoundError`       | `NOT_FOUND`             | Yes    | "Resource not found"        |
| `RateLimitError`      | `RATE_LIMITED`          | Yes    | "Too many requests"         |
| `DatabaseError`       | `DATABASE_ERROR`        | No     | "Database operation failed" |
| `InternalServerError` | `INTERNAL_SERVER_ERROR` | No     | "Something went wrong"      |

## Error Converters

Use helpers when calling better-auth or database APIs that throw.

```ts
// Good: convert external errors into ActionError subclasses
import { fromBetterAuthError } from "@/lib/next-action-handler/error/better-auth-error";
import { toDatabaseError } from "@/lib/next-action-handler/error/database-error";

export function toAuthError(error: unknown) {
  return fromBetterAuthError(error, {
    enumerationSafe: true,
    genericMessage: "Invalid credentials",
  });
}

export function toDbError(error: unknown) {
  return toDatabaseError(error, "Database operation failed");
}
```

## Logging

Logging is automatic. `logActionExecution` runs only when there is no `serverError`. `logActionError` runs for all errors.

## References

- Copy-paste patterns: [references/patterns.md](references/patterns.md)
