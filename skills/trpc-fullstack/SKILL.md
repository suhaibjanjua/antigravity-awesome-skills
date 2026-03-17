---
name: trpc-fullstack
description: "Build end-to-end type-safe APIs with tRPC — routers, procedures, middleware, subscriptions, and Next.js/React integration patterns."
category: framework
risk: none
source: community
date_added: "2026-03-17"
author: suhaibjanjua
tags: [typescript, trpc, api, fullstack, nextjs, react, type-safety]
tools: [claude, cursor, gemini]
---

# tRPC Full-Stack

## Overview

tRPC lets you build fully type-safe APIs without writing a schema or code-generation step. Your TypeScript types flow from the server router directly to the client — so every API call is autocompleted, validated at compile time, and refactoring-safe. Use this skill when building TypeScript monorepos, Next.js apps, or any project where the server and client share a codebase.

## When to Use This Skill

- Use when building a TypeScript full-stack app (Next.js, Remix, Express + React) where the client and server share a single repo
- Use when you want end-to-end type safety on API calls without REST/GraphQL schema overhead
- Use when adding real-time features (subscriptions) to an existing tRPC setup
- Use when designing multi-step middleware (auth, rate limiting, tenant scoping) on tRPC procedures
- Use when migrating an existing REST/GraphQL API to tRPC incrementally

## Core Concepts

### Routers and Procedures

A **router** groups related **procedures** (think: endpoints). Procedures are typed functions — `query` for reads, `mutation` for writes, `subscription` for real-time streams.

### Input Validation with Zod

All procedure inputs are validated with Zod schemas. The validated, typed input is available in the procedure handler as `ctx.input` — no manual parsing.

### Context

`context` is shared state passed to every procedure — auth session, database client, request headers, etc. It is built once per request in `createContext()`.

### Middleware

Middleware chains run before a procedure. Use them for authentication, logging, and request enrichment. They can extend the context for downstream procedures.

---

## How It Works

### Step 1: Install and Initialize

```bash
npm install @trpc/server @trpc/client @trpc/react-query @tanstack/react-query zod
```

Create the tRPC instance and reusable builders:

```typescript
// src/server/trpc.ts
import { initTRPC, TRPCError } from '@trpc/server';
import { type Context } from './context';
import { ZodError } from 'zod';

const t = initTRPC.context<Context>().create({
  errorFormatter({ shape, error }) {
    return {
      ...shape,
      data: {
        ...shape.data,
        zodError:
          error.cause instanceof ZodError ? error.cause.flatten() : null,
      },
    };
  },
});

export const router = t.router;
export const publicProcedure = t.procedure;
export const middleware = t.middleware;
```

### Step 2: Define Context

```typescript
// src/server/context.ts
import { type CreateNextContextOptions } from '@trpc/server/adapters/next';
import { getServerAuthSession } from './auth'; // your auth helper
import { db } from './db';

export async function createContext({ req, res }: CreateNextContextOptions) {
  const session = await getServerAuthSession({ req, res });
  return { session, db, req, res };
}

export type Context = Awaited<ReturnType<typeof createContext>>;
```

### Step 3: Build an Auth Middleware and Protected Procedure

```typescript
// src/server/trpc.ts (continued)
const enforceAuth = middleware(({ ctx, next }) => {
  if (!ctx.session?.user) {
    throw new TRPCError({ code: 'UNAUTHORIZED' });
  }
  return next({
    ctx: {
      // Narrows type: session is non-null from here
      session: { ...ctx.session, user: ctx.session.user },
    },
  });
});

export const protectedProcedure = t.procedure.use(enforceAuth);
```

### Step 4: Create Routers

```typescript
// src/server/routers/post.ts
import { z } from 'zod';
import { router, publicProcedure, protectedProcedure } from '../trpc';
import { TRPCError } from '@trpc/server';

export const postRouter = router({
  list: publicProcedure
    .input(
      z.object({
        limit: z.number().min(1).max(100).default(20),
        cursor: z.string().optional(),
      })
    )
    .query(async ({ ctx, input }) => {
      const posts = await ctx.db.post.findMany({
        take: input.limit + 1,
        cursor: input.cursor ? { id: input.cursor } : undefined,
        orderBy: { createdAt: 'desc' },
      });
      const nextCursor =
        posts.length > input.limit ? posts.pop()!.id : undefined;
      return { posts, nextCursor };
    }),

  byId: publicProcedure
    .input(z.object({ id: z.string() }))
    .query(async ({ ctx, input }) => {
      const post = await ctx.db.post.findUnique({ where: { id: input.id } });
      if (!post) throw new TRPCError({ code: 'NOT_FOUND' });
      return post;
    }),

  create: protectedProcedure
    .input(
      z.object({
        title: z.string().min(1).max(200),
        body: z.string().min(1),
      })
    )
    .mutation(async ({ ctx, input }) => {
      return ctx.db.post.create({
        data: { ...input, authorId: ctx.session.user.id },
      });
    }),

  delete: protectedProcedure
    .input(z.object({ id: z.string() }))
    .mutation(async ({ ctx, input }) => {
      const post = await ctx.db.post.findUnique({ where: { id: input.id } });
      if (!post) throw new TRPCError({ code: 'NOT_FOUND' });
      if (post.authorId !== ctx.session.user.id)
        throw new TRPCError({ code: 'FORBIDDEN' });
      return ctx.db.post.delete({ where: { id: input.id } });
    }),
});
```

### Step 5: Compose the Root Router and Export Types

```typescript
// src/server/root.ts
import { router } from './trpc';
import { postRouter } from './routers/post';
import { userRouter } from './routers/user';

export const appRouter = router({
  post: postRouter,
  user: userRouter,
});

// Export the type for the client — never import the appRouter itself on the client
export type AppRouter = typeof appRouter;
```

### Step 6: Mount the API Handler (Next.js App Router)

```typescript
// src/app/api/trpc/[trpc]/route.ts
import { fetchRequestHandler } from '@trpc/server/adapters/fetch';
import { appRouter } from '@/server/root';
import { createContext } from '@/server/context';

const handler = (req: Request) =>
  fetchRequestHandler({
    endpoint: '/api/trpc',
    req,
    router: appRouter,
    createContext: () => createContext({ req } as any),
  });

export { handler as GET, handler as POST };
```

### Step 7: Set Up the Client (React Query)

```typescript
// src/utils/trpc.ts
import { createTRPCReact } from '@trpc/react-query';
import type { AppRouter } from '@/server/root';

export const trpc = createTRPCReact<AppRouter>();
```

```typescript
// src/app/providers.tsx
'use client';
import { QueryClient, QueryClientProvider } from '@tanstack/react-query';
import { httpBatchLink } from '@trpc/client';
import { useState } from 'react';
import { trpc } from '@/utils/trpc';

export function TRPCProvider({ children }: { children: React.ReactNode }) {
  const [queryClient] = useState(() => new QueryClient());
  const [trpcClient] = useState(() =>
    trpc.createClient({
      links: [
        httpBatchLink({
          url: '/api/trpc',
          headers: () => ({ 'x-trpc-source': 'react' }),
        }),
      ],
    })
  );

  return (
    <trpc.Provider client={trpcClient} queryClient={queryClient}>
      <QueryClientProvider client={queryClient}>{children}</QueryClientProvider>
    </trpc.Provider>
  );
}
```

---

## Examples

### Example 1: Fetching Data in a Component

```typescript
// components/PostList.tsx
'use client';
import { trpc } from '@/utils/trpc';

export function PostList() {
  const { data, isLoading, error } = trpc.post.list.useQuery({ limit: 10 });

  if (isLoading) return <p>Loading…</p>;
  if (error) return <p>Error: {error.message}</p>;

  return (
    <ul>
      {data?.posts.map((post) => (
        <li key={post.id}>{post.title}</li>
      ))}
    </ul>
  );
}
```

### Example 2: Mutation with Optimistic UI

```typescript
'use client';
import { trpc } from '@/utils/trpc';

export function CreatePost() {
  const utils = trpc.useUtils();

  const createPost = trpc.post.create.useMutation({
    onSuccess: () => {
      // Invalidate and refetch the post list
      utils.post.list.invalidate();
    },
  });

  const handleSubmit = (e: React.FormEvent<HTMLFormElement>) => {
    e.preventDefault();
    const form = e.currentTarget;
    const data = new FormData(form);
    createPost.mutate({
      title: data.get('title') as string,
      body: data.get('body') as string,
    });
    form.reset();
  };

  return (
    <form onSubmit={handleSubmit}>
      <input name="title" placeholder="Title" required />
      <textarea name="body" placeholder="Body" required />
      <button type="submit" disabled={createPost.isPending}>
        {createPost.isPending ? 'Creating…' : 'Create Post'}
      </button>
      {createPost.error && <p>{createPost.error.message}</p>}
    </form>
  );
}
```

### Example 3: Server-Side Caller (SSR / Server Components)

```typescript
// app/posts/page.tsx (Next.js Server Component)
import { appRouter } from '@/server/root';
import { createCallerFactory } from '@trpc/server';
import { createContext } from '@/server/context';

const createCaller = createCallerFactory(appRouter);

export default async function PostsPage() {
  // Direct call — no HTTP overhead
  const caller = createCaller(await createContext({} as any));
  const { posts } = await caller.post.list({ limit: 20 });

  return (
    <ul>
      {posts.map((post) => (
        <li key={post.id}>{post.title}</li>
      ))}
    </ul>
  );
}
```

### Example 4: Real-Time Subscriptions (WebSocket)

```typescript
// server/routers/notifications.ts
import { observable } from '@trpc/server/observable';
import { EventEmitter } from 'events';

const ee = new EventEmitter();

export const notificationRouter = router({
  onNew: protectedProcedure.subscription(({ ctx }) => {
    return observable<{ message: string; at: Date }>((emit) => {
      const onNotification = (data: { message: string }) => {
        emit.next({ message: data.message, at: new Date() });
      };

      const channel = `user:${ctx.session.user.id}`;
      ee.on(channel, onNotification);
      return () => ee.off(channel, onNotification);
    });
  }),
});
```

```typescript
// Client usage
const { data: notification } = trpc.notification.onNew.useSubscription(
  undefined,
  {
    onData(data) {
      toast(data.message);
    },
  }
);
```

---

## Best Practices

- ✅ **Export only `AppRouter` type** from server code — never import `appRouter` on the client
- ✅ **Validate all inputs with Zod** — never trust raw `input` without a schema
- ✅ **Use `createCallerFactory`** for Server Component / SSR data fetching to avoid HTTP round-trips
- ✅ **Split routers by domain** (posts, users, billing) and merge in `root.ts`
- ✅ **Extend context in middleware** rather than querying the DB multiple times per request
- ✅ **Use `utils.invalidate()`** after mutations to keep the cache fresh
- ❌ **Don't use tRPC over `fetch` in Server Components** — use the server-side caller instead
- ❌ **Don't put business logic in the route handler** — keep it in the procedure or a service layer
- ❌ **Don't skip Zod input schemas** even for simple mutations — they enforce type safety at runtime
- ❌ **Don't share the tRPC client instance globally** — create it per-provider to avoid stale closures

---

## Security & Safety Notes

- Always enforce authorization in `protectedProcedure` — never rely on client-side checks alone
- Validate all input shapes with Zod, including pagination cursors and IDs, to prevent injection via malformed inputs
- Avoid exposing internal error details to clients — use `TRPCError` with a public-safe `message` and keep stack traces server-side only
- Rate-limit public procedures using middleware to prevent abuse

---

## Common Pitfalls

- **Problem:** "Type error: AppRouter is not assignable to AnyRouter"
  **Solution:** Ensure you import `AppRouter` as a `type` import (`import type { AppRouter }`) on the client, not the full module

- **Problem:** Mutations not reflecting in the UI after success
  **Solution:** Call `utils.<router>.<procedure>.invalidate()` in `onSuccess` to trigger a refetch via React Query

- **Problem:** Server-side caller throws "No context" errors in App Router
  **Solution:** Pass a properly constructed context to `createCaller()` — do not reuse request-bound contexts across different server function calls

- **Problem:** "Cannot find module '@trpc/server/adapters/next'" with App Router
  **Solution:** Use `@trpc/server/adapters/fetch` and the `fetchRequestHandler` for Next.js 13+ App Router; the `nextjs` adapter is for Pages Router only

- **Problem:** Subscriptions not connecting
  **Solution:** Subscriptions require a WebSocket or SSE link. Add `splitLink` to route subscriptions to `wsLink` and queries/mutations to `httpBatchLink`

---

## Related Skills

- `@typescript-expert` — Deep TypeScript patterns used inside tRPC routers and generic utilities
- `@react-patterns` — React hooks patterns that pair with `trpc.*.useQuery` and `useMutation`
- `@test-driven-development` — Write procedure unit tests using `createCallerFactory` without an HTTP server
- `@security-auditor` — Review tRPC middleware chains for auth bypass and input validation gaps

## Additional Resources

- [tRPC Official Docs](https://trpc.io/docs)
- [create-t3-app](https://create.t3.gg) — Production Next.js starter with tRPC wired in
- [tRPC GitHub](https://github.com/trpc/trpc)
- [TanStack Query Docs](https://tanstack.com/query/latest)
