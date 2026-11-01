# Sleep-time consolidation of a memory graph without an LLM

**First public disclosure:** 2026-01-28, consolidation pass, hippograph [`a091221`](https://github.com/artemMprokhorov/hippograph/commit/a091221) (the recency half-life used by the same system is public from 2026-01-26, [`a96a1e5`](https://github.com/artemMprokhorov/hippograph/commit/a96a1e5))
**Status of disclosure:** working code (MIT), plus design text in README / ROADMAP files

All times are UTC. "Pushed" times come from GitHub's own push log (`/activity`) or from GH Archive; see [README.md](README.md#how-to-verify-dates).

## What it is

A background "sleep" job that maintains a personal memory graph (notes as nodes, typed weighted edges) using only
graph math and embeddings: no language model is called. It adds links instead of summarising or deleting notes,
recomputes graph structure (PageRank, communities, topics), lets old unused edges fade, and protects the database with
a snapshot before every run. Work is split into a frequent cheap pass ("light sleep") and a rarer heavy pass
("deep sleep"), triggered by time or by the number of new notes.

## How it worked as disclosed

**1. Consolidation by linking, not compression**: [`a091221`](https://github.com/artemMprokhorov/hippograph/commit/a091221), file
[`src/memory_consolidation.py`](https://github.com/artemMprokhorov/hippograph/blob/a091221/src/memory_consolidation.py);
design note [`docs/MEMORY_CONSOLIDATION.md`](https://github.com/artemMprokhorov/hippograph/blob/d853e2d/docs/MEMORY_CONSOLIDATION.md) ([`d853e2d`](https://github.com/artemMprokhorov/hippograph/commit/d853e2d)).
- *Thematic clusters:* greedy pass over all note embeddings; notes with cosine ≥ 0.75 to a seed join its cluster;
  clusters of ≥ 3 notes get all-to-all edges of type `consolidation`, weight 0.9.
- *Temporal chains:* notes grouped by category and ordered by time; consecutive notes with a gap ≤ 7 days form a chain;
  chains of ≥ 3 get sequential `temporal_chain` edges, weight 0.95.
- Code comment: "CRITICAL: This is NOT compression. Original notes preserved intact."

**2. Recency half-life (used in retrieval, softened for protected memories later)**: [`a96a1e5`](https://github.com/artemMprokhorov/hippograph/commit/a96a1e5), [`ac9da23`](https://github.com/artemMprokhorov/hippograph/commit/ac9da23),
file [`src/graph_engine.py`](https://github.com/artemMprokhorov/hippograph/blob/ac9da23/src/graph_engine.py).
`recency = max(0.1, 0.5 ** (age_days / HALF_LIFE_DAYS))`, `HALF_LIFE_DAYS = 30` (env-configurable); from `ac9da23` the
age is measured from `last_accessed` (fallback: creation time), and the floor 0.1 keeps old notes reachable. The factor
multiplies a note's activation score.

**3. Maintenance daemon ("sleep-time compute")**: [`ca0392a`](https://github.com/artemMprokhorov/hippograph/commit/ca0392a), file
[`src/sleep_compute.py`](https://github.com/artemMprokhorov/hippograph/blob/ca0392a/src/sleep_compute.py); MCP tool `sleep_compute(dry_run)` in
[`c296a1c`](https://github.com/artemMprokhorov/hippograph/commit/c296a1c). Modes `--once`, `--interval N` (hours, default 6), `--dry-run`. One cycle:
1. consolidation (step 1 above);
2. PageRank + community detection recomputed over the whole graph (NetworkX);
3. orphan detection: notes with ≤ 1 edge are reported (not deleted);
4. stale edge decay: edges created more than `STALE_EDGE_DAYS` = 90 days ago with weight > 0.3 get `weight *= 0.95`;
5. near-duplicate scan: cosine ≥ 0.95 within a sliding window of 50 notes (reported).

The design goal is stated in [`ROADMAP_PERSONAL.md`](https://github.com/artemMprokhorov/hippograph/blob/ea6f160/ROADMAP_PERSONAL.md) ([`ea6f160`](https://github.com/artemMprokhorov/hippograph/commit/ea6f160)):
"Pure graph math — NetworkX, zero LLM cost."

**4. Light sleep / deep sleep and triggers**: hippograph-pro [`8533620`](https://github.com/artemMprokhorov/hippograph-pro/commit/8533620),
[`ROADMAP_PRO.md` §8 "Sleep-Wake Cycle Architecture"](https://github.com/artemMprokhorov/hippograph-pro/blob/8533620/ROADMAP_PRO.md):
- *Light sleep* (fast, about every 50 new notes): stale edge decay, duplicate scan, PageRank recalculation;
- *Deep sleep* (heavy, daily): entity re-extraction on old notes, typed-relation building, cluster consolidation via
  community detection, extractive cluster summaries, contradiction detection.

Code the same day: deep-sleep relation extraction step ([`777802c`](https://github.com/artemMprokhorov/hippograph-pro/commit/777802c), "Step 2.5") and a
scheduler ([`b7983dd`](https://github.com/artemMprokhorov/hippograph-pro/commit/b7983dd), [`src/sleep_scheduler.py`](https://github.com/artemMprokhorov/hippograph-pro/blob/b7983dd/src/sleep_scheduler.py)) with two
triggers: every `SLEEP_INTERVAL_HOURS` = 6 and after `SLEEP_NOTE_THRESHOLD` = 50 new notes (a counter bumped on every
add); runs in a background thread; `GET /api/sleep/status`, `POST /api/sleep/trigger`. The README
([`725e01e`](https://github.com/artemMprokhorov/hippograph-pro/blob/725e01e/README.md#sleep-time-compute)) then described: "Light sleep (every 50 notes): stale edge decay,
PageRank recalculation, duplicate scan, anchor importance boost / Deep sleep (daily): … relation extraction, conflict
detection, snapshot + rollback". *Note:* in the disclosed code both triggers call the same full pipeline
(`run_all`); the light/deep split is disclosed as a description, not as two separate code paths.

**5. Snapshot before sleep, manual rollback, no overwrite of existing edges**: [`cb7df86`](https://github.com/artemMprokhorov/hippograph-pro/commit/cb7df86)
(pushed as [`c51ff90`](https://github.com/artemMprokhorov/hippograph-pro/commit/c51ff90)), file
[`src/sleep_compute.py`](https://github.com/artemMprokhorov/hippograph-pro/blob/cb7df86/src/sleep_compute.py):
- before every live (non-dry) run the database file is copied to `memory_snapshot_<timestamp>.db`; the newest
  `MAX_SNAPSHOTS` = 7 are kept;
- `restore_snapshot()` exists but is never called automatically; on errors in critical steps the run prints the
  restore command and returns `rollback_available` ("never auto-rollback, human decides");
- when sleep finds a relation for a pair that already has an edge of another type, it does not overwrite it but logs
  the candidate into an `edge_history` table (`conflict_type = 'type_conflict'`).

**6. Rebuilding topics during sleep**: hippograph-pro:
- [`06b2a7b`](https://github.com/artemMprokhorov/hippograph-pro/commit/06b2a7b) (pushed as [`f87c023`](https://github.com/artemMprokhorov/hippograph-pro/commit/f87c023)), [`src/extractive_summary.py`](https://github.com/artemMprokhorov/hippograph-pro/blob/06b2a7b/src/extractive_summary.py):
  for each thematic cluster, the representative note is chosen by TF-IDF term coverage weighted by intra-cluster
  PageRank and stored in `cluster_summaries` (no text is generated);
- [`fcd9175`](https://github.com/artemMprokhorov/hippograph-pro/commit/fcd9175), [`src/sleep_compute.py`](https://github.com/artemMprokhorov/hippograph-pro/blob/fcd9175/src/sleep_compute.py) (`step_topic_linking_tfidf`,
  `step_topic_linking_kmeans`): each cluster gets a topic node (category `abstract-topic`) labelled with its top
  three terms, and member notes get `BELONGS_TO` edges; a second variant clusters all embeddings with k-means,
  `k = max(10, n // 30)` (capped at `n // 3`), fixed seed 42; topic nodes are excluded from search results. Old
  `BELONGS_TO` edges are reset on every run;
- [`a22cc4b`](https://github.com/artemMprokhorov/hippograph-pro/commit/a22cc4b) (pushed as [`86c325f`](https://github.com/artemMprokhorov/hippograph-pro/commit/86c325f)): the rebuild made idempotent: old topic nodes and
  their edges are deleted before re-clustering, and synthetic node types are kept out of the k-means pool.

## Disclosure timeline

| Date (UTC) | Repo | Commit | What was added | Evidence |
|---|---|---|---|---|
| 2026-01-26 | hippograph | [`a96a1e5`](https://github.com/artemMprokhorov/hippograph/commit/a96a1e5) | recency factor, half-life 30 days | push 01:42:51Z ([activity](https://github.com/artemMprokhorov/hippograph/activity)); GH Archive [2026-01-26-1](https://data.gharchive.org/2026-01-26-1.json.gz) PushEvent, `public: true` |
| 2026-01-26 | hippograph | [`ac9da23`](https://github.com/artemMprokhorov/hippograph/commit/ac9da23) | half-life from `last_accessed`, floor 0.1 | push 02:02:05Z; GH Archive [2026-01-26-2](https://data.gharchive.org/2026-01-26-2.json.gz), `public: true` |
| 2026-01-28 | hippograph | [`a091221`](https://github.com/artemMprokhorov/hippograph/commit/a091221), [`d853e2d`](https://github.com/artemMprokhorov/hippograph/commit/d853e2d) | consolidation: thematic clusters + temporal chains, design note | push 19:22:43Z / 19:23:38Z ([activity](https://github.com/artemMprokhorov/hippograph/activity)); GH Archive has no events for that hour, the next captured push [`bfc9310`](https://github.com/artemMprokhorov/hippograph/commit/bfc9310) (2026-02-04T18:11:04Z, [file](https://data.gharchive.org/2026-02-04-18.json.gz)) contains it |
| 2026-02-12 | hippograph | [`2b4e5b8`](https://github.com/artemMprokhorov/hippograph/commit/2b4e5b8) | one-line idea: "Graph-wide rollback (point-in-time snapshots)" (enterprise roadmap) | push 16:24:58Z; GH Archive [2026-02-12-16](https://data.gharchive.org/2026-02-12-16.json.gz) |
| 2026-02-18 | hippograph | [`ea6f160`](https://github.com/artemMprokhorov/hippograph/commit/ea6f160) | "sleep-time compute" without LLM, design | on remote before 23:16:50Z (activity) |
| 2026-02-18 | hippograph | [`ca0392a`](https://github.com/artemMprokhorov/hippograph/commit/ca0392a), [`c296a1c`](https://github.com/artemMprokhorov/hippograph/commit/c296a1c) | 5-step daemon, edge decay ×0.95 after 90 days, MCP tool | push 23:33:00Z; GH Archive [2026-02-18-23](https://data.gharchive.org/2026-02-18-23.json.gz), `public: true`; Wayback [2026-03-18](https://web.archive.org/web/20260318170705/https://github.com/artemMprokhorov/hippograph) (README lists sleep-time compute) |
| 2026-02-26 | hippograph-pro | [`8533620`](https://github.com/artemMprokhorov/hippograph-pro/commit/8533620) | light / deep sleep design | push 00:10:23Z; GH Archive [2026-02-26-0](https://data.gharchive.org/2026-02-26-0.json.gz), `public: true` |
| 2026-02-26 | hippograph-pro | [`777802c`](https://github.com/artemMprokhorov/hippograph-pro/commit/777802c), [`b7983dd`](https://github.com/artemMprokhorov/hippograph-pro/commit/b7983dd) | deep-sleep relation step; scheduler (6 h / 50 notes) | push 12:18:12Z / 12:27:11Z; GH Archive [2026-02-26-13](https://data.gharchive.org/2026-02-26-13.json.gz): next push has `before = b7983dd` |
| 2026-02-27 | hippograph-pro | [`dce87ec`](https://github.com/artemMprokhorov/hippograph-pro/commit/dce87ec) (pushed as [`fe1a7a8`](https://github.com/artemMprokhorov/hippograph-pro/commit/fe1a7a8)) | roadmap: sleep conflict resolution + rollback | push 16:24:20Z; GH Archive [2026-02-27-16](https://data.gharchive.org/2026-02-27-16.json.gz), `public: true` |
| 2026-02-27 | hippograph-pro | [`cb7df86`](https://github.com/artemMprokhorov/hippograph-pro/commit/cb7df86) (pushed as [`c51ff90`](https://github.com/artemMprokhorov/hippograph-pro/commit/c51ff90)) | snapshot before sleep, keep 7, manual restore, `edge_history` | push 17:06:38Z; GH Archive [2026-02-27-17](https://data.gharchive.org/2026-02-27-17.json.gz): push 17:08:15Z with `before = c51ff90`; fork copy [iflow-mcp](https://github.com/iflow-mcp/artemmprokhorov-hippograph-pro/commit/cb7df86) |
| 2026-02-28 | hippograph-pro | [`725e01e`](https://github.com/artemMprokhorov/hippograph-pro/commit/725e01e) (pushed as [`c750703`](https://github.com/artemMprokhorov/hippograph-pro/commit/c750703)) | README: light / deep sleep, snapshot + rollback | push 21:21:49Z; Wayback [2026-03-04](https://web.archive.org/web/20260304011737/https://github.com/artemMprokhorov/hippograph-pro) shows the text |
| 2026-03-03 | hippograph-pro | [`06b2a7b`](https://github.com/artemMprokhorov/hippograph-pro/commit/06b2a7b) (pushed as [`f87c023`](https://github.com/artemMprokhorov/hippograph-pro/commit/f87c023)) | extractive cluster representatives (TF-IDF + PageRank) | push 23:02:56Z; ancestor of the Wayback 2026-03-04 head |
| 2026-03-22 | hippograph-pro | [`fbd9c79`](https://github.com/artemMprokhorov/hippograph-pro/commit/fbd9c79) | online consolidation at write time (variant) | push 21:29:21Z; GH Archive [2026-03-22-21](https://data.gharchive.org/2026-03-22-21.json.gz), `public: true` |
| 2026-03-24 | hippograph-pro | [`fcd9175`](https://github.com/artemMprokhorov/hippograph-pro/commit/fcd9175) | topic nodes + `BELONGS_TO`, TF and k-means variants | push 02:45:17Z; GH Archive [2026-03-24-2](https://data.gharchive.org/2026-03-24-2.json.gz): next push has `before = fcd9175`; forks [vaavaa](https://github.com/vaavaa/hippograph-pro/commit/fcd9175), [iflow-mcp](https://github.com/iflow-mcp/artemmprokhorov-hippograph-pro/commit/fcd9175) |
| 2026-04-08 | hippograph-pro | [`a22cc4b`](https://github.com/artemMprokhorov/hippograph-pro/commit/a22cc4b) (pushed as [`86c325f`](https://github.com/artemMprokhorov/hippograph-pro/commit/86c325f)) | idempotent topic rebuild | push 22:22:09Z (activity only) |

## Variants disclosed

- **Online consolidation** ([`fbd9c79`](https://github.com/artemMprokhorov/hippograph-pro/commit/fbd9c79)): at write time the new note is linked by `consolidation` edges to
  up to 15 ANN neighbours with cosine ≥ 0.75, without waiting for sleep (`ONLINE_CONSOLIDATION`, default on).
- **Protected memories during decay**: from 2026-02-27 edges touching notes of protected categories are skipped by
  stale-edge decay, and the recency half-life is softened per category: see
  [07-anchors-and-importance.md](07-anchors-and-importance.md).
- **Deep-sleep steps added later** (same pipeline): typed relations from a zero-shot relation extractor
  ([`777802c`](https://github.com/artemMprokhorov/hippograph-pro/commit/777802c)), contradiction candidates ([`c547d78`](https://github.com/artemMprokhorov/hippograph-pro/commit/c547d78), pushed as [`2d691dc`](https://github.com/artemMprokhorov/hippograph-pro/commit/2d691dc), 2026-03-03), keyword anchors
  ([`82f7b50`](https://github.com/artemMprokhorov/hippograph-pro/commit/82f7b50), pushed as [`fe3af56`](https://github.com/artemMprokhorov/hippograph-pro/commit/fe3af56), 2026-04-07, see doc 07), intention upkeep
  ([`b05871f`](https://github.com/artemMprokhorov/hippograph-pro/commit/b05871f), pushed as [`3a2938c`](https://github.com/artemMprokhorov/hippograph-pro/commit/3a2938c), 2026-04-08, see doc 09).
- **Dry run everywhere**: every step reports what it would change without writing (`--dry-run`, `dry_run=true`).

## Prior work and our contribution

Builds on: complementary learning systems (McClelland, McNaughton & O'Reilly 1995); sleep-time compute for agents
(Letta, arXiv 2504.13171, 2025); LLM-free "dream-inspired" consolidation with decay, clustering, associations and
archiving (mcp-memory-service, 2025-07-28); recency half-life (Generative Agents, 2023).
Our contribution (first disclosed 2026-01-28, hippograph a091221; daemon 2026-02-18, ca0392a; snapshots 2026-02-27,
hippograph-pro cb7df86): a strictly non-destructive offline consolidation pass for an agent's memory graph, it only
adds or re-weights edges (cluster and temporal-chain links, gentle decay of long-unused edges, topic nodes labelled
by TF-IDF and rebuilt each run) and never summarises, compresses or deletes the user's notes; the database is
snapshotted before every run, and an edge whose type would change is logged to an edge history instead of being
overwritten.

## Citation

Prokhorov, Artem. *HippoGraph* (formerly *neural-memory-graph*), `src/memory_consolidation.py` and
`src/sleep_compute.py`. GitHub, https://github.com/artemMprokhorov/hippograph, commit
`a091221ceb44433abbedbb7846c5ca9ac33e1150` (2026-01-28) and `ca0392a95c3cffc1338ab6290075e01e7f9289bd` (2026-02-18).
MIT License. Archived: Software Heritage, https://archive.softwareheritage.org/browse/origin/?origin_url=https://github.com/artemMprokhorov/hippograph
(SWHID `swh:1:rev:a091221ceb44433abbedbb7846c5ca9ac33e1150`).

Prokhorov, Artem. *HippoGraph Pro*, `ROADMAP_PRO.md` §8 and `src/sleep_scheduler.py`. GitHub,
https://github.com/artemMprokhorov/hippograph-pro, commits `8533620eccbb7f9e8721d0f0a986c8fcdf1c8cbd` and
`b7983ddab445221284dfd154c450fa1deef70a11` (2026-02-26); snapshot/rollback `cb7df86d1a6c462b0d9cca719373b07a64a59425`
(originally pushed as `c51ff901c92e6ec510956113440eaad275e51232`, 2026-02-27). MIT License. Software Heritage:
https://archive.softwareheritage.org/browse/origin/?origin_url=https://github.com/artemMprokhorov/hippograph-pro
