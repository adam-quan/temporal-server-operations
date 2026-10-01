## Timer Task Scheduling Lag Critical

**Severity:** Critical
**Component:** history
**Dashboard panel:** [Timer Task Scheduling Latency](https://github.com/tsurdilo/temporal-metrics/blob/main/observability/dashboards/server/temporal-server-readme.md) — panel ID 325

### What this alert detects

The timer queue's **ack level** is more than 15 minutes behind its **read position**, at p99 across shards, held for 15 minutes. The measurement is `shardinfo_scheduled_queue_lag_bucket{task_category="timer"}`.

The ack level is the fire time of the oldest timer task that has not finished. The read position is how far ahead the queue has read. The gap between them is what this alert measures.

### What it does not detect

It is **not** how late an individual timer fired. One task that cannot complete holds the ack level on its own, and the number will be large while every other timer on the shard runs on time.

### Why it matters

Nothing at or above the ack level is ever deleted. While it is stuck, the `timer_tasks` rows for those shards keep accumulating — **for every namespace on those shards**, not only the one holding it up. A queue pinned for weeks is how a task table reaches hundreds of millions of rows. That is usually the damage, not timer lateness.

### Two things to know before you act

**It has a floor.** The reader always reads ahead of now, so the gap is never zero even on an idle cluster with no workflows at all — several hundred seconds is normal there. Take a reading on a quiet cluster and treat that as your zero. The 900s threshold on this alert sits above that floor.

**It saturates at 1000s** (16.7 min), the top histogram bucket. A line pinned at 1000 means "at least that, and possibly far more". The number stops growing; the backlog does not. Do not read a flat line at the top as the problem having stabilised.

### Triage steps

1. Open **Scheduled Queue Lag per Pod** (panel 2110) — this breaks the same metric out by pod, so you can see whether it is the whole cluster or a few hosts. Panel 325 is the cluster-wide view this alert fires on.
2. If it is a few pods: pick a shard owned by one of them and read its queue state with `tdbg shard describe`. Look at the timer category's readers and their scopes — the un-acked scope with the oldest `InclusiveMin` fire time is the one holding the ack level, and its predicate names the namespace.
3. Decide which case you are in:
   - **A few scattered un-acked scopes, seconds wide.** Tasks are failing and retrying. Check **Task Errors** and the history task DLQ panels — a task retrying forever rather than being dead-lettered holds the position indefinitely.
   - **One wide scope, days or weeks long.** A real backlog that is not draining fast enough. Go to step 4.
4. For a real backlog, check whether it is stuck or throttled. Look at timer task throughput against **Task Scheduler Throttling by Namespace**: throughput well below the configured limit with few errors means it is draining as fast as it is permitted to, and the limits are the constraint — not a fault.
5. Check **Persistence Latencies** (panel 71) and **Shard Lock Latency** (panel 14) to rule out the database or shard-lock contention as the reason tasks are slow to complete.
6. Check whether the namespace holding the position still exists. A deleted namespace is renamed to `<name>-deleted-<first 5 hex of its ID>`, and its timers keep their original namespace ID in the queue state — so the scope's ID may not match any namespace you recognise by name.

### Relevant dynamic config

- `history.timerProcessorSchedulerWorkerCount` — task slots for timer processing per history host (default 512). Raising it only helps if slots are the bottleneck, not if the scheduler's rate limits are.
- `history.timerProcessorMaxPollHostRPS` — ceiling on timer **read calls** per second per host (default 0, meaning derive it from the persistence rate). This is calls, not tasks: multiply by `history.timerTaskBatchSize` for the task rate it allows.
- `history.timerTaskBatchSize` — tasks returned per read call (default 100).
- `history.taskSchedulerGlobalNamespaceMaxQPS` / `history.taskSchedulerNamespaceMaxQPS` — the per-namespace scheduling limits. If one namespace owns the backlog, these are what govern how fast it drains.

For how these interact and how to size them, see the [History Task Processing Tuning playbook](https://github.com/tsurdilo/temporal-metrics/blob/main/playbooks/history-task-processing-tuning.md) — in particular [Tuning how fast tasks are loaded](https://github.com/tsurdilo/temporal-metrics/blob/main/playbooks/history-task-processing-tuning.md#7-tuning-how-fast-tasks-are-loaded) and [The task scheduler's rate limits](https://github.com/tsurdilo/temporal-metrics/blob/main/playbooks/history-task-processing-tuning.md#4-the-task-schedulers-rate-limits).
