---
name: architector
description: Design and implement features using the project's hybrid Clean Architecture. Use when adding features, defining use-cases, creating inbound/outbound adapters, or making architectural decisions for the full-stack Next.js app.
---

# Architector Skill

Guide for architectural decisions and feature implementation in the template.

## Architecture Overview

```text
app/ui -> ui/server-state | feature-local actions.ts -> inbound adapters -> use-cases -> outbound adapters -> domain
```

## Layer Roles

- `src/domain/` — pure business schemas, invariants, helpers
- `src/use-cases/` — application scenarios, ports, feature-local types
- `src/ui/server-state/` — TanStack Query keys/hooks, SSR prefetch helpers, cache integration
- feature-local `actions.ts` — thin direct wrappers around Server Actions when TanStack Query is unnecessary
- `src/adapters/inbound/next/` — Server Actions and route-handler wiring
- `src/adapters/outbound/` — Supabase repositories, external APIs, transport
- `src/app/` — thin Next.js entrypoints only
- `src/ui/` — presentation components and view hooks
- `src/infrastructure/` — auth, i18n, config, logging, shared technical glue

## Core Rules

- Domain depends on nothing except domain
- Use-cases must not know about `use server`, `NextRequest`, `NextResponse`, `revalidatePath`
- UI must not import outbound adapters directly
- Server Actions are inbound adapters, not use-cases
- `ui/server-state` may import inbound adapters; `ui/components` may not

## Recommended Build Order

Follow: **Domain -> Use-Cases -> Outbound Adapters -> Inbound Adapters -> Server-State -> UI**

## Example Vertical Slice: Work Items

### 1. Domain

```ts
import { boolean, nullable, object, pipe, string, trim, minLength, type InferOutput } from 'valibot'

export const WorkItemSchema = object({
  id: string(),
  title: pipe(string(), trim(), minLength(1)),
  description: nullable(string()),
  archived: boolean(),
})

export type WorkItem = InferOutput<typeof WorkItemSchema>
```

### 2. Use-Cases

```ts
import type { WorkItemRepository } from './ports'

export async function listWorkItems(
  deps: { workItems: WorkItemRepository },
  filters: { archived?: boolean }
) {
  return deps.workItems.list(filters)
}
```

Typical feature structure:

```text
src/use-cases/work-items/
├── work-items.ts
├── ports.ts
└── types.ts
```

### 3. Outbound Adapters

```ts
export function createSupabaseWorkItemRepository(
  supabase: SupabaseServerClient
): WorkItemRepository {
  return {
    list: (filters) => listWorkItemsOperation(supabase, filters),
    create: (input) => createWorkItemOperation(supabase, input),
  }
}
```

### 4. Inbound Adapters

```ts
'use server'

export const listWorkItemsAction = withAdminAuth(async (ctx, filters) => {
  const workItems = createSupabaseWorkItemRepository(ctx.supabase)
  return listWorkItems({ workItems }, filters)
})
```

### 5. Server-State

```ts
export function useWorkItems(filters: WorkItemFilters) {
  return useQuery({
    queryKey: workItemKeys.list(filters),
    queryFn: () => listWorkItemsAction(filters),
  })
}
```

### 6. UI

- page entrypoint in `src/app/(protected)/admin/work-items/page.tsx`
- dashboard component under `_internal/ui/WorkItemsDashboard`
- form modal under `_internal/ui/WorkItemFormModal`
- labels reference panel under `_internal/ui/LabelsPanel`

## File Placement Guide

| Need                                      | Put it in                                   |
| ----------------------------------------- | ------------------------------------------- |
| Business rule                             | `src/domain/<feature>/`                     |
| Application scenario                      | `src/use-cases/<feature>/`                  |
| Query keys / query hooks / SSR prefetch   | `src/ui/server-state/<feature>/`            |
| One-off direct Server Action call from UI | feature-local `actions.ts`                  |
| Server Action                             | `src/adapters/inbound/next/server-actions/` |
| Route handler                             | `src/adapters/inbound/next/route-handlers/` |
| Supabase persistence                      | `src/adapters/outbound/supabase/`           |
| External HTTP client                      | `src/adapters/outbound/api/`                |
| Presentation component                    | `src/ui/` or `src/app/**/_internal/ui/`     |

## Checklist

- [ ] Domain schema exists
- [ ] Use-case module exists
- [ ] Port types exist where persistence is abstracted
- [ ] Outbound repository/gateway created
- [ ] Inbound Server Action or route handler created
- [ ] `ui/server-state/<feature>/` created for React Query integration
- [ ] UI uses `composeHooks`
- [ ] i18n messages added
- [ ] No `ui/app -> outbound adapter` import leaks

## References

- `docs/ARCHITECTURE/ARCHITECTURE.md`
- `docs/ARCHITECTURE/USE_CASES.md`
- `docs/ARCHITECTURE/DATA_ACCESS.md`
- `docs/ARCHITECTURE/FOLDER_STRUCTURE.md`
