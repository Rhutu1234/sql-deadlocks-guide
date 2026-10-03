# Deadlocks (SQL Server)

*A deep-dive walkthrough of database deadlocks — covering the wait-for graph as the precise definition of a deadlock, the specific, named patterns that cause the overwhelming majority of real-world deadlocks (cycle deadlocks, bookmark-lookup deadlocks, conversion deadlocks), how SQL Server's deadlock monitor detects a cycle and chooses a victim, capturing and reading an actual deadlock graph, how index design directly determines deadlock likelihood, `DEADLOCK_PRIORITY`, and a systematic, step-by-step process for eliminating a recurring deadlock rather than just retrying around it forever.*

---

## Table of Contents

1. [Introduction](#introduction)
2. [The Precise Definition: A Cycle in the Wait-For Graph](#1-the-precise-definition-a-cycle-in-the-wait-for-graph)
3. [How SQL Server Detects a Deadlock](#2-how-sql-server-detects-a-deadlock)
4. [How the Victim Is Chosen](#3-how-the-victim-is-chosen)
5. [Pattern 1: The Classic Cycle Deadlock](#4-pattern-1-the-classic-cycle-deadlock)
6. [Pattern 2: The Bookmark/Key Lookup Deadlock](#5-pattern-2-the-bookmarkkey-lookup-deadlock)
7. [Pattern 3: The Conversion Deadlock](#6-pattern-3-the-conversion-deadlock)
8. [Pattern 4: Same-Table, Different Order — Caused by a Missing Index](#7-pattern-4-same-table-different-order--caused-by-a-missing-index)
9. [Capturing a Deadlock Graph](#8-capturing-a-deadlock-graph)
10. [Reading a Deadlock Graph](#9-reading-a-deadlock-graph)
11. [How Isolation Level Affects Deadlock Likelihood](#10-how-isolation-level-affects-deadlock-likelihood)
12. [DEADLOCK_PRIORITY](#11-deadlock_priority)
13. [A Systematic Elimination Process](#12-a-systematic-elimination-process)
14. [Retry Logic: The Safety Net, Not the Fix](#13-retry-logic-the-safety-net-not-the-fix)
15. [Common Pitfalls](#14-common-pitfalls)
16. [Quick Reference Table](#quick-reference-table)
17. [Conclusion](#conclusion)

---

## Introduction

This series' Transactions guide's Section 10 and this series' Threading guide's Section 5 both introduce deadlocks — a circular wait where each party holds something the other needs — as a general concept, applicable equally to an in-process `lock` and a database row lock. This guide goes considerably deeper into the database-specific version: SQL Server's precise definition of a deadlock as a cycle in a directed wait-for graph, the handful of *named, recognizable patterns* that account for the overwhelming majority of real-world deadlocks (not every deadlock looks like the textbook "transfer money between two accounts" example), how to actually capture and read a deadlock graph rather than guessing at the cause, and a systematic process for eliminating a recurring deadlock at its root rather than permanently retrying around it.

```plaintext
Session 51: holds a lock on Row A, WAITS for a lock on Row B
Session 52: holds a lock on Row B, WAITS for a lock on Row A
   ↓
This is a CYCLE in the wait-for graph — SQL Server's deadlock monitor
  detects it, picks a VICTIM (Section 3), kills that session's
  transaction, and lets the other proceed.
```

---

## 1. The Precise Definition: A Cycle in the Wait-For Graph

### Every blocked session is an edge in a graph: "I am waiting for YOU"

```plaintext
At any moment, SQL Server can construct a directed graph where each NODE
  is a session, and an EDGE from Session A to Session B means "A is
  currently blocked, waiting on a lock held by B." Most of the time this
  graph is a simple chain or tree — A waits for B, who isn't waiting for
  anyone — and resolves naturally once B finishes and releases its lock.
```

### A deadlock exists precisely when this graph contains a CYCLE

```plaintext
A → waits for → B → waits for → A
```

This is the exact, formal condition — not "two transactions running slowly against each other," not "a long-running query," but specifically a cycle: a path through the wait-for graph that returns to its own starting point. This is worth holding as the precise mental model for everything else in this guide, because every named pattern in Sections 4-7 is simply a different, concrete way a real application can produce exactly this graph shape — the underlying condition is always identical.

### Why "ordinary blocking" and "deadlock" are genuinely different problems

```plaintext
Blocking (no deadlock): Session B waits patiently for Session A to
  finish and release its lock — slow, possibly frustrating, but it WILL
  resolve on its own once A completes.
Deadlock: NEITHER session can EVER finish, because each is waiting on
  the OTHER — without intervention, this is a permanent, unresolvable standstill.
```

This distinction matters because the fixes are different — ordinary blocking is addressed by reducing lock duration or improving concurrency (shorter transactions, better indexes, appropriate isolation); a deadlock *requires* something to break the cycle, which is precisely why SQL Server's deadlock monitor (Section 2) exists as an active, independent mechanism rather than simply "wait longer."

---

## 2. How SQL Server Detects a Deadlock

### A background thread, the Lock Monitor, periodically scans for cycles

```plaintext
SQL Server runs a dedicated background thread that checks the wait-for
  graph for cycles at a regular interval — by default, every 5 SECONDS,
  though this interval SHRINKS automatically (down toward a 100ms
  floor) once a deadlock IS found, specifically so that a BURST of
  related deadlocks (several sessions deadlocking against each other in
  quick succession) gets resolved quickly rather than each one waiting
  out the full 5-second cycle independently.
```

This is worth knowing precisely because it explains a genuinely common observation: the *first* deadlock in a cluster of related ones can take up to 5 seconds to be detected and resolved, while subsequent ones (from the same underlying contention) resolve far faster, since the monitor has already shifted into its faster-polling mode.

### Why detection, not prevention, is the chosen strategy

```plaintext
SQL Server does NOT try to prevent deadlocks from ever occurring (which
  would require either extremely conservative, concurrency-killing
  locking, or perfect foreknowledge of every transaction's future lock
  needs) — it accepts that deadlocks are a normal, occasional
  consequence of genuinely concurrent locking, DETECTS them when they
  happen, and resolves them automatically by sacrificing one participant.
```

This is a deliberate, reasonable engineering trade-off worth understanding as such, not a limitation — this series' Threading guide's Section 5 covers the alternative, prevention-based approach (a consistent lock-acquisition order) for in-process code, and the exact same prevention technique applies here too (Section 4) — but the database additionally provides automatic detection and resolution as a safety net for cases prevention alone doesn't catch.

---

## 3. How the Victim Is Chosen

### The default rule: the transaction that's done the LEAST work to undo is sacrificed

```plaintext
SQL Server estimates the relative COST of rolling back each participant
  in the cycle — specifically, how much LOG has been generated by each
  transaction so far — and chooses the one that's CHEAPEST to roll back
  as the victim, on the reasoning that undoing less work is less
  wasteful than undoing more.
```

This is worth knowing precisely because it means the victim is **not** necessarily the transaction that "caused" the problem, and isn't chosen by any notion of fairness or priority (unless `DEADLOCK_PRIORITY`, Section 11, is explicitly set) — it's a purely cost-based heuristic, which is exactly why any application code touching a table that's ever involved in deadlocks needs to be prepared to be the victim itself, regardless of how small or seemingly innocent its own transaction looks.

### What actually happens to the victim

```plaintext
The victim's ENTIRE transaction is rolled back automatically (per this
  series' Transactions guide's Section 11 undo mechanism), its locks
  are released, and SQL Server raises error 1205 in that session —
  WHICHEVER OTHER transaction(s) were part of the cycle can now proceed,
  since the resource they were waiting on has just been freed.
```

---

## 4. Pattern 1: The Classic Cycle Deadlock

### Two transactions, two resources, opposite acquisition order

```sql
-- Session 51:
BEGIN TRAN;
UPDATE Accounts SET Balance = Balance - 100 WHERE Id = 1;  -- X lock on row 1
-- (pause)
UPDATE Accounts SET Balance = Balance + 100 WHERE Id = 2;  -- WAITS for row 2

-- Session 52, running CONCURRENTLY:
BEGIN TRAN;
UPDATE Accounts SET Balance = Balance - 50 WHERE Id = 2;   -- X lock on row 2
-- (pause)
UPDATE Accounts SET Balance = Balance + 50 WHERE Id = 1;    -- WAITS for row 1  →  CYCLE
```

This is precisely the pattern this series' Transactions guide's Section 10 introduces, restated here as the first of several named patterns worth being able to recognize at a glance, since real-world deadlocks almost always resemble one of a small handful of shapes rather than being genuinely novel each time.

### The fix: a CONSISTENT acquisition order, applied universally

```sql
-- Both sessions now touch accounts in ASCENDING Id order, regardless of
-- which account is logically the "source" or "destination" of the transfer
UPDATE Accounts SET Balance = Balance - @amount WHERE Id = @lowerId;
UPDATE Accounts SET Balance = Balance + @amount WHERE Id = @higherId;
```

Exactly this series' Threading guide's Section 5 lock-ordering discipline, applied to database rows instead of in-process `lock` objects — if every transaction that touches both row 1 and row 2 always acquires row 1's lock first, the circular wait this pattern depends on becomes structurally impossible, since neither transaction can ever be holding row 2 while waiting for row 1.

---

## 5. Pattern 2: The Bookmark/Key Lookup Deadlock

### A reader and a writer colliding over the SAME row, via two DIFFERENT access paths

```plaintext
Session 51 (a SELECT, using a non-clustered index): per this series' SQL
  Indexes guide's Section 7, this SEEKS the non-clustered index first,
  then performs a KEY LOOKUP back to the clustered index to retrieve
  additional columns — acquiring a SHARED lock on the non-clustered
  index entry FIRST, then attempting one on the CLUSTERED index row.
Session 52 (an UPDATE, modifying an indexed column): acquires an
  EXCLUSIVE lock on the CLUSTERED index row first (to perform the
  update), then needs to update the NON-CLUSTERED index entry too
  (since the indexed column changed) — requiring a lock on THAT structure next.
```

This produces the exact same cycle shape as Section 4's pattern, but the two "resources" being contended for aren't two different *rows* — they're the *same logical row*, accessed through two different physical structures (the non-clustered index and the clustered index/heap) in opposite order by the reader versus the writer, which is precisely why this series' SQL Indexes guide's Section 7 covering-index technique is directly relevant here too.

### Why covering indexes reduce this specific pattern's likelihood

```plaintext
A COVERING index (per this series' SQL Indexes guide's Section 7)
  eliminates the key lookup entirely for queries it fully satisfies — if
  Session 51's query never needs to touch the clustered index at all
  (every column it needs is already in the covering non-clustered
  index), there's no second resource for it to contend with Session
  52's update over, closing this specific deadlock pattern at its root,
  not just making it less likely.
```

---

## 6. Pattern 3: The Conversion Deadlock

### Two sessions each holding a SHARED lock, both trying to UPGRADE it to EXCLUSIVE on the same resource

```sql
-- Both sessions run essentially the SAME statement, CONCURRENTLY:
UPDATE Accounts SET Balance = Balance + 10 WHERE Id = 1;
-- Internally, SQL Server first takes a SHARED lock to READ the current
-- value and evaluate the WHERE clause, then needs to CONVERT it to an
-- EXCLUSIVE lock to actually perform the write.
```

```plaintext
Session 51: holds a SHARED lock on row 1, wants to CONVERT to EXCLUSIVE.
Session 52: ALSO holds a SHARED lock on the SAME row 1, ALSO wants to convert.
→ Neither can convert, because the OTHER session's shared lock is
  blocking the conversion — a cycle, even though BOTH sessions are
  doing the exact same, single-row operation.
```

This is a genuinely counter-intuitive pattern worth knowing exists specifically because it doesn't fit the "two resources, opposite order" mental model at all — it can occur with a single row and a single statement type, purely from the shared-to-exclusive lock conversion step, and it's a real, recognized SQL Server deadlock category distinct from the classic cycle pattern.

### Why this is comparatively rare in practice, and what tends to trigger it

```plaintext
Conversion deadlocks are most commonly seen with UPDATE statements under
  REPEATABLE READ or SERIALIZABLE isolation (per this series'
  Transactions guide's Section 8), where shared locks are held LONGER
  (for the whole transaction, not just the duration of the read), giving
  a genuine WINDOW for two sessions to both acquire the shared lock
  before either attempts the conversion — under the more common READ
  COMMITTED, the shared-lock window is typically too brief for this to
  occur as often.
```

---

## 7. Pattern 4: Same-Table, Different Order — Caused by a Missing Index

### A deadlock that LOOKS like it involves many rows, but traces back to one missing index

```sql
UPDATE Orders SET Status = 'Shipped' WHERE CustomerId = 42;
```

```plaintext
If NO useful index exists on CustomerId, per this series' SQL Indexes
  guide's Section 4, this UPDATE performs a full SCAN of the table —
  acquiring (and, depending on lock escalation, per that guide's
  discussion and this series' Transactions guide's Section 8, possibly
  ESCALATING to) locks across a much WIDER range of rows than the
  logically-intended "just this customer's orders."
```

```plaintext
Two such UPDATEs, each targeting a DIFFERENT customer but each FORCED to
  scan the WHOLE table due to the missing index, can acquire their
  respective locks in an order that depends purely on PHYSICAL ROW
  ORDER rather than any logical relationship — producing a deadlock
  between two transactions that, from the application's point of view,
  never "should" have conflicted at all, since they're logically
  touching entirely different customers' data.
```

This is a genuinely important, often-missed pattern worth internalizing as its own category — it's not that the *application logic* has a lock-ordering bug (Section 4's fix doesn't directly apply, since there's no deliberate ordering decision to reconsider); the deadlock is a direct, indirect symptom of a *missing index*, and the actual fix is adding the index that lets each `UPDATE` seek directly to its own customer's rows (per this series' SQL Indexes guide's Section 3) rather than scanning — which is precisely why Section 12's systematic process treats checking for missing/inadequate indexing as an early, high-value diagnostic step, not an afterthought.

---

## 8. Capturing a Deadlock Graph

### The system health session: on by default, capturing every deadlock automatically

```sql
SELECT CAST(xe.target_data AS XML) AS DeadlockGraphs
FROM sys.dm_xe_session_targets xe
JOIN sys.dm_xe_sessions s ON xe.event_session_address = s.address
WHERE s.name = 'system_health'
  AND xe.target_name = 'ring_buffer';
```

SQL Server's built-in `system_health` Extended Events session captures deadlock graphs automatically, with no setup required — this is genuinely the first place to look after a deadlock error surfaces, since it's always running and retains recent history without needing to have anticipated the specific deadlock in advance.

### A dedicated Extended Events session, for ongoing, deliberate monitoring

```sql
CREATE EVENT SESSION DeadlockMonitor ON SERVER
ADD EVENT sqlserver.xml_deadlock_report
ADD TARGET package0.event_file(SET filename = N'DeadlockMonitor');

ALTER EVENT SESSION DeadlockMonitor ON SERVER STATE = START;
```

For an application with recurring deadlock issues worth tracking deliberately over time (rather than relying on `system_health`'s rolling, limited retention), a dedicated session writing to a file target gives a durable, queryable history — worth setting up specifically once Section 12's systematic process identifies a pattern that needs sustained observation to fully characterize.

### The older trace flag, still occasionally seen in legacy environments

```sql
DBCC TRACEON(1222, -1);  -- writes deadlock graph details to the SQL Server error log
```

Worth recognizing this if encountered in an older environment, though Extended Events (above) is the modern, recommended, and lower-overhead approach — trace flag 1222 predates Extended Events and writes less structured output to the error log rather than a queryable XML document.

---

## 9. Reading a Deadlock Graph

### The three-part structure: victim list, process list, resource list

```xml
<deadlock>
  <victim-list>
    <victimProcess id="process1" />
  </victim-list>
  <process-list>
    <process id="process1" ... waitresource="KEY: ..." lockMode="X" ...>
      <executionStack>...</executionStack>
      <inputbuf>UPDATE Accounts SET Balance = Balance + @amount WHERE Id = @lowerId</inputbuf>
    </process>
    <process id="process2" ... waitresource="KEY: ..." lockMode="X" ...>
      <inputbuf>UPDATE Accounts SET Balance = Balance - @amount WHERE Id = @higherId</inputbuf>
    </process>
  </process-list>
  <resource-list>
    <keylock objectname="MyDb.dbo.Accounts" ... >
      <owner-list><owner id="process2" mode="X" /></owner-list>
      <waiter-list><waiter id="process1" mode="X" /></waiter-list>
    </keylock>
    ...
  </resource-list>
</deadlock>
```

This is the actual, structured record of the exact scenario Section 1's wait-for graph describes, in concrete form: `<victim-list>` names which process was killed; each `<process>` entry shows the actual SQL statement (`<inputbuf>`) that was running, which transaction isolation level, and what resource it was waiting on; each `<resource>` entry shows precisely which process currently *owns* the lock and which is *waiting* for it — this `owner`/`waiter` pairing, cross-referenced across every resource listed, is exactly how you reconstruct the specific cycle that occurred.

### The practical reading process

```plaintext
1. Identify the VICTIM from <victim-list>.
2. For EACH process, read its <inputbuf> — this tells you the ACTUAL
   statement each participant was running, which is usually enough to
   recognize which of Sections 4-7's named PATTERNS you're looking at.
3. Cross-reference the resource-list's owner/waiter pairs against the
   process IDs to confirm the EXACT cycle — which process was waiting
   on which specific lock, held by which other process.
4. Check isolationlevel on each process — per Section 10, this directly
   affects how long locks were held and therefore how LIKELY this
   specific collision was.
```

This is worth treating as a genuinely mechanical, learnable procedure — a deadlock graph contains everything needed to definitively identify the pattern and the responsible statements; the skill is in reading it methodically rather than guessing from the error message alone, which, by itself, rarely names enough detail to diagnose the real cause.

---

## 10. How Isolation Level Affects Deadlock Likelihood

### Higher isolation levels hold locks longer, widening the collision window

```plaintext
Per this series' Transactions guide's Section 8: READ COMMITTED releases
  shared locks almost immediately after each read; REPEATABLE READ and
  SERIALIZABLE hold them for the WHOLE transaction — the LONGER a lock
  is held, the LONGER the window during which another transaction can
  attempt to acquire a conflicting lock on the same resource, directly
  increasing the probability that TWO such attempts collide into a cycle.
```

This is the direct, quantitative link between this series' Transactions guide's isolation-level trade-off and this guide's own subject — choosing `SERIALIZABLE` "to be safe," per that guide's Section 7 caution, doesn't just cost raw throughput; it measurably increases deadlock frequency too, which is a second, additional reason (beyond blocking alone) to apply strong isolation narrowly rather than as a blanket default.

### Snapshot/row-versioning isolation as a genuine deadlock-reduction technique

```plaintext
Per this series' Transactions guide's Section 9: under snapshot
  isolation, READERS take no shared locks at all — which means readers
  can NEVER participate in a lock-based deadlock cycle, since they're
  not contending for locks in the first place. This doesn't eliminate
  WRITER-versus-WRITER deadlocks (Section 4's pattern can still occur
  between two writers), but it directly eliminates Section 5's reader-
  versus-writer bookmark lookup deadlock pattern entirely.
```

---

## 11. DEADLOCK_PRIORITY

### Overriding the default "cheapest to roll back" victim selection

```sql
SET DEADLOCK_PRIORITY LOW;   -- this SESSION should be PREFERRED as the victim, if a deadlock occurs
-- or a numeric value from -10 to 10, for finer-grained relative priority
```

This lets specific, known transactions explicitly volunteer (or refuse to volunteer) as the deadlock victim, overriding Section 3's default cost-based heuristic — genuinely useful for a background/batch process that's safe and cheap to retry, run alongside latency-sensitive, user-facing transactions that genuinely should not be the one sacrificed.

### A real, deliberate use case

```sql
-- In a nightly batch job, touching the same tables user-facing requests also touch:
SET DEADLOCK_PRIORITY LOW;
BEGIN TRAN;
    -- bulk maintenance work
COMMIT;
```

Worth applying deliberately and narrowly — a background reporting or cleanup job explicitly marking itself as the preferred victim means that if it ever does collide with a real-time user request, the user's transaction survives and the batch job (which can simply retry on its own schedule, per Section 13) absorbs the rollback instead, which is a genuinely sensible, low-cost way to protect latency-sensitive paths without needing to eliminate the underlying contention entirely.

---

## 12. A Systematic Elimination Process

### The order worth following, from cheapest-to-check to most involved

```plaintext
1. CAPTURE the actual deadlock graph (Section 8) — never guess at the
   cause from the bare error message alone.
2. READ it (Section 9) to identify which NAMED PATTERN (Sections 4-7)
   it matches — this immediately suggests the likely category of fix.
3. If it's Pattern 4 (Section 7): check whether an appropriate INDEX
   exists on the columns involved, per this series' SQL Indexes guide —
   this is often the SINGLE highest-leverage fix, eliminating the
   deadlock by eliminating the unnecessary table scan causing it.
4. If it's Pattern 1 (Section 4): identify whether the colliding
   transactions can be rewritten to acquire resources in a CONSISTENT order.
5. If it's Pattern 2 (Section 5): consider a COVERING index to eliminate
   the key lookup, per this series' SQL Indexes guide's Section 7.
6. Check the ISOLATION LEVEL (Section 10) in use — is SERIALIZABLE or
   REPEATABLE READ genuinely needed here, or would READ COMMITTED (or
   snapshot isolation) suffice and reduce the lock-holding window?
7. Only AFTER attempting a genuine, root-cause fix, apply retry logic
   (Section 13) as the remaining safety net for whatever residual
   deadlock risk the fix didn't fully eliminate.
```

This ordering is worth following deliberately rather than jumping straight to retry logic — Section 13 makes the case directly that retry-only is a real, if common, anti-pattern: it treats a diagnosable, often fixable root cause as an unavoidable fact of life, when a meaningful fraction of real-world deadlocks (Pattern 4 especially) trace back to a missing index or a correctable acquisition order that, once fixed, eliminates the deadlock's actual *cause*, not just its symptom.

---

## 13. Retry Logic: The Safety Net, Not the Fix

### Why retry is always necessary, even after root-cause elimination

```plaintext
Even a perfectly-ordered, well-indexed system can never reduce deadlock
  probability to ABSOLUTE zero under genuine concurrent load — retry
  logic is therefore always a legitimate, necessary SAFETY NET, exactly
  as this series' Transactions guide's Section 10 covers — the mistake
  is treating it as the PRIMARY defense rather than the backstop for
  whatever Section 12's systematic elimination process doesn't fully close out.
```

### The concrete retry pattern, revisited with this guide's own depth

```csharp
for (int attempt = 0; attempt < 3; attempt++)
{
    try
    {
        using var transaction = await connection.BeginTransactionAsync(IsolationLevel.ReadCommitted);
        await ExecuteWorkAsync(transaction);
        await transaction.CommitAsync();
        break;
    }
    catch (SqlException ex) when (ex.Number == 1205 && attempt < 2)
    {
        await Task.Delay(TimeSpan.FromMilliseconds(Random.Shared.Next(20, 100) * (attempt + 1)));
        // a RANDOMIZED backoff, specifically to avoid two retrying sessions
        // colliding AGAIN on their very next, simultaneous attempt
    }
}
```

The randomized component in the backoff is worth calling out specifically — if both sides of a deadlock retry after an identical, fixed delay, they have a real chance of colliding again immediately upon their synchronized retry; a small, randomized jitter (per this series' Rate Limiter guide's own jitter discussion for a structurally similar reason) spreads retries apart in time, genuinely reducing the chance of an immediate repeat collision.

---

## 14. Common Pitfalls

| Pitfall | Why it hurts | Better approach |
|---|---|---|
| Diagnosing a deadlock from the bare error message alone | The error names the victim but not the actual statements, resources, or cycle involved | Always capture and read the actual deadlock graph (Sections 8-9) before attempting a fix |
| Treating every deadlock as the classic "two accounts, opposite order" pattern | Many real-world deadlocks (bookmark lookups, conversion deadlocks, missing-index scans) have different root causes and different fixes | Recognize which of the named patterns (Sections 4-7) a specific deadlock actually matches before choosing a remedy |
| Assuming a recurring deadlock between two seemingly-unrelated transactions is an application logic bug | It's often an indirect symptom of a missing index forcing a table scan, not a deliberate, flawed acquisition order | Check indexing on the columns involved before assuming the application's locking logic itself needs restructuring (Section 7) |
| Relying on retry logic alone, indefinitely | Treats a diagnosable, often-fixable root cause as unavoidable, leaving real throughput and latency cost on the table | Use Section 12's systematic process to eliminate the root cause first; keep retry logic as the remaining safety net (Section 13) |
| Retrying with a fixed, non-randomized delay | Two colliding sessions retrying after the same fixed delay can collide again immediately | Add randomized jitter to retry backoff (Section 13) |
| Defaulting to `SERIALIZABLE`/`REPEATABLE READ` without a specific need | Directly increases deadlock likelihood by widening the lock-holding window, beyond just reducing throughput | Use `READ COMMITTED` (or snapshot isolation) by default; reserve stronger isolation for operations that specifically need it (Section 10) |
| Assuming the victim transaction is always the one that "caused" the problem | Victim selection is based purely on rollback cost, with no relation to fault or causation | Investigate every participant in the deadlock graph, not just whichever one happened to be rolled back (Section 3) |
| Never considering `DEADLOCK_PRIORITY` for known, low-stakes batch workloads | A background job can repeatedly sacrifice a latency-sensitive, user-facing transaction purely by chance | Set a lower `DEADLOCK_PRIORITY` on background/batch work that collides with user-facing transactions (Section 11) |

---

## Quick Reference Table

| Concept | Mechanism | Key Point |
|---|---|---|
| Formal definition | Cycle in the wait-for graph | Ordinary blocking resolves on its own; a cycle never does without intervention |
| Detection | Background Lock Monitor | Runs every ~5s by default, faster once a deadlock is found |
| Victim selection | Lowest estimated rollback cost | Not based on fault — any participant can be chosen |
| Classic cycle | Two resources, opposite acquisition order | Fix: consistent acquisition order everywhere (Section 4) |
| Bookmark lookup | Reader/writer colliding via index vs. clustered index | Fix: a covering index removing the lookup (Section 5) |
| Conversion deadlock | Shared-to-exclusive lock upgrade collision | More common under REPEATABLE READ/SERIALIZABLE (Section 6) |
| Missing-index pattern | A scan acquires locks in physical, not logical, order | Fix: add the missing index (Section 7) |
| Capture | `system_health` Extended Events session | Always running by default; no setup required |
| Victim protection | `SET DEADLOCK_PRIORITY LOW/HIGH/n` | Lets known, low-stakes work volunteer as the preferred victim |

---

## Conclusion

A deadlock is, formally, nothing more than a cycle in a wait-for graph — and the practical craft of dealing with them well comes from recognizing that real-world deadlocks cluster into a small number of named, recognizable shapes, not an infinite variety of unique scenarios. The classic two-resource cycle this series' Transactions guide introduces is only one of them; bookmark-lookup deadlocks, conversion deadlocks, and the missing-index pattern that produces a deadlock between transactions that never should have logically collided at all are each genuinely common, each have a specific diagnostic signature in an actual deadlock graph, and each have a specific, often root-cause-eliminating fix — a consistent acquisition order, a covering index, or simply the index that was missing all along.

The discipline this guide argues for throughout — capture the actual graph, identify the pattern, fix the root cause, and keep retry logic as the remaining safety net rather than the primary defense — is what separates a system that's merely tolerant of occasional deadlocks from one that's been deliberately, measurably hardened against the specific patterns that actually occur in it. Retry logic alone will always work, in the sense that it eventually succeeds; it just leaves real, fixable throughput and latency cost sitting on the table, for exactly the same reason this series' SQL Indexes and Execution Plans guides argue throughout: a specific, diagnosed problem deserves a specific fix, not a general-purpose workaround applied indefinitely.

---

*Found this useful? Feel free to star the repo, open an issue with corrections, or share the two-completely-unrelated-customers'-orders-deadlocking-against-each-other discovery that traced back to one missing index and made Pattern 4 click far better than any textbook two-account example ever could.*
