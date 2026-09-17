# CLAUDE.md

This file guides Claude Code (and any teammate) when working in this repository.

## Project Overview

A fault-tolerant, scalable background job queue and **push-based** worker execution engine, built for a hackathon. A central dispatcher tracks worker availability and actively pushes tasks to idle workers (rather than workers polling/pulling). Core requirements: at-least-once execution, delayed/scheduled jobs, worker heartbeat detection with automatic task reassignment, Dead-Letter Queue (DLQ) reprocessing, and a real-time observability dashboard.

## Tech Stack

- **Backend:** Node.js (Express or Fastify — pick one and stay consistent across all services)
- **Persistence:** PostgreSQL (source of truth for tasks, workers, execution history)
- **Worker communication:** Plain HTTP (dispatcher pushes via `POST /assign` to each worker's own small HTTP server)
- **Realtime dashboard:** WebSockets (`ws` or Socket.IO) or SSE, pushed from a metrics aggregator
- **Frontend:** Plain HTML/JS or a lightweight Next.js page — keep it simple, this is secondary to the backend

Do not introduce Redis/RabbitMQ/NATS unless a specific bottleneck demands it — Postgres alone (using `SELECT ... FOR UPDATE` or a conditional `UPDATE ... WHERE status='IDLE'`) is sufficient for atomic worker claiming at hackathon scale.

## Architecture (push model — read this before touching dispatcher code)

1. **Task Submission API** inserts a row into `tasks` (status: `pending`, optional `run_at`).
2. **Scheduler loop** polls for tasks whose `run_at <= now()` and status is `pending`/`scheduled` → marks them `ready`.
3. **Dispatcher loop**:
   - Finds a `ready` task and an `IDLE` worker.
   - **Atomically claims the worker**: `UPDATE workers SET status='BUSY' WHERE id=$1 AND status='IDLE'` — if 0 rows affected, another push already claimed it; retry with the next idle worker. This is the single most important correctness rule in this codebase — never assign a worker without this conditional update.
   - Pushes the task via `POST /assign` to the worker's registered address.
   - If the HTTP push fails (worker unreachable), mark the worker `DEAD` and requeue the task — do not leave it in `BUSY` limbo.
4. **Worker process**:
   - Exposes `POST /assign` (accept task, respond immediately, execute asynchronously).
   - Sends a heartbeat (`POST /heartbeat` or equivalent) every few seconds.
   - On completion, reports back (`task done` + `I'm idle again`) so the dispatcher can immediately push any pending task to it.
5. **Watchdog** scans for workers whose last heartbeat exceeds the timeout → marks them `DEAD`, requeues any task they were holding.
6. **Retry / DLQ**: failed tasks retry with exponential backoff up to `max_retries`; beyond that they move to the `dead_letter` table with failure reason + attempt history. A replay endpoint moves a DLQ task back to `pending`.
7. **Dashboard** subscribes to queue depth, worker states, and task status counts, pushed live over WebSocket/SSE.

## Data Model (baseline — extend as needed, but don't rename existing columns without telling the team)

```
tasks (
  id, payload, status, priority, run_at,
  idempotency_key, retries, max_retries,
  assigned_worker_id, created_at, updated_at
)

workers (
  id, address, status,          -- IDLE | BUSY | DEAD
  last_heartbeat, created_at
)

dead_letter (
  id, task_id, payload, failure_reason,
  attempt_history, created_at
)
```

## Team Ownership (5 people)

- **Person 1 — API & Schema:** `POST /tasks`, task/worker table migrations, validation.
- **Person 2 — Dispatcher:** the claim-and-push loop. Owns correctness of the atomic claim.
- **Person 3 — Worker process:** `/assign` endpoint, heartbeat sender, task execution simulation.
- **Person 4 — Failure handling:** watchdog, retry/backoff, DLQ table + replay endpoint.
- **Person 5 — Dashboard:** metrics aggregation, WebSocket/SSE push, live UI, "kill worker" demo control.

## Commands

Adjust these once `package.json` scripts are finalized — placeholders below:

```bash
npm install              # install deps at repo root (or per-service if using workspaces)
npm run dev:api          # start task submission API
npm run dev:dispatcher   # start dispatcher/scheduler loop
npm run dev:worker       # start a worker instance (run multiple with different ports)
npm run dev:dashboard    # start dashboard server
npm run db:migrate       # apply Postgres schema
```

If using npm workspaces, keep each service (`api/`, `dispatcher/`, `worker/`, `dashboard/`) as a separate workspace so team members don't collide on shared `node_modules` state.

## Conventions

- All services log worker/task state transitions to stdout with a timestamp and status — this is what will let you debug a live failover demo quickly.
- Never mutate `workers.status` outside the atomic `UPDATE ... WHERE status=...` pattern described above.
- Task execution in the worker must be idempotent-aware: check `idempotency_key` against a recently-completed set before applying side effects, since at-least-once delivery means duplicate pushes can happen after a reassignment.
- Keep the heartbeat interval and dead-worker threshold as environment variables (e.g. `HEARTBEAT_INTERVAL_MS`, `HEARTBEAT_TIMEOUT_MS`) — you'll want to tune these live before the demo.

## Testing Priorities (matches mentor evaluation criteria)

1. Kill a worker process mid-task → confirm the task is detected as orphaned and reassigned to another idle worker.
2. Submit a burst of tasks with only 1–2 workers available → confirm no task starves indefinitely once a worker frees up.
3. Submit delayed tasks with varying `run_at` offsets → measure actual dispatch time vs. requested time.
4. Force a task to fail repeatedly → confirm it lands in `dead_letter` after `max_retries`, and that the replay endpoint successfully requeues it.
5. Confirm the dashboard reflects all of the above in real time, not on a delay.