---
name: component-creator
description: ALWAYS use when working with React components or hooks. Use for creating, editing, or refactoring UI components, forms, tables, custom hooks, or any React elements. Enforces composeHooks pattern, Mantine UI, translations, and proper file structure.
---

# Component Creator Skill

Creates React components following the template's Smart/Dumb separation.

## Workflow

1. Create the file structure
2. Build a pure `View`
3. Put view-model logic into `lib.ts`
4. Use `ui/server-state` for TanStack Query concerns
5. Use feature-local `actions.ts` only for thin one-off Server Action calls
6. Export with `composeHooks`
7. Add i18n and `data-testid` where the interaction is critical

## Standard Structure

```text
ComponentName/
├── index.tsx
├── lib.ts
├── messages.json
└── styles.module.css
```

Add `interfaces.ts` when the component accumulates many types.

## Boundaries

### `lib.ts` may import

- `@/ui/server-state/**`
- `@/ui/hooks/**`
- `@/domain/**`
- `@/lib/**`
- `./actions`

### `lib.ts` must NOT import

- `@/adapters/inbound/**`
- `@/adapters/outbound/**`
- `@/app/**`

If the code needs:

- `useQuery`, `useMutation`, query keys, cache invalidation -> `src/ui/server-state/<feature>/`
- a one-off direct Server Action call -> local `actions.ts`
- business orchestration -> `src/use-cases/<feature>/`

## Core Pattern

```tsx
// index.tsx
import { Button, Card, Stack, Text } from '@mantine/core'
import { composeHooks } from '@/ui/hooks/compose-hooks'
import { TranslationText } from '@/ui/components/TranslationText'
import { useWorkItemCardProps, type WorkItemCardProps, type WorkItemCardViewProps } from './lib'
import messages from './messages.json'

export function WorkItemCardView({
  title,
  description,
  isPriority,
  onEdit,
}: WorkItemCardViewProps) {
  return (
    <Card withBorder>
      <Stack gap="xs">
        <Text fw={600}>{title}</Text>
        <Text size="sm" c="dimmed">
          {description}
        </Text>
        {isPriority ? <TranslationText {...messages.priority} size="xs" /> : null}
        <Button onClick={onEdit} data-testid="work-item-card-edit-btn">
          <TranslationText {...messages.edit} />
        </Button>
      </Stack>
    </Card>
  )
}

export const WorkItemCard = composeHooks<WorkItemCardViewProps, WorkItemCardProps>(
  WorkItemCardView
)(useWorkItemCardProps)
```

```ts
// lib.ts
import { useCallback } from 'react'
import { useWorkItem } from '@/ui/server-state/work-items/queries'

export type WorkItemCardProps = {
  workItemId: string
  onEdit?: (id: string) => void
}

export type WorkItemCardViewProps = {
  title: string
  description: string
  isPriority: boolean
  onEdit: () => void
}

export function useWorkItemCardProps({
  workItemId,
  onEdit,
}: WorkItemCardProps): WorkItemCardViewProps {
  const { data } = useWorkItem(workItemId)

  const handleEdit = useCallback(() => {
    onEdit?.(workItemId)
  }, [onEdit, workItemId])

  return {
    title: data?.title ?? '',
    description: data?.description ?? '',
    isPriority: data?.is_priority ?? false,
    onEdit: handleEdit,
  }
}
```

## Rules

- `index.tsx` is presentation only
- `lib.ts` assembles view props only
- all user-facing text goes through translations
- prefer Mantine props or CSS Modules over inline styles
- mock `ui/server-state` in component tests, not adapters
- add `data-testid` to destructive or workflow-critical controls

## Reference

- `docs/ARCHITECTURE/COMPONENT_PATTERNS.md`
