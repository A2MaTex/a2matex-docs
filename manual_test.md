Use **Postman against your running Runtime**. Phase H expands `/runs` to multiple devices and adds group-wide Pause, Resume and Cancel.

You can test with one Waydroid device first. Proving concurrent execution and device isolation requires two independent Android targets.

| API                               | Purpose                                                                      |
| --------------------------------- | ---------------------------------------------------------------------------- |
| `POST /runs`                      | Start one workflow on the selected devices, with separate inputs per device. |
| `GET /runs/{runGroupId}`          | Read current progress or saved results.                                      |
| `GET /runs/active`                | Find the current active group; returns 404 when none exists.                 |
| `POST /runs/{runGroupId}/actions` | Pause, Resume or Cancel the entire group.                                    |
| `GET /artifacts/{artifactId}`     | Download a committed screenshot.                                             |

All paths below are relative to the connection file’s `apiBaseUrl`, which already includes `/api/v1`.

**1. Start Runtime and configure Postman**

From your Runtime repository:

```bash
make run
```

Wait for `runtime_ready`. Read the generated connection file using the [authentication guide](/home/dino/dev/a2matex/a2matex-plan/components/a2matex-runtime/integration/local_authentication.md).

Create these Postman environment values manually:

| Variable        | Value                               |
| --------------- | ----------------------------------- |
| `API_BASE_URL`  | The connection file’s `apiBaseUrl`. |
| `SESSION_TOKEN` | Its `sessionToken`.                 |
| `RUN_GROUP_ID`  | Leave empty until starting a run.   |

For every request, select **Authorization → Bearer Token** and enter:

```text
{{SESSION_TOKEN}}
```

For JSON POST requests, use **Body → raw → JSON**. Leave query parameters empty. GET requests and device refresh have no body.

Send:

```http
GET {{API_BASE_URL}}/runtime-info
```

Expect **200**. Refresh the token after restarting Runtime.

**2. Refresh and select devices**

Send:

```http
POST {{API_BASE_URL}}/devices/refresh
```

Body: **none**.

Expected shape:

```json
{
  "devices": [
    {
      "adbSerial": "192.168.240.112:5555",
      "status": "READY"
    }
  ]
}
```

Use the exact serial returned by your Runtime. The IP above is illustrative.

`GET /devices` reads the cached inventory; **POST `/devices/refresh` performs discovery**.

**3. Compile the prepared workflow**

Open [compile-multi-device.json](/home/dino/dev/a2matex/a2matex-runtime/examples/h15/compile-multi-device.json), copy its complete contents, and send:

```http
POST {{API_BASE_URL}}/workflow-compilation
```

Expect `validation.status` to be `VALID`. Copy the returned **`runtimeIr` object**.

The workflow performs:

```text
Start → Home → set private score → set shared threshold=50 → Delay
      → Switch score < threshold
      → low_wait / high_wait
      → Screenshot → remember private artifact ID → End
```

Compilation does not execute any Android action.

**4. Start a run**

Send:

```http
POST {{API_BASE_URL}}/runs
```

Use this request structure. **Replace the empty `runtimeIr` object with the complete compiled object**, and replace both serial placeholders:

```json
{
  "requestId": "{{$guid}}",
  "runtimeIr": {},
  "devices": [
    {
      "adbSerial": "REPLACE_WITH_FIRST_READY_SERIAL",
      "inputs": {
        "score": 25,
        "holdMs": 5000
      }
    },
    {
      "adbSerial": "REPLACE_WITH_SECOND_READY_SERIAL",
      "inputs": {
        "score": 75,
        "holdMs": 5000
      }
    }
  ]
}
```

For one Waydroid device, remove the second entry.

`{{$guid}}` is Postman’s built-in UUID generator. For the replay test later, replace it with the **literal `runGroup.requestId` returned by the first response**.

Expect **201 Created** immediately. This means admission succeeded; execution continues independently.

Copy `runGroup.id` into `RUN_GROUP_ID`. Actions use this ID—not `requestId` or a device execution ID.

**5. Test successful execution**

Poll:

```http
GET {{API_BASE_URL}}/runs/{{RUN_GROUP_ID}}
```

Send approximately once per second until terminal. This is a snapshot API, not SSE.

For two devices, acceptance means:

| Evidence        | Expected result                                                        |
| --------------- | ---------------------------------------------------------------------- |
| Android screens | Home command dispatched on each device.                                |
| Group           | `SUCCEEDED`.                                                           |
| Summary         | `total=2`, `succeeded=2`; all other counts zero.                       |
| Members         | Two distinct execution IDs, paired with the submitted serials.         |
| Branches        | Score25 visits `low_wait`; score75 visits `high_wait`.                 |
| Attempts        | Exactly 10 successful attempts per device.                             |
| Screenshots     | Two distinct artifact IDs, each owned by its device’s capture attempt. |
| Terminal fields | `nextNodeId=null`, `finishedAt` populated, `terminalError=null`.       |

With one device, expect one successful member, 10 attempts and one screenshot.

**6. Test Pause, Resume and Cancel**

Prepare the action requests **before starting another run**, because `holdMs=5000` provides a five-second control window.

All three use:

```http
POST {{API_BASE_URL}}/runs/{{RUN_GROUP_ID}}/actions
```

| Test                   | Body and procedure                                              | Acceptance                                                              |
| ---------------------- | --------------------------------------------------------------- | ----------------------------------------------------------------------- |
| Pause                  | Send `{"action":"PAUSE"}` while the `hold` Delay runs.          | **202**; poll until group becomes `PAUSED`.                             |
| Duplicate Pause        | After confirmed PAUSED, send PAUSE again.                       | **200**; no additional attempts or events.                              |
| Resume                 | Send `{"action":"RESUME"}` after confirmed PAUSED.              | **202**; same execution IDs continue and eventually succeed.            |
| Active Cancel          | Start a fresh run and send `{"action":"CANCEL"}` during `hold`. | **202**; unfinished work becomes `CANCELLED`; no subsequent screenshot. |
| Paused Cancel          | Start another run, Pause, wait for PAUSED, then CANCEL.         | **202**; completed attempts stay successful; no successor executes.     |
| Duplicate Cancel       | Repeat after cancellation cleanup finishes.                     | **200**. During cleanup it can still return **202**.                    |
| Cancel succeeded run   | Target an earlier SUCCEEDED group.                              | **409**, containing `error` and `snapshot`.                             |
| Fresh run after Cancel | Submit `/runs` with a new request ID.                           | **201**; Runtime remains usable without restarting.                     |

For this fixture, when Pause is submitted during `hold`, a confirmed paused member should have:

```text
start             SUCCEEDED
home              SUCCEEDED
set_score         SUCCEEDED
set_threshold     SUCCEEDED
hold              SUCCEEDED

nextNodeId = choose
```

**Pause lets the active Delay finish.** It prevents the next node from starting. Resume retains the live variables and completed outputs.

For active Cancel at that same position, the first four attempts remain successful and `hold` becomes cancelled. Cancelling after PAUSED preserves all five successful attempts.

If the workflow finishes before your action arrives, a terminal-state conflict is valid. Repeat with a fresh run. **202 confirms submission, not completion.**

**7. Test rejection, replay and failure behavior**

Use a fresh request ID for each rejection case, and run admission tests when no other group is active.

| Test                                                             | Expected acceptance                                          |
| ---------------------------------------------------------------- | ------------------------------------------------------------ |
| Remove Bearer token                                              | **401 `UNAUTHORIZED`**.                                      |
| Use `{"action":"pause"}` or add an extra action field            | **400 `INVALID_REQUEST`**. Action names must be uppercase.   |
| Use malformed group ID                                           | **400**.                                                     |
| Use a valid unknown group UUID                                   | **404**.                                                     |
| Submit an empty device list                                      | **400**; no execution created.                               |
| Duplicate a serial in the selection                              | **400**; no execution created.                               |
| Set one device’s `score` to 101                                  | **422**; the complete selection is rejected.                 |
| Include one unavailable serial                                   | **422** with availability diagnostics; no partial run.       |
| Start a different request while a group is active or paused      | **409 `RUNTIME_BUSY`**.                                      |
| Repost the original admitted request with its literal request ID | **200**, same group/member IDs; no repeated Android actions. |
| GET `/runs/active` after completion                              | **404**; GET by the saved group ID still returns its result. |

To test disconnect isolation with two independent targets, use the dedicated scenario in the [H15 guide](/home/dino/dev/a2matex/a2matex-runtime/examples/h15/README.md). Disconnect A after admission but before its next Android operation. Acceptance is A failing with the appropriate device error while B continues.

Disconnect is **command-observed**: losing a device during Delay does not immediately change its execution status. A later failing ADB operation triggers diagnosis.

**8. Inspect SQLite and download screenshots**

The normal Linux database is:

```text
${XDG_STATE_HOME:-$HOME/.local/state}/a2matex/runtime/execution.sqlite3
```

Open it read-only in DataGrip. Replace the group-ID placeholder in this query:

```sql
SELECT
    d.adb_serial,
    a.sequence,
    a.node_id,
    a.status,
    a.output_json,
    a.error_json
FROM node_attempts a
JOIN device_executions d ON d.id = a.device_execution_id
WHERE d.run_group_id = 'PASTE_RUN_GROUP_ID'
ORDER BY d.admission_ordinal, a.sequence;
```

Find screenshot IDs and ownership:

```sql
SELECT
    d.adb_serial,
    f.id AS artifact_id,
    f.width,
    f.height,
    f.byte_length
FROM artifacts f
JOIN node_attempts a ON a.id = f.attempt_id
JOIN device_executions d ON d.id = a.device_execution_id
WHERE d.run_group_id = 'PASTE_RUN_GROUP_ID';
```

Then request:

```http
GET {{API_BASE_URL}}/artifacts/<artifact_id>
```

Expect **200**, `image/png`, the expected dimensions and matching byte length. Use Postman’s response preview or save response.

Flow and Global variable values remain in RAM; their absence from SQLite is expected. Attempt-history HTTP endpoints belong to Phase I.

For the deterministic eight-scenario acceptance check without real devices:

```bash
make build
python3 cmd/agent/testdata/h15_acceptance.py bin/a2matex-runtime --results-dir test_demo/h15-manual-check
```

Choose a new results directory. It contains readable requests, snapshots, SQLite records, PNGs and dispatch journals. It uses controlled ADB endpoints and does not operate Waydroid.

AGENTS.md loaded.
