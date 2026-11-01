# Spreading activation over a personal memory graph, seeded by semantic / ANN retrieval

**First public disclosure:** 2026-01-23 (description), hippograph [`813d0a1`](https://github.com/artemMprokhorov/hippograph/commit/813d0a1f7fe95c153df83080f4d8140203881861); 2026-01-26 (working code), hippograph [`a96a1e5`](https://github.com/artemMprokhorov/hippograph/commit/a96a1e51a1edc25759da249cfdd37c1d9ddb9a49)
**Status of disclosure:** working code (MIT License)

All times are UTC. Git author dates in these repositories are in UTC−3, so `git log` may show the previous calendar day.
"Pushed" times come from GitHub's server-side push log for the repository (`GET /repos/{owner}/{repo}/activity`, saved on 2026-10-31).

## What it is

A memory store for an AI assistant in which every note is a node in a graph. The nodes are linked by weighted edges, for example semantic-similarity edges and shared-entity edges.
A query is answered in two stages. First, the notes most similar to the query embedding receive an initial activation. From 2026-01-27 these seed notes come from an approximate-nearest-neighbour (ANN) index.
Second, that activation spreads along the graph edges for a fixed number of iterations, and it decays at each hop. The highest-scoring notes are returned, so the result includes notes that are associated with the query but not lexically similar to it.

## How it worked as disclosed

**Description, 2026-01-23** ([`docs/GRAPH_FEATURES.md` @ 813d0a1](https://github.com/artemMprokhorov/hippograph/blob/813d0a1f7fe95c153df83080f4d8140203881861/docs/GRAPH_FEATURES.md)): the query is embedded and similar notes are found by vector similarity. Activation then spreads along graph edges and boosts the notes connected to the similar notes. The document states the algorithm in four steps: "Initial activation", "Spread to neighbors", "Decay", "Threshold".

**Working code, 2026-01-26** ([`src/graph_engine.py` @ a96a1e5](https://github.com/artemMprokhorov/hippograph/blob/a96a1e51a1edc25759da249cfdd37c1d9ddb9a49/src/graph_engine.py), `search_with_activation`, lines 110-194):
1. The query is embedded (default `sentence-transformers/all-MiniLM-L6-v2`, 384-d). Every note with cosine ≥ 0.3 gets initial activation = cosine.
2. The spreading step repeats `ACTIVATION_ITERATIONS` = 3 times. Every node with activation ≥ 0.1 sends `spread = activation × edge_weight × ACTIVATION_DECAY` to each neighbour, with a decay of 0.7.
3. A neighbour that is not yet active takes the spread value. An active neighbour takes `max(a, a + 0.5·spread)`.
4. Each score is multiplied by a recency factor `0.5^(age_days / 30)`, and the top-k results are returned.

The graph is built in the same commit (`add_note_with_links`). It has bidirectional `entity` edges with weight 0.6 between notes that share an extracted entity. It also has bidirectional `semantic` edges between notes with cosine ≥ 0.5, at most 5 per note, with weight = cosine.

**ANN seeding, normalization and damping, 2026-01-27**
- [`dcb7d32`](https://github.com/artemMprokhorov/hippograph/commit/dcb7d323c7e6cfbf0ac9aaf9bb819aaf9c92d463) replaced the linear scan in step 1 with an index search ([`src/ann_index.py`](https://github.com/artemMprokhorov/hippograph/blob/dcb7d323c7e6cfbf0ac9aaf9bb819aaf9c92d463/src/ann_index.py), FAISS `IndexFlatIP` on L2-normalized vectors). The seed set is the top `limit × 3` candidates, with a linear-scan fallback.
- [`6b4b9e9`](https://github.com/artemMprokhorov/hippograph/commit/6b4b9e9c55e96bc8c786be5af34a5cc3643d6b77) changed the update rule. In each iteration a node keeps `activation × decay`, and each neighbour adds `activation × edge_weight × decay`.
- 6b4b9e9 also added a normalization step: after every iteration all activations are divided by the maximum, so the maximum is 1.0. Nodes below 0.01 are skipped. The commit message reports that this fixes the unbounded growth seen before ("2520+").
- After spreading, each score is multiplied by a recency factor and an importance factor. The recency factor uses the last access time, with a floor of 0.1. The importance factor is critical 2.0 / normal 1.0 / low 0.5, plus an access-count boost.
- [`e54312a`](https://github.com/artemMprokhorov/hippograph/commit/e54312aed4df64ca7e33518d1e0c4988222c7c63) keeps the whole adjacency list in RAM, `{node_id: [(neighbor_id, weight, edge_type)]}`, so that spreading does not query SQL on each hop ([`src/graph_cache.py`](https://github.com/artemMprokhorov/hippograph/blob/e54312aed4df64ca7e33518d1e0c4988222c7c63/src/graph_cache.py)).

**2026-01-28** [`4037dcc`](https://github.com/artemMprokhorov/hippograph/commit/4037dcca0fe972b307df596b43a7d5768d82e38d) moved the seed index to hnswlib HNSW (cosine space, incremental `add_items`), so new notes become seedable without a rebuild.

## Disclosure timeline

| Date (UTC) | Repo | Commit | What was added | Evidence |
|---|---|---|---|---|
| 2026-01-23 00:25 | hippograph (then `neural-memory-graph`) | [813d0a1](https://github.com/artemMprokhorov/hippograph/commit/813d0a1f7fe95c153df83080f4d8140203881861) | algorithm described in README and GRAPH_FEATURES.md | push log 00:25:29Z (merge 12ff9ef). GH Archive `2026-01-23-0`: `CreateEvent` 00:24:08Z and `PushEvent` 00:25:29Z, both `public: true` |
| 2026-01-26 01:42 | hippograph | [a96a1e5](https://github.com/artemMprokhorov/hippograph/commit/a96a1e51a1edc25759da249cfdd37c1d9ddb9a49) | working engine: cosine seeding, 3 iterations, decay 0.7, semantic/entity edges | push log 01:42:51Z. GH Archive `2026-01-26-1`: `PushEvent` head a96a1e5, `public: true` |
| 2026-01-27 21:43 | hippograph | [dcb7d32](https://github.com/artemMprokhorov/hippograph/commit/dcb7d323c7e6cfbf0ac9aaf9bb819aaf9c92d463), [6b4b9e9](https://github.com/artemMprokhorov/hippograph/commit/6b4b9e9c55e96bc8c786be5af34a5cc3643d6b77) | ANN seeding; per-iteration max-normalization and damping | push log 21:43:30Z (head 009f209). GH Archive has no events for this hour. Public by 2026-01-28 13:02Z (GH Archive `2026-01-28-13`, head 46e69ce, a descendant) |
| 2026-01-27 23:51 | hippograph | [e54312a](https://github.com/artemMprokhorov/hippograph/commit/e54312aed4df64ca7e33518d1e0c4988222c7c63) | in-memory graph cache for traversal | push log 23:51:49Z. GH Archive `2026-01-28-13` (descendant) |
| 2026-01-28 13:21 | hippograph | [4037dcc](https://github.com/artemMprokhorov/hippograph/commit/4037dcca0fe972b307df596b43a7d5768d82e38d) | hnswlib HNSW seed index | push log 13:21:35Z. GH Archive `2026-01-28-13`: `PushEvent` 13:36:29Z with before = 4037dcc, `public: true` |
| 2026-02-04 18:11 | hippograph | [bfc9310](https://github.com/artemMprokhorov/hippograph/commit/bfc9310fd711bc7219135362f639f5fc85bb7373) | (all of the above as ancestors) | GH Archive `2026-02-04-18`, `public: true`. The GitHub-signed web-flow commit [6adb2b2](https://github.com/artemMprokhorov/hippograph/commit/6adb2b229c22a14bf54d5fedeb81826f897a3139) (2026-02-05 01:29Z) descends from all of them |
| 2026-03-05 18:25 / 18:29 | hippograph-pro | [ba9bbca](https://github.com/artemMprokhorov/hippograph-pro/commit/ba9bbca6cf8d91d4767e91feb85463de844cd619), [c5cfcb1](https://github.com/artemMprokhorov/hippograph-pro/commit/c5cfcb1ac5ef0489cb3187a4cb81242f4a2366f6) | subgraph sampling; community-aware routing | originally pushed as [bf3eb87](https://github.com/artemMprokhorov/hippograph-pro/commit/bf3eb87bce5c0b721254b0e630dd8cec62b30a53) (18:25:21Z) and [a0891a1](https://github.com/artemMprokhorov/hippograph-pro/commit/a0891a148545f29122f809aa3cab6cb2ce69b99f) (18:29:47Z). File content is identical to the current hashes. Ancestors of the Wayback 2026-03-13 head; also in forks |
| 2026-03-14 00:11 | hippograph-pro | [8822671](https://github.com/artemMprokhorov/hippograph-pro/commit/8822671ba723598d3663e0f48aaf3e402585d065) | ×1.5 spread along temporal edges for temporal queries | originally pushed as [4ef484e](https://github.com/artemMprokhorov/hippograph-pro/commit/4ef484e6e253348c30cd0fbbaef8de52b8a5a082). GH Archive `2026-03-14-0`: `PushEvent` 00:11:05Z, `public: true`. Content is identical |

Independent snapshots:
- hippograph: Wayback [2026-03-18 17:07Z](https://web.archive.org/web/20260318170705/https://github.com/artemMprokhorov/hippograph) shows 146 commits, head 169fa4f, which contains every hippograph commit above.
- hippograph-pro: Wayback [2026-03-13 06:10Z](https://web.archive.org/web/20260313061019/https://github.com/artemMprokhorov/hippograph-pro) shows 82 commits, head 13a8397. The GitHub API parent chain confirms bf3eb87 and a0891a1 as ancestors of 13a8397.

**Note on hippograph-pro hashes.** The hippograph-pro history was rewritten on 2026-03-14 12:51Z (force push) and again on 2026-04-10. Commits from 2026-02-26 13:13Z to 2026-03-14 therefore have a new hash on `main`.
GitHub still serves the original hashes listed above, and the file contents of the cited commits are byte-identical (checked file by file on 2026-10-31).
The current hashes were public from 2026-03-14 12:51Z (GH Archive `2026-03-14-12`, `public: true`). They are also present in two independent forks: [vaavaa/hippograph-pro](https://github.com/vaavaa/hippograph-pro), created 2026-03-26, and [iflow-mcp/artemmprokhorov-hippograph-pro](https://github.com/iflow-mcp/artemmprokhorov-hippograph-pro), created 2026-04-01.

## Variants disclosed

- **Seed source:** linear cosine scan with threshold 0.3 (a96a1e5); FAISS index, top `limit×3` (dcb7d32); hnswlib HNSW (4037dcc).
- **Update rule:** max/accumulate rule without normalization (a96a1e5); additive spreading with per-iteration max-normalization and damping (6b4b9e9). From 2026-02-10 the normalized spreading score is blended with the semantic score (see [02-blended-scoring.md](02-blended-scoring.md)).
- **Subgraph sampling (hippograph-pro, 2026-03-05,** [ba9bbca](https://github.com/artemMprokhorov/hippograph-pro/commit/ba9bbca6cf8d91d4767e91feb85463de844cd619)**):** spreading is limited to the ANN seed nodes plus their direct neighbours (`SA_SUBGRAPH_ENABLED`, default true).
- **Community-aware routing (2026-03-05,** [c5cfcb1](https://github.com/artemMprokhorov/hippograph-pro/commit/c5cfcb1ac5ef0489cb3187a4cb81242f4a2366f6)**):** spreading is further limited to the graph communities that contain the seeds, plus one hop outside them (`SA_COMMUNITY_ROUTING`).
- **Edge-type-dependent spreading:**
  - Spread along `TEMPORAL_BEFORE/AFTER` edges is ×1.5 when the query is temporal (8822671, 2026-03-14).
  - On 2026-03-16, `CONTRADICTS` edges first carried negative (inhibitory) spread ([da9d3c4](https://github.com/artemMprokhorov/hippograph-pro/commit/da9d3c406aac8ec640e9020f4dd79c27a3b63e4b)). This was replaced 4 minutes later by a score penalty ([a6b6465](https://github.com/artemMprokhorov/hippograph-pro/commit/a6b6465827e86876b0af709581752cb6596a0e0c)).
  - See [04-typed-edges.md](04-typed-edges.md).
- **Inhibition inside the spreading loop:** suppression of non-winning nodes per community on the last iteration (2026-03-27). See [02-blended-scoring.md](02-blended-scoring.md).

Later developed in An Airy Spirit (github.com/Seventhwalllab/anairyspirit).

## Prior work and our contribution

Builds on: spreading activation over semantic networks (Collins & Loftus 1975; Anderson 1983); search results as
seeds for spreading activation (Rocha et al., WWW 2004); vector-search seeds expanded over a graph (HippoRAG 2024);
spreading activation in agent memory seeded by semantic hits (Hindsight/TEMPR, arXiv 2512.12818, 2025-12-14;
SYNAPSE, arXiv 2601.02744, 2026-01-06).
No distinct contribution claimed. This record documents the date and form of our implementation only.

## Citation

Prokhorov, Artem (Artëm). *HippoGraph* (formerly *neural-memory-graph*), MIT License. https://github.com/artemMprokhorov/hippograph, commit `a96a1e51a1edc25759da249cfdd37c1d9ddb9a49`, publicly pushed 2026-01-26T01:42:51Z (description: commit `813d0a1f7fe95c153df83080f4d8140203881861`, 2026-01-23T00:25:29Z).
Software Heritage: https://archive.softwareheritage.org/browse/origin/?origin_url=https://github.com/artemMprokhorov/hippograph, revision `swh:1:rev:a96a1e51a1edc25759da249cfdd37c1d9ddb9a49`.
The Software Heritage copy was first archived on 2026-10-31. It preserves the content, but it is not evidence of the earlier dates.
For the hippograph-pro variants: https://github.com/artemMprokhorov/hippograph-pro, commits `ba9bbca`, `c5cfcb1` (2026-03-05). Software Heritage origin: https://archive.softwareheritage.org/browse/origin/?origin_url=https://github.com/artemMprokhorov/hippograph-pro.
