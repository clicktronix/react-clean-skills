# Supabase Persistence Boundaries

**Impact: HIGH**

Supabase access belongs behind outbound adapters or server-only infrastructure helpers.

Use-case owns the port:

```ts
export type UsersRepository = { update(input: UpdateUser): Promise<User> }
```

Outbound adapter implements it:

```ts
export function createSupabaseUsersRepository(client): UsersRepository {
  return { update: async (input) => mapUserRow(await updateRow(client, input)) }
}
```

Inbound/DAL composes it with the concrete client. UI and use-cases do not import Supabase clients.

RLS guardrails:

- enable RLS for user/tenant data.
- prefer `(select auth.uid())` in policies.
- pair update `using` with `with check` so users cannot self-promote.
- use private `security definer` helpers for role/membership lookups that would recurse through protected tables.

For exact SQL syntax and performance details, fetch current Supabase docs. The architectural rule is stable: Supabase is an outbound implementation detail.

## Bulk Writes Via RPC

Replace `Promise.all(N × update())` with a single `SECURITY INVOKER` RPC that takes `jsonb` and unpacks via `jsonb_to_recordset`. One round-trip, atomic, RLS still applied.

```sql
CREATE FUNCTION public.reorder_in_column(updates jsonb) RETURNS SETOF campaigns
LANGUAGE sql SECURITY INVOKER SET search_path = public AS $$
  UPDATE public.campaigns c SET column_id = u.column_id, position = u.position
  FROM jsonb_to_recordset(updates) AS u(id uuid, column_id uuid, position int)
  WHERE c.id = u.id RETURNING c.*;
$$;
```

The adapter calls `client.rpc('reorder_in_column', { updates })` and parses the rows through the domain schema.

## Error Mapping

Adapters must not `throw new Error(error.message)` from Postgres errors. Raw messages, details, and hints leak schema and end up in user-facing toasts. Map known Postgres SQLSTATE codes to typed `ApiError` subclasses (`ForbiddenError` for `42501`, `ConflictError` for `23505`, `NotFoundError` for `PGRST116`, `ValidationError` for `23502`, etc.). Put the raw payload into an `extra`/`responseBody` field for server logs only.

## Explicit Column Selection

Avoid `select('*')` on hot read paths. Migrations add or remove columns silently, network bandwidth is wasted, and internal flags leak. Maintain a per-entity column constant and select it explicitly; the same constant is used by `.select('...')` and by the row mapper, so adding a column is one diff.

Reference: Supabase as outbound adapter plus RLS as database-side authority.
