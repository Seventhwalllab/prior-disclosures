# Typed edges in a personal memory graph

**First public disclosure:**
- 2026-01-23: `semantic` and `entity` edge types, described — hippograph [`813d0a1`](https://github.com/artemMprokhorov/hippograph/commit/813d0a1f7fe95c153df83080f4d8140203881861).
- 2026-01-26: the same edge types as working code — hippograph [`a96a1e5`](https://github.com/artemMprokhorov/hippograph/commit/a96a1e51a1edc25759da249cfdd37c1d9ddb9a49).
- The other types followed between 2026-01-28 and 2026-03-31; see the timeline below.

**Status of disclosure:** working code for every type listed here (MIT License).

All times are UTC. Git author dates are in UTC−3. "Pushed" times are from GitHub's server-side push log (`/activity`, saved 2026-10-31).

## What it is

Every edge in the memory graph carries a `weight` and an `edge_type`. Each type is produced by its own rule:
- at write time (`add_note`), or
- by a background "sleep" pass over the stored notes.

Spreading activation runs over all of these edges (see [01](01-spreading-activation.md)). Some types also have a specific effect on retrieval, such as a boost, a penalty, or an exclusion from the results.
The edge types disclosed:
- **Similarity and entity links:** `semantic`, `entity`.
- **Consolidation and time:** `consolidation`, `temporal_chain`, `TEMPORAL_BEFORE` / `TEMPORAL_AFTER`.
- **Conflict and update:** `CONTRADICTS`, `SUPERSEDES`.
- **Affect:** `EMOTIONAL_RESONANCE`.
- **Abstraction and structure:** `GENERALIZES` / `INSTANTIATES`, `BELONGS_TO` (topic nodes), `PART_OF` (atomic facts and chunks).

## How it worked as disclosed

| Type | Introduced | Rule as disclosed | Weight / direction | Retrieval effect |
|---|---|---|---|---|
| `entity` | [a96a1e5](https://github.com/artemMprokhorov/hippograph/blob/a96a1e51a1edc25759da249cfdd37c1d9ddb9a49/src/graph_engine.py) (2026-01-26) | on `add_note`: link to every note that shares an extracted entity | 0.6, both directions | spreading |
| `semantic` | a96a1e5 | on `add_note`: up to 5 most similar notes with cosine ≥ 0.5 | weight = cosine, both directions | spreading |
| `consolidation` | [a091221](https://github.com/artemMprokhorov/hippograph/blob/a091221ceb44433abbedbb7846c5ca9ac33e1150/src/memory_consolidation.py) (2026-01-28) | thematic clusters (cosine ≥ 0.75, at least 3 notes), all pairs in a cluster | 0.9 | spreading |
| `temporal_chain` | a091221 | within each category, notes in time order with gaps ≤ 7 days; chains of at least 3 notes; consecutive notes linked | 0.95, earlier → later | spreading |
| `TEMPORAL_BEFORE` / `TEMPORAL_AFTER` | hippograph-pro [8822671](https://github.com/artemMprokhorov/hippograph-pro/commit/8822671ba723598d3663e0f48aaf3e402585d065) (2026-03-14) | on `add_note`: the 6 nodes nearest by extracted event time `t_event_start`; also a backfill script | 0.4; BEFORE from earlier to later, AFTER the reverse | spread ×1.5 on these edges when the query is temporal |
| `CONTRADICTS` | [da9d3c4](https://github.com/artemMprokhorov/hippograph-pro/commit/da9d3c406aac8ec640e9020f4dd79c27a3b63e4b) (2026-03-16) | created for each pair found by the contradiction detector (see [05](05-contradiction-detector.md)) | −severity, both directions | first negative spread; from [a6b6465](https://github.com/artemMprokhorov/hippograph-pro/commit/a6b6465827e86876b0af709581752cb6596a0e0c) a ×0.5 score penalty if the contradicting note is active (> 0.3) |
| `EMOTIONAL_RESONANCE` | [74078e0](https://github.com/artemMprokhorov/hippograph-pro/commit/74078e0c34958e85d5f9e89a85b92e890562d63b) (2026-03-16) | sleep step: notes sharing at least 2 `emotional_tone` tags; at most 5 per note | Jaccard of the tag sets, both directions | spreading |
| `GENERALIZES` / `INSTANTIATES` | [4dc70d9](https://github.com/artemMprokhorov/hippograph-pro/commit/4dc70d9f871e1a1835b2f4286be75b0fe1ac4c36) (2026-03-16) | sleep step: a "concrete" category note (lesson, crisis, debug, session-summary, …) paired with an "abstract" category note (protocol, skill, architecture-decision, …) at cosine ≥ 0.65; at most 3 per note | weight = cosine; GENERALIZES concrete → abstract, INSTANTIATES abstract → concrete | spreading |
| `SUPERSEDES` | [243f149](https://github.com/artemMprokhorov/hippograph-pro/commit/243f149c0d50a9fcbc2238a88a501bc2cd729ec7) (2026-03-22) | sleep step: pairs with cosine ≥ 0.85 inside a 100-note sliding window, at least 1 shared entity, different timestamps; the older note must not be critical; the older note is set to `importance=low` | weight = cosine, newer → older | ×0.3 penalty, removed the same day in [49abe5b](https://github.com/artemMprokhorov/hippograph-pro/commit/49abe5b1696d2b120da3b7c9516eab0f48bde4d8) ("All 4 penalty configs hurt"); the edges were kept |
| `BELONGS_TO` | [fcd9175](https://github.com/artemMprokhorov/hippograph-pro/commit/fcd9175fe72744cbbc3871e2781abf94e10eb7d1) (2026-03-24) | sleep step: one topic node (category `abstract-topic`) per community cluster, labelled by its top-3 TF-IDF terms; a second variant uses k-means on embeddings with k = n/30 | TF-IDF variant 0.5 note → topic and 0.4 topic → note; k-means variant 0.6 and 0.5 | topic nodes take part in spreading but are excluded from results |
| `PART_OF` (facts) | [bb98ac5](https://github.com/artemMprokhorov/hippograph-pro/commit/bb98ac5fdf1c24dbe3f0d787e9c893b33a7227df) (2026-03-24) | sleep step: up to 5 "atomic-fact" nodes per eligible note (regex numeric facts and spaCy subject-verb-object triples) | 0.8, both directions | fact nodes take part in spreading but are excluded from results |
| `PART_OF` (chunks) | [f0ada94](https://github.com/artemMprokhorov/hippograph-pro/commit/f0ada9470da6fd99dac6aedf7b209637f3816d47) (2026-03-31) | on `add_note`: if late chunking is enabled, notes of 600 characters or more are split into chunks of about 400 characters with about 200 characters of overlap; each chunk becomes a node with its own embedding in the ANN index | 0.9, both directions | chunks are retrievable. From 2026-04-07, chunks of the same parent inhibit each other (see [02](02-blended-scoring.md)) |

The 2026-01-23 description ([`docs/GRAPH_FEATURES.md` @ 813d0a1](https://github.com/artemMprokhorov/hippograph/blob/813d0a1f7fe95c153df83080f4d8140203881861/docs/GRAPH_FEATURES.md)) already defined an edge as having a "Weight — Connection strength (0.0-1.0)" and a "Type — Connection reason (semantic, entity-based)", both directions tracked.

## Disclosure timeline

| Date (UTC) | Repo | Commit | What was added | Evidence |
|---|---|---|---|---|
| 2026-01-23 00:25 | hippograph | [813d0a1](https://github.com/artemMprokhorov/hippograph/commit/813d0a1f7fe95c153df83080f4d8140203881861) | typed, weighted, bidirectional edges described | push log 00:25:29Z. GH Archive `2026-01-23-0`, `public: true` |
| 2026-01-26 01:42 | hippograph | [a96a1e5](https://github.com/artemMprokhorov/hippograph/commit/a96a1e51a1edc25759da249cfdd37c1d9ddb9a49) | `semantic`, `entity` | push log 01:42:51Z. GH Archive `2026-01-26-1`: `PushEvent` head a96a1e5, `public: true` |
| 2026-01-28 19:22 | hippograph | [a091221](https://github.com/artemMprokhorov/hippograph/commit/a091221ceb44433abbedbb7846c5ca9ac33e1150) | `consolidation`, `temporal_chain` | push log 19:22:43Z. GH Archive has no events for this hour. Public by 2026-02-04 18:11Z (GH Archive `2026-02-04-18`, bfc9310 a descendant). The GitHub-signed commit 6adb2b2 is also a descendant |
| 2026-03-14 00:11 | hippograph-pro | [8822671](https://github.com/artemMprokhorov/hippograph-pro/commit/8822671ba723598d3663e0f48aaf3e402585d065) | `TEMPORAL_BEFORE/AFTER` | originally pushed as [4ef484e](https://github.com/artemMprokhorov/hippograph-pro/commit/4ef484e6e253348c30cd0fbbaef8de52b8a5a082). GH Archive `2026-03-14-0`: `PushEvent` 00:11:05Z, `public: true`. File content is identical. The current hash has been public since 12:51Z (GH Archive `2026-03-14-12`) and is in both forks |
| 2026-03-16 00:18 | hippograph-pro | [da9d3c4](https://github.com/artemMprokhorov/hippograph-pro/commit/da9d3c406aac8ec640e9020f4dd79c27a3b63e4b) | `CONTRADICTS` | push log 00:18:38Z. Public by 22:16Z (GH Archive `2026-03-16-22`, a descendant). Same hash in the [vaavaa fork](https://github.com/vaavaa/hippograph-pro/commit/da9d3c406aac8ec640e9020f4dd79c27a3b63e4b) |
| 2026-03-16 01:04 | hippograph-pro | [74078e0](https://github.com/artemMprokhorov/hippograph-pro/commit/74078e0c34958e85d5f9e89a85b92e890562d63b) | `EMOTIONAL_RESONANCE` | push log 01:04:37Z. GH Archive `2026-03-16-22` (a descendant). In forks |
| 2026-03-16 01:27 | hippograph-pro | [4dc70d9](https://github.com/artemMprokhorov/hippograph-pro/commit/4dc70d9f871e1a1835b2f4286be75b0fe1ac4c36) | `GENERALIZES`, `INSTANTIATES` | push log 01:27:05Z. GH Archive `2026-03-16-22` (a descendant). In forks |
| 2026-03-22 17:54 | hippograph-pro | [243f149](https://github.com/artemMprokhorov/hippograph-pro/commit/243f149c0d50a9fcbc2238a88a501bc2cd729ec7) | `SUPERSEDES` (branch `feat/supersedes-edge`, merged in 9d7864f at 19:01Z) | push log branch creation 17:54:39Z. GH Archive `2026-03-22-17`: `CreateEvent` 17:54:39Z, `public: true`. In forks |
| 2026-03-24 02:45 | hippograph-pro | [fcd9175](https://github.com/artemMprokhorov/hippograph-pro/commit/fcd9175fe72744cbbc3871e2781abf94e10eb7d1) | `BELONGS_TO`, abstract-topic nodes | push log 02:45:17Z. GH Archive `2026-03-24-2`: `PushEvent` 02:50:18Z with before = fcd9175, `public: true`. In forks |
| 2026-03-24 12:45 | hippograph-pro | [bb98ac5](https://github.com/artemMprokhorov/hippograph-pro/commit/bb98ac5fdf1c24dbe3f0d787e9c893b33a7227df) | `PART_OF` (atomic facts) | push log 12:45:25Z. GH Archive `2026-03-24-12`: `PushEvent` 12:45:26Z head bb98ac5, `public: true`. In forks |
| 2026-03-31 00:10 | hippograph-pro | [f0ada94](https://github.com/artemMprokhorov/hippograph-pro/commit/f0ada9470da6fd99dac6aedf7b209637f3816d47) | `PART_OF` (overlap chunks) | originally pushed as [2334bd9](https://github.com/vaavaa/hippograph-pro/commit/2334bd976b25c33e77f30b3e4b6611fb32bd30a6) at 00:10:46Z, with an identical tree. GH Archive `2026-03-31-0`: `PushEvent` head 2334bd9, `public: true`. In both forks |

Additional independent evidence for hippograph-pro:
- The forks [vaavaa/hippograph-pro](https://github.com/vaavaa/hippograph-pro) (created 2026-03-26) and [iflow-mcp/artemmprokhorov-hippograph-pro](https://github.com/iflow-mcp/artemmprokhorov-hippograph-pro) (created 2026-04-01) contain every commit from 8822671 to 2334bd9 above.
- Wayback [2026-04-06](https://web.archive.org/web/20260406082821/https://github.com/artemMprokhorov/hippograph-pro/blob/main/AGENT_PROMPT.md) shows head 0ef8eee, which is a descendant of all of them.

hippograph is confirmed by Wayback [2026-03-18](https://web.archive.org/web/20260318170705/https://github.com/artemMprokhorov/hippograph) (head 169fa4f).

## Variants disclosed

- **Creation time:** at write time (`semantic`, `entity`, `TEMPORAL_*`, `PART_OF` chunks) or in a background sleep pass (`consolidation`, `temporal_chain`, `EMOTIONAL_RESONANCE`, `GENERALIZES/INSTANTIATES`, `SUPERSEDES`, `BELONGS_TO`, `PART_OF` facts). `CONTRADICTS` comes from the contradiction-detection sleep step.
- **Signed weights:** `CONTRADICTS` used a negative weight and negative spread (da9d3c4). This was replaced by a multiplicative penalty (a6b6465).
- **Synthetic nodes linked by typed edges:**
  - topic nodes (`BELONGS_TO`) and atomic-fact nodes (`PART_OF`) take part in spreading but are not returned;
  - chunk nodes (`PART_OF`) are returned.
- **Two topic-linking variants:** TF-IDF over community clusters and k-means over embeddings (fcd9175).
- **Edge type as a retrieval modifier:**
  - ×1.5 spread on temporal edges for temporal queries (8822671);
  - penalties for `CONTRADICTS` (kept) and `SUPERSEDES` (removed);
  - chunk-sibling inhibition through `PART_OF` (2026-04-07).
- **Out of scope here:** entity-relation types extracted by spaCy/GLiNER2 (hippograph-pro, 2026-02-26/28) are not covered by this document.

Later developed in An Airy Spirit (github.com/Seventhwalllab/anairyspirit).

## Prior work and our contribution

Builds on: typed links in agent-memory graphs (Zep/Graphiti, arXiv 2501.13956, 2025; Hindsight/TEMPR, 2025; MAGMA,
arXiv 2601.03236, 2026-01-06; mcp-memory-service relationship ontology, 2026-01-17; CerebroCortex, 2026-02-05, with
temporal, semantic, affective, contradicts, derived_from and part_of links); emotion nodes in associative memory
(Bower 1981); lexical hierarchies (WordNet); topic nodes over clusters (GraphRAG, 2024).
Our contribution (first disclosed 2026-03-16 and 2026-03-22, hippograph-pro 74078e0, 4dc70d9, 243f149): rules that
build three edge types without any generative model, in an offline pass — emotional-resonance edges between notes
sharing at least two emotional-tone tags, weighted by Jaccard and capped per note; paired generalizes/instantiates
edges between notes of designated concrete-experience and abstract-rule categories above a cosine threshold; and
supersedes edges from a later note to an earlier one when they are highly similar and share at least one entity.

## Citation

Prokhorov, Artem (Artëm). *HippoGraph*, MIT License. https://github.com/artemMprokhorov/hippograph, commits `a96a1e51a1edc25759da249cfdd37c1d9ddb9a49` (2026-01-26) and `a091221ceb44433abbedbb7846c5ca9ac33e1150` (2026-01-28).
Prokhorov, Artem (Artëm). *HippoGraph Pro*, MIT License. https://github.com/artemMprokhorov/hippograph-pro, commits `8822671`, `da9d3c4`, `74078e0`, `4dc70d9`, `243f149`, `fcd9175`, `bb98ac5` and `f0ada94` (original `2334bd976b25c33e77f30b3e4b6611fb32bd30a6`), dated 2026-03-14 to 2026-03-31.
Software Heritage (first archived 2026-10-31; preserves the content but does not prove the earlier dates):
- https://archive.softwareheritage.org/browse/origin/?origin_url=https://github.com/artemMprokhorov/hippograph
- https://archive.softwareheritage.org/browse/origin/?origin_url=https://github.com/artemMprokhorov/hippograph-pro
