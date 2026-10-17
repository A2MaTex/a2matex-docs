# Project Module Feature Spec

**Status:** Implemented, migrated, permissioned, and production-equivalent CRUD smoke-tested;
dedicated multi-user checks pending  
**Phase:** `a2matex-be/phase2.md` / Studio workflow checklist phase 5  
**Owner:** Backend; consumed by Studio

## 1. Purpose and boundary

`Project` is the Publisher-owned cloud container that Studio creates and opens. This phase stores
only identity, metadata, ownership and lifecycle.

Out of scope: `ProjectDraft`, graph/Runtime IR JSON, draft conflict/version handling, assets,
validation/test summaries, workflow revisions, submissions, execution history, public delete and
unarchive.

## 2. Contract decisions for approval

These values resolve the remaining choices in `phase2.md` and become final when this spec is
approved:

- Statuses: `ACTIVE`, `ARCHIVED`.
- One Project belongs to exactly one `PublisherAccount`; its trimmed name is unique within that
  Publisher account using case-insensitive comparison, including archived Projects.
- Name is trimmed and must contain `1..255` characters.
- Repeating cloud creation with the same active normalized Publisher/name returns the existing
  `{ id }`; an archived match returns `409 Error.ProjectArchived`.
- Renaming to another Project's normalized name returns `409 Error.ProjectNameAlreadyExists`.
- Description is optional; when supplied it is trimmed, limited to 4,000 characters, and an empty
  result is stored as `NULL`.
- Create is unscoped and assigned only to Publisher; ownership is always derived from the
  authenticated user's active, non-deleted Publisher account. `OWN` update/archive requires the
  same active Publisher account. Unrestricted Admin update/archive bypasses owner Publisher status
  but still obeys Project lifecycle rules.
- Blocking a Publisher removes the Publisher role but retains the non-deleted `PublisherAccount` as
  the ownership anchor. The remaining Customer role can read owned Projects but cannot mutate them.
- `UserStatus.BANNED` denies all authenticated access; `PublisherAccountStatus.BLOCKED` removes
  Publisher capability without removing read-only ownership access.
- Archived Projects remain readable but immutable.
- Archive is idempotent: archiving an already archived accessible Project returns the standard
  success response.
- List returns all statuses when the filter is omitted; optional `status=ACTIVE|ARCHIVED` selects one
  status.
- Ordinary Project operations do not use a Redis distributed lock.

## 3. Data model

```text
Project
  id UUID PK
  publisherAccountId UUID FK PublisherAccount, required
  name VARCHAR(255), required
  description TEXT, nullable
  status ProjectStatus, default ACTIVE
  createdById UUID FK User, required
  updatedById UUID FK User, nullable
  deletedById UUID FK User, nullable
  createdAt TIMESTAMPTZ, required
  updatedAt TIMESTAMPTZ, nullable
  deletedAt TIMESTAMPTZ, nullable
```

All foreign keys use `NO ACTION`. Add the partial index below; add no speculative detail index
unless the reviewed query plan shows the UUID primary key is insufficient.

Strict Zod API contracts trim and reject blank names, normalize empty descriptions to `NULL`, and
enforce the `1..255` name and `4,000` description limits. This phase does not duplicate those text
rules as database check constraints.

```sql
CREATE INDEX "Project_publisherAccountId_status_idx"
ON "Project" ("publisherAccountId", status)
WHERE "deletedAt" IS NULL;
```

The SQL-first migration stops after review. The user deploys it; only then run `prisma db pull`,
generate the client and verify the resulting relations.

## 4. API contract

All routes use Bearer authentication, route permissions, strict Zod input, the global response
envelope and existing pagination format.

| Method and route                          | Behavior                                                                                                                                 |
| ----------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------- |
| `POST /api/v1/projects`                   | Unscoped Publisher capability; create under caller's active Publisher account and return `{ id }`. Body: `name`, optional `description`. |
| `GET /api/v1/projects`                    | `OWN` lists owned Projects; Admin unrestricted scope lists all matching Projects.                                                        |
| `GET /api/v1/projects/:projectId`         | `OWN` reads an owned Project; Admin unrestricted scope can read any Project.                                                             |
| `PUT /api/v1/projects/:projectId`         | Update `name` and/or `description` on an active Project and return the standard success message.                                         |
| `PUT /api/v1/projects/:projectId/archive` | Atomically transition active to archived and return the standard success message; repeat is idempotent.                                  |

Read responses expose `id`, `publisherAccountId`, `name`, `description`, `status`, `createdAt`, and
`updatedAt`. Create returns `{ id }`; update/archive return the standard success message. Do not
expose audit user IDs, `deletedAt`, graph data or publication data. List order is `updatedAt DESC
NULLS LAST, createdAt DESC, id DESC` through Prisma ORM.

## 5. Authorization and domain behavior

- The controller passes the authenticated user ID to create. Existing-resource operations also pass
  `@ActivePermissionScopes()` to the service.
- The Project module consumes scopes only where an existing resource must be filtered. It does not
  inspect role names, Redis cache values or the complete role-permission payload.
- Scope precedence is unrestricted `null` over `OWN`; no matching permission is denied by the guard.
- For unrestricted access, do not resolve the caller's Publisher account. Admin can list, read,
  update and archive across ownership boundaries while lifecycle rules still apply.
- For `OWN` reads, resolve the caller's non-deleted Publisher account by user ID regardless of
  `ACTIVE` or `BLOCKED` status. Customer without a Publisher account receives an empty list or
  not-found detail.
- For create, derive ownership from the authenticated user and require an `ACTIVE`, non-deleted
  Publisher account. For `OWN` mutations, require the same active account. Unrestricted Admin
  mutations do not resolve or validate the owner Publisher account.
- `OWN` detail/update/archive queries include both `projectId` and `publisherAccountId`;
  unrestricted queries omit the owner predicate. All paths exclude soft-deleted records.
- Cross-owner `OWN` access and missing IDs return the same not-found error to prevent resource
  enumeration.
- Invalid UUID/input is rejected before the service runs.
- Mutating an archived Project returns a stable conflict error; archive itself remains idempotent.
- Do not add routine create/update/archive success logs. Use centralized request/error logging for
  failures, and never log descriptions or future workflow/private input payloads.

Stable errors:

- Missing and cross-owner Project: `404 Error.ProjectNotFound`.
- Metadata mutation of an archived Project: `409 Error.ProjectArchived`.
- Invalid request data: `422` from the global Zod validation pipe.
- Missing route permission: `403` from `AccessTokenGuard`.

## 6. Repository and transaction rules

- Follow existing Prisma repository/module/controller patterns.
- Do not query the database inside loops.
- Accept a typed Project access predicate from the service; do not interpret role names or raw
  permission records in the repository.
- Resolve a Publisher account only for `OWN` or create flows, never for unrestricted Admin access.
- Set `createdById` at create; set `updatedById` and `updatedAt` on every mutation.
- Update only an owned, non-deleted `ACTIVE` row.
- Archive with an atomic conditional update. If no row changes, perform one ownership-safe lookup to
  distinguish an already archived owned Project from not found; never reveal cross-owner existence.
- No distributed lock is needed because conditional database writes enforce the lifecycle.

## 7. Permissions and implementation sequence

1. Approve section 2 and stable error codes.
2. Create/review the forward-only SQL migration; do not deploy it from implementation tooling.
3. After user deployment, introspect Prisma and implement contracts, repository, service,
   controller and module registration.
4. Create route permissions through `POST /api/v1/permissions`; do not run the permission seed.
5. Assign the unscoped create permission only to Publisher, Customer and Publisher `OWN` read
   permissions, Publisher `OWN` update/archive permissions, and Admin unrestricted
   list/detail/update/archive permissions without removing existing IDs.
6. Update Publisher blocking so the same transaction soft-deletes the Publisher `UserRole` while
   retaining Customer membership and the `PublisherAccount`; reactivation restores the role.
7. Run focused contract/service checks, build, lint, formatting and HTTP smoke tests.

## 8. Acceptance checklist

- [x] Repeating cloud creation with the same normalized Publisher/name returns the existing active
      Project ID; an archived match returns `409 Error.ProjectArchived` through the same recovery
      path.
- [ ] Active Publisher can create, list, read, update and archive only its own Projects. CRUD is
      verified; second-Publisher isolation is pending.
- [ ] Blocked former Publisher can list/read owned Projects through Customer `OWN` permission but
      cannot create, update, archive, save drafts or submit.
- [ ] Admin can list/read/update/archive any Project without resolving a Publisher account.
- [x] Admin plus Publisher scope aggregation resolves to unrestricted access.
- [x] Blank/oversized names and invalid UUIDs return validation errors. Oversized-description
      verification remains pending.
- [ ] Customer and blocked/deleted Publisher cannot create or mutate.
- [ ] Cross-owner access is indistinguishable from not found.
- [x] Archived Project is readable, immutable and repeat archive is idempotent.
- [ ] Soft-deleted Projects never appear in normal reads.
- [ ] Pagination and deterministic sorting are stable.
- [x] Permission assignment preserves every existing role permission.
- [x] Routine Project mutations do not add success logs; centralized request/error logging handles
      failures.
- [x] Migration deployment, foreign keys, index and audit fields are verified before database success
      is claimed.

Verification note: use the compiled server (`npm run start:prod`) for DTO-validation proof. The
current `tsx` development launchers do not emit Nest decorator metadata, so implicit global DTO
validation is not reliable there.
