# 02 — Authentication

**Goal:** deny-by-default bearer-token auth, where each route declares which principal may call it.
**ADRs:** 012.

## Contracts (`src/auth/`)

```ts
export type PrincipalKind = 'agent' | 'google-script' | 'admin';
export interface Principal { kind: PrincipalKind }

export const Public = () => SetMetadata(IS_PUBLIC, true);
export const RequirePrincipal = (...kinds: PrincipalKind[]) => SetMetadata(PRINCIPALS, kinds);
export const CurrentPrincipal = createParamDecorator(/* returns req.principal */);
```

## Rules

1. `AuthGuard` is registered as `APP_GUARD` (global).
2. Routes marked `@Public()` pass without a token.
3. Every other route MUST have `@RequirePrincipal(...)`. A non-public route without it → **500 at boot**: validate in `onModuleInit` by scanning controllers with `DiscoveryService` + `Reflector` and throw if any handler lacks both decorators.
4. Token extraction: header `Authorization: Bearer <token>` only (no query parameters).
5. Token → kind: compare against `IRIS_TOKEN_AGENT`, `IRIS_TOKEN_GOOGLE_SCRIPT` and `IRIS_TOKEN_ADMIN` with `crypto.timingSafeEqual` on SHA-256 digests of both sides (equal length). Compare against **all three** every time (no early exit).
6. Missing or unknown token → `401 {"message":"Unauthorized"}`. Known token with the wrong kind for the route → `403 {"message":"Forbidden"}`.
7. Attach `req.principal`. Never log the token.
8. `GET /health` is `@Public()`.
9. The Habitica webhook route (spec 09) is `@Public()` and checks its path secret with the same constant-time helper `safeEqual(a: string, b: string): boolean` exported from `src/auth/safe-equal.ts`. On mismatch → **404**.

## Tests

- Unit `safe-equal.spec.ts`: equal, different, different lengths.
- E2E `auth.e2e-spec.ts` with a test-only controller registered in the testing module (`/v1/_test/agent` requires `agent`, `/v1/_test/admin` requires `admin`): no token → 401; wrong token → 401; agent token on the admin route → 403; correct → 200; `/health` without a token → 200.
- Unit: the boot validation throws for a handler without decorators.

## Done when

Lint, build, unit and e2e tests pass.
