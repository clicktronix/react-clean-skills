---
name: architector
description: Design or refactor React/Next.js features using the Fullstack AI Template hybrid Clean Architecture profile. Use when adding a feature, defining domain schemas, use-cases, ports, inbound/outbound adapters, server-state, or deciding where code belongs. Enforce strict layer boundaries; adapt stack details only when the target repo has explicit conflicting conventions.
---

# Architector Skill

Apply the Fullstack AI Template architecture profile. The skill is portable across projects, but it is not loose: use these defaults unless the target repository has its own `AGENTS.md`, `CLAUDE.md`, or architecture docs that explicitly override a path, library, or naming convention.

## Source Priority

1. Target repository instructions and architecture docs.
2. Existing vertical slices in the target repository.
3. This default profile from `fullstack-ai-template`.

When the target repo has no equivalent docs, use this profile exactly.

## Default Profile

- Framework: Next.js App Router, React, TypeScript.
- Domain validation: Valibot schemas and inferred types.
- Persistence: Supabase outbound adapters.
- Server state: TanStack Query in `src/ui/server-state/<feature>/`.
- UI: Mantine + CSS Modules, route-local `_internal/ui/` for page-specific UI.
- i18n: locale/message files plus `TranslationText`.
- Component logic: `composeHooks(View)(useProps)`.

Equivalent stacks are allowed only as substitutions for the same layer role. Do not delete the layer.

## Glossary

- **Port** — a TypeScript `type` alias describing exactly what a use-case needs from the outside world (a repository contract, an email sender, a clock). Ports live in `use-cases/<feature>/ports.ts`. Use-cases depend on ports; outbound adapters implement them; tests pass fake objects that satisfy the same type.
- **Adapter** — concrete implementation of a port (Supabase repository, REST client, in-memory fake).
- **Use-case** — a plain async function that accepts a `deps` object (satisfying one or more ports) plus an input, validates with a domain schema, orchestrates calls, and returns a domain result.
- **Inbound adapter** — Next.js Server Action or route handler that composes dependencies (creates the concrete outbound instances), acquires auth/session context, calls the use-case, and triggers framework-level cache invalidation.
- **Feature-local `actions.ts`** — a file co-located with UI that re-exports or thin-wraps a Server Action for UI components that don't need TanStack Query semantics (no cache, no optimistic update, no shared state).

## Dependency Flow

```text
app/ui -> ui/server-state | feature-local actions.ts -> inbound adapters -> use-cases -> outbound adapters -> domain
```

Rules behind the flow:

- `src/use-cases/**` must not import `app`, `ui`, inbound adapters, React hooks, TanStack Query, or framework request/response APIs.
- `src/ui/server-state/**` is the only UI-facing layer allowed to call inbound adapters.
- Feature-local `actions.ts` is allowed only for thin direct Server Action wrappers when TanStack Query semantics are unnecessary.
- UI must not import outbound adapters directly.
- `app/` entrypoints stay thin.
- Use-cases define ports; outbound adapters implement them. The use-case must depend on the port contract, not concrete infrastructure.

If an implementation violates these rules, stop and move the code to the correct layer before continuing.

## Layer Map

| Layer | Default path | Purpose | Must not do |
| --- | --- | --- | --- |
| Domain | `src/domain/` | schemas, inferred types, invariants, pure helpers | import outside `domain` |
| Use-Cases | `src/use-cases/` | application scenarios, ports, feature-local types, lightweight orchestration | use `use server`, `NextRequest`, `NextResponse`, `revalidatePath`, React hooks, TanStack Query |
| Outbound | `src/adapters/outbound/` | repositories, DB operations, external APIs, transport | depend on inbound or UI |
| Inbound | `src/adapters/inbound/next/` | Server Actions, route handlers, auth/session context, request mapping, cache revalidation | contain business logic |
| Server-State | `src/ui/server-state/` | TanStack Query keys/hooks, mutations, SSR prefetch, cache orchestration | be imported by non-UI code |
| UI | `src/app/`, `src/ui/` | App Router pages, components, view hooks, providers, themes | import outbound adapters directly |
| Infrastructure | `src/infrastructure/` | auth, i18n, logging, config, shared technical glue | contain feature business logic |

## Feature Build Order

Create or update layers in this order:

1. Domain
2. Use-cases
3. Outbound adapters
4. Inbound adapters
5. UI server-state
6. UI
7. Tests

Do not start in UI for a feature that introduces new business data or workflow rules. For purely presentational UI, use the component skill instead.

## File Placement

```text
src/
├── domain/<feature>/
├── use-cases/<feature>/
│   ├── <feature>.ts
│   ├── ports.ts
│   └── types.ts
├── adapters/
│   ├── inbound/next/server-actions/<feature>.ts
│   └── outbound/<backend>/
│       ├── <feature>.operations.ts
│       └── <feature>.repository.ts
├── ui/server-state/<feature>/
│   ├── keys.ts
│   ├── queries.ts
│   ├── mutations.ts
│   └── prefetch.ts
└── app/**/_internal/ui/
```

Use route handlers only when HTTP entrypoints are actually needed.

## Domain

Create `src/domain/<feature>/<entity>.ts`.

Include:

- validation schema
- inferred TypeScript types
- pure helpers and invariants

Valibot example:

```ts
import {
  array,
  boolean,
  isoTimestamp,
  nullable,
  object,
  optional,
  picklist,
  pipe,
  string,
  trim,
  minLength,
  type InferOutput,
} from 'valibot'

export const WorkItemStatusSchema = picklist(['active', 'archived'])

export const WorkItemSchema = object({
  id: string(),
  title: pipe(string(), trim(), minLength(1)),
  description: nullable(string()),
  status: WorkItemStatusSchema,
  is_priority: boolean(),
  label_ids: array(string()),
  created_at: pipe(string(), isoTimestamp()),
  updated_at: pipe(string(), isoTimestamp()),
})

export const CreateWorkItemSchema = object({
  title: pipe(string(), trim(), minLength(1)),
  description: optional(nullable(string())),
  is_priority: optional(boolean()),
  label_ids: optional(array(string())),
})

export type WorkItem = InferOutput<typeof WorkItemSchema>
export type CreateWorkItem = InferOutput<typeof CreateWorkItemSchema>
```

## Use-Cases

Create:

- `src/use-cases/<feature>/ports.ts`
- `src/use-cases/<feature>/<feature>.ts`
- `src/use-cases/<feature>/types.ts` only when needed

Use-cases receive dependencies explicitly and validate commands with domain schemas.

```ts
import { parse } from 'valibot'
import { CreateWorkItemSchema, type CreateWorkItem } from '@/domain/work-item/work-item'
import type { WorkItemsRepository } from './ports'

type WorkItemsDeps = {
  workItems: WorkItemsRepository
}

export async function createWorkItem(
  deps: WorkItemsDeps,
  input: CreateWorkItem
) {
  const validated = parse(CreateWorkItemSchema, input)
  return deps.workItems.create(validated)
}
```

Port example:

```ts
export type WorkItemsRepository = {
  list: (params: WorkItemListParams) => Promise<PaginatedWorkItemsResult>
  getById: (id: string) => Promise<WorkItem>
  create: (input: CreateWorkItem) => Promise<WorkItem>
  update: (id: string, input: UpdateWorkItem) => Promise<WorkItem>
  archive: (id: string) => Promise<WorkItem>
  restore: (id: string) => Promise<WorkItem>
}
```

## Outbound Adapters

All persistence and external I/O belongs in `src/adapters/outbound/`.

- `*.operations.ts`: low-level DB/RPC/API calls and row mapping.
- `*.repository.ts`: factory that satisfies a use-case port.

```ts
export function createSupabaseWorkItemsRepository(
  supabase: SupabaseServerClient,
  userId: string
): WorkItemsRepository {
  return {
    list: (params) => listWorkItemsOperation(supabase, params),
    getById: (id) => getWorkItemOperation(supabase, id),
    create: (input) => createWorkItemOperation(supabase, userId, input),
    update: (id, input) => updateWorkItemOperation(supabase, id, input),
    archive: (id) => archiveWorkItemOperation(supabase, id),
    restore: (id) => restoreWorkItemOperation(supabase, id),
  }
}
```

Validate external payloads into domain types. Do not leak raw database rows into UI.

## Inbound Adapters

Server Actions and route handlers compose dependencies, apply auth/session context, map request input, call use-cases, and handle framework cache revalidation.

```ts
'use server'

export const createWorkItemAction = withAdminAuthContext(
  async (ctx, input: CreateWorkItem): Promise<WorkItem> => {
    const result = await createWorkItem(
      { workItems: createSupabaseWorkItemsRepository(ctx.supabase, ctx.userId) },
      input
    )
    revalidateWorkItemsRoutes()
    return result
  }
)
```

Do not put business decisions in inbound adapters. Delegate to use-cases.

### Auth Context

Inbound adapters must acquire the authenticated session before instantiating outbound adapters that need it. Use one of two patterns:

- **Higher-order wrapper** (`withAdminAuthContext`, `withAuthContext`, or project equivalent) — takes an async handler `(ctx, input) => result` and supplies a `ctx` containing the authenticated Supabase client and user id. Reject unauthenticated callers.
- **Imperative helper** (`requireAuth()` or `requireAdmin()`) — called at the top of a route handler; returns the session or throws/redirects.

If the target repo does not have such helpers, create them under `src/infrastructure/auth/` before adding new Server Actions. Never read `cookies()` directly inside a Server Action.

For pages, use the imperative form in a Server Component:

```tsx
// app/(protected)/admin/work-items/page.tsx
import { requireAdmin } from '@/infrastructure/auth/require-admin'
import { WorkItemsDashboard } from './_internal/ui/WorkItemsDashboard'

export default async function WorkItemsPage() {
  await requireAdmin() // redirects if unauthenticated
  return <WorkItemsDashboard />
}
```

### Cache Revalidation

After any mutation Server Action, call `revalidatePath` or `revalidateTag` for the affected routes/tags. Centralize per-feature invalidation in a helper so multiple actions stay consistent:

```ts
// src/adapters/inbound/next/server-actions/work-items.revalidate.ts
import { revalidatePath } from 'next/cache'

export function revalidateWorkItemsRoutes() {
  revalidatePath('/admin/work-items')
  revalidatePath('/admin/work-items/[id]', 'page')
}
```

Revalidation belongs in the inbound adapter, never in use-cases or outbound adapters.

## Feature-Local Actions

When UI needs to call a Server Action directly without TanStack Query (no cache entry, no optimistic update, no background refetch), place a thin wrapper next to the component:

```ts
// src/app/(protected)/profile/_internal/ui/UserEditForm/actions.ts
'use server'
export { updateCurrentUserProfileAction } from '@/adapters/inbound/next/server-actions/users'
```

The UI hook imports `./actions` and calls it inside a form submit. Use this only when TanStack Query would add noise (one-off operations, forms that reset on success, settings saves). Prefer server-state for anything cached, list-invalidating, or optimistic.

## Server-State

Use `src/ui/server-state/<feature>/` for all TanStack Query concerns:

- `keys.ts`: query key factory
- `queries.ts`: query options and query hooks
- `mutations.ts`: mutation hooks and invalidation
- `prefetch.ts`: SSR hydration helpers when needed

```ts
export const workItemKeys = {
  all: ['work-items'] as const,
  lists: () => [...workItemKeys.all, 'list'] as const,
  list: (params?: WorkItemListParams) => [...workItemKeys.lists(), params ?? {}] as const,
  details: () => [...workItemKeys.all, 'detail'] as const,
  detail: (id: string) => [...workItemKeys.details(), id] as const,
}
```

```ts
export function useWorkItems(params?: WorkItemListParams) {
  return useAuthenticatedQuery({
    queryKey: workItemKeys.list(params),
    queryFn: () => getWorkItemsAction(params),
    staleTime: STALE_TIME.FREQUENT_DATA,
    gcTime: GC_TIME.FREQUENT_DATA,
  })
}
```

Components consume server-state hooks, not inbound adapters.

## UI

- Page entrypoints in `app/` stay thin.
- Page-specific UI goes under route-local `_internal/ui/`.
- Shared UI goes under `src/ui/components/`.
- Components with logic use `composeHooks`; pure presentation components may skip it.
- User-facing text goes through the project i18n layer.
- Critical e2e controls get `data-testid`.

## Tests

Add tests proportional to the touched layers:

- domain and use-case unit tests for business rules
- repository/inbound tests when mapping or auth behavior is non-trivial
- server-state hook tests where cache behavior matters
- one e2e smoke path for a new user-facing feature

### Patterns by Layer

**Domain** — pure, no setup:

```ts
import { expect, test } from 'bun:test'
import { parse } from 'valibot'
import { CreateWorkItemSchema } from '@/domain/work-item/work-item'

test('CreateWorkItemSchema rejects empty title', () => {
  expect(() => parse(CreateWorkItemSchema, { title: '' })).toThrow()
})
```

**Use-case** — inject a fake repository satisfying the port. Never mock Supabase or fetch directly:

```ts
import { expect, test } from 'bun:test'
import { createWorkItem } from '@/use-cases/work-items/work-items'
import type { WorkItemsRepository } from '@/use-cases/work-items/ports'

const fakeRepo: Pick<WorkItemsRepository, 'create'> = {
  create: async (input) => ({ id: '1', ...input, created_at: '', updated_at: '' }),
}

test('createWorkItem forwards validated input', async () => {
  const result = await createWorkItem({ workItems: fakeRepo as WorkItemsRepository }, { title: 'hello' })
  expect(result.title).toBe('hello')
})
```

**Server-state hook** — wrap in `QueryClientProvider`, mock the inbound Server Action:

```tsx
import { renderHook, waitFor } from '@testing-library/react'
import { expect, mock, test } from 'bun:test'
import { useWorkItems } from '@/ui/server-state/work-items/queries'

mock.module('@/adapters/inbound/next/server-actions/work-items', () => ({
  getWorkItemsAction: async () => ({ items: [{ id: '1', title: 'A' }] }),
}))
```

**UI component** — mock `@/ui/server-state/**`, never outbound adapters.

## Anti-Patterns

```ts
// ❌ UI importing outbound adapter directly
import { listWorkItemsOperation } from '@/adapters/outbound/supabase/work-items.operations'

// ❌ Use-case importing inbound adapter or Next primitives
'use server' // in a use-case file
import { NextResponse } from 'next/server'
import { revalidatePath } from 'next/cache'

// ❌ Domain reaching into framework
import { getServerSession } from '@/infrastructure/auth/server-session'

// ❌ Inbound adapter with business rules
export const createWorkItemAction = async (input) => {
  if (input.title.length < 3) throw new Error('too short') // belongs in domain schema
  // ...
}

// ❌ Use-case holding a concrete Supabase client
export async function createWorkItem(supabase: SupabaseClient, input) { ... }
// ✅ should accept a port: deps: { workItems: WorkItemsRepository }
```

## Final Checklist

- [ ] Domain schema and inferred types exist
- [ ] Use-case module exists and has no UI/framework imports
- [ ] Ports exist where persistence or external I/O is abstracted
- [ ] Outbound repository implements the port
- [ ] Inbound Server Action or route handler composes dependencies
- [ ] Server-state keys/queries/mutations/prefetch exist when UI needs server data
- [ ] UI consumes only server-state hooks or thin local actions
- [ ] No UI to outbound adapter import leak
- [ ] `app/` entrypoints remain thin
- [ ] Tests cover the layer where behavior lives
