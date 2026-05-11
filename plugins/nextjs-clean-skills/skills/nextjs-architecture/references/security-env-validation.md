# Validate Environment Variables

**Impact: HIGH**

Centralize environment access.

Pattern:

- `src/infrastructure/env/server.ts` validates server-only variables, either at module load or lazily on first access
- `src/infrastructure/env/client.ts` exposes only safe `NEXT_PUBLIC_*` variables
- use Valibot or another schema validator
- forbid direct `process.env` outside env modules with lint rules

Choose eager or lazy parsing deliberately, but keep validation centralized. Eager parsing fails fast at boot/build; lazy parsing avoids touching server-only values on paths that never need them.

For Supabase projects, prefer current key names and accept legacy names only as compatibility aliases:

- browser-safe: `NEXT_PUBLIC_SUPABASE_PUBLISHABLE_KEY`, with `NEXT_PUBLIC_SUPABASE_ANON_KEY` as a legacy fallback.
- server-only: `SUPABASE_SECRET_KEY`, with `SUPABASE_SERVICE_ROLE_KEY` as a legacy fallback.

Never prefix secrets with `NEXT_PUBLIC_`. Secret/service-role keys must be imported only from server-only modules.

Reference: Next.js environment variable bundling model and project env boundary.
