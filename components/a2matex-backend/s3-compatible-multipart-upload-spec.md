# S3-Compatible Multipart Upload Specification

Status: implementation handoff

Audience: developers who are new to backend and object-storage workflows

Reference implementation: HRM commit `4ea1aeb42ad3a7134e04bc5a492b9bfacb20c281` on `feat/s3-stream`

## 1. Purpose

This document describes the multipart-upload feature implemented by the reference commit and how to reproduce the same focused feature in another project using S3-compatible storage.

The feature supports:

- Creating an S3 multipart-upload session.
- Generating presigned `PUT` URLs for file parts.
- Uploading large file parts directly from the client to S3.
- Listing the parts already accepted by S3 so an upload can resume.
- Completing the multipart upload and creating the application's permanent file record.
- Aborting an upload.
- Cleaning up expired multipart uploads.
- Generating a short-lived presigned `GET` URL for the completed file.

This specification intentionally stays close to the reference commit. It does not introduce a queue, event system, additional session tables, an expanded state machine, or a new storage architecture.

## 2. Why multipart upload is different from CRUD

A normal create API usually receives one request and writes one database row. Multipart upload spans several requests and two systems:

```text
Application database                   S3-compatible storage
--------------------                   ---------------------
Upload-session row                     Multipart upload
Permanent file row                     Temporary file parts
                                       Completed file object
```

The database and S3 cannot share one transaction. For example, S3 may successfully assemble the object and then the database insert may fail. The implementation therefore needs small compensation actions, such as aborting an S3 session when its database row cannot be created or deleting an assembled object when the permanent file record cannot be saved.

## 3. Terms

### Bucket

A bucket is the top-level storage container. It is similar to a private container, not a normal local directory.

### Object key

An object key is the name used to store one file inside a bucket, for example:

```text
uploads/123e4567-e89b-12d3-a456-426614174000.pdf
```

The reference flow generates a UUID for the filename so two users uploading `report.pdf` do not overwrite each other.

### Multipart upload

Multipart upload allows one large file to be split into numbered pieces. S3 keeps the pieces temporarily and assembles them only after the backend calls `CompleteMultipartUpload`.

### Part number

Each part has a number that determines its position in the final file. Part 1 contains the beginning of the file, part 2 follows it, and so on.

Amazon S3 supports part numbers 1 through 10,000. Every part except the last normally needs to be at least 5 MiB. Verify these limits with the selected S3-compatible provider.

### Upload ID

When the backend starts a multipart upload, S3 returns an opaque upload ID. S3 uses the combination of object key and upload ID to identify that multipart upload.

### Presigned URL

A presigned URL is a temporary URL that permits one specific S3 operation without exposing the application's S3 access key and secret to the client.

For this feature, each upload URL allows the client to upload one specific part number. Anyone who obtains the URL can use it until it expires, so it must not be logged or shared.

### ETag

After accepting a part, S3 returns an ETag response header. The client keeps the ETag and sends it to the backend during completion. S3 uses the submitted part numbers and ETags to assemble the file.

Treat the ETag as an opaque storage value. Do not assume it is always an MD5 checksum.

### Upload session

The upload-session database row connects the application's file metadata to S3's upload ID and object key. It also records whether the upload is still active, completed, or aborted.

### Expiration

Uploaded parts consume storage even before the file is completed. A session therefore has an expiry time. A scheduled job aborts expired sessions and marks them as aborted.

## 4. High-level flow

```text
1. Client -> Backend: send file metadata and requested part numbers
2. Backend -> Database: reuse or create an UPLOADING session
3. Backend -> S3: create multipart upload when the session is new
4. Backend -> S3: generate presigned URLs for the requested parts
5. Backend -> Client: return upload ID, file key, and URLs
6. Client -> S3: PUT every part directly
7. Client -> Backend: ask which parts S3 has received, if resuming
8. Client -> Backend: send part numbers and ETags to complete
9. Backend -> S3: assemble the parts
10. Backend -> Database: mark session COMPLETED and create file record
11. Backend -> Client: return the permanent file metadata
```

The large file bytes travel from the client directly to S3. They do not pass through the backend.

## 5. Minimal data model

Use the same small session record as the reference commit:

```sql
CREATE TABLE IF NOT EXISTS static_upload_sessions (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  file_hash TEXT NOT NULL,
  upload_id TEXT NOT NULL,
  file_key TEXT NOT NULL,
  status VARCHAR(50),
  expires_at TIMESTAMP,

  created_at TIMESTAMP NOT NULL DEFAULT NOW(),
  updated_at TIMESTAMP,
  deleted_at TIMESTAMP,
  created_by UUID REFERENCES users(id),
  updated_by UUID REFERENCES users(id),
  deleted_by UUID REFERENCES users(id)
);

CREATE INDEX IF NOT EXISTS idx_static_upload_sessions_file_hash
ON static_upload_sessions(file_hash)
WHERE deleted_at IS NULL;

CREATE UNIQUE INDEX idx_unique_uploading_session
ON static_upload_sessions(file_hash)
WHERE status = 'UPLOADING';
```

If the target project does not have shared audit fields or a `users` table, map `created_by`, `updated_by`, and `deleted_by` to its existing ownership convention. Do not add a second ownership model only for this feature.

### Field meanings

| Field | Meaning |
| --- | --- |
| `id` | Application database identifier for the session |
| `file_hash` | Hash of stable client metadata used to find an existing active session |
| `upload_id` | Opaque multipart-upload ID returned by S3 |
| `file_key` | Object key generated by the backend |
| `status` | `UPLOADING`, `COMPLETED`, or `ABORTED` |
| `expires_at` | Time after which cleanup may abort the session |
| Audit fields | Existing project convention for creator, updater, and soft deletion |

The reference commit calculates `file_hash` from:

```text
user ID + filename + file size + last-modified timestamp
```

It then applies MD5 to that metadata string. This hash is used only to recognize the same user's upload attempt. It is not a hash of the file bytes and must not be described as file-integrity verification.

## 6. Session statuses

Keep the three statuses used by the reference commit:

| Status | Meaning |
| --- | --- |
| `UPLOADING` | S3 multipart upload exists and may receive parts |
| `COMPLETED` | S3 assembled the object and the permanent file record was created |
| `ABORTED` | The multipart upload was cancelled or cleaned up |

Normal transitions are:

```text
UPLOADING -> COMPLETED
UPLOADING -> ABORTED
```

No additional state machine is required for this implementation.

## 7. Storage operations

Add the multipart methods to the project's existing storage abstraction. Do not split them into multiple new interfaces unless the target project already follows that pattern.

The active flow needs these operations:

```go
GetMultipartSessionTTL() int
CreateMultipartUpload(key, contentType string) (uploadID string, err error)
GeneratePresignedURLForPart(key, uploadID string, partNumber int) (url string, err error)
CompleteMultipartUpload(key, uploadID string, parts []Part) error
AbortMultipartUpload(key, uploadID string) error
ListUploadedParts(key, uploadID string) ([]Part, error)
```

Completion reuses the project's existing object-metadata and delete-object methods. Presigned download reuses its existing share/presign-GET method. They are not new multipart abstractions.

The session TTL comes directly from storage configuration through `GetMultipartSessionTTL`. It does not require a separate service or table.

The reference commit also added `ListAllMultipartUploads`, but the implemented cleanup flow does not use it. It is not required to reproduce the feature described here.

Use the method names and context style already established by the target project. If its storage methods already accept `context.Context`, preserve that pattern.

### Supporting types

```go
type Part struct {
	PartNumber int
	ETag       string
}
```

## 8. API contract

The paths below match the reference commit. Another project may rename the common prefix while keeping the request and response behavior.

### 8.1 Initialize or resume and get part URLs

```http
POST /api/v1/statics/files/multipart/urls
Content-Type: application/json
```

Request:

```json
{
  "prefix": "uploads",
  "content_type": "application/pdf",
  "file_extension": "pdf",
  "file_name": "report.pdf",
  "file_size": 73400320,
  "last_modified": 1793000000000,
  "part_numbers": [1, 2, 3]
}
```

Response:

```json
{
  "upload_id": "provider-upload-id",
  "file_key": "uploads/123e4567-e89b-12d3-a456-426614174000.pdf",
  "urls": [
    {"part_number": 1, "url": "https://storage.example/..."},
    {"part_number": 2, "url": "https://storage.example/..."},
    {"part_number": 3, "url": "https://storage.example/..."}
  ]
}
```

Rules from the reference implementation:

- Require at least one part number.
- Return at most 100 URLs in one request.
- Generate each presigned URL with a short expiry; the reference uses 15 minutes.
- Return the same active session when the same `file_hash` is found and is not expired.
- Generate a new UUID-based object key when no active session exists.

Minimal validation for reimplementation:

- Part numbers must be unique and between 1 and the provider maximum.
- File size must be positive and within the application's configured limit.
- Content type and extension must follow the target project's allowed-file policy.
- Restrict `prefix` to server-approved values. Do not allow an arbitrary bucket namespace.

### 8.2 Upload each part directly to S3

For every returned URL, the client performs:

```http
PUT <presigned-url>

<raw bytes for this part>
```

The client stores the ETag returned by S3. Browser CORS must expose the `ETag` response header.

The client may upload several parts concurrently, but it should use a small bounded number such as 3 to 6 rather than starting every part at once.

### 8.3 Get uploaded parts

```http
GET /api/v1/statics/files/multipart/uploaded-parts?upload_id=...&file_key=...
```

Response:

```json
{
  "parts": [
    {"part_number": 1, "etag": "\"etag-value-1\""},
    {"part_number": 2, "etag": "\"etag-value-2\""}
  ]
}
```

The backend calls S3 `ListParts`. It must follow pagination because S3 returns a limited number of parts per page.

The client compares this result with the local part list and uploads only the missing parts.

### 8.4 Complete the upload

```http
POST /api/v1/statics/files/multipart/complete
Content-Type: application/json
```

Request:

```json
{
  "upload_id": "provider-upload-id",
  "file_key": "uploads/123e4567-e89b-12d3-a456-426614174000.pdf",
  "file_name": "report.pdf",
  "parts": [
    {"part_number": 1, "etag": "\"etag-value-1\""},
    {"part_number": 2, "etag": "\"etag-value-2\""}
  ]
}
```

Response:

```json
{
  "file": {
    "static_id": "123e4567-e89b-12d3-a456-426614174000",
    "filename": "report.pdf",
    "file_type": "application/pdf",
    "file_size": 73400320
  }
}
```

The backend sorts parts by part number before calling S3 completion.

### 8.5 Abort

```http
POST /api/v1/statics/files/multipart/abort
Content-Type: application/json
```

```json
{
  "upload_id": "provider-upload-id",
  "file_key": "uploads/123e4567-e89b-12d3-a456-426614174000.pdf"
}
```

The backend aborts the S3 multipart upload and then marks its database session `ABORTED`.

### 8.6 Clean up stale uploads

The reference commit supports both:

- A scheduled job configured by `cleanUpStaleUploads`.
- A manual `POST /api/v1/statics/files/multipart/cleanup` endpoint.

The job selects sessions where:

```text
status = UPLOADING
expires_at < current time
```

It aborts each matching S3 multipart upload and bulk-updates successfully aborted session IDs to `ABORTED`.

The default session TTL in the reference commit is 48 hours.

The manual cleanup endpoint is optional in another project. If retained, protect it with an administrator or operations permission.

### 8.7 Generate a presigned download URL

```http
GET /api/v1/statics/files/{id}/presigned-url
```

Response:

```json
{
  "url": "https://storage.example/..."
}
```

The backend resolves the permanent file record by ID and generates a short-lived presigned `GET` URL. The reference commit uses five minutes.

## 9. Detailed backend behavior

### 9.1 Initialize or reuse a session

1. Build `file_hash` from user ID, filename, file size, and last-modified time.
2. Query for a matching `UPLOADING` session.
3. If it exists and has not expired, reuse it.
4. Otherwise, acquire the existing Redis distributed lock for that `file_hash`.
5. Query the database again after acquiring the lock because another request may have created the session.
6. Generate a UUID for the future static-file ID.
7. Build the object key from approved prefix, UUID, and extension.
8. Call S3 `CreateMultipartUpload`.
9. Insert the `UPLOADING` database row with its expiry.
10. If the database insert fails, call S3 abort so temporary storage is not orphaned.
11. Generate URLs for the requested part numbers and return them.

The distributed lock reduces concurrent creation of two S3 sessions for the same metadata. The partial unique index remains the database protection against two `UPLOADING` rows with the same `file_hash`.

### 9.2 List uploaded parts

1. Receive `upload_id` and `file_key`.
2. Call S3 `ListParts` with pagination.
3. Map S3 part numbers and ETags to the API response.

For a new project, perform a database lookup first and confirm that the exact `upload_id + file_key` pair belongs to a visible session. This uses the same table and avoids adding new architecture.

### 9.3 Complete

1. Validate that `file_key` contains the UUID used as the future static-file ID.
2. Map client parts to storage parts.
3. Sort by part number.
4. Call S3 `CompleteMultipartUpload`.
5. Start the compensation guard: if later database work fails, call `DeleteObject(file_key)`.
6. Load the upload session using `upload_id`.
7. Change its status to `COMPLETED`.
8. Call `HeadObject`/`GetObjectMetadata` to read final size and content type.
9. Create the permanent static/file record using the UUID from the key.
10. Commit the local database transaction.
11. Return the permanent file metadata.

The database updates in steps 7 and 9 belong to one local transaction. S3 completion in step 4 is outside that transaction in real system terms, even if the use-case middleware wraps the Go function.

### 9.4 Abort

1. Call S3 `AbortMultipartUpload`.
2. Find the database session by `upload_id + file_key`.
3. Change its status to `ABORTED`.

For a new project, query and authorize the database session before calling S3. This prevents one user from providing another upload's identifiers.

### 9.5 Cleanup

1. Query expired `UPLOADING` sessions.
2. Call S3 abort for each session.
3. Continue when one abort fails so the other sessions are still processed.
4. Collect successfully aborted session IDs.
5. Bulk-update those sessions to `ABORTED`.
6. Log the number of successful and failed aborts.

Object-storage calls necessarily occur once per upload session. Database updates should remain batched rather than executed inside the loop.

## 10. Data-consistency evaluation

The reference design is intentionally small. It handles the most common inconsistencies with compensation rather than adding a workflow engine.

| Failure point | Reference behavior | Remaining risk |
| --- | --- | --- |
| S3 session created, DB insert fails | Abort the new S3 session | Abort may also fail; provider lifecycle cleanup should be configured as a backup |
| URL generation fails after session creation | Return an error; client can request URLs again | Session remains `UPLOADING`, which is acceptable until retry or expiry |
| S3 completion fails | Return an error; DB is unchanged | Client can inspect uploaded parts and retry completion |
| S3 completes, DB work fails | Delete the completed object | Delete may fail and leave an orphan object |
| S3 abort succeeds, DB update fails | Return an error | DB may still say `UPLOADING` even though the S3 session no longer exists |
| Cleanup aborts S3, bulk DB update fails | Return an error | Those rows may be retried although S3 has already removed the sessions |
| Session expires before cleanup changes its status | Initialization ignores it as expired | The unique `UPLOADING` index can prevent inserting a replacement until cleanup marks the old row `ABORTED` |

### Minimal safeguards for the new project

These safeguards do not require more tables, statuses, or infrastructure:

- Configure a bucket lifecycle rule to abort incomplete multipart uploads after the agreed retention period.
- Before list, complete, or abort, load the database session and verify the `upload_id + file_key` pair and current user ownership.
- Treat provider “upload not found” as an idempotent success during abort/cleanup only after confirming the matching DB session.
- If an expired row blocks a replacement session, abort it and mark it `ABORTED` before creating the replacement.
- Keep the partial unique index for active sessions.
- Log enough identifiers to investigate compensation failures, but never log presigned URLs or credentials.

These are targeted corrections to the same design, not a new workflow.

## 11. Security and validation

The original commit placed the static multipart routes in the authorization-skip map. Do not copy that behavior unless the target application's security model explicitly makes these routes safe.

Minimum requirements:

- Authenticate initiate, list, complete, abort, and download operations.
- Restrict manual cleanup to operations/admin access.
- Verify session ownership before acting on `upload_id` and `file_key`.
- Restrict allowed upload prefixes, file extensions, content types, and maximum file size.
- Validate part numbers as unique positive integers within provider limits.
- Limit each URL-generation request to a bounded number such as 100.
- Use short presigned URL expirations.
- Keep the bucket private.
- Store S3 credentials outside source code and sample configuration.
- Never log a full presigned URL because its query string contains a temporary signature.
- Do not trust the original filename as an object key.

Browser CORS must allow the approved frontend origin and `PUT`, and must expose `ETag`. CORS permits browser access; it does not replace backend authorization.

## 12. Client implementation

1. Choose a part size that satisfies the configured provider minimum. A common starting value is 8 to 16 MiB.
2. Calculate the total number of parts.
3. Call the URL endpoint with the first batch of part numbers.
4. Slice the local file and upload each part to its matching presigned URL.
5. Save each successful part's ETag.
6. Request additional URL batches until every part is covered.
7. If the page reloads or the network fails, call the uploaded-parts endpoint and continue with missing parts.
8. Send all part numbers and ETags to the completion endpoint.
9. If the user cancels, call abort.

The client should retry temporary network failures with bounded exponential backoff. When a presigned URL expires, request a new URL for that part rather than restarting the whole upload.

## 13. Logging and operations

Follow the target project's existing log style. Useful fields include:

- User ID
- Session database ID
- Upload ID, when the logging policy permits it
- File key
- Part number
- Old and new session status
- Provider request ID
- Error category
- Cleanup success/failure counts

Do not log:

- File contents
- S3 access key or secret
- Authorization headers
- Full presigned URLs

Recommended operational metrics:

- Sessions created, completed, aborted, and expired
- Presigned URL generation failures
- Completion failures
- Cleanup successes and failures
- Number and age of `UPLOADING` sessions

## 14. Implementation order

1. Add storage multipart methods using the existing storage provider.
2. Add the small upload-session table, entity, repository, and indexes.
3. Implement initialization/session reuse and URL generation.
4. Implement paginated uploaded-part lookup.
5. Implement completion and permanent file creation.
6. Implement abort.
7. Implement scheduled cleanup.
8. Implement presigned download URL generation.
9. Add route authorization and CORS configuration.
10. Validate the full flow against the actual S3-compatible provider.

No message broker, new dependency, or extra worker system is required. Use the project's existing cron/scheduler mechanism for cleanup.

## 15. Test cases

Confirm test scope with the project owner before implementation. At minimum, manually or automatically verify:

- A multi-part file is assembled with the exact original bytes.
- URL requests with zero or more than 100 part numbers are rejected.
- Duplicate and out-of-range part numbers are rejected.
- The same file metadata reuses the current active session.
- Concurrent initialization creates only one `UPLOADING` database session.
- A failed DB insert after S3 creation triggers abort.
- Uploaded-parts pagination returns every part.
- A partially uploaded file resumes without re-uploading completed parts.
- Completion sorts parts correctly and creates one permanent file record.
- DB failure after completion attempts to delete the S3 object.
- Abort changes the session to `ABORTED`.
- Cleanup aborts only expired `UPLOADING` sessions and bulk-updates successful IDs.
- Another user cannot list, complete, abort, or download the upload.
- Browser CORS exposes the ETag header.
- Expired upload and download URLs fail as expected.

## 16. Acceptance criteria

- File parts upload directly from client to S3.
- The backend returns no more than 100 part URLs per request.
- The client can query uploaded parts and resume.
- Completion creates the S3 object and permanent file record.
- Abort removes temporary parts and updates session status.
- The scheduled job cleans expired sessions.
- Download uses a short-lived presigned URL.
- The implementation uses only the three session statuses from the reference commit.
- The implementation uses the existing storage abstraction and scheduler.
- Authentication and session ownership are enforced.
- S3 and database failure paths are logged and compensated as described.
- The selected provider's multipart limits, pagination, CORS, and lifecycle behavior are verified.

## 17. Design assessment of the reference commit

### Good decisions to retain

- Direct client-to-S3 upload avoids backend memory and bandwidth pressure.
- URL batches keep responses bounded.
- The database session makes resume, abort, and cleanup possible.
- Redis locking plus the unique partial index reduces duplicate active sessions.
- `ListParts` uses pagination.
- Cleanup bulk-updates database rows rather than updating them inside the storage loop.
- Completion reads final size and type from S3 metadata.
- Compensation is attempted when DB persistence fails.
- Presigned download avoids proxying large completed files through the backend.

### Known limitations to document

- The metadata MD5 identifies an upload attempt, not the actual file content.
- The client currently supplies `upload_id` and `file_key`; the backend must verify them against the DB session.
- S3 and DB operations cannot be one atomic transaction.
- Compensation can fail and requires operational monitoring.
- Expired `UPLOADING` rows can temporarily conflict with the unique active-session index.
- The original route configuration skipped authorization.
- The commit added no focused tests for the new flow.

These limitations do not require redesigning the feature. The minimal safeguards in this document are sufficient for a focused reimplementation unless the target project's scale or business requirements prove otherwise.

## 18. References

- [Amazon S3 multipart upload overview](https://docs.aws.amazon.com/AmazonS3/latest/userguide/mpuoverview.html)
- [Amazon S3 multipart limits](https://docs.aws.amazon.com/AmazonS3/latest/userguide/qfacts.html)
- [ListParts API and pagination](https://docs.aws.amazon.com/AmazonS3/latest/API/API_ListParts.html)
- [CompleteMultipartUpload API](https://docs.aws.amazon.com/AmazonS3/latest/API/API_CompleteMultipartUpload.html)
- [AbortMultipartUpload API](https://docs.aws.amazon.com/AmazonS3/latest/API/API_AbortMultipartUpload.html)
- [Abort incomplete multipart uploads with lifecycle rules](https://docs.aws.amazon.com/AmazonS3/latest/userguide/mpu-abort-incomplete-mpu-lifecycle-config.html)
- [Presigned URL behavior and security](https://docs.aws.amazon.com/AmazonS3/latest/userguide/using-presigned-url.html)
- [Browser CORS and exposing ETag](https://docs.aws.amazon.com/sdk-for-javascript/v2/developer-guide/cors.html)
