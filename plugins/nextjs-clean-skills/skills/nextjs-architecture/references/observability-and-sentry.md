# Observability And Sentry

**Impact: HIGH**

Error reporting belongs in infrastructure, not in domain or use-case code. Split helpers by runtime:

- server runtime: server-only lazy loader for inbound adapters, route handlers, and Server Actions.
- client runtime: client-safe wrapper for Error Boundaries and UI events.

Do not import a server-only helper into a Client Component or client error boundary.

## Lazy Loader

Do not import `@sentry/nextjs` at module top in shared modules. The SDK is heavy and pulls in OpenTelemetry. Wrap it in a lazy loader so tests without a DSN and code paths that never report do not pay the import cost.

```ts
let p: Promise<typeof import('@sentry/nextjs') | null> | null = null
export function getSentry() {
  p ??= import('@sentry/nextjs').catch(() => null)
  return p
}
```

Callers handle the `null` case (no DSN, dynamic import failure) without throwing.

## PII Redaction

`sendDefaultPii: false` only stops Sentry's auto-attached request fields. The same payloads still travel through `event.message`, `event.exception.values[].value`, breadcrumbs, contexts, and `event.user.email`. Add a `beforeSend` that walks the event and rewrites email, phone, UUID, JWT, and provider key shapes to placeholder tokens.

Apply the same redaction to client-side `Replay` events if Replay is enabled; otherwise set `replaysSessionSampleRate: 0` explicitly so a future flag flip cannot accidentally enable it.

## User Context Without Email

`Sentry.setUser({ id, email })` is the most common PII leak — the email goes raw into `event.user` and is not touched by message/breadcrumb scrubbers. Identify users by id only; if domain-level triage matters, add an `email_domain` tag derived from the local-part split. On logout, call `setUser(null)` and clear scope tags.

## Route Handler Capture

Capture unexpected errors once at the inbound boundary. Route handlers return generic public-safe responses; Server Actions use a shared safe-action wrapper or `handleActionError`. Do not wrap every nested function manually or duplicate-capture the same exception.

```ts
try { return await handler(req) }
catch (error) {
  getSentry().then((s) => s?.captureException(error, { tags: { route, method } }))
  return new Response('Internal Server Error', { status: 500 })
}
```

Use-cases throw typed `ApiError`s; inbound adapters decide which become user-visible status codes and which become 500 + Sentry events. Do not echo `error.message`, stack frames, or DB hints to the client.

Reference: project observability boundary; Sentry SDK lazy + privacy posture.
