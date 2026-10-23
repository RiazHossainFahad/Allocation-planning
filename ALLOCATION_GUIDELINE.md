# Standard-order allocation: user manual

Audience: developers, support and operations who run, tune or debug allocation of standard (B2B / B2C) orders.
Scope: the whole feature on branch `5033-iqh-new-allocation-engine`: the journaled (crash-safe) engine, the pool lock,
the large-order path, the request marker, recovery, configuration and rollout.

Everything tunable lives in `config/allocation.php` and is driven by `.env` keys. A change takes effect only after
the steps in [Applying a change](#9-applying-a-change).

---

## 1. What this feature does, in one page

**Allocation** reserves stock (`inventory_balances.available_qty` moves to `allocated_qty`) for an order's lines and
creates its picking tasks.

**The problem it fixes.** A ~4,700-line order was killed by the Horizon 180 s timeout in the middle of allocation. The
stock it had reserved stayed reserved with no picking tasks ("orphans") and `orders.is_allocating` stayed at 1 forever.

**The solution has four parts:**

| Part | What it is | One-line effect |
|---|---|---|
| **Journaled engine** | A second allocator that writes every reservation together with a journal row. | A run that dies can be undone exactly. Stock never leaks. |
| **Pool lock** | One allocator at a time per stock pool (Redis lock). | No two allocators reserve from the same pool at once. |
| **Large-order path** | Orders over a line threshold are allocated by their own cron, not by the shared instant queue. | A 5,000-line order can no longer be killed by the 180 s timeout. |
| **Recovery** | A sweeper and the allocation crons compensate runs that died. | Dead runs give their stock back by themselves. |

The **legacy engine** (the old allocator, `AllocateStandardOrderHelper`) still exists. Which engine a pool uses is a
**rollout setting** (section 6). Which *path* an order takes (instant or large cron) depends **only on its size**
(section 3). The two decisions are independent.

---

## 2. Concepts

### Stock pool
A pool is `(organization, warehouse, customer)`, where the customer is resolved through its aggregator (aggregator
children share the aggregator's pool). Every allocator of one pool competes for the same `inventory_balances` rows.
Engine choice, locking and deferral are all decided per pool, never per order.

### Engines

| | Legacy | Journaled |
|---|---|---|
| Crash behavior | Run dies: reserved stock stays reserved (orphans). | Run dies: its journal is used to give every unit back. |
| Lost race (another writer takes stock first) | Clamps to 0 and can over-allocate. | Detected, the chunk is planned again from fresh data. |
| Pool lock | Held for the whole run, cannot be renewed. | Taken per chunk of SKUs, renewed while planning. |
| Runtime cap | None. | Yes for large orders (`max_order_runtime_s`). |
| Exceptions | Swallowed inside the helper. | Run is compensated, exception rethrown. |
| Selection | One query set per line. | `per_line` (same as legacy) or `batched` (few queries per 200 SKUs). |

The journaled engine reuses the legacy code for *what* to allocate (picking rules, passes, partial handling, cartons).
For the same stock it picks the same stock. Only the way the choice is made durable differs. A differential test
harness checks legacy, `per_line` and `batched` against each other on thousands of random worlds.

### Run and journal
Every journaled allocation is a **run** (table `allocation_runs`, portal DB) with a **journal** of exactly what it
reserved (`allocation_run_reservations`: run, balance, qty).

* `status`: `running`, `completed` or `compensated`.
* `reason` (why a run was compensated): `lock_wait`, `lost_race`, `runtime_cap`, `exception`, `killed`, `expired`.
* One order can have only one `running` run at a time (unique `active_order_id`).
* A run holds a **lease** (`lease_until`, renewed while it works). A `running` run whose lease expired is a dead run.
* **Fencing:** every stock write first locks the run row and checks it is still `running` with the worker's token. A
  worker whose run was already compensated cannot write anything.

### Chunks
A journaled run works in chunks of `chunk_sku_count` SKUs (default 200). For each chunk it takes the pool lock, plans,
reserves (a conditional `UPDATE` plus journal rows in one short transaction), and releases the lock. The picking
tasks, master inventories, order-line statuses and run completion are written in **one atomic transaction** at the
end. All-or-nothing orders are compensated if a later chunk cannot be satisfied.

### Request marker (`orders.allocation_requested_at`)
A timestamp set when someone explicitly asks for a **large** order to be allocated (the manual click). It means
"allocate this one now, I asked for it." It makes the large cron take the order:
* without waiting for the retry schedule, and
* even in a warehouse that has automatic allocation off.

The marker is **one-shot**; see [section 4.5](#45-what-happens-to-the-marker).

### Attempts, counters and locking
`orders.allocation_counter` counts failed attempts and `last_allocation_tried_at` is the time of the last try.
Out of stock is a failed attempt. At 18 failed attempts the order is locked by the system
(`is_locked = 1`, "Locked by system for reaching allocation threshold.") and no cron touches it. A successful
allocation resets the counter and unlocks. **Deferrals are not attempts** (busy pool, lost race, runtime cap).

The retry schedule that applies to automatic allocation:

| Counter | Wait before next try |
|---|---|
| never tried | immediate |
| 0 to 5 | 2 hours |
| 6 to 13 | 6 hours |
| 14 to 17 | 24 hours |

---

## 3. Who allocates an order

### 3.1 The routing rule: size only

An order is **large** when its **allocatable lines** are **more than** `allocation.large_orders.allocatable_line_threshold`
(default **2000**). Equal to the threshold counts as small. Allocatable lines are counted live: order lines whose
status is Confirmed, Out of stock or Partially allocated. An order that gets mostly allocated drops under the
threshold by itself and moves back to the normal path.

The engine does **not** change the routing: a large order is routed to the large cron whether its pool is journaled
or legacy, and each path runs whichever engine the pool is configured for.

### 3.2 Entry points

| Entry point | Handles | Engine used | Notes |
|---|---|---|---|
| `process:standard-order` (cron, every 10 min at :04, :14, ...) | **Small** orders in warehouses with automatic allocation, older than 5 minutes, due by the retry schedule | per config | Skips busy pools for the rest of the tick. |
| `allocation:process-large-orders` (cron, every 30 min by default) | **Large** orders that carry a request, or sit in an auto-allocation warehouse and are due by the retry schedule | per config; journaled runs always use `batched` selection | Sequential, bounded batch. |
| Manual click, `POST instant_order_allocation` | Marks a large order, or dispatches a small one to the instant queue | per config | See 3.3. |
| Instant listener (queue `iqh-instant-allocation`) | A small order dispatched by the click | per config | Short lock wait, re-dispatches when the pool is busy. |
| New SKU added to an order, skip-task switch | One line | per config | Take the pool lock too (the skip switch waits at most `skip_wait_seconds`). |

The `iqh-instant-allocation` queue has **one worker** shared with ASN auto-close and RCO allocation. That is why a
large order must never run there.

### 3.3 The manual click (`instant_order_allocation`)

The endpoint keeps its validations (order not locked, status Confirmed / Out of stock / Partially allocated, not
already allocating, creation completed, picking strategy rules exist). Then, for standard orders:

* **Large order:** resets the order's stored `partial_allocation_handling` override, stamps `allocation_requested_at`,
  sets **no** `is_allocating` flag, dispatches nothing, and answers
  `{"message": "Order is queued for large-order allocation.", "queued_for_large_order_allocation": true}`.
  Clicking again just stamps again.
* **Small order:** clears any stale marker, sets `is_allocating = 1` and dispatches the instant event, exactly as before.

The order payload carries `allocation_requested_at`, and `Order::isQueuedForLargeOrderAllocation()` is true while the
marker is set and the order is not allocating. The frontend shows "queued for large-order allocation" from that.

The instant listener repeats the size check as a backstop (an order can grow past the threshold after it was queued):
a large order reaching it is stamped, its flag is released, no attempt is counted and no stock is touched.

---

## 4. How a large order is allocated

### 4.1 The large cron

`allocation:process-large-orders` runs every `cron_expression` (default every 30 minutes) with `withoutOverlapping`.

1. It first compensates any journaled run whose lease expired.
2. It selects open standard orders (status Confirmed / Out of stock / Partially allocated, not locked, creation
   completed, not allocating, older than 5 minutes) that have more allocatable lines than the threshold, and that either
   * carry a request (any warehouse, any retry state), or
   * sit in a warehouse with automatic allocation **and** are due by the retry schedule.
3. Oldest first (`insertTime`, then `ordersId`). It takes at most `orders_per_run` orders and runs them **one after another**.
4. A busy pool defers that pool for the rest of the run; the batch is topped up from other pools.
5. It stops **starting** orders after `cron_start_budget_s`. A running order is never cut short by that budget.
6. One order may run at most `max_order_runtime_s`. Past it the run is compensated and the order deferred.

Unmarked large orders in a warehouse **without** automatic allocation are never picked: nobody asked for them.

### 4.2 Batched selection

With `batched` selection the engine prefetches candidate stock per query-shape group, keeps a per-run registry of raw
available qty, plans in memory, and applies each chunk with one conditional multi-row `UPDATE`. A 5,000-line order is
about 274 statements instead of tens of thousands. The cron forces `batched` for journaled pools, whatever
`journaled.selection` says.

### 4.3 A large order on a legacy pool

It is allocated by the legacy engine, with weaker guarantees: no journal, the pool lock is held for the whole run with
a fixed TTL (`legacy_run_ttl_cron_seconds`), no runtime cap, and the overlap mutex bound is not guaranteed. The cron
logs a `warning` for every such order and runs the orphan check afterwards. **Move the pools that receive large
orders to the journaled engine first** (rollout phase 3).

### 4.4 Outcomes of an attempt

| Outcome | Order | Counters | Marker |
|---|---|---|---|
| Fully allocated | tasks created, status Allocated, order unlocked | counter reset to 0 | cleared |
| Partially allocated | tasks created for what was available, status Partially allocated | counts as a failed attempt (the order is not fully allocated) | cleared |
| Out of stock (run completes with nothing to allocate) | stays Out of stock | counts as a failed attempt | cleared |
| Pool busy at start, or the pool lock could not be re-taken for a later chunk (`lock_wait`) | untouched | unchanged (deferral) | **kept** |
| Another run of this order is already live | untouched | unchanged | **kept** |
| Lost race too many times (`lost_race`) | run compensated | unchanged (deferral) | cleared, except outside auto warehouses |
| Runtime cap reached (`runtime_cap`) | run compensated | unchanged (deferral) | cleared, except outside auto warehouses |
| Fenced (recovery compensated a slow live run) | nothing written | unchanged | cleared, except outside auto warehouses |
| Crash (exception) | run compensated, flag released | free for the first `free_compensations_per_order` per 24 h, then counts as an attempt | cleared, except outside auto warehouses |
| Nothing can be tried (no picking strategy rules, or partial-allocation handling forbids another run) | untouched | unchanged | cleared |

### 4.5 What happens to the marker

The marker is a **one-shot request**. After a command (`process:standard-order` or the large cron) has tried an order:

* it is **kept** only if the pool kept the run from trying (pool busy, mid-run `lock_wait`, another run live);
* it is **cleared** in every other case, **except** when the order's warehouse does **not** have automatic
  allocation and the run did not finish (deferral or crash). There the marker is the only thing that selects the
  order, so clearing it would strand an order the user was told is queued.

A journaled run that completes also clears the marker in its own commit. The marker is cleared by whoever tries the
order, so legacy runs (which have no commit to clear it in) are covered too.

Consequences worth knowing:
* A crashing order in a non-auto warehouse keeps being retried each cron, but each repeated crash counts as an
  attempt after the free ones, so it locks at 18.
* Deferrals never lock an order. After `max_consecutive_deferrals` in a row the cron logs a `critical` line so a human looks.
* A marked order that shrinks under the threshold in a non-auto warehouse is taken by neither cron (small orders are
  not the large cron's, and `process:standard-order` only reads auto warehouses). Clicking allocate again clears the
  stale marker and dispatches it.

---

## 5. What protects stock

* **Atomic reserve:** each reservation is a conditional update plus a journal row in one transaction. If the stock
  moved, the update affects fewer rows and the chunk is re-planned (a "lost race").
* **Atomic finish:** tasks, inventories, line statuses and run completion commit together, fenced by the run row.
* **Compensation:** `AllocationRunRecovery::compensate()` reads the journal, gives every quantity back and marks the
  run `compensated`. It runs when a run fails, when its lease expires (every allocation cron tick and the sweeper),
  and when the queue marks the job failed or killed.
* **Pool lock** (Redis key `alloc:{org}:{warehouse}:{poolCustomer}`): serializes allocators of one pool. Correctness
  never depends on it alone; the conditional update, row locks and run fencing hold even if the lock is lost.
* **Lock release never masks a result:** if releasing the lock fails after a run committed, the failure is logged and
  the lock expires by its TTL. The run is still reported as successful.

---

## 6. Rollout and engine selection

`AllocationEngineResolver` decides the engine **per pool**, in this order:

1. `ALLOCATION_JOURNALED_ENABLED` is false -> legacy for everyone.
2. The cache key `allocation:journaled:kill` is set -> legacy for everyone (emergency stop, no deploy).
3. The pool's customer is in `ALLOCATION_JOURNALED_EXCEPT_CUSTOMERS` -> legacy.
4. The pool matches **any** of `..._ORGANIZATIONS`, `..._WAREHOUSES`, `..._CUSTOMERS` -> journaled; otherwise legacy.

The lists are a union: each phase adds an id. The customer lists compare the aggregator-resolved customer.

### Recommended phases

| Phase | Scope | Advance when |
|---|---|---|
| 0 | Deploy with `ALLOCATION_JOURNALED_ENABLED=false`. Run both migrations. Smoke test. | Migrations ran, nothing changed for users. |
| 1 | Staging, everything journaled, `per_line`. | Differential, crash and concurrency tests are green. |
| 2 | Staging, `batched`, plus a 5,000-line scale test. | Same harness is green at scale. |
| 3 | Production: one customer that receives large orders, `per_line`. | Days with no orphans, no stale runs, every compensation explained. |
| 4 | Same customer on `batched`. | Same. |
| 5 | Next customer, then warehouse 1, then organization 1, then everyone. Then schedule removal of legacy. | Same, per step. |

Rollback at any phase: flip the env value (and apply it, section 9) or set the kill key. No data migration is needed.

### Emergency kill switch
```php
Cache::put('allocation:journaled:kill', 1);   // everyone back to legacy, takes effect on the next allocation
Cache::forget('allocation:journaled:kill');   // resume
```
(`php artisan tinker`). A journaled run that is in flight at that moment is not interrupted. If it dies, the legacy allocator (or the
sweeper) compensates its journal once its lease has expired, before allocating that order.

---

## 7. Configuration reference

After any change see [section 9](#9-applying-a-change). "Raise" and "lower" describe the effect on behavior.

### 7.1 Journaled engine (`allocation.journaled`)

| Env key | Default | What it controls | If you raise it | If you lower it |
|---|---|---|---|---|
| `ALLOCATION_JOURNALED_ENABLED` | `false` | Master switch for the journaled engine. | n/a | `false` = every pool legacy. |
| `ALLOCATION_JOURNALED_ORGANIZATIONS` | empty | Comma list of organization ids that use the journaled engine. | Add ids = more pools journaled. | Remove ids = back to legacy. |
| `ALLOCATION_JOURNALED_WAREHOUSES` | empty | Same, by warehouse id. | | |
| `ALLOCATION_JOURNALED_CUSTOMERS` | empty | Same, by stock-pool customer id. | | |
| `ALLOCATION_JOURNALED_EXCEPT_CUSTOMERS` | empty | Deny list. Wins over every list above. Use to pull one customer back to legacy. | | |
| `ALLOCATION_JOURNALED_SELECTION` | `per_line` | `per_line` = reference, safest. `batched` = fast. Large-order cron ignores it and always uses `batched`. Anything else falls back to `per_line` with a warning. | | |
| `ALLOCATION_JOURNALED_LEASE` | `600` | Seconds a run counts as alive without a heartbeat. Must exceed the slowest chunk. | Dead runs are recovered later, slow chunks are safer. | Dead runs are recovered sooner; a slow chunk may be mistaken for dead (it is then fenced and compensated). |
| `ALLOCATION_JOURNALED_CHUNK_SKUS` | `200` | SKUs planned per chunk. | Fewer queries, longer pool-lock holds, more memory. | Fairer to other orders of the pool, more queries. |
| `ALLOCATION_JOURNALED_RESERVE_CHUNK` | `500` | Balance rows updated per reserve transaction. | Fewer statements. | Shorter row locks for pickers and putaway. |
| `ALLOCATION_JOURNALED_TASK_INSERT_ROWS` | `500` | Reserved (see the header of the config file). | | |
| `ALLOCATION_JOURNALED_LOST_RACE_RETRIES` | `3` | Re-plans of a chunk when another writer took the stock. Exhausted = run compensated, order deferred. | Busy pools succeed more often, runs last longer. | Fail fast. |

Fixed values (edit the config file to change): `deadlock_max_attempts` 3, `deadlock_base_wait_ms` 100,
`kill_cache_key` `allocation:journaled:kill`.

### 7.2 Pool lock (`allocation.pool_lock`)

| Env key | Default | What it controls |
|---|---|---|
| `ALLOCATION_POOL_LOCK_STORE` | `redis` | Cache store holding the lock. Must be shared by web, queue and cron processes. Keep it on Redis. |
| `ALLOCATION_POOL_LOCK_CRON_WAIT` | `30` | Seconds a cron waits for a busy pool before deferring that pool for the tick. |
| `ALLOCATION_POOL_LOCK_INSTANT_WAIT` | `8` | Same for the instant listener. Keep it short: one worker serves every customer. |
| `ALLOCATION_POOL_LOCK_REDISPATCH_DELAY` | `20` | Seconds before a busy instant allocation re-queues itself. |
| `ALLOCATION_POOL_LOCK_MAX_REDISPATCHES` | `30` | Re-queues before the job fails visibly (`AllocationPoolBusyException`). 30 x 20 s is about 10 minutes. |
| `ALLOCATION_POOL_LOCK_CHUNK_WAIT` | `60` | Mid-run wait of a cron run to re-take the lock for its next chunk. Exhausted = the run is deferred (`lock_wait`). |
| `ALLOCATION_POOL_LOCK_INSTANT_CHUNK_WAIT` | `10` | Same for instant runs. |
| `ALLOCATION_POOL_LOCK_SKIP_WAIT` | `10` | Wait for HTTP callers (skip-task switch) before answering "could not switch". |
| `ALLOCATION_POOL_LOCK_CHUNK_TTL` | `60` | Lock lifetime per journaled chunk (extended while planning). Also the longest a dead holder can block its pool. |
| `ALLOCATION_POOL_LOCK_LEGACY_TTL_INSTANT` | `240` | Lock lifetime for a legacy run on the instant queue (the queue kills at 180 s). |
| `ALLOCATION_POOL_LOCK_LEGACY_TTL_CRON` | `3600` | Lock lifetime for a legacy run started by a cron, **including legacy large orders**. Must exceed your longest legacy run or mutual exclusion is lost mid-run. |

### 7.3 Large orders (`allocation.large_orders`)

| Env key | Default | What it controls | If you raise it | If you lower it |
|---|---|---|---|---|
| `ALLOCATION_LARGE_ORDER_ALLOCATABLE_LINES` | `2000` | Threshold: an order with **more** allocatable lines is large. | More orders use the fast instant path; bigger orders risk the 180 s kill. | More orders wait for the 30-minute cron. `0` makes every order large. |
| `ALLOCATION_LARGE_ORDER_CRON` | `*/30 * * * *` | Schedule of the large cron. | Large orders wait longer. | Less waiting, more cron load. |
| `ALLOCATION_LARGE_ORDER_PER_RUN` | `10` | Orders taken per run, sequentially. | More throughput per run, longer runs. | |
| `ALLOCATION_LARGE_ORDER_START_BUDGET` | `1500` | Seconds after which no new order is started. Running orders are not killed. | | |
| `ALLOCATION_LARGE_ORDER_MAX_RUNTIME` | `3600` | Hard cap for **one** order. Past it the run is compensated and deferred. Checked between chunks. | | An order that needs longer is deferred again and again. |
| `ALLOCATION_LARGE_ORDER_OVERLAP_EXPIRES` | `100` | Minutes a killed cron blocks the next run (Laravel `withoutOverlapping`). **Must be at least (start budget + max runtime) / 60 = 85**. A test enforces it. | | Below 85 two crons could overlap after a kill. |

If you change the start budget or max runtime, change the overlap value with them.

### 7.4 Recovery (`allocation.recovery`)

| Env key | Default | What it controls |
|---|---|---|
| `ALLOCATION_RECOVERY_CRON` | `0 * * * *` | Schedule of `allocation:recover-stale-runs`, the last-resort sweeper. Allocation cron ticks also compensate expired runs. |
| `ALLOCATION_RECOVERY_FREE_COMPENSATIONS` | `2` | Compensated runs per order per 24 h that do **not** count as failed attempts (treated as deploy or OOM kills). Later ones count, so a crashing "poison" order locks instead of looping. |
| `ALLOCATION_RECOVERY_MAX_DEFERRALS` | `10` | Consecutive deferrals after which a `critical` log is written. Never locks the order. |
| `ALLOCATION_RECOVERY_LEGACY_AUTO_RESTORE` | `false` | `false` = legacy orphan recovery only **reports**. **Keep `false`** until the fulfillment-task and RCO reservers also take the pool lock (release "2b", not done); otherwise it could release another allocator's in-flight reservation. |
| `ALLOCATION_RECOVERY_ORPHAN_MIN_AGE` | `300` | Auto-restore skips balances updated more recently than this. |
| `ALLOCATION_RECOVERY_RETENTION_DAYS` | `14` | Finished runs and journal rows are deleted by the sweeper after this many days. `0` deletes them right away. |

### 7.5 What each change affects

| You change | What happens |
|---|---|
| Enable journaled for a pool | From the next allocation on, that pool's orders use the journaled engine. Large orders there become crash-safe and capped. Nothing else changes for users. |
| Set the kill key / disable journaled | All pools go to legacy at once. A large order then runs on legacy with weaker guarantees (warning logged). |
| `selection` to `batched` | Small-order cron and listener runs on journaled pools use the fast selection. The large cron was already `batched`. Same allocation result, fewer queries. |
| Lower the line threshold | More orders are marked by the click and left to the large cron. Their manual allocation takes up to one cron interval. |
| Raise the line threshold | More orders run on the instant queue / standard cron. Risk of the 180 s kill returns for orders near the new limit. |
| Raise `chunk_sku_count` | Larger chunks hold the pool lock longer; other orders of that customer wait longer. |
| Lower `lease_seconds` | Dead runs are freed sooner. Must stay above the slowest chunk or live runs get fenced. |
| Change cron expressions | Large cron or sweeper runs more or less often. Clear the schedule cache afterwards (section 9). |
| Turn `legacy_auto_restore` on | **Do not** until 2b is done. |

---

## 8. Operating it

### 8.1 Scheduler

| Command | Schedule | Purpose |
|---|---|---|
| `process:standard-order` | at :04, :14, :24, :34, :44, :54 | Small orders in auto warehouses. |
| `allocation:process-large-orders` | `ALLOCATION_LARGE_ORDER_CRON` (30 min) | Large orders. |
| `allocation:recover-stale-runs` | `ALLOCATION_RECOVERY_CRON` (hourly) | Compensates expired runs, purges old finished runs. |

`php artisan allocation:recover-stale-runs --dry-run` lists expired runs and how much stock they hold without changing anything.

### 8.2 Logs (channel `wms`, `storage/logs/wms-YYYY-MM-DD.log`)

| Message contains | Meaning |
|---|---|
| `allocation:process-large-orders started / finished` | One line per large cron run, with counts: allocated, deferred, failed, skipped pool. |
| `Large order allocation started / finished / deferred` | Per order, with seconds, run id, lost races. |
| `large order allocated by the legacy engine` | A large order is running without journal or cap. Move that pool to journaled. |
| `journaled allocation deferred (<reason>)` | A run was compensated for a deliberate reason (not an error). |
| `journaled run was fenced` | Recovery compensated this run; the worker wrote nothing. |
| `could not compensate allocation run` (critical) | Compensation failed. The lease expiring is the backstop; the sweeper retries. |
| `has been deferred N times in a row` (critical) | Look at this order: its pool is busy, or it keeps losing races or hitting the cap. |
| `could not release the allocation pool lock` | Redis error on release. The lock expires by TTL; the run result is unaffected. |
| `Allocation pool lock expired during a legacy run` | Mutual exclusion was lost during a legacy run. Raise `legacy_run_ttl_cron_seconds` or move the pool to journaled. |

### 8.3 Looking at the data

```sql
-- runs of an order, newest first
SELECT id, status, reason, selection, line_count, lost_races, started_at, finished_at
FROM portal.allocation_runs WHERE order_id = :order_id ORDER BY id DESC;

-- runs that are alive right now, and dead ones the sweeper has not reached yet
SELECT id, order_id, lease_until, lease_until < NOW() AS expired
FROM portal.allocation_runs WHERE status = 'running';

-- what a run holds
SELECT balance_id, qty FROM portal.allocation_run_reservations WHERE run_id = :run_id;

-- large orders waiting on a request
SELECT ordersId, warehouseId, allocation_requested_at FROM portal.orders WHERE allocation_requested_at IS NOT NULL;
```

### 8.4 A stuck order: where to look

| Symptom | Likely cause | What to do |
|---|---|---|
| User clicked allocate on a big order, nothing happened for a while | It is queued for the large cron (up to one cron interval, plus busy-pool deferrals). | Check `allocation_requested_at` and the large cron log lines. |
| "Allocation is already started" | `is_allocating = 1`. A run is live, or a worker died without recovery. | Look for a `running` row in `allocation_runs`. If none and the flag stays, a legacy run died: check orphans before clearing the flag by hand. |
| Order keeps being deferred | Its pool is busy, or lost races / runtime cap. | Look for the `critical` deferral log; check what else allocates in that pool; raise `max_order_runtime_s` or the lost-race retries if justified. |
| Order is locked ("Locked by system for reaching allocation threshold") | 18 failed attempts. | Fix the cause (stock, rules), then unlock the order as before. |
| `allocated_qty` reserved with no tasks | Orphans from a legacy run that died. | Journaled runs cannot leave orphans. Legacy orphan recovery only reports until 2b is done. |

---

## 9. Applying a change

1. Edit `.env` (never commit it; new keys go in `.env.example`).
2. `php artisan config:cache` (config is cached).
3. `php artisan horizon:terminate` (queue workers keep the old config until they restart).
4. After changing a **cron expression, the overlap value or after a deploy/restart**: `php artisan schedule:clear-cache`
   (a stale `withoutOverlapping` mutex can block the large cron for up to `overlap_expires_after_minutes`).

### First deployment checklist

1. Run the migrations on the `portal` connection:
   * `2026_10_23_120000_create_allocation_runs_tables.php` (`allocation_runs`, `allocation_run_reservations`)
   * `2026_10_23_130000_add_allocation_requested_at_to_orders_table.php` (nullable column, instant change on MySQL 8)
2. Keep `ALLOCATION_JOURNALED_ENABLED=false` for the first deploy. Behavior is then unchanged.
3. Make sure Redis is reachable from web, queue and cron (the pool lock needs it).
4. Follow the rollout phases in section 6.

---

## 10. Known limitations

* **Legacy engine on large orders** has weaker guarantees (section 4.3). Journaled first for pools that get large orders.
* **Release 2b is not done:** fulfillment-task, RCO and B2B reservers do not take the pool lock yet. Until they do,
  keep `legacy_auto_restore=false`.
* **Stranded marker:** see the last bullet of section 4.5.
* **Fairness:** the large cron is global and oldest-first. One organization requesting many large orders can use up
  most of a run's capacity.
* **The runtime cap is checked between chunks**, so a single chunk is never interrupted.
* **Open review items** (tracked, not yet changed): a failed order-status refresh after a committed run is counted as
  a failed attempt; one failing run stops the whole expired-run sweep; the listener labels every non-run result as
  "pool busy"; legacy `lock_lost` is only logged; carton numbers are concatenated into one `orderByRaw`; some log lines
  include whole order objects or stack traces with arguments.
* **Empty env values** become `0` (`(int) ""`): an empty line threshold makes every order large, an empty retention
  deletes finished runs immediately. Leave a key out instead of leaving it empty.

---

## 11. Tests

Allocation tests live in `tests/Feature/WMS/Allocation` and need Redis and the testing database
(`iqh_portal_test`) with the journal tables and the marker column (run the two migrations with
`APP_ENV=testing php artisan migrate --env=testing --path=<migration>`).

* All allocation tests: `php artisan test tests/Feature/WMS/Allocation`
* Differential acceptance run (legacy vs `per_line` vs `batched`): `ALLOCATION_DIFF_SCENARIOS=1000 php artisan test --filter AllocationEngineDifferentialTest`
* Replay a seed: `ALLOCATION_DIFF_SEED_START=<seed> ALLOCATION_DIFF_SCENARIOS=1 php artisan test --filter AllocationEngineDifferentialTest`
* Tests using Mockery aliases or real models run in separate processes. Never run DDL inside a `DatabaseTransactions` test.

## 12. Where the code lives

| Area | Path |
|---|---|
| Engines, factory, resolver | `app/Helpers/Allocation/Journaled/` (`StandardOrderAllocatorFactory`, `AllocationEngineResolver`, `LegacyStandardOrderAllocator`, `JournaledStandardOrderAllocator`, `BatchedJournaledStandardOrderAllocator`) |
| Journal, recovery | `AllocationRunJournal`, `AllocationRunRecovery`, `LegacyRunRecovery` (same folder) |
| Pool lock | `AllocationPoolLock`, `RedisPoolLock`, `PoolLockHandle` |
| Routing and marker | `LargeOrderRouting`, `AllocationRequest` |
| Exceptions | `app/Exceptions/Custom/Allocation*Exception.php` |
| Commands | `app/Console/Commands/Allocation/` (`ProcessStandardOrderCommand`, `ProcessLargeOrdersCommand`, `RecoverStaleAllocationRunsCommand`) |
| Entry points | `OrderController::instant_order_allocation`, `InstantAllocationStandardOrderEventListener` |
| Retry schedule and lock threshold | `app/Helpers/Allocation/AllocationAttemptHelper.php` |
| Configuration | `config/allocation.php`, `.env.example`, schedule in `app/Console/Kernel.php` |
