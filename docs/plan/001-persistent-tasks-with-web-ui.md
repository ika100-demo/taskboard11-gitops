---
plan_id: 001-persistent-tasks-with-web-ui
spec: 001-persistent-tasks-with-web-ui
feature: Tasks stored in Postgres so they survive restarts, and a web task page to list and create them
gitops_app: ika100-demo/taskboard11-gitops
status: completed
repos:
- id: taskboard11-gitops
  shape: gitops-app
  summary: Declare the taskboard11 Postgres database, give it to the Task API and wire the Task API URL
    into the web UI
  acs:
  - AC-001.1
  - AC-001.2
  - AC-001.3
  gitops:
  - resource: postgres
  - uses: postgres
    component: taskboard11-api
  - env:
      TASK_API_URL: http://taskboard11-api
    component: taskboard11-web
  depends_on: []
  done: true
- id: taskboard11-api
  shape: service-python
  summary: Store tasks in Postgres so they survive restarts, and answer 503 while the database is unreachable
  acs:
  - AC-001.1
  - AC-001.2
  criteria:
  - Given tasks were created through `POST /tasks` against the database in `DATABASE_URL`, when the service
    process is restarted against the same database, then `GET /tasks` returns 200 with each of those tasks,
    with the same `id`, `title` and `due_date`, in creation order (system AC-001.1) (replaces AC-001.7)
  - Given `DATABASE_URL` points to an empty database, when the service starts, then it creates its own
    schema and `POST /tasks` then `GET /tasks` work, and a second start against the same database keeps
    the existing tasks (system AC-001.1)
  - 'Given the database is unreachable, when a client sends `POST /tasks` or `GET /tasks`, then the response
    is 503 with the body `{"detail": "database unavailable"}` within 5 seconds, `GET /ready` returns 503
    and `GET /health` stays 200 (system AC-001.2)'
  - Given a `POST /tasks` was answered 503 while the database was unreachable, when the database is reachable
    again, then `GET /ready` returns 200 within 10 seconds without a restart of the service, and `GET
    /tasks` does not contain the task sent during the outage (system AC-001.2)
  - Given the database is unreachable when the service starts, when a client calls `GET /health` and `GET
    /ready`, then the process is running, `/health` returns 200 and `/ready` returns 503, and the task
    endpoints answer 503, never an empty list from memory (system AC-001.2)
  arguments: |
    Persistence replaces the in-memory TaskStore; the request/response format of POST/GET /tasks stays exactly as
    in spec 001 (AC-001.1 to AC-001.6 of that spec stay active). Only AC-001.7 of spec 001 is superseded.
    Connection comes from DATABASE_URL (PG* also set by the platform). Creation order must be stable across
    restarts: order by a monotonically increasing column (e.g. a bigserial or created_at plus sequence), never by id.
    Tests run against a real Postgres (the repo's devbox / a test container), not a mock.
  depends_on:
  - taskboard11-gitops
  done: true
- id: taskboard11-web
  shape: web-nextjs
  summary: Task page on the start page that lists tasks and creates them through the Task API
  acs:
  - AC-001.3
  - AC-001.4
  - AC-001.5
  - AC-001.6
  - AC-001.7
  criteria:
  - Given the Task API's `GET /tasks` returns tasks A, B and C in that order, when a user opens `/`, then
    the page lists A before B before C, each with its `title` and its `due_date` as `YYYY-MM-DD`, or "No
    due date" when `due_date` is null (system AC-001.3)
  - Given `GET /tasks` returns `[]`, when a user opens `/`, then the page shows "No tasks yet" and the
    create form with a title field, a due date field (calendar date, may be empty or in the past) and
    a submit button (system AC-001.4)
  - 'Given the page is open, when a user enters the title "Buy milk" and the due date 2026-12-01 (or leaves
    the due date empty) and submits, then `POST /tasks` is sent with `{"title": "Buy milk", "due_date":
    "2026-12-01"}` (or `"due_date": null`), and on 201 "Buy milk" with 2026-12-01 (or "No due date") appears
    at the end of the list without a page reload and the form is cleared (system AC-001.5)'
  - Given the page is open, when a user submits a title that is empty, only whitespace, or longer than
    200 characters, then the page shows "Title must be 1 to 200 characters", keeps the entered title and
    due date, and no `POST /tasks` is sent (system AC-001.6)
  - Given the Task API is unreachable, does not answer within 5 seconds, or answers `GET /tasks` with
    any status other than 200, when a user opens `/`, then the page shows "Tasks could not be loaded.
    Try again later." instead of the list, and the create form is still shown (system AC-001.7)
  - Given the Task API is unreachable, does not answer within 5 seconds, or answers `POST /tasks` with
    any status other than 201 (including 422 and 503), when a user submits a valid form, then the page
    shows "Task could not be saved. Try again.", keeps the entered title and due date, and the list is
    unchanged (system AC-001.7)
  arguments: |
    The task page replaces the starter page at `/`. The web server calls the Task API (server components and a
    server action or a same-origin route handler); the browser never calls the Task API directly, so there is no
    CORS and TASK_API_URL stays server-only (never NEXT_PUBLIC_*). Read TASK_API_URL at request time and render
    the list dynamically (no static caching of GET /tasks). Document TASK_API_URL in docs/env-vars.md.
    The smoke test that creates a task is tagged @smoke @writes (dev only); keep a read-only @smoke check that `/` loads.
  depends_on:
  - taskboard11-gitops
  done: true
gitops_pin: []
---

## taskboard11-gitops — Declare the taskboard11 Postgres database, give it to the Task API and wire the Task API URL into the web UI

**Shape:** `gitops-app` · **Criteria:** AC-001.1, AC-001.2, AC-001.3 · **Depends on:** []

No resource exists in `system.yaml` yet. The plan's `gitops:` operations, in run order:

1. `resource: postgres` declares the system's Postgres database (CloudNativePG cluster, in the cluster, per A7), created in every environment where a component that uses it runs: today dev only.
2. `uses: postgres` on `taskboard11-api` gives the API `DATABASE_URL` and `PGHOST`, `PGPORT`, `PGDATABASE`, `PGUSER`, `PGPASSWORD` from the database's app secret. Only the API uses it; the web UI never gets database credentials (A1).
3. `env: TASK_API_URL=http://taskboard11-api` on `taskboard11-web`: the API's in-cluster Service (port 80 to the API's port 8080) in the same namespace.

Both components are already exposed (`expose.host`), so no `expose` operation is needed. This entry comes first (level 1) because dev tracks `main`: the new API image would fail readiness without `DATABASE_URL`, and the new web image would show the load error without `TASK_API_URL`. Applying it before them is harmless: the in-memory API ignores the new variables and the starter page ignores `TASK_API_URL`. It owns the deployment side of AC-001.1 (a database outlives API pods), AC-001.2 (the readiness probe `/ready` already exists and takes the API out of the Service) and AC-001.3 (the web UI can reach the API).

## taskboard11-api — Store tasks in Postgres so they survive restarts, and answer 503 while the database is unreachable

**Shape:** `service-python` · **Criteria:** AC-001.1, AC-001.2 · **Depends on:** [taskboard11-gitops]

The API owns the persistence (AC-001.1) and the outage behaviour (AC-001.2): no memory fallback (A2), the service starts and stays alive without the database, `/ready` reflects database reachability, and a write answered 503 must not be committed. Its new spec supersedes AC-001.7 of `001-create-and-list-tasks-in-memory` ("empty list after restart"); the other criteria of that spec, the request and response format, stay active and are its regression tests. Test concerns: restart is tested by creating the app twice against the same real Postgres; the outage by stopping or pointing at an unreachable database and checking the 5-second bound.

## taskboard11-web — Task page on the start page that lists tasks and creates them through the Task API

**Shape:** `web-nextjs` · **Criteria:** AC-001.3, AC-001.4, AC-001.5, AC-001.6, AC-001.7 · **Depends on:** [taskboard11-gitops]

The web UI owns everything a user sees: the list in API order (A4), the empty state, the create form with client-side title validation using the API's rules (A5), the exact texts (A6) and both error states. It talks only to the Task API contract below, so it builds in parallel with the API: unit and component tests stub the Task API; the e2e tests run against a stub server on `TASK_API_URL`. The part of AC-001.5 "`GET /tasks` contains the task" is checked through the contract (the web sent the right `POST /tasks` and got 201) and by the dev smoke test against the real API.

## Contract

The Task API keeps the format of its spec `001-create-and-list-tasks-in-memory` unchanged; this spec adds only 503 and the readiness behaviour. Base URL for the web UI: `TASK_API_URL` (dev: `http://taskboard11-api`), called from the web server only. JSON, UTF-8, `Content-Type: application/json`.

**Task object**

| Field | Type | Notes |
|---|---|---|
| `id` | string | server-assigned, opaque, non-empty; never sent by the client (ignored if sent) |
| `title` | string | 1 to 200 characters, at least one non-whitespace character, stored and returned exactly as sent |
| `due_date` | string or null | calendar date `YYYY-MM-DD`; past dates allowed; `null` when there is none |

**`POST /tasks`** — create a task. Request body `{"title": "<title>", "due_date": "YYYY-MM-DD" | null}` (`due_date` may be omitted, which equals `null`).

| Status | Body | Meaning |
|---|---|---|
| 201 | the Task object | created and stored in the database |
| 422 | FastAPI validation body `{"detail": [...]}` | title or due date invalid; nothing created |
| 503 | `{"detail": "database unavailable"}` | database unreachable; nothing created, the task never appears later |

**`GET /tasks`** — list all tasks, oldest first (creation order), no paging.

| Status | Body | Meaning |
|---|---|---|
| 200 | JSON array of Task objects (`[]` when none) | |
| 503 | `{"detail": "database unavailable"}` | database unreachable |

**Probes** (platform, not the web UI): `GET /health` 200 `{"status": "ok"}` while the process runs, independent of the database; `GET /ready` 200 `{"status": "ready"}` when the database answers, else 503 `{"status": "unavailable"}`.

**Web UI to user** (inside the web repo, listed for completeness): the task page is `/`; texts exactly "No tasks yet", "No due date", "Title must be 1 to 200 characters", "Tasks could not be loaded. Try again later.", "Task could not be saved. Try again."

### Errors

- Error bodies of the Task API are JSON with a `detail` field: a string for 503, the FastAPI validation list for 422. The web UI never shows `detail` to the user; it maps outcomes to its fixed texts.
- The web UI treats any response the contract does not list for a call (other 4xx, 5xx, a 200 to `POST`, a 201 to `GET`, a body that is not the expected JSON shape), a connection error and a timeout the same as 503: on `GET /tasks` it shows "Tasks could not be loaded. Try again later."; on `POST /tasks` it shows "Task could not be saved. Try again." and keeps the entered values. A 422 on `POST` (which the client-side check should prevent) is also shown as "Task could not be saved. Try again."
- The web UI logs each failed call once (`@/lib/logger`, with status or error kind, never the task title), and does not retry automatically.

### Timeouts

- Web server to Task API, `GET /tasks` and `POST /tasks`: 5 seconds total per call; on expiry the call is aborted and handled as an error above. No retries.
- Task API to Postgres: connect timeout 3 seconds, statement timeout 3 seconds, so a task endpoint answers 503 within 5 seconds when the database is down or hangs; the readiness check uses the same bound and answers within 5 seconds.
- Database down at API start: the API starts anyway (liveness 200, readiness 503) and reconnects on the next request or readiness check; once the database is back `/ready` turns 200 within 10 seconds without a restart.
- Task API down or not ready: Kubernetes removes it from the Service, so the web UI gets a connection error or timeout and shows its error texts; the web UI's own readiness does not depend on the Task API.

## gitops-app PR

One PR in this repo carries the three `gitops:` operations (run by `/system:ship` with `cplat resource add` and `cplat component set`, then `devbox run render`) and lands before the API and web merges. No version pins: both components run in dev only, which tracks `main`, and per the spec's non-goals neither may be promoted to staging or prod before the authentication spec is done, so `gitops_pin` is empty.

After Argo reconciles dev, verify: the CloudNativePG cluster of the system is healthy in the dev namespace; the `taskboard11-api` pod has `DATABASE_URL` and the `PG*` variables and `/ready` returns 200 after the new API image is deployed; the `taskboard11-web` pod has `TASK_API_URL=http://taskboard11-api`; then the dev smoke tests of both repos pass (the API's create-then-list, the web's create through the form), and a task created before a rollout restart of `taskboard11-api` is still listed after it.
