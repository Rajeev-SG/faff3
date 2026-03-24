# Review: GH05T-97/faff3-backend-revamp PR 4

## Context

- External repository: `GH05T-97/faff3-backend-revamp`
- Pull request: `#4`
- URL: <https://github.com/GH05T-97/faff3-backend-revamp/pull/4>
- Review submitted: `CHANGES_REQUESTED` on 2026-03-24

## Review summary

The change improves one real issue by moving client-owned IDs off the request body and onto `req.user.id`, but it also removes server-side role authorization from the protected route groups. As submitted, any authenticated user can reach admin, client, and freelancer routes that were previously gated by role.

## Findings

### 1. Role-based authorization was removed without a replacement

The PR replaces every `RoleMiddleware` check in the protected admin, client, and freelancer routers with `AuthMiddleware.authenticate`.

That is not a safe substitution in this codebase. The current auth middleware only:

1. validates a bearer token
2. loads the user
3. attaches `req.user`

It does not verify that the caller has the role required by the route.

Relevant code paths reviewed:

- `src/middleware/auth.middleware.ts`
- `src/routes/protected_routes/admin.routes.ts`
- `src/routes/protected_routes/client.routes.ts`
- `src/routes/protected_routes/freelancer.routes.ts`

Impact:

- any authenticated user can now hit `/admin/*`
- any authenticated user can now hit `/client/*`
- any authenticated user can now hit `/freelancer/*`

That is a privilege-escalation regression, not just a refactor.

## 2. Admin-only handlers can now be reached by non-admin users

The admin router now uses authentication without an admin-role check, and `AdminAuthController.signup` now trusts `req.user.role` as the allowed role value.

That means a non-admin token can reach the admin signup flow and satisfy the role validation with its own role instead of being rejected at the route boundary.

Relevant code paths reviewed:

- `src/routes/protected_routes/admin.routes.ts`
- `src/controllers/adminAuth.controller.ts`

Impact:

- admin-only behavior is no longer enforced at the route level
- the controller no longer guarantees that the caller is actually an admin

## Recommended fix

Restore a server-side role authorization layer after authentication. Either of these approaches would address the regression:

1. keep a role middleware and update it to read `req.user.role` instead of trusting request-body role input
2. add a new authorization middleware that runs after `AuthMiddleware.authenticate` and explicitly checks for the required role per route

## Validation notes

- Reviewed the submitted diff with `gh pr diff`
- Inspected the current PR state with `gh pr view`
- Cross-checked the live auth implementation in the cloned repo under `/Users/rajeev/Projects/faff3-backend-revamp`
- No application runtime or automated tests were executed for this review note
