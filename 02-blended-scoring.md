# Blended retrieval scoring over a memory graph: semantic + spreading + BM25 + temporal, RRF, hub penalty, lateral inhibition

**First public disclosure:** 2026-02-11, hippograph [`d86ccd6`](https://github.com/artemMprokhorov/hippograph/commit/d86ccd61e8ecba756291dc69105480785f60fa9d) (semantic + spreading blend). The full four-signal blend was public on 2026-02-18 ([`e76ef82`](https://github.com/artemMprokhorov/hippograph/commit/e76ef82d3f13486173404f2b72724d2e04cf8aaf)).
**Status of disclosure:** working code (MIT License). RRF was first described on 2026-02-18 and published as code on 2026-02-20.

All times are UTC. Git author dates are in UTC−3. "Pushed" times are from GitHub's server-side push log (`/activity`, saved 2026-10-31).

## What it is

The final rank of a memory note is not the raw spreading-activation value (see [01-spreading-activation.md](01-spreading-activation.md)). It is a weighted blend of several independently normalized signals:
- semantic similarity of the query to the note,
- the note's spreading-activation score in the graph,
- a BM25 keyword score,
- a temporal-overlap score between the time range in the query and the event time of the note.

Reciprocal Rank Fusion (RRF) is offered as an alternative to the weighted blend. Several post-blend corrections follow:
- a penalty for "hub" notes with many entities,
- penalties driven by edge types,
- lateral inhibition, in which within each graph community the top note keeps its score and the others are suppressed.

## How it worked as disclosed

**2026-02-11, blend of two signals** ([`src/graph_engine.py` @ d86ccd6](https://github.com/artemMprokhorov/hippograph/blob/d86ccd61e8ecba756291dc69105480785f60fa9d/src/graph_engine.py)): the raw semantic similarities of the seed notes are preserved. Spreading scores and semantic scores are each divided by their maximum. Then `final = α·semantic + (1−α)·spreading`, with `BLEND_ALPHA` = 0.6. The commit states that this "Fixes hub dominance". The deployment was tuned to α = 0.7 the same day ([`c68549a`](https://github.com/artemMprokhorov/hippograph/commit/c68549a)).

**2026-02-11, hub penalty** ([`0d9f8f9`](https://github.com/artemMprokhorov/hippograph/commit/0d9f8f9d86206b1765d373583d16482365e3c54e)): after blending, a note linked to more than 20 entities is multiplied by `20 / entity_count`. The commit gives 0.8 at 25 entities and 0.48 at 42 entities.

**2026-02-12, third signal: BM25** ([`01f4187`](https://github.com/artemMprokhorov/hippograph/commit/01f4187688244cebb7fce752f8f1796990ca3a47)):
- The commit adds an Okapi BM25 inverted index in pure Python ([`src/bm25_index.py`](https://github.com/artemMprokhorov/hippograph/blob/01f4187688244cebb7fce752f8f1796990ca3a47/src/bm25_index.py), k1 = 1.5, b = 0.75). The index is built at startup and updated for each new note.
- BM25 scores are divided by their maximum, and `final = α·semantic + β·spreading + γ·BM25` with `β = 1 − α − γ`.
- The candidate set is the union of graph-activated notes and BM25 hits. γ is 0 by default (`BLEND_GAMMA`).

**2026-02-18, fourth signal: temporal** ([`e76ef82`](https://github.com/artemMprokhorov/hippograph/commit/e76ef82d3f13486173404f2b72724d2e04cf8aaf)):
- Event-time ranges (`t_event_start/end`) are extracted from notes and from the query ([`src/temporal_extractor.py`](https://github.com/artemMprokhorov/hippograph/blob/e76ef82d3f13486173404f2b72724d2e04cf8aaf/src/temporal_extractor.py), explicit and relative dates, English and Russian).
- The temporal score is the fraction of the query range covered by the note range. Then `final = α·sem + β·spread + γ·BM25 + δ·temporal` with `β = 1 − α − γ − δ`.
- In [`b7a3724`](https://github.com/artemMprokhorov/hippograph/commit/b7a3724), the same day, temporal words are stripped from the query, and δ is set to 0.15 automatically when the query is temporal and δ is otherwise 0.

**RRF.** RRF was described on 2026-02-18 in the hippograph roadmap ([`99abd0d`](https://github.com/artemMprokhorov/hippograph/commit/99abd0d3f279878a236c845973b309b7cbbf1a29)). It was published as code on 2026-02-20 in hippograph-pro [`78945a6`](https://github.com/artemMprokhorov/hippograph-pro/commit/78945a68687fa64a5bc67faf2e4604f643347ca4) ([`src/rrf_fusion.py`](https://github.com/artemMprokhorov/hippograph-pro/blob/78945a68687fa64a5bc67faf2e4604f643347ca4/src/rrf_fusion.py)).
- Each of the four signals is ranked separately, and `RRF(d) = Σ_r 1/(k + rank_r(d))` with k = 60. This is a standard method; the module cites Cormack et al. 2009.
- The setting `FUSION_METHOD=blend|rrf` selects the method; the default is `blend`.
- The pipeline order is: blend or RRF → hub penalty → optional cross-encoder rerank (see [03-reranker.md](03-reranker.md)) → sort.

**2026-03-16, lateral inhibition** (hippograph-pro [`04eb35e`](https://github.com/artemMprokhorov/hippograph-pro/commit/04eb35e8c8e0b3082fe330f577a72b709333d09b)):
- Communities are computed by greedy modularity at a higher resolution (`COMMUNITY_RESOLUTION` = 2.0; the commit reports 104 sub-clusters).
- After blending, the scored notes are grouped by community. In each group with at least 2 members, the top note keeps its score and every other member is multiplied by `(1 − INHIBITION_STRENGTH)`, default 0.3.
- The commit reports top-5 diversity rising from 3.2 to 4.8 communities.

**2026-03-27, late-stage inhibition** (hippograph-pro [`da9ba44`](https://github.com/artemMprokhorov/hippograph-pro/commit/da9ba444fc45d3992c685093350c5d5058667090)):
- The same per-community suppression is also applied inside the spreading loop, on the third (last) iteration, with factor `1 − 0.5·INHIBITION_STRENGTH`.
- The post-blend step is kept. The commit reports deployment with strength 0.05, chosen by grid search.

## Disclosure timeline

| Date (UTC) | Repo | Commit | What was added | Evidence |
|---|---|---|---|---|
| 2026-02-11 02:20 | hippograph | [d86ccd6](https://github.com/artemMprokhorov/hippograph/commit/d86ccd61e8ecba756291dc69105480785f60fa9d) | semantic + spreading blend, both max-normalized | push log 02:20:28Z. Public by 13:48Z: GH Archive `2026-02-11-13`, `PushEvent` head c266126 (a descendant), `public: true` |
| 2026-02-11 13:15 | hippograph | [0d9f8f9](https://github.com/artemMprokhorov/hippograph/commit/0d9f8f9d86206b1765d373583d16482365e3c54e) | entity-count hub penalty | push log 13:15:13Z. GH Archive `2026-02-11-13`: `PushEvent` 13:48:12Z with before = 0d9f8f9, `public: true` |
| 2026-02-12 16:51 | hippograph | [01f4187](https://github.com/artemMprokhorov/hippograph/commit/01f4187688244cebb7fce752f8f1796990ca3a47) | BM25 as third signal | push log 16:51:18Z. GH Archive `2026-02-12-17`: `PushEvent` 17:00:32Z with before = 01f4187, `public: true` |
| 2026-02-18 13:05 | hippograph | [e76ef82](https://github.com/artemMprokhorov/hippograph/commit/e76ef82d3f13486173404f2b72724d2e04cf8aaf) | temporal-overlap signal δ | push log 13:05:00Z. GH Archive `2026-02-18-13`: `PushEvent` head e76ef82, `public: true` |
| 2026-02-18 22:17 | hippograph | [99abd0d](https://github.com/artemMprokhorov/hippograph/commit/99abd0d3f279878a236c845973b309b7cbbf1a29) | RRF described as an alternative to the weighted blend | push log 22:17:30Z. Wayback [2026-03-18](https://web.archive.org/web/20260318170705/https://github.com/artemMprokhorov/hippograph) (head 169fa4f, a descendant) |
| 2026-02-20 12:58 | hippograph-pro | [78945a6](https://github.com/artemMprokhorov/hippograph-pro/commit/78945a68687fa64a5bc67faf2e4604f643347ca4) | RRF fusion code; full blend pipeline carried over | push log 12:58:22Z. GH Archive `2026-02-26-0`: `PushEvent` head 8533620 (a descendant), `public: true`. Wayback [2026-03-04](https://web.archive.org/web/20260304011737/https://github.com/artemMprokhorov/hippograph-pro) |
| 2026-03-16 00:22 | hippograph-pro | [a6b6465](https://github.com/artemMprokhorov/hippograph-pro/commit/a6b6465827e86876b0af709581752cb6596a0e0c) | CONTRADICTS penalty ×0.5 in the blend | push log 00:22:33Z. GH Archive `2026-03-16-22` (a descendant), `public: true`. Same hash in forks |
| 2026-03-16 22:06 | hippograph-pro | [04eb35e](https://github.com/artemMprokhorov/hippograph-pro/commit/04eb35e8c8e0b3082fe330f577a72b709333d09b) | lateral inhibition via sub-communities | push log 22:06:51Z. GH Archive `2026-03-16-22`: `PushEvent` 22:16:18Z with before = 04eb35e, `public: true`. Same hash in [vaavaa fork](https://github.com/vaavaa/hippograph-pro/commit/04eb35e8c8e0b3082fe330f577a72b709333d09b) |
| 2026-03-24 22:36 | hippograph-pro | [c9b5b1b](https://github.com/artemMprokhorov/hippograph-pro/commit/c9b5b1ba2fb581d38de8466a7a55f9bdf72fb599) | BM25 enabled in deployment (γ = 0.15) | originally pushed as [124e05b](https://github.com/vaavaa/hippograph-pro/commit/124e05b7d4d20d84be50adfd22ba75e053e3ef5d) (22:36:20Z), which is in both forks. The only difference is one line in `benchmark/bm25_grid.py` |
| 2026-03-27 21:31 | hippograph-pro | [da9ba44](https://github.com/artemMprokhorov/hippograph-pro/commit/da9ba444fc45d3992c685093350c5d5058667090) | late-stage inhibition inside spreading | originally pushed as [46360a0](https://github.com/vaavaa/hippograph-pro/commit/46360a073591a621979e931ae5b8efe4432a3fb8). GH Archive `2026-03-27-21`: `PushEvent` 21:31:29Z head 46360a0, `public: true`. In both forks. Ancestor of the Wayback [2026-04-06](https://web.archive.org/web/20260406082821/https://github.com/artemMprokhorov/hippograph-pro/blob/main/AGENT_PROMPT.md) head 0ef8eee |

**Note on hippograph-pro hashes.** The hippograph-pro history was rewritten on 2026-04-10. Commits from 2026-03-24 22:36Z onward therefore have new hashes on `main`.
The original hashes listed above are still served by GitHub. They are also kept in the independent forks [vaavaa/hippograph-pro](https://github.com/vaavaa/hippograph-pro) (created 2026-03-26) and [iflow-mcp/artemmprokhorov-hippograph-pro](https://github.com/iflow-mcp/artemmprokhorov-hippograph-pro) (created 2026-04-01).
Original and rewritten trees differ only in one configuration line (checked 2026-10-31). The GH Archive hour `2026-02-11-2` has no events for these repos, so the earliest blend push is confirmed only by the GitHub push log and by later descendant events.

## Variants disclosed

- **Weights:** α = 0.6 (d86ccd6) and α = 0.7 (c68549a). γ = 0 by default, then 0.15 with α = 0.7 in hippograph-pro ([707e0dc](https://github.com/artemMprokhorov/hippograph-pro/commit/707e0dc); original [b4a80f0](https://github.com/vaavaa/hippograph-pro/commit/b4a80f0)). δ = 0 by default, 0.1 in the benchmark ([f1d45ab](https://github.com/artemMprokhorov/hippograph/commit/f1d45ab)), and 0.15 automatically for temporal queries (b7a3724).
- **Fusion:** weighted linear blend, or RRF with k = 60 (78945a6, `FUSION_METHOD`).
- **Multipliers before the blend** (since 2026-01-27, see 01): recency `max(0.1, 0.5^(age/30d))`, and importance critical 2.0 / normal 1.0 / low 0.5 plus an access boost. The importance boost was reduced to 1.5 and the access boost to 20% in [171232a](https://github.com/artemMprokhorov/hippograph/commit/171232a) (2026-02-06).
- **Penalties after the blend:**
  - hub penalty `20/entity_count` for notes with more than 20 entities (0d9f8f9);
  - CONTRADICTS: ×0.5 if a contradicting note is itself scored above 0.3 (a6b6465);
  - SUPERSEDES: ×0.3 for a superseded note. This was added in [243f149](https://github.com/artemMprokhorov/hippograph-pro/commit/243f149c0d50a9fcbc2238a88a501bc2cd729ec7) and removed the same day in [49abe5b](https://github.com/artemMprokhorov/hippograph-pro/commit/49abe5b1696d2b120da3b7c9516eab0f48bde4d8), because all tested penalty values lowered recall.
- **Lateral inhibition:**
  - post-blend winner-takes-most per sub-community, strength 0.3 (04eb35e);
  - also inside the last spreading iteration (da9ba44);
  - chunk-aware inhibition on 2026-04-07, where chunks of the same parent note (linked by `PART_OF`) suppress each other with strength 0.6. This is hippograph-pro [88a1828](https://github.com/artemMprokhorov/hippograph-pro/commit/88a1828f4d480a5b52ff5d505bac4a80ad43a120), originally pushed 2026-04-07 20:05:02Z as [29649ba](https://github.com/artemMprokhorov/hippograph-pro/commit/29649ba0e23a61ec0f05efbf69aeb83742791105). No fork copy exists, so the only evidence is the GitHub push log.

Later developed in An Airy Spirit (github.com/Seventhwalllab/anairyspirit).

## Prior work and our contribution

Builds on: linear score fusion (Fox & Shaw 1994), BM25 (Robertson et al. 1994), Reciprocal Rank Fusion (Cormack et
al. 2009); semantic + BM25 + graph + temporal retrieval fused by RRF in agent memory (Hindsight/TEMPR, arXiv
2512.12818, 2025-12-14; Zep, arXiv 2501.13956, 2025-01-20); hub dampening (fan effect; HippoRAG node specificity,
2024); lateral inhibition in agent-memory retrieval (SYNAPSE, arXiv 2601.02744, 2026-01-06); result diversification
(MMR, 1998).
Our contribution (first disclosed 2026-03-16, hippograph-pro 04eb35e): lateral inhibition scoped to graph
communities, after score fusion, candidates compete only within the community detected on the memory graph; the
top-scoring note of each community keeps its score and every other member is multiplied by (1 − s). The weighted
blend, the RRF option, the entity-count hub penalty and the temporal-overlap term (2026-02-11 … 02-20) are disclosed
as implementation, without a novelty claim.

## Citation

Prokhorov, Artem (Artëm). *HippoGraph*, MIT License. https://github.com/artemMprokhorov/hippograph, commits `d86ccd61e8ecba756291dc69105480785f60fa9d` (2026-02-11), `01f4187688244cebb7fce752f8f1796990ca3a47` (2026-02-12) and `e76ef82d3f13486173404f2b72724d2e04cf8aaf` (2026-02-18).
Prokhorov, Artem (Artëm). *HippoGraph Pro*, MIT License. https://github.com/artemMprokhorov/hippograph-pro, commits `78945a68687fa64a5bc67faf2e4604f643347ca4` (RRF, 2026-02-20) and `04eb35e8c8e0b3082fe330f577a72b709333d09b` (lateral inhibition, 2026-03-16).
Software Heritage (first archived 2026-10-31; preserves the content but does not prove the earlier dates):
- https://archive.softwareheritage.org/browse/origin/?origin_url=https://github.com/artemMprokhorov/hippograph
- https://archive.softwareheritage.org/browse/origin/?origin_url=https://github.com/artemMprokhorov/hippograph-pro
- revision `swh:1:rev:d86ccd61e8ecba756291dc69105480785f60fa9d`
