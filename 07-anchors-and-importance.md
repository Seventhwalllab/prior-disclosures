# Importance levels, non-decaying anchor categories, anchor policies, keyword anchors and entity merging

**First public disclosure:** 2026-01-26, importance `critical / normal / low`, hippograph [`ac9da23`](https://github.com/artemMprokhorov/hippograph/commit/ac9da23); anchor categories 2026-02-27; anchor policies 2026-03-05; keyword anchors 2026-04-07
**Status of disclosure:** working code (MIT) for all parts; anchor policies by tag / note id / pattern only as a roadmap idea

All times are UTC; for rewritten hippograph-pro commits both the current hash and the hash as first pushed are given
(see [README.md](README.md#how-to-verify-dates)).

## What it is

Every note in the memory graph carries an importance level that scales its retrieval score. Some note categories
("anchors": identity, relationships, protocols, milestones, …) are exempt from, or strongly slowed in, time decay; the
sleep job keeps them at the highest importance and skips their edges during edge decay. The set of protected categories
is a hard-coded baseline, extended by user-defined policies stored in the database and by automatic discovery. Separately,
each long note can get a small "keyword anchor" node (its named entities and numeric facts) that is easy to hit by vector
search and redirects to the parent note. Entity nodes are de-duplicated by normalised name and can be merged on request.

## How it worked as disclosed

**1. Importance levels**: hippograph [`ac9da23`](https://github.com/artemMprokhorov/hippograph/commit/ac9da23), files
[`src/graph_engine.py`](https://github.com/artemMprokhorov/hippograph/blob/ac9da23/src/graph_engine.py) (`importance_factor`),
[`src/database.py`](https://github.com/artemMprokhorov/hippograph/blob/ac9da23/src/database.py), MCP tools `set_importance`, `add_note(importance=…)`.
- column `importance TEXT DEFAULT 'normal'`, values `critical | normal | low` (auto-migration for old databases);
- after spreading activation each note's score is multiplied by `recency_factor(...)` and by
  `importance_factor = base + min(0.5, 0.025 · access_count)` with base `critical 2.0 / normal 1.0 / low 0.5`;
- retuned in [`171232a`](https://github.com/artemMprokhorov/hippograph/commit/171232a) (2026-02-06) to `1.5 / 1.0 / 0.7` and access boost `min(0.2, 0.01 · access_count)`.

**2. Anchor categories exempt from decay**: hippograph-pro [`d115f80`](https://github.com/artemMprokhorov/hippograph-pro/commit/d115f80) (pushed as [`a30167a`](https://github.com/artemMprokhorov/hippograph-pro/commit/a30167a)),
files [`src/graph_engine.py`](https://github.com/artemMprokhorov/hippograph-pro/blob/d115f80/src/graph_engine.py), [`src/sleep_compute.py`](https://github.com/artemMprokhorov/hippograph-pro/blob/d115f80/src/sleep_compute.py).
- category `anchor`: recency factor is always 1.0 (no decay);
- per-category decay multipliers `m`: self-reflection, relational-context, gratitude 0.1; milestone 0.15; protocol,
  security, breakthrough 0.2; applied as `recency = max(0.1, 1 − (1 − base_decay) · m)`;
- sleep: edges with an endpoint in a protected category are excluded from stale-edge decay (×0.95);
- new sleep step "anchor importance boost": notes of protected categories with importance ≠ `critical` are set to
  `critical`.
- README feature table from 2026-02-28 ([`725e01e`](https://github.com/artemMprokhorov/hippograph-pro/blob/725e01e/README.md)): "Anchor Protection … Critical memories exempt from decay".

**3. User-defined anchor policies**: idea [`6121dd7`](https://github.com/artemMprokhorov/hippograph-pro/commit/6121dd7) (pushed as [`62c6ab0`](https://github.com/artemMprokhorov/hippograph-pro/commit/62c6ab0)), code
[`f7704a9`](https://github.com/artemMprokhorov/hippograph-pro/commit/f7704a9) (pushed as [`7057b7e`](https://github.com/artemMprokhorov/hippograph-pro/commit/7057b7e)), files
[`src/database.py`](https://github.com/artemMprokhorov/hippograph-pro/blob/f7704a9/src/database.py), [`src/mcp_sse_handler.py`](https://github.com/artemMprokhorov/hippograph-pro/blob/f7704a9/src/mcp_sse_handler.py), [`src/sleep_compute.py`](https://github.com/artemMprokhorov/hippograph-pro/blob/f7704a9/src/sleep_compute.py).
- table `anchor_policies(id, category UNIQUE, policy_type DEFAULT 'protect', description, created_at, created_by)`;
- MCP tools `list_anchor_policies`, `add_anchor_policy(category, description)`, `remove_anchor_policy(category)`;
  policy `protect` = exempt from stale-edge decay + kept at `critical`;
- the hard-coded baseline (`anchor, self-reflection, relational-context, gratitude, milestone, protocol, security,
  breakthrough`) cannot be removed; sleep uses the union of baseline and database policies.
- The roadmap item (2026-02-27) also proposed policy rules by tag, by explicit note ids and by content pattern, with
  decay / importance overrides; only the per-category rule was implemented.

**4. Automatic discovery of protected categories**: [`2d0d873`](https://github.com/artemMprokhorov/hippograph-pro/commit/2d0d873), [`b8e96f7`](https://github.com/artemMprokhorov/hippograph-pro/commit/b8e96f7).
Third layer on top of baseline and policies: a category is protected if it contains at least 3 (from `b8e96f7`: at
least 1) `critical` notes, or if its name contains one of `learned, identity, protocol, security, consciousness,
critical-lesson, self, anchor`.

**5. Keyword anchors (materialised keywords)**: [`d9d60a0`](https://github.com/artemMprokhorov/hippograph-pro/commit/d9d60a0) (pushed as [`8244f10`](https://github.com/artemMprokhorov/hippograph-pro/commit/8244f10)),
[`c7f4aa5`](https://github.com/artemMprokhorov/hippograph-pro/commit/c7f4aa5) (pushed as [`3f86353`](https://github.com/artemMprokhorov/hippograph-pro/commit/3f86353)), [`ad89bb1`](https://github.com/artemMprokhorov/hippograph-pro/commit/ad89bb1) (pushed as
[`f064e9f`](https://github.com/artemMprokhorov/hippograph-pro/commit/f064e9f)), [`82f7b50`](https://github.com/artemMprokhorov/hippograph-pro/commit/82f7b50) (pushed as [`fe3af56`](https://github.com/artemMprokhorov/hippograph-pro/commit/fe3af56)); files
[`EXPERIMENTS.md`](https://github.com/artemMprokhorov/hippograph-pro/blob/d9d60a0/EXPERIMENTS.md), [`src/graph_engine.py`](https://github.com/artemMprokhorov/hippograph-pro/blob/c7f4aa5/src/graph_engine.py),
[`src/sleep_compute.py`](https://github.com/artemMprokhorov/hippograph-pro/blob/82f7b50/src/sleep_compute.py) (`step_keyword_anchors`).
- keys for a note: named entities (multilingual spaCy model, length > 2) from the first 1000 characters, plus numeric
  facts matched by regex (e.g. `name: 42%`, `Recall@5 = 0.9`); fallback: up to 4 words longer than 6 letters; at most
  7 keys;
- anchor node text `KEY: k1 | k2 | …`, own embedding (normalised), category `keyword-anchor`, importance `low`, tags
  `anchor parent-<id> cat-<category>`; edge `PART_OF` anchor → parent; added to the ANN index;
- skipped for notes shorter than 80 characters and for synthetic categories (chunks, topics, atomic facts, anchors);
- retrieval ("small-to-big"): when an anchor is among the hits, its `PART_OF` parent is returned instead, scored
  `max(existing, 1.2 · anchor_activation)`;
- two placements were disclosed: inline at write time (`c7f4aa5`) and batch in sleep after consolidation (`82f7b50`,
  "step 1d": all old anchors removed and rebuilt). EXPERIMENTS.md reports batch anchors created after consolidation
  outperforming inline ones.

**6. Entity de-duplication and merging**: hippograph [`a96a1e5`](https://github.com/artemMprokhorov/hippograph/commit/a96a1e5) (`get_or_create_entity` looks up
`LOWER(name)`), hippograph-pro [`e8f1719`](https://github.com/artemMprokhorov/hippograph-pro/commit/e8f1719) (pushed as [`568c8ce`](https://github.com/artemMprokhorov/hippograph-pro/commit/568c8ce)), file
[`src/database.py`](https://github.com/artemMprokhorov/hippograph-pro/blob/e8f1719/src/database.py):
- on ingestion, entities are matched by `lower(name)` and `entity_type` and stored lower-case (different types stay
  different entities);
- `list_entity_candidates` (read-only) groups existing entities by `lower(name), entity_type` with count > 1;
- `merge_entities(keep_id, remove_id)` moves every note→entity link from `remove_id` to `keep_id` (skipping links
  that already exist), then deletes `remove_id`; notes themselves are never touched; the tool description asks for a
  snapshot first.

## Disclosure timeline

| Date (UTC) | Repo | Commit | What was added | Evidence |
|---|---|---|---|---|
| 2026-01-26 | hippograph | [`a96a1e5`](https://github.com/artemMprokhorov/hippograph/commit/a96a1e5) | entity lookup by lower-cased name | push 01:42:51Z ([activity](https://github.com/artemMprokhorov/hippograph/activity)); GH Archive [2026-01-26-1](https://data.gharchive.org/2026-01-26-1.json.gz), `public: true` |
| 2026-01-26 | hippograph | [`ac9da23`](https://github.com/artemMprokhorov/hippograph/commit/ac9da23) | importance critical/normal/low, `set_importance` | push 02:02:05Z; GH Archive [2026-01-26-2](https://data.gharchive.org/2026-01-26-2.json.gz), `public: true`; Wayback [2026-03-18](https://web.archive.org/web/20260318170705/https://github.com/artemMprokhorov/hippograph) README: "Importance scoring (critical/normal/low)" |
| 2026-02-06 | hippograph | [`171232a`](https://github.com/artemMprokhorov/hippograph/commit/171232a) | retuned multipliers | push 12:25:14Z |
| 2026-02-27 | hippograph-pro | [`c18def7`](https://github.com/artemMprokhorov/hippograph-pro/commit/c18def7) (pushed as [`65bd1e3`](https://github.com/artemMprokhorov/hippograph-pro/commit/65bd1e3)) | roadmap: anchor memory (no decay for `anchor`, 0.1× decay for identity/relational, sleep raises importance) | push 00:26:40Z |
| 2026-02-27 | hippograph-pro | [`d115f80`](https://github.com/artemMprokhorov/hippograph-pro/commit/d115f80) (pushed as [`a30167a`](https://github.com/artemMprokhorov/hippograph-pro/commit/a30167a)) | anchor categories, per-category decay, edge-decay exemption, importance boost | push 16:46:41Z; GH Archive [2026-02-27-16](https://data.gharchive.org/2026-02-27-16.json.gz): push 16:50:56Z with `before = a30167a`; fork [vaavaa](https://github.com/vaavaa/hippograph-pro/commit/d115f80) |
| 2026-02-27 | hippograph-pro | [`6121dd7`](https://github.com/artemMprokhorov/hippograph-pro/commit/6121dd7) (pushed as [`62c6ab0`](https://github.com/artemMprokhorov/hippograph-pro/commit/62c6ab0)) | roadmap: user-defined anchor policies | push 17:15:27Z; GH Archive [2026-02-27-21](https://data.gharchive.org/2026-02-27-21.json.gz): push with `before = 62c6ab0` |
| 2026-02-28 | hippograph-pro | [`725e01e`](https://github.com/artemMprokhorov/hippograph-pro/commit/725e01e) (pushed as [`c750703`](https://github.com/artemMprokhorov/hippograph-pro/commit/c750703)) | README: "Critical memories exempt from decay" | push 21:21:49Z; Wayback [2026-03-04](https://web.archive.org/web/20260304011737/https://github.com/artemMprokhorov/hippograph-pro) |
| 2026-03-05 | hippograph-pro | [`e8f1719`](https://github.com/artemMprokhorov/hippograph-pro/commit/e8f1719) (pushed as [`568c8ce`](https://github.com/artemMprokhorov/hippograph-pro/commit/568c8ce)) | entity resolution: normalisation, candidates, merge | push 18:45:13Z; GH Archive [2026-03-05-19](https://data.gharchive.org/2026-03-05-19.json.gz): push 19:23:41Z with `before = 568c8ce`; Wayback [2026-03-13](https://web.archive.org/web/20260313061019/https://github.com/artemMprokhorov/hippograph-pro) |
| 2026-03-05 | hippograph-pro | [`f7704a9`](https://github.com/artemMprokhorov/hippograph-pro/commit/f7704a9) (pushed as [`7057b7e`](https://github.com/artemMprokhorov/hippograph-pro/commit/7057b7e)) | `anchor_policies` table + 3 MCP tools | push 19:23:40Z; GH Archive [2026-03-05-19](https://data.gharchive.org/2026-03-05-19.json.gz), `public: true`; Wayback [2026-03-13](https://web.archive.org/web/20260313061019/https://github.com/artemMprokhorov/hippograph-pro) README: "User-Defined Anchor Policies" |
| 2026-03-21 | hippograph-pro | [`2d0d873`](https://github.com/artemMprokhorov/hippograph-pro/commit/2d0d873), [`b8e96f7`](https://github.com/artemMprokhorov/hippograph-pro/commit/b8e96f7) | auto-discovered protected categories | push 02:25:49Z / 02:33:30Z; GH Archive [2026-03-21-2](https://data.gharchive.org/2026-03-21-2.json.gz), `public: true`; forks [vaavaa](https://github.com/vaavaa/hippograph-pro/commit/b8e96f7), [iflow-mcp](https://github.com/iflow-mcp/artemmprokhorov-hippograph-pro/commit/b8e96f7) |
| 2026-04-05 | hippograph-pro | [`c327a5e`](https://github.com/artemMprokhorov/hippograph-pro/commit/c327a5e) (pushed as [`f58c011`](https://github.com/artemMprokhorov/hippograph-pro/commit/f58c011)) | name only: `KEYWORD_ANCHOR_ENABLED` in `.env.example` | push 19:58:28Z; fork [vaavaa](https://github.com/vaavaa/hippograph-pro/commit/f58c011) (fork pushed 2026-04-06) |
| 2026-04-07 | hippograph-pro | [`d9d60a0`](https://github.com/artemMprokhorov/hippograph-pro/commit/d9d60a0) (pushed as [`8244f10`](https://github.com/artemMprokhorov/hippograph-pro/commit/8244f10)) | keyword anchors described, experiment results | push 01:43:25Z; GH Archive [2026-04-07-1](https://data.gharchive.org/2026-04-07-1.json.gz), `public: true` |
| 2026-04-07 | hippograph-pro | [`c7f4aa5`](https://github.com/artemMprokhorov/hippograph-pro/commit/c7f4aa5) (pushed as [`3f86353`](https://github.com/artemMprokhorov/hippograph-pro/commit/3f86353)), [`ad89bb1`](https://github.com/artemMprokhorov/hippograph-pro/commit/ad89bb1) (pushed as [`f064e9f`](https://github.com/artemMprokhorov/hippograph-pro/commit/f064e9f)) | inline keyword anchors + small-to-big retrieval | push 03:40:06Z / 03:51:27Z; GH Archive [2026-04-07-3](https://data.gharchive.org/2026-04-07-3.json.gz): push 03:43:24Z with `before = 3f86353` |
| 2026-04-07 | hippograph-pro | [`82f7b50`](https://github.com/artemMprokhorov/hippograph-pro/commit/82f7b50) (pushed as [`fe3af56`](https://github.com/artemMprokhorov/hippograph-pro/commit/fe3af56), re-pushed as [`873b4d7`](https://github.com/artemMprokhorov/hippograph-pro/commit/873b4d7)) | batch keyword anchors in sleep | push 17:33:13Z; GH Archive [2026-04-07-17](https://data.gharchive.org/2026-04-07-17.json.gz): push 17:56:08Z with `before = 873b4d7` |

## Variants disclosed

- **Importance through category**: the old repos never make `critical` itself non-decaying; non-decay comes from the
  category (anchor protection), and protection in turn sets `critical` (boost step) and is inferred from `critical`
  notes (auto-discovery).
- **Decay switch for benchmarks**: `DISABLE_CATEGORY_DECAY=true` in the benchmark compose file
  ([`9cdb624`](https://github.com/artemMprokhorov/hippograph-pro/commit/9cdb624), pushed as [`59cdc9b`](https://github.com/artemMprokhorov/hippograph-pro/commit/59cdc9b), 2026-03-07), category decay is a production behaviour,
  disabled for external datasets.
- **Pending intentions** reuse the same protection: kept `critical` while pending, set to `low` when done: see
  [09-intentions.md](09-intentions.md).
- **Inline vs batch keyword anchors**, as described in item 5.

## Prior work and our contribution

Builds on: importance and recency scoring (Generative Agents, 2023); non-decaying core memory (MemGPT, 2023);
memory-type-specific retention and protected tags (mcp-memory-service, 2025-07-28); multi-vector and parent-document
retrieval (LangChain, 2023; Dense X Retrieval, 2023); entity resolution.
Our contribution (first disclosed 2026-03-21, hippograph-pro 2d0d873; user policies 2026-03-05, f7704a9): the set of
decay-exempt ("anchor") categories is the union of a fixed baseline, user policies added through MCP tools, and
categories discovered in each offline pass: any category that already contains a note marked critical, or whose
name contains a protective keyword; notes in these categories are kept at critical importance and their edges are
excluded from edge decay. Keyword anchors and entity merging: no distinct contribution claimed.

## Citation

Prokhorov, Artem. *HippoGraph* (formerly *neural-memory-graph*), importance scoring in `src/graph_engine.py`. GitHub,
https://github.com/artemMprokhorov/hippograph, commit `ac9da2397dfccb21a2e510aa189b6df612858fae` (2026-01-26). MIT License.
Software Heritage: https://archive.softwareheritage.org/browse/origin/?origin_url=https://github.com/artemMprokhorov/hippograph

Prokhorov, Artem. *HippoGraph Pro*: anchor memory `d115f80a53bb5102ae228b6e18bff2ca8bf22465` (first pushed as
`a30167afb11edf422eaa3efb85aade3037defe8c`, 2026-02-27); anchor policies `f7704a9768f1673d6fe0d7b73ffec4a7ba82bc2e`
(first pushed as `7057b7e92d03c496cf283d596a8cce43ae35b08b`, 2026-03-05); entity resolution
`e8f171963b19b31a594c4888f18ebbff30c941a3` (first pushed as `568c8cef104ed3cb8c09181389fc99212dffabba`, 2026-03-05); keyword
anchors `c7f4aa517785b0d52567c1ab2c1d9f193b5a5b6d` (first pushed as `3f863539c865ad961726259af13993d3447a85b6`, 2026-04-07).
GitHub, https://github.com/artemMprokhorov/hippograph-pro. MIT License. Software Heritage:
https://archive.softwareheritage.org/browse/origin/?origin_url=https://github.com/artemMprokhorov/hippograph-pro
