# Data Ownership, Cache, And TanStack

**Impact: HIGH**

Pick one owner for each read path.

| Need | Default owner |
| --- | --- |
| Read-heavy dashboard/list/detail | RSC + server-only DAL + serializable props |
| UI create/update/delete | Server Action -> use-case -> cache invalidation |
| External service/API command | Route Handler -> use-case -> JSON response |
| Realtime, polling, infinite scroll, optimistic client lifecycle | TanStack Query |
| URL-shareable filters/tabs/paging | URL search params, not a client store |

Do not back the same read with both RSC props and `useQuery`. Do not use TanStack Query in Server Components. Do not wrap a Server Action in `useMutation` only to invalidate a TanStack key when the affected read is RSC-owned.

Cache Components and tag APIs are framework syntax. The architecture decision is simpler:

- RSC-owned reads invalidate server cache tags.
- TanStack-owned reads invalidate/update TanStack keys.
- mixed pages must document which subset each owner controls.

For `cacheLife`, `cacheTag`, `revalidateTag`, `updateTag`, and `router.refresh()` syntax, fetch current Next.js docs.

## RSC DAL Hybrid Read

For reference data that needs synchronous first paint **and** client-side optimistic CRUD, combine RSC DAL fetch with TanStack `initialData`:

1. Server DAL fetches with `'use cache'` + `cacheTag(...)` and returns serializable rows.
2. RSC passes rows as `initialData` props to a Client island.
3. Client `useQuery` is keyed by feature and receives `initialData` plus an explicit freshness decision.
4. Mutations call a Server Action that runs the use-case and then `revalidateTag(tag, 'max')`.

```ts
useQuery({
  queryKey: keys.list(),
  queryFn: getListAction,
  initialData,
  initialDataUpdatedAt: initialData ? serverFetchedAt : undefined,
})
```

Use `initialDataUpdatedAt: serverFetchedAt` when the server timestamp is known. Use `initialDataUpdatedAt: 0` only when the seed is intentionally "render now, refetch immediately". If one query is consumed by multiple islands or TanStack should own freshness, prefer `prefetchQuery` + `HydrationBoundary` instead of hand-passing `initialData`.

Do not apply this hybrid to interactive search/filter lists (pure TanStack) or to mostly-static pages without writes (pure RSC props).

Reference: Next.js RSC/cache ownership; TanStack Query client async lifecycle.
