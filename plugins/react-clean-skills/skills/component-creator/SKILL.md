---
name: component-creator
description: Create or refactor React UI components using the Fullstack AI Template component profile. Enforce composeHooks Smart/Dumb separation for any component with logic, strict file layout, no barrel exports, server-state boundaries, i18n messages, and data-testid rules. Use for components, forms, tables, view hooks, and UI state wiring.
---

# Component Creator Skill

Apply the Fullstack AI Template component profile. This is portable across React projects, but the defaults are strict. Use the target repo's existing equivalent only when it clearly replaces the same role.

## Source Priority

1. Target repository component docs and nearby components.
2. Target repository `AGENTS.md` / `CLAUDE.md`.
3. This default profile from `fullstack-ai-template`.

## Non-Negotiable Defaults

- Components with logic must use Smart/Dumb separation through `composeHooks(View)(useProps)`.
- `index.tsx` contains the View and exported Smart component. It is not a barrel file.
- `lib.ts` contains hook/view-model logic.
- `interfaces.ts` is created when types exceed five definitions or are shared across files.
- `messages.json` is used for component-local i18n when the project uses this convention.
- `styles.module.css` is used when styling cannot be expressed through UI-library props/classes.
- No `interface`; use `type`.
- No classes.
- No `any`.
- No inline `style={{}}`.
- No barrel exports.
- Critical interactive controls used in e2e get stable `data-testid`.

## State Placement Decision Tree

Before writing any stateful code, pick the layer by kind of state:

| Kind of state                                           | Where it lives                                  |
| ------------------------------------------------------- | ----------------------------------------------- |
| Server data (fetched, cached, revalidated)              | TanStack Query in `src/ui/server-state/<feature>/` |
| Form state (controlled inputs + validation)             | Mantine Form (`useForm`) inside `lib.ts`        |
| Page-level UI (filters, selection, sort, drawer state)  | Zustand store in `src/ui/stores/` or `src/app/**/_internal/stores/` |
| Global cross-page UI (theme, locale, session hint)      | React Context provider in `src/ui/providers/`   |
| Component-local ephemeral state (open/close, hover)     | `useState` in `lib.ts`                          |
| Derived data from one of the above                      | `useMemo` in `lib.ts`                           |

Do not put server data in Zustand/Context/useState. Do not store form drafts in server-state. When in doubt, prefer the rightmost column that fits; promote to a broader layer only when another component needs the same state.

## `_internal/` Segment Convention

Next.js App Router segments use an `_internal/` folder for code that is private to that segment:

```text
src/app/(protected)/admin/work-items/
├── page.tsx                     # thin entrypoint
└── _internal/
    ├── ui/
    │   ├── WorkItemsDashboard/  # page-specific, not imported from outside
    │   └── WorkItemFormModal/
    └── stores/                  # optional page-level Zustand store
```

Nothing under `_internal/` may be imported from outside its owning segment. This is a segment boundary, not a physical folder — enforce it in code review. Shared UI goes under `src/ui/components/` instead.

## When composeHooks Is Required

Use `composeHooks` when the component contains or depends on:

- `useState`, `useEffect`, `useMemo`, `useCallback`, refs, timers, or derived local state
- `useQuery`, `useMutation`, server-state hooks, cache invalidation
- Zustand or other page/global UI stores
- i18n formatting hooks such as `useIntl`
- action handlers that transform inputs before calling parent callbacks
- multiple sources of data or state

Allowed exceptions:

- Pure presentation component that only renders props.
- Generic components where `composeHooks` cannot preserve generic type parameters; document the exception in a short comment.
- Error boundary fallbacks or framework-specific components where hook separation would break lifecycle semantics; document why.

## Standard Structure

```text
Component/
├── index.tsx                  # View + Smart component, no re-exports
├── lib.ts                     # hooks, view-model assembly, local utilities
├── interfaces.ts              # if > 5 types or shared types
├── messages.json              # optional i18n keys
└── styles.module.css          # optional styles
```

Do not create `index.ts` barrel files. Import components directly:

```ts
import { UserCard } from '@/ui/components/UserCard'
```

## Boundary Rules

Before writing a hook, choose the correct layer:

| Concern | Correct location |
| --- | --- |
| View props assembly for one component | `Component/lib.ts` |
| Reusable UI-only logic | `src/ui/hooks/use-*.ts` |
| TanStack Query, query keys, invalidation, optimistic updates | `src/ui/server-state/<feature>/` |
| Thin direct Server Action wrappers without server-state | feature-local `actions.ts` |
| Application orchestration | `src/use-cases/<feature>/` |

### `Component/lib.ts` may import

- `@/ui/server-state/**`
- `@/ui/hooks/**`
- `@/ui/stores/**`
- `@/domain/**`
- `@/lib/**`
- `./actions`
- `./messages.json`
- local `./interfaces`

### `Component/lib.ts` must not import

- `@/adapters/inbound/**`
- `@/adapters/outbound/**`
- `@/adapters/api/**`
- `@/adapters/supabase/**`
- `@/adapters/transport/**`
- `@/app/**`

If a component needs server data, consume `ui/server-state` hooks. If it needs a one-off Server Action call without TanStack Query semantics, add a local `actions.ts` wrapper next to the component or hook.

## Workflow

1. Inspect nearby components and identify whether this is route-local UI or shared UI.
2. Create the standard file structure.
3. Define external props and view props as `type`.
4. Implement a pure `View` in `index.tsx`.
5. Implement `use<Component>Props` in `lib.ts`.
6. Move query/mutation/cache logic to `src/ui/server-state/<feature>/` before using it in `lib.ts`.
7. Add messages and `TranslationText` or the project i18n equivalent for user-facing text.
8. Add `data-testid` for modal/drawer triggers, destructive confirmations, table filters, save/create/delete actions, async controls, and e2e-critical controls.
9. Export the Smart component with explicit generics when external props differ from view props.

## Core Pattern

```tsx
// WorkItemCard/index.tsx
import { Badge, Button, Card, Group, Stack, Text } from '@mantine/core'
import { TranslationText } from '@/ui/components/TranslationText'
import { composeHooks } from '@/ui/hooks/compose-hooks'
import type { WorkItemCardProps, WorkItemCardViewProps } from './interfaces'
import { useWorkItemCardProps } from './lib'
import messages from './messages.json'

export function WorkItemCardView({
  title,
  description,
  isPriority,
  isLoading,
  onEdit,
  onArchive,
}: WorkItemCardViewProps) {
  return (
    <Card withBorder>
      <Stack gap="xs">
        <Group justify="space-between">
          <Text fw={600}>{title}</Text>
          {isPriority ? (
            <Badge color="orange">
              <TranslationText {...messages.priority} />
            </Badge>
          ) : null}
        </Group>
        <Text size="sm" c="dimmed">
          {description}
        </Text>
        <Group gap="xs">
          <Button
            variant="light"
            onClick={onEdit}
            loading={isLoading}
            data-testid="work-item-card-edit-btn"
          >
            <TranslationText {...messages.edit} />
          </Button>
          <Button
            color="red"
            variant="subtle"
            onClick={onArchive}
            data-testid="work-item-card-archive-btn"
          >
            <TranslationText {...messages.archive} />
          </Button>
        </Group>
      </Stack>
    </Card>
  )
}

export const WorkItemCard = composeHooks<WorkItemCardViewProps, WorkItemCardProps>(
  WorkItemCardView
)(useWorkItemCardProps)
```

```ts
// WorkItemCard/interfaces.ts
export type WorkItemCardProps = {
  workItemId: string
  onEdit?: (id: string) => void
}

export type WorkItemCardViewProps = {
  title: string
  description: string
  isPriority: boolean
  isLoading: boolean
  onEdit: () => void
  onArchive: () => void
}
```

```ts
// WorkItemCard/lib.ts
import { useCallback } from 'react'
import { useArchiveWorkItem } from '@/ui/server-state/work-items/mutations'
import { useWorkItem } from '@/ui/server-state/work-items/queries'
import type { WorkItemCardProps, WorkItemCardViewProps } from './interfaces'

export function useWorkItemCardProps({
  workItemId,
  onEdit,
}: WorkItemCardProps): WorkItemCardViewProps {
  const { data, isLoading } = useWorkItem(workItemId)
  const archiveWorkItem = useArchiveWorkItem()

  const handleEdit = useCallback(() => {
    onEdit?.(workItemId)
  }, [onEdit, workItemId])

  const handleArchive = useCallback(() => {
    archiveWorkItem.mutate(workItemId)
  }, [archiveWorkItem, workItemId])

  return {
    title: data?.title ?? '',
    description: data?.description ?? '',
    isPriority: data?.is_priority ?? false,
    isLoading,
    onEdit: handleEdit,
    onArchive: handleArchive,
  }
}
```

```json
{
  "priority": {
    "id": "workItemCard.priority",
    "defaultMessage": "Priority"
  },
  "edit": {
    "id": "workItemCard.edit",
    "defaultMessage": "Edit"
  },
  "archive": {
    "id": "workItemCard.archive",
    "defaultMessage": "Archive"
  }
}
```

## composeHooks Rules

- The hook receives all external props automatically; destructure only the props it needs.
- Props passed by the parent override hook results.
- When the hook transforms external props into different view props, use explicit generics:

```ts
export const WorkItemsList = composeHooks<WorkItemsListViewProps, WorkItemsListProps>(
  WorkItemsListView
)(useWorkItemsListProps)
```

- `composeHooks` can compose up to five hooks. Keep them ordered by dependency: data, selection/state, actions, derived labels.
- Do not pass every prop manually from component to hook.

## Server-State Guidance

Move code to `src/ui/server-state/<feature>/` if it contains:

- `useQuery`
- `useMutation`
- query key factories
- optimistic cache updates
- query invalidation
- realtime invalidation
- stream cache synchronization

`Component/lib.ts` stays focused on view-model assembly and interaction wiring.

## i18n Rules

- Put component-local messages in `messages.json` when the project follows this convention.
- Use `TranslationText` in JSX when possible.
- Use the i18n formatter in `lib.ts` only when the label must be computed before rendering.
- Do not leave user-facing strings hardcoded in committed UI unless the target project explicitly does not use i18n.

## Forms

For template-style forms:

- use Mantine Forms (`useForm`)
- validate with `createMantineValidator(ValibotSchema)`
- keep form state and submit wiring in `lib.ts`
- submit through server-state mutations or a thin local `actions.ts`
- keep the form View presentational

```tsx
// WorkItemFormModal/index.tsx
import { Button, Modal, Stack, Textarea, TextInput } from '@mantine/core'
import { TranslationText } from '@/ui/components/TranslationText'
import { composeHooks } from '@/ui/hooks/compose-hooks'
import type { WorkItemFormModalProps, WorkItemFormModalViewProps } from './interfaces'
import { useWorkItemFormModalProps } from './lib'
import messages from './messages.json'

export function WorkItemFormModalView({
  opened,
  onClose,
  form,
  onSubmit,
  isSubmitting,
}: WorkItemFormModalViewProps) {
  return (
    <Modal opened={opened} onClose={onClose} title={<TranslationText {...messages.title} />}>
      <form onSubmit={form.onSubmit(onSubmit)}>
        <Stack>
          <TextInput
            label={<TranslationText {...messages.titleLabel} />}
            {...form.getInputProps('title')}
            data-testid="work-item-form-title"
          />
          <Textarea
            label={<TranslationText {...messages.descriptionLabel} />}
            {...form.getInputProps('description')}
          />
          <Button type="submit" loading={isSubmitting} data-testid="work-item-form-submit">
            <TranslationText {...messages.submit} />
          </Button>
        </Stack>
      </form>
    </Modal>
  )
}

export const WorkItemFormModal = composeHooks<WorkItemFormModalViewProps, WorkItemFormModalProps>(
  WorkItemFormModalView
)(useWorkItemFormModalProps)
```

```ts
// WorkItemFormModal/lib.ts
import { useForm } from '@mantine/form'
import { CreateWorkItemSchema, type CreateWorkItem } from '@/domain/work-item/work-item'
import { createMantineValidator } from '@/lib/forms/create-mantine-validator'
import { useCreateWorkItem } from '@/ui/server-state/work-items/mutations'
import type { WorkItemFormModalProps, WorkItemFormModalViewProps } from './interfaces'

export function useWorkItemFormModalProps({
  opened,
  onClose,
}: WorkItemFormModalProps): WorkItemFormModalViewProps {
  const createWorkItem = useCreateWorkItem()
  const form = useForm<CreateWorkItem>({
    initialValues: { title: '', description: null, is_priority: false, label_ids: [] },
    validate: createMantineValidator(CreateWorkItemSchema),
  })

  const onSubmit = async (values: CreateWorkItem) => {
    await createWorkItem.mutateAsync(values)
    form.reset()
    onClose()
  }

  return { opened, onClose, form, onSubmit, isSubmitting: createWorkItem.isPending }
}
```

If `createMantineValidator` does not exist in the target project, create it under `src/lib/forms/`:

```ts
import { safeParse, type BaseSchema } from 'valibot'

export function createMantineValidator<T>(schema: BaseSchema<T, T, unknown>) {
  return (values: unknown) => {
    const result = safeParse(schema, values)
    if (result.success) return {}
    return Object.fromEntries(result.issues.map((i) => [i.path?.[0]?.key ?? '', i.message]))
  }
}
```

## Loading & Error States

Render explicit states in the View; do not show stale data while loading new data without indication.

- **Layout-preserving load** — use `Skeleton` with matching dimensions:

  ```tsx
  if (isLoading) return <Skeleton height={40} width="100%" />
  ```

- **Full-page load** — use `Loader` or `Center` + `Loader`:

  ```tsx
  if (isLoading) return <Center h={200}><Loader /></Center>
  ```

- **Recoverable error** — render inline `Alert` with retry:

  ```tsx
  if (error) return <Alert color="red" title={<TranslationText {...messages.errorTitle} />}>
    <Button onClick={onRetry}><TranslationText {...messages.retry} /></Button>
  </Alert>
  ```

- **Unrecoverable error** — let a parent `ErrorBoundary` catch (do not show raw `Error.message` to users).
- **Mutation pending** — use `loading` prop on the submit `Button`, disable destructive affordances during flight.

Do not render a blank screen for `isLoading`; do not render stale data for `error`. Map the `isLoading`/`error` pair explicitly in `lib.ts` into view props so the View stays declarative.

## Styling Alternatives to Inline Styles

When a layout or color cannot be expressed via a Mantine component prop, pick in this order:

1. **Mantine style props** — `c="blue.6"`, `bg="dark.7"`, `p="md"`, `gap="xs"`, `justify="space-between"`.
2. **Mantine layout primitives** — `Stack`, `Group`, `Flex`, `SimpleGrid`, `Grid.Col span={...}`.
3. **CSS Modules** — `import styles from './styles.module.css'` then `className={styles.primaryColumn}`. Use for custom grids, container queries, or animations.
4. **Palette import** — `import { darkColorScales } from '@/ui/themes/palette-dark'` when a Mantine color token is insufficient.
5. **CSS variables in module** — `color: var(--mantine-color-gray-0)` inside `.module.css`.

Never write `style={{ ... }}` in committed UI. If you catch yourself doing it, create a `styles.module.css` and move the rule there.

## Tests and Selectors

Component tests:

- mock `@/ui/server-state/**`
- avoid mocking inbound adapters from component tests
- assert rendered output and callbacks, not transport mechanics

Add `data-testid` for:

- modal and drawer triggers
- destructive confirmations
- table filter triggers and filter inputs
- save/create/delete actions
- async controls such as submit, cancel, refresh, history, and new item

## Final Checklist

- [ ] Component placement matches shared vs route-local usage
- [ ] `index.tsx` is not a barrel file
- [ ] Logic lives in `lib.ts`, not the View
- [ ] Components with logic use `composeHooks` or document an allowed exception
- [ ] External props and view props are typed with `type`
- [ ] Explicit generics are used when external props differ from view props
- [ ] Server data logic lives in `ui/server-state`
- [ ] `lib.ts` has no inbound/outbound/app imports
- [ ] User-facing text uses i18n
- [ ] e2e-critical controls have `data-testid`
- [ ] No inline styles, classes, `any`, or barrel exports were introduced
