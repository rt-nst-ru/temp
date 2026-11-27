# YouTube DBMS Project — 30 Viva Questions

> **Legend:** 🟢 Easy · 🟡 Medium · 🔴 Hard
> Tags indicate the **phase**, **topic area**, and relevant **query IDs** where applicable.

---

## Q1 🟢 Easy
**Phase:** 1 · **Tags:** `problem-understanding`, `stakeholders`

**Q:** Problem P5 says "subscribers who never watched" always returns empty. Which stakeholder is affected, and is this a customer-facing or reporting/analytics issue?

**Expected Answer:** The **Creators / YouTube Studio** team is affected. It's a **reporting/analytics** issue — creators use this insight to understand engagement gaps. The report returning empty suggests a query bug (likely the `NOT IN` with NULLs problem in R13), not that every subscriber has watched.

---

## Q2 🟢 Easy
**Phase:** 2 · **Tags:** `data-types`, `schema-design`

**Q:** What PostgreSQL data type should `video_id` be, and why not `INTEGER` or `SERIAL`?

**Expected Answer:** `CHAR(11)` or `VARCHAR(11)`. YouTube video IDs are 11-character alphanumeric strings (e.g., `__21WjTDc9K`). They contain letters, digits, underscores, and hyphens — not integers. Using an integer type would reject all seed data.

---

## Q3 🟢 Easy
**Phase:** 2 · **Tags:** `constraints`, `CHECK`

**Q:** Give three examples of CHECK constraints the project requires.

**Expected Answer:** *(Any 3 from):*
- `video_id` is exactly 11 characters: `CHECK (char_length(video_id) = 11)`
- `handle` starts with `@`: `CHECK (handle LIKE '@%')`
- `country_code` is exactly 2 characters: `CHECK (char_length(country_code) = 2)`
- Shorts ≤ 60 seconds: `CHECK (NOT is_short OR duration_sec <= 60)`
- `visibility` in `('PUBLIC','PRIVATE','UNLISTED','SCHEDULED')`
- `reaction` in `('LIKE','DISLIKE')`
- `revenue_inr >= 0`

---

## Q4 🟢 Easy
**Phase:** 2 · **Tags:** `keys`, `primary-key`

**Q:** What is the primary key of `subscriptions`? Why isn't a single column enough?

**Expected Answer:** Composite PK: `(user_id, channel_id)`. A user can subscribe to many channels, and a channel has many subscribers, but a user subscribes to a given channel at most once. Neither column alone is unique.

---

## Q5 🟢 Easy
**Phase:** 2 · **Tags:** `NULL`, `data-analysis`

**Q:** Why is `user_id` nullable in `watch_events`? Give the business reason.

**Expected Answer:** Logged-out (anonymous) viewers generate watch events too. The project description says "signed-in or not" generate watch events. When a viewer isn't signed in, there's no user to reference, so `user_id` is NULL.

---

## Q6 🟢 Easy
**Phase:** 3 · **Tags:** `benchmarking`, `methodology`

**Q:** Why must WRITE queries (W01–W05) be measured inside `BEGIN; … ROLLBACK;`?

**Expected Answer:** To prevent modifying the data. If INSERTs/UPDATEs actually commit, subsequent runs would fail (duplicate keys) or measure on altered data. ROLLBACK undoes the change so each run starts from the same state, giving comparable timings.

---

## Q7 🟢 Easy
**Phase:** 3 · **Tags:** `EXPLAIN`, `scan-types`

**Q:** What does `Seq Scan` mean in an EXPLAIN plan, and when is it expected?

**Expected Answer:** Sequential Scan reads every row in the table from disk. It's expected when no suitable index exists, or when the query must examine a large fraction of the table (e.g., R12 groups *all* watch events by device). For small tables it can also be faster than an index scan.

---

## Q8 🟡 Medium
**Phase:** 2 · **Tags:** `normalization`, `redundancy`, `3NF`

**Q:** `videos.view_count` and `channels.subscriber_count` are derived/redundant columns. Explain why, and argue whether keeping them is justified.

**Expected Answer:** `view_count` can be computed as `COUNT(*) FROM watch_events WHERE video_id = ?`, and `subscriber_count` as `COUNT(*) FROM subscriptions WHERE channel_id = ?`. They violate 3NF because they're transitively dependent on computed aggregates. However, keeping them is **justified** for performance — the watch page (R02) and channel page (R01) are real-time/viewer-facing and can't afford to aggregate millions of rows on every page load. W03 and W04 exist specifically to maintain these counters.

---

## Q9 🟡 Medium
**Phase:** 2 · **Tags:** `foreign-keys`, `ON-DELETE`

**Q:** What ON DELETE action would you choose for `watch_events.video_id → videos.video_id`, and why?

**Expected Answer:** `CASCADE` or `RESTRICT`, with reasoning:
- **CASCADE**: Deleting a video removes all its watch events. Makes sense if YouTube purges a video completely.
- **RESTRICT**: Prevents deleting a video that has watch events, forcing explicit cleanup first. Safer for auditing.
Both are defensible. `SET NULL` is bad here because watch events without a video are meaningless. The key is to justify the choice with a business reason.

---

## Q10 🟡 Medium
**Phase:** 2 · **Tags:** `constraints`, `CHECK`, `business-rules`

**Q:** A CHECK says "a reply can't be pinned." How would you express this, and what columns does it use?

**Expected Answer:**
```sql
CHECK (parent_comment_id IS NULL OR is_pinned = FALSE)
```
If `parent_comment_id` is not null, the comment is a reply. Only top-level comments (where `parent_comment_id IS NULL`) may be pinned. This enforces the YouTube business rule.

---

## Q11 🟡 Medium
**Phase:** 2 · **Tags:** `constraints`, `NULL`, `business-rules`

**Q:** Why is `published_at` nullable in `videos`, and what constraint ties it to `visibility`?

**Expected Answer:** `PRIVATE` videos are never published, so `published_at` is NULL for them. `SCHEDULED` videos have a future `published_at` (the go-live time). A useful CHECK:
```sql
CHECK (
  (visibility = 'PRIVATE'  AND published_at IS NULL) OR
  (visibility != 'PRIVATE' AND published_at IS NOT NULL)
)
```
This ensures private videos don't have a publish timestamp and published/scheduled videos do.

---

## Q12 🟡 Medium
**Phase:** 3 · **Tags:** `query-analysis`, `scan-types`

**Q:** R04 uses `ILIKE '%tutorial%'`. Why can't a standard B-tree index help here, and what index type would you later use?

**Expected Answer:** B-tree indexes support prefix matching (`LIKE 'tutorial%'`) but not infix patterns with a leading wildcard (`%tutorial%`). The leading `%` means the search term can appear anywhere in the string, so the B-tree can't narrow the search. In Phase 4, a **GIN trigram index** (`pg_trgm` extension) would handle this: `CREATE INDEX ... USING gin (title gin_trgm_ops)`.

---

## Q13 🟡 Medium
**Phase:** 4 · **Tags:** `indexing`, `composite-index`, `column-order`

**Q:** For R05 (user's watch history), you'd create an index on `watch_events`. What columns, in what order, and why that order?

**Expected Answer:** `CREATE INDEX ON watch_events (user_id, started_at DESC)`. Order matters:
1. `user_id` first — equality filter (`WHERE user_id = 1042`) narrows to that user's rows.
2. `started_at DESC` second — the remaining rows are already sorted for `ORDER BY started_at DESC LIMIT 50`, so Postgres reads the first 50 directly. No sort step needed. Reversing the order would require scanning all timestamps to filter by user.

---

## Q14 🟡 Medium
**Phase:** 4 · **Tags:** `partial-index`, `indexing`

**Q:** Explain what a partial index is and give one query from the workload where it's beneficial.

**Expected Answer:** A partial index indexes only rows matching a `WHERE` condition, making it smaller and faster. Example for R06:
```sql
CREATE INDEX ON videos (published_at) WHERE visibility = 'SCHEDULED';
```
Only a tiny fraction of videos are `SCHEDULED`. The index is small and directly serves R06's filter. Another example: `WHERE is_short = TRUE` for R10, or `WHERE is_made_for_kids = TRUE` for R18.

---

## Q15 🟡 Medium
**Phase:** 4 · **Tags:** `covering-index`, `Index-Only-Scan`

**Q:** What is a covering index (`INCLUDE`), and how does it produce an Index Only Scan?

**Expected Answer:** A covering index stores extra columns in the index leaf pages via `INCLUDE(...)`. When all columns the query needs are in the index (key + included), Postgres reads *only the index*, never visiting the heap table — producing an **Index Only Scan**. Example for R24:
```sql
CREATE INDEX ON subscriptions (channel_id, subscribed_at) INCLUDE (user_id);
```
This avoids a table lookup for each matching row, which is much faster.

---

## Q16 🟡 Medium
**Phase:** 4 · **Tags:** `NULL-bug`, `query-correctness`, `R13`

**Q:** The project says "at least one of the 30 queries returns a wrong answer because of a subtle SQL rule about NULLs." Which query, and what's the bug?

**Expected Answer:** **R13**. It uses `NOT IN (subquery)`. The subquery `SELECT w.user_id FROM watch_events ...` returns NULLs (logged-out viewers). In SQL, `x NOT IN (..., NULL, ...)` is **UNKNOWN** (never TRUE), so the entire result is **empty** — explaining problem P5. The fix is to replace `NOT IN` with `NOT EXISTS`:
```sql
WHERE NOT EXISTS (
  SELECT 1 FROM watch_events w JOIN videos v ON ...
  WHERE v.channel_id = 217 AND w.user_id = s.user_id
)
```

---

## Q17 🟡 Medium
**Phase:** 4 · **Tags:** `query-rewrite`, `sargable`

**Q:** R07 uses `DATE(w.started_at) BETWEEN ...`. Why is this non-sargable, and how would you rewrite it?

**Expected Answer:** Wrapping a column in a function (`DATE(...)`) prevents the optimizer from using a B-tree index on `started_at`. It must evaluate the function for every row → Seq Scan. Sargable rewrite:
```sql
WHERE w.started_at >= '2026-06-01' AND w.started_at < '2026-07-01'
```
Now a B-tree index on `started_at` can directly seek to the range. Alternatively, create an **expression index** on `DATE(started_at)`, but the range rewrite is preferred.

---

## Q18 🟡 Medium
**Phase:** 4 · **Tags:** `query-rewrite`, `pagination`, `R23`

**Q:** R23 uses `OFFSET 30000 LIMIT 25`. Why is this inefficient, and what's the alternative?

**Expected Answer:** `OFFSET N` forces Postgres to scan and discard N rows before returning the next 25. At offset 30000, it reads 30025 rows and throws away 30000 — extremely wasteful and gets worse for later pages. **Keyset pagination** is the fix: instead of OFFSET, filter using the last-seen value:
```sql
WHERE created_at < :last_seen_created_at
ORDER BY created_at DESC LIMIT 25
```
With an index on `(created_at DESC)`, this is O(1) regardless of page number.

---

## Q19 🟡 Medium
**Phase:** 4 · **Tags:** `write-overhead`, `index-tradeoff`

**Q:** After adding indexes for read optimization, some WRITE queries may get slower. Why, and which write queries are most at risk?

**Expected Answer:** Every INSERT/UPDATE must also update all relevant indexes. More indexes → more writes per mutation. Most at risk:
- **W02** (insert watch event): `watch_events` likely has the most indexes (user_id, video_id, started_at, etc.)
- **W04** (update all channels' subscriber_count): touches every row in `channels`; each update must modify index entries
- **W01** (insert video): multiple indexes on `videos`

The report should quantify this overhead and argue whether the read improvement justifies it.

---

## Q20 🔴 Hard
**Phase:** 2 · **Tags:** `ER-diagram`, `relationships`, `cardinality`

**Q:** Describe the relationship between `users`, `channels`, `videos`, and `watch_events`. Include cardinalities and explain why `watch_events` connects to `users` differently than `subscriptions` does.

**Expected Answer:**
- `users` 1:N `channels` (one user can own multiple channels, via `owner_user_id`)
- `channels` 1:N `videos` (one channel has many videos)
- `users` 1:N `watch_events` (one user generates many watch events) — but `user_id` is **nullable** (anonymous views)
- `users` M:N `channels` via `subscriptions` junction table — `user_id` is **NOT NULL** here because only signed-in users can subscribe

The key difference: `watch_events.user_id` is nullable (anonymous viewing allowed), while `subscriptions.user_id` is NOT NULL (subscribing requires a signed-in user). This distinction is what causes the R13 NULL bug.

---

## Q21 🔴 Hard
**Phase:** 4 · **Tags:** `BRIN-index`, `indexing`, `large-tables`

**Q:** What is a BRIN index, when is it appropriate, and which table in this project is the best candidate?

**Expected Answer:** BRIN (Block Range INdex) stores min/max values per block range rather than per row. It's tiny but only effective when data is **physically ordered** by the indexed column. `watch_events` is the best candidate — rows are inserted chronologically, so `started_at` values correlate strongly with physical order. A BRIN on `watch_events(started_at)` is much smaller than a B-tree and helps time-range queries (R07, R10, R11, R20). On the large dataset (4M rows), the size advantage is dramatic. However, BRIN won't help for point lookups or if the table is heavily updated out-of-order.

---

## Q22 🔴 Hard
**Phase:** 4 · **Tags:** `materialized-view`, `summary-table`, `R19`

**Q:** R19 has two correlated subqueries per channel row. How would you optimize it, and when would a materialized view be appropriate?

**Expected Answer:** R19 runs a subquery for each ELIGIBLE channel — O(channels × videos) or worse. Options:
1. **Rewrite with JOINs and GROUP BY** to process all channels in one pass.
2. **Materialized view** pre-computing per-channel stats:
```sql
CREATE MATERIALIZED VIEW channel_stats AS
SELECT v.channel_id, MAX(v.uploaded_at) AS last_upload, COUNT(w.*) AS total_views
FROM videos v LEFT JOIN watch_events w ON w.video_id = v.video_id
GROUP BY v.channel_id;
```
Refresh it periodically (`REFRESH MATERIALIZED VIEW CONCURRENTLY`). Appropriate because this is a **partner report** (analytics, not real-time) — stale-by-minutes data is acceptable. The report explicitly says materialized views are allowed for heavy reports that don't need live data.

---

## Q23 🔴 Hard
**Phase:** 5 · **Tags:** `scalability`, `partitioning`, `DDL`

**Q:** Explain how you'd partition `watch_events` for the 100× scenario. Write the DDL and explain the primary key constraint.

**Expected Answer:**
```sql
CREATE TABLE watch_events (
    watch_id    BIGINT NOT NULL,
    user_id     INTEGER,
    video_id    CHAR(11) NOT NULL,
    started_at  TIMESTAMPTZ NOT NULL,
    watch_seconds INTEGER NOT NULL,
    device      VARCHAR(10) NOT NULL,
    traffic_source VARCHAR(20) NOT NULL,
    PRIMARY KEY (watch_id, started_at)  -- partition key MUST be in PK
) PARTITION BY RANGE (started_at);

CREATE TABLE watch_events_2025_q1 PARTITION OF watch_events
    FOR VALUES FROM ('2025-01-01') TO ('2025-04-01');
-- ... more quarterly partitions
```
**Key rule:** In PostgreSQL, the partition key must be part of the primary key (or the table can't have a PK). This means `watch_id` alone can't be the PK — you need `(watch_id, started_at)`. **Partition pruning:** queries with `WHERE started_at >= ... AND started_at < ...` only scan relevant partitions, verified via `EXPLAIN` showing `Prune: ...`.

---

## Q24 🔴 Hard
**Phase:** 5 · **Tags:** `scalability`, `sharding`, `tradeoffs`

**Q:** If you had to shard `watch_events` across multiple servers, would you shard by user_id, video_id, channel_id, or time? Discuss the tradeoffs with respect to queries R05, R07, and R09.

**Expected Answer:**

| Shard key | R05 (user history) | R07 (channel daily views) | R09 (channel avg watch %) |
|-----------|-------------------|--------------------------|--------------------------|
| `user_id` | ✅ Single-shard | ❌ Cross-shard (need all shards for a channel's videos) | ❌ Cross-shard |
| `video_id` | ❌ Cross-shard | ⚠️ Needs gathering from all shards with that channel's videos | ⚠️ Same issue |
| `channel_id` | ❌ Cross-shard | ✅ Single-shard | ✅ Single-shard |
| `time` | ❌ Cross-shard for history | ✅ Single-shard (June 2026 → one partition) | ❌ Cross-shard |

No single key is perfect. **Time-based** sharding is common in practice (matches partitioning, easy archival), but Studio queries (per-channel) become cross-shard. A **hybrid approach** (time-partition within shards + route Studio queries to replicas with per-channel materialized views) is the best answer.

---

## Q25 🔴 Hard
**Phase:** 5 · **Tags:** `scalability`, `growth-analysis`

**Q:** When data grows 20×, some queries grow by O(log n) and others by O(n). Categorize R01, R12, and R23 and explain why.

**Expected Answer:**
- **R01** (channel page, 20 newest videos): With an index on `(channel_id, published_at DESC)`, this is an index seek + 20-row limit. Growth: **O(log n)** — stays near-constant because index depth grows logarithmically. Should stay under 5 ms.
- **R12** (logged-out share per device): Aggregates *all* watch events with `GROUP BY device`. No filter to reduce rows. Growth: **O(n) — grows ≈20×**. At 4M rows, still manageable (full table scan). At 100×, would need a BRIN or materialized view.
- **R23** (OFFSET 30000 pagination): `OFFSET` is O(offset) — it scans 30000 rows every time regardless of table size. At 20× data, the sort itself takes longer (more rows to sort), so growth is **≈20× or worse**. With keyset pagination, it becomes O(log n).

---

## Q26 🟢 Easy
**Phase:** 2 · **Tags:** `schema`, `load-order`

**Q:** Why must CSV files be loaded in a specific order? Give the first three tables you'd load.

**Expected Answer:** Foreign keys enforce referential integrity — a child row can't reference a parent that doesn't exist yet. Load **parents before children**:
1. `users` (no FK dependencies)
2. `channels` (FK → users)
3. `videos` (FK → channels)

Then `subscriptions`, `watch_events`, `video_reactions`, `comments`, `playlists`, `playlist_items`, `ad_impressions` — each after its parent tables are loaded.

---

## Q27 🟡 Medium
**Phase:** 3 · **Tags:** `EXPLAIN`, `BUFFERS`, `benchmarking`

**Q:** What does `EXPLAIN (ANALYZE, BUFFERS)` show that plain `EXPLAIN` does not? Why does the project require it?

**Expected Answer:** 
- Plain `EXPLAIN`: Shows the **estimated** plan (expected rows, cost) without running the query.
- `EXPLAIN ANALYZE`: Actually **runs** the query and shows **actual** rows, loops, and **Execution Time**.
- `BUFFERS`: Shows **shared buffer hits** vs **reads** (how much came from cache vs disk).

The project requires `ANALYZE` for real timing, and `BUFFERS` helps diagnose whether slowness is CPU-bound or I/O-bound. The raw plans serve as evidence that timings aren't fabricated.

---

## Q28 🔴 Hard
**Phase:** 4 · **Tags:** `VACUUM`, `ANALYZE`, `maintenance`

**Q:** What do VACUUM and ANALYZE do, and why does the project run `ANALYZE` after loading data? What happens to query plans if you skip it?

**Expected Answer:**
- **VACUUM**: Reclaims space from dead tuples (deleted/updated rows). Without it, table bloat grows and scans slow down. `VACUUM FULL` rewrites the table but locks it.
- **ANALYZE**: Collects column statistics (histograms, most-common values, null fractions) that the query planner uses for cost estimates.

After bulk-loading, the statistics are empty or stale. Without `ANALYZE`, the planner uses default assumptions (e.g., assumes 200 rows in a 200,000-row table), leading to **terrible plan choices** — it might choose nested-loop joins instead of hash joins, or skip indexes. Running `ANALYZE` after load ensures the planner has accurate statistics for Phase 3 baseline measurements.

---

## Q29 🟡 Medium
**Phase:** 5 · **Tags:** `scalability`, `read-replicas`, `caching`

**Q:** Which of the 30 queries would you send to a read replica, and which must hit the primary? Why?

**Expected Answer:**
**Read replica** (can tolerate slight lag):
- All report/analytics queries: R10, R11, R12, R16, R17, R18, R19, R20, R22, R23, R25
- Studio analytics: R07, R08, R09, R14, R24
- Search: R04

**Must hit primary**:
- All WRITE queries: W01–W05 (must write to primary)
- Real-time viewer-facing reads that need freshest data: R01, R02, R03, R05, R15, R21 (channel page, watch page, subscriptions feed, history, comments, playlist — users expect immediate consistency after their own writes)
- R06 (scheduled publishing — must see latest state to avoid missing a go-live)
- R13 (creator insights can go either way)

---

## Q30 🔴 Hard
**Phase:** 4 · **Tags:** `expression-index`, `query-rewrite`, `R20`

**Q:** R20 uses `started_at::date` in both WHERE and GROUP BY. Explain two different ways to optimize this, with their tradeoffs.

**Expected Answer:**

**Option 1 — Expression index:**
```sql
CREATE INDEX ON watch_events ((started_at::date));
```
Indexes the computed expression directly. Postgres can use it for `WHERE started_at::date BETWEEN ...` and `GROUP BY started_at::date`. Tradeoff: extra storage and write overhead for the expression index; the original query stays unchanged.

**Option 2 — Sargable rewrite:**
```sql
WHERE started_at >= '2026-06-01' AND started_at < '2026-07-01'
```
with `GROUP BY started_at::date` (or `DATE(started_at)`). This lets a plain B-tree or BRIN index on `started_at` handle the WHERE filter efficiently. The GROUP BY still needs the cast, but after the range filter narrows to ~1/18th of the data, it's cheap. Tradeoff: requires query rewrite (must put in `06_optimized_queries.sql`) and prove same results.

**Best approach:** Combine both — sargable WHERE + expression index for GROUP BY, or just the sargable rewrite if a BRIN index on `started_at` already exists.

---

## Summary

| Difficulty | Count | Questions |
|:----------:|:-----:|-----------|
| 🟢 Easy | 8 | Q1, Q2, Q3, Q4, Q5, Q6, Q7, Q26 |
| 🟡 Medium | 13 | Q8, Q9, Q10, Q11, Q12, Q13, Q14, Q15, Q16, Q17, Q18, Q19, Q27, Q29 |
| 🔴 Hard | 9 | Q20, Q21, Q22, Q23, Q24, Q25, Q28, Q30 |

| Phase | Questions |
|:-----:|-----------|
| 1 | Q1 |
| 2 | Q2, Q3, Q4, Q5, Q8, Q9, Q10, Q11, Q20, Q26 |
| 3 | Q6, Q7, Q12, Q27 |
| 4 | Q13, Q14, Q15, Q16, Q17, Q18, Q19, Q22, Q28, Q30 |
| 5 | Q21, Q23, Q24, Q25, Q29 |
