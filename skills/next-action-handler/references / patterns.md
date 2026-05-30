# next-action-handler: Copy-Paste Patterns

## 1. Full server action + client component

### Action file (`actions/posts.ts`)

```ts
"use server";

// Good: authed action with metadata and typed errors
import { z } from "zod";
import { authedActionClient } from "@/lib/next-action-handler/safe-action";
import {
  NotFoundError,
  ForbiddenError,
} from "@/lib/next-action-handler/error/errors";
import { toDatabaseError } from "@/lib/next-action-handler/error/database-error";
import { db } from "@/lib/db";

export const deletePost = authedActionClient
  .metadata({ actionName: "deletePost" })
  .inputSchema(z.object({ postId: z.string().uuid() }))
  .action(async ({ parsedInput, ctx }) => {
    const post = await db.posts.findById(parsedInput.postId).catch((error) => {
      throw toDatabaseError(error, "Failed to fetch post");
    });

    if (!post) throw new NotFoundError("Post not found");
    if (post.authorId !== ctx.user.id) throw new ForbiddenError();

    await db.posts.delete(post.id).catch((error) => {
      throw toDatabaseError(error, "Failed to delete post");
    });

    return { deleted: true };
  });
```

### Client component (`components/DeletePostButton.tsx`)

```tsx
"use client";

// Good: check validationErrors before serverError
import { useState } from "react";
import { deletePost } from "@/actions/posts";

export function DeletePostButton({ postId }: { postId: string }) {
  const [error, setError] = useState<string | null>(null);

  async function handleClick() {
    setError(null);
    const result = await deletePost({ postId });

    if (result.validationErrors) {
      setError("Invalid input");
      return;
    }

    if (result.serverError) {
      setError(result.serverError.message);
      return;
    }

    // result.data.deleted === true
  }

  return (
    <>
      <button onClick={handleClick}>Delete</button>
      {error && <p className="text-red-500">{error}</p>}
    </>
  );
}
```

---

## 2. Client error handling with `useAction`

```tsx
"use client";

// Good: handle validationErrors and serverError
import { useAction } from "next-safe-action/hooks";
import { toast } from "sonner";
import { updateProfile } from "@/actions/profile";

export function ProfileForm() {
  const { execute, isPending } = useAction(updateProfile, {
    onSuccess: () => {
      toast.success("Profile updated");
    },
    onError: ({ error }) => {
      if (error.validationErrors) {
        toast.error("Invalid input");
        return;
      }
      if (error.serverError) {
        toast.error(error.serverError.message);
      }
    },
  });

  return (
    <button
      onClick={() => execute({ displayName: "Alice" })}
      disabled={isPending}
    >
      Save
    </button>
  );
}
```

---

## 3. Output validation handling in `safe-action.ts`

```ts
// Good: keep PublicServerError shape and handle output validation errors
import "server-only";
import z from "zod";
import {
  createSafeActionClient,
  DEFAULT_SERVER_ERROR_MESSAGE,
} from "next-safe-action";
import { logActionError, logActionExecution } from "./log/logger";
import { normalizeError } from "./error/normalize-error";
import { InternalServerError } from "./error/errors";
import { requireUser } from "../auth-helpers";

const OUTPUT_VALIDATION_SERVER_ERROR_MESSAGE =
  "Unexpected response. Please try again.";

function isActionOutputDataValidationError(error: unknown): error is Error {
  return (
    error instanceof Error &&
    (error.name === "ActionOutputDataValidationError" ||
      error.constructor?.name === "ActionOutputDataValidationError")
  );
}

export const actionClient = createSafeActionClient({
  defineMetadataSchema: () =>
    z.object({
      actionName: z.string(),
    }),
  handleServerError(error, ctx) {
    if (isActionOutputDataValidationError(error)) {
      const normalized = normalizeError(
        new InternalServerError("Action output validation failed", error),
      );

      logActionError({
        action: ctx.metadata.actionName,
        error: normalized,
      });

      return {
        code: normalized.code,
        message: OUTPUT_VALIDATION_SERVER_ERROR_MESSAGE,
      };
    }

    const normalized = normalizeError(error);

    logActionError({
      action: ctx.metadata.actionName,
      error: normalized,
    });

    return {
      code: normalized.code,
      message: normalized.expose
        ? normalized.message
        : DEFAULT_SERVER_ERROR_MESSAGE,
    };
  },
}).use(async ({ next, metadata }) => {
  const startedAt = Date.now();

  const result = await next();

  if (!result.serverError) {
    logActionExecution({
      action: metadata.actionName,
      durationMs: Date.now() - startedAt,
    });
  }

  return result;
});

export const authedActionClient = actionClient.use(async ({ next }) => {
  const user = await requireUser();
  return next({ ctx: { user } });
});
```

---

## 4. Unauthenticated action (public form)

```ts
"use server";

// Good: public action with metadata
import { z } from "zod";
import { actionClient } from "@/lib/next-action-handler/safe-action";
import { BadRequestError } from "@/lib/next-action-handler/error/errors";
import { db } from "@/lib/db";

export const subscribeToNewsletter = actionClient
  .metadata({ actionName: "subscribeToNewsletter" })
  .inputSchema(z.object({ email: z.string().email("Invalid email address") }))
  .action(async ({ parsedInput }) => {
    const existing = await db.subscribers.findByEmail(parsedInput.email);
    if (existing) throw new BadRequestError("Email already subscribed");

    await db.subscribers.create({ email: parsedInput.email });
    return { subscribed: true };
  });
```

---

## 5. auth-helpers.ts with better-auth

```ts
// Good: requireUser used by authedActionClient
import { headers } from "next/headers";
import { auth } from "@/lib/auth";
import { UnauthorizedError } from "@/lib/next-action-handler/error/errors";

export async function requireUser() {
  const session = await auth.api.getSession({ headers: await headers() });
  if (!session?.user) throw new UnauthorizedError("You must be logged in");
  return session.user;
}
```

---

## 6. ValidationError with field-level errors

```ts
// Good: use ValidationError for custom server-side validation
import { ValidationError } from "@/lib/next-action-handler/error/errors";

throw new ValidationError("Passwords do not match", {
  confirmPassword: ["Must match the password field"],
});
```

On the client, `ValidationError.fields` is not forwarded via `serverError`. Schema failures still show up in `result.validationErrors`.
