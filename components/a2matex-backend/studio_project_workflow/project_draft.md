# Project Draft Feature Spec

**Status:** Backend complete; migrated, permissioned, unit-tested, and compiled HTTP/PostgreSQL verified
**Phase:** `a2matex-be/phase3.md` / Studio workflow checklist phase 6  
**Owner:** Backend; authored by Studio and read later by Web submission

## 1. Purpose and boundary

`ProjectDraft` stores the latest mutable Studio workflow snapshot for one Project. It prevents
silent overwrite between Studio sessions through optimistic versioning.

This phase persists JSON objects but does not validate graph topology, node semantics, bindings,
Runtime IR consistency, submission eligibility, assets, credentials, or execution history.

## 2. Decisions

- A Project has zero or one current draft. The first save creates it lazily.
- Studio creates local work without a cloud record. The first explicit Save to Cloud creates the
  Project, persists its returned Backend ID locally, and then saves the current content as draft
  version `1`; no empty draft is created.
- Cloud Project creation is retry-safe through the Publisher account plus case-insensitive normalized
  Project name. The Backend does not trust a client-generated identifier as resource authority.
- Cloud is authoritative for synchronized state. A different device downloads the cloud Project and
  draft instead of attaching an unrelated local Project.
- `GET` on an accessible Project with no draft returns `200` and `{ draft: null }`.
- First save uses `expectedVersion: 0` and creates version `1`; later saves use the current positive
  version.
- Changed content increments the version exactly once. An identical retry returns the same compact
  synchronization metadata without changing version, `savedAt`, or `savedById`.
- Stale differing content returns `409 Error.ProjectDraftVersionConflict` with
  `details: { currentVersion }`.
- The Backend computes `contentHash`; Studio never supplies version or audit fields.
- Draft content is limited to 1 MiB of canonical UTF-8 JSON.
- Archived Project drafts remain readable but cannot be saved.
- Customer/Publisher may read owned drafts. Only an active Publisher may save an owned draft.
  Admin may read any draft but does not edit Publisher workflow content in the MVP.
- Ordinary saves use PostgreSQL conditional writes, not a Redis lock.

## 3. Data model

```text
ProjectDraft
  id UUID PK
  projectId UUID FK Project, unique, required
  version INTEGER, required, starts at 1
  editorGraph JSONB object, required
  inputSchema JSONB object, required
  runtimeIr JSONB object, nullable
  validationSummary JSONB object, required, default {}
  testSummary JSONB object, required, default {}
  contentHash VARCHAR(64), required, lowercase SHA-256
  savedAt TIMESTAMPTZ, required
  savedById UUID FK User, required
  createdAt TIMESTAMPTZ, required
  updatedAt TIMESTAMPTZ, nullable
  deletedAt TIMESTAMPTZ, nullable
```

Foreign keys use `NO ACTION`. Standard row lifecycle timestamps are present, but this phase exposes
no draft delete API and stores no draft history; Project lifecycle remains the user-visible boundary.
Database checks require object-shaped JSON and a 64-character lowercase hash. Use `INTEGER`, not
`BIGINT`, so API versions remain ordinary safe JSON numbers.

## 4. API

All responses use the global envelope.

| Method and route                        | Permission and behavior                                                                                                   |
| --------------------------------------- | ------------------------------------------------------------------------------------------------------------------------- |
| `GET /api/v1/projects/:projectId/draft` | Customer/Publisher `OWN`; Admin unrestricted. Returns `{ projectStatus, draft: ProjectDraftResponse \| null }`.           |
| `PUT /api/v1/projects/:projectId/draft` | Publisher `OWN` only. Creates or atomically saves using `expectedVersion`, then returns compact synchronization metadata. |

Save body:

```json
{
  "expectedVersion": 0,
  "editorGraph": {},
  "inputSchema": {},
  "runtimeIr": null,
  "validationSummary": {},
  "testSummary": {}
}
```

The `GET` response always exposes `projectStatus`. A present draft exposes the five content fields
plus `id`, `projectId`, `version`, `contentHash`, and `savedAt`; `savedById` remains an internal audit
field. The `PUT` response returns only `id`, `version`, `contentHash`, and `savedAt`; this explicit
mutation-response exception avoids a second read while preventing the Backend from echoing up to
1 MiB of content. Input is strict: every non-null content field must be a JSON object and
`expectedVersion` must be a non-negative safe integer.

`validationSummary` and `testSummary` are informational snapshots in this phase. They are never
trusted as proof that a draft may be finalized or submitted.

## 5. Hash and idempotency

Canonicalize the object containing exactly `editorGraph`, `inputSchema`, `runtimeIr`,
`validationSummary`, and `testSummary`: recursively sort object keys, preserve array order and
preserve JSON scalar values. Compute lowercase SHA-256 using `node:crypto`.

The canonical UTF-8 bytes are also used for the 1 MiB size check. This keeps size and hash behavior
deterministic across request key orderings without adding a dependency.

The HTTP JSON parser uses a named 2 MiB transport ceiling so it can parse the envelope before the
feature applies the exact 1 MiB canonical-content limit. The transport ceiling is not the business
limit.

## 6. Concurrency algorithm

1. Resolve Project access and lifecycle before saving.
2. For `expectedVersion = 0`, insert version `1`. If the unique `projectId` constraint races,
   read the winner: matching hash is an idempotent retry returning compact synchronization metadata;
   otherwise return conflict.
3. For an existing draft, issue one conditional update on `projectId + expectedVersion` and only
   when the stored hash differs. Increment version and replace content/audit fields in that write.
4. If nothing updates, read the current row once. Matching hash is an idempotent retry; differing
   hash returns conflict with `currentVersion`.

Studio does not compare or merge workflow graphs in the MVP. On conflict it fetches the cloud draft
into a separate tab. If the user explicitly chooses the local content, Studio retries the normal
save using the freshly fetched cloud version; another concurrent winner produces another conflict.

The unique constraint and conditional update are the correctness boundary. No process-local or
Redis lock is used.

## 7. Authorization and errors

- Extract a feature-local Project access policy from the current private `ProjectService` access
  methods and reuse its typed predicate; repository code does not inspect role names.
- Unrestricted `null` wins over `OWN` for reads.
- `OWN` reads resolve a non-deleted Publisher account even when blocked.
- Saves require Publisher `OWN` permission and an active, non-deleted Publisher account;
  unrestricted `null` does not grant draft editing.
- Missing/cross-owner Project: `404 Error.ProjectNotFound`.
- Archived save: `409 Error.ProjectArchived`.
- Stale differing save: `409 Error.ProjectDraftVersionConflict` with
  `details: { currentVersion }`. The shared exception boundary preserves only explicitly supplied
  safe details.
- Oversized content: `413 Error.ProjectDraftPayloadTooLarge`.
- Invalid UUID, version, or JSON shape: `422` through an explicit `CustomZodValidationPipe` on Draft
  parameters and bodies, so behavior is consistent in compiled and `tsx` development runtimes.
- Missing route permission: `403` through `AccessTokenGuard`.

Do not add routine successful-save logs. Centralized request/error logging handles failures and must
never include draft JSON, credentials, secrets, or workflow input values.

## 8. Acceptance

- [x] First save creates version `1`; subsequent changed saves increment once.
- [x] Identical retries are successful no-ops, including after a lost response.
- [x] Stale differing saves cannot overwrite the winner.
- [x] Real-PostgreSQL concurrent saves allow only one winner.
- [x] Accessible missing draft returns `draft: null`.
- [x] Customer/Publisher access is resolved through the shared ownership policy; blocked Publishers
      retain read ownership but fail the active-Publisher save policy.
- [x] Cross-owner repository predicates and compiled HTTP read/save behavior are indistinguishable
      from not found.
- [x] Unrestricted Admin reads are supported across owners. Admin receives no unrestricted save;
      an account that also has Publisher remains subject to Publisher `OWN` for `PUT`.
- [x] Archived Project draft is readable and immutable.
- [x] Object shape, payload size, hash, version, and audit fields are enforced by code and focused
      tests; database constraints were deployed separately by the user.
- [x] Permission assignment preserves existing grants: Customer/Publisher have owned read, Publisher
      has owned save, and Admin has unrestricted read.
- [x] A blocked Publisher retains owned read through Customer but receives `403` for save/create;
      reactivation restores Publisher capability.
- [x] Compiled HTTP validation returns `422` for malformed UUIDs and invalid versions.
- [x] Failures are logged centrally without workflow payloads; routine successful saves add no logs.

## 9. Deferred

Cloud graph-validation rules begin in checklist phase 7, but Draft Save does not invoke them. The
future submission worker validates the immutable submitted revision. Studio conflict-tab/explicit-
overwrite UX remains a client integration task. Runtime IR consistency, immutable revisions,
submission, assets, credentials, and execution history remain separate later modules.
