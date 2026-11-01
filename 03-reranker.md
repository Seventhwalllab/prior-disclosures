# Cross-encoder reranking stage in a graph-memory retrieval pipeline

**First public disclosure:** 2026-02-12 — hippograph [`7502438`](https://github.com/artemMprokhorov/hippograph/commit/75024382806b6254c52d11a77949c325870bb220) (working code). It was announced as a plan the same day in [`2b4e5b8`](https://github.com/artemMprokhorov/hippograph/commit/2b4e5b87325c76efc81986804d7cb059c7ec2291). A broader idea, "multi-factor" reranking, appeared on 2026-02-04 in [`63ebb09`](https://github.com/artemMprokhorov/hippograph/commit/63ebb09348ad7e14e98f4ceda9960beb06db8bb2).
**Status of disclosure:** working code (MIT License). It was off by default until 2026-03-25 and enabled in the deployment config from 2026-03-25.

All times are UTC. Git author dates are in UTC−3. "Pushed" times are from GitHub's server-side push log (`/activity`, saved 2026-10-31).

## What it is

The graph-memory search first produces a candidate list. The list comes from spreading activation and the blended score: semantic, spreading, BM25 and temporal (see [01](01-spreading-activation.md) and [02](02-blended-scoring.md)).
The top N candidates are then re-scored by a cross-encoder that reads each (query, note text) pair together. The cross-encoder score is min-max normalized and mixed with the original blended score, and the list is re-sorted.
Cross-encoder reranking is a well-known IR technique. What is disclosed here is its placement as a stage after graph spreading and blend fusion, together with its parameters.

## How it worked as disclosed

**Idea, 2026-02-04** ([`63ebb09`](https://github.com/artemMprokhorov/hippograph/commit/63ebb09348ad7e14e98f4ceda9960beb06db8bb2), `ROADMAP_ENTERPRISE.md`): "Advanced Search Reranking". The plan is to combine graph distance, recency, importance and entity overlap, with learning-to-rank from feedback. It does not mention a cross-encoder.

**Plan, 2026-02-12** ([`ROADMAP_PERSONAL.md` @ 2b4e5b8](https://github.com/artemMprokhorov/hippograph/blob/2b4e5b87325c76efc81986804d7cb059c7ec2291/ROADMAP_PERSONAL.md)): after the initial blend, rerank the top-N candidates with a sentence-transformers cross-encoder ("ms-marco-MiniLM-L6-v2 or similar"), going from top-20 to top-5. The enterprise roadmap in the same commit names "BGE-m3 or ms-marco".

**Code, 2026-02-12** ([`src/reranker.py` @ 7502438](https://github.com/artemMprokhorov/hippograph/blob/75024382806b6254c52d11a77949c325870bb220/src/reranker.py); Step 6.5 in [`src/graph_engine.py`](https://github.com/artemMprokhorov/hippograph/blob/75024382806b6254c52d11a77949c325870bb220/src/graph_engine.py)):
1. After the blended score (and the hub penalty), take the top `RERANK_TOP_N` = 20 notes by score, each with its content.
2. Score each `(query, content)` pair with `sentence_transformers.CrossEncoder(RERANK_MODEL)`. The default model is `cross-encoder/ms-marco-MiniLM-L-6-v2`, loaded lazily on first use.
3. Min-max normalize the raw cross-encoder scores to [0, 1]; if all scores are equal, use 0.5.
4. `final = (1 − RERANK_WEIGHT)·blend_score + RERANK_WEIGHT·rerank_score`, with `RERANK_WEIGHT` = 0.3. Write the result back into the score map and re-sort.
5. `RERANK_ENABLED` = false by default. If the model is disabled or fails to load, the stage passes the scores through unchanged.

The commit message states the order as "Semantic → Spreading Activation → BM25 → Blend → Rerank → Results".

**Benchmark use, 2026-02-13** ([`d42bd87`](https://github.com/artemMprokhorov/hippograph/commit/d42bd87ef8edce5ba543caefda8692fbd7964076)): reranking was enabled in the LOCOMO benchmark container. [`4d6cbb0`](https://github.com/artemMprokhorov/hippograph/commit/4d6cbb0) reports the benchmark result with "hybrid+reranking" in BENCHMARK.md at that commit.

**Model change and deployment, 2026-03-25** (hippograph-pro [`ca15c87`](https://github.com/artemMprokhorov/hippograph-pro/commit/ca15c87a56d36216b7de3a6b09c3d3f7079e3da8), `docker-compose.yml`): `RERANK_ENABLED=true`, `RERANK_MODEL=BAAI/bge-reranker-v2-m3`, `RERANK_TOP_N=20`, `RERANK_WEIGHT=0.5`. The commit message reports an internal benchmark (PCB) going from 46.7% to 66.7%.

## Disclosure timeline

| Date (UTC) | Repo | Commit | What was added | Evidence |
|---|---|---|---|---|
| 2026-02-04 23:31 | hippograph | [63ebb09](https://github.com/artemMprokhorov/hippograph/commit/63ebb09348ad7e14e98f4ceda9960beb06db8bb2) | idea: multi-factor reranking (no cross-encoder) | push log 23:31:19Z (force push). GH Archive `2026-02-04-23`: `PushEvent` 23:31:20Z head 63ebb09, `public: true` |
| 2026-02-12 16:24 | hippograph | [2b4e5b8](https://github.com/artemMprokhorov/hippograph/commit/2b4e5b87325c76efc81986804d7cb059c7ec2291) | plan: cross-encoder rerank top-20 → top-5 | push log 16:24:58Z. GH Archive `2026-02-12-16`: `PushEvent` 16:24:59Z head 2b4e5b8, `public: true` |
| 2026-02-12 17:00 | hippograph | [7502438](https://github.com/artemMprokhorov/hippograph/commit/75024382806b6254c52d11a77949c325870bb220) | working reranker stage after the blend | push log 17:00:32Z. GH Archive `2026-02-12-17`: `PushEvent` 17:00:32Z head 7502438, `public: true` |
| 2026-02-13 13:11 | hippograph | [d42bd87](https://github.com/artemMprokhorov/hippograph/commit/d42bd87ef8edce5ba543caefda8692fbd7964076) | reranking enabled in the benchmark | push log 13:11:22Z. Wayback [2026-03-18](https://web.archive.org/web/20260318170705/https://github.com/artemMprokhorov/hippograph) (head 169fa4f, a descendant) |
| 2026-02-20 12:58 | hippograph-pro | [78945a6](https://github.com/artemMprokhorov/hippograph-pro/commit/78945a68687fa64a5bc67faf2e4604f643347ca4) | same stage carried into hippograph-pro (after blend/RRF + hub penalty) | push log 12:58:22Z. Wayback [2026-03-04](https://web.archive.org/web/20260304011737/https://github.com/artemMprokhorov/hippograph-pro) (a descendant) |
| 2026-03-25 00:28 | hippograph-pro | [ca15c87](https://github.com/artemMprokhorov/hippograph-pro/commit/ca15c87a56d36216b7de3a6b09c3d3f7079e3da8) | bge-reranker-v2-m3 enabled, weight 0.5, top-N 20 | originally pushed as [b72e819](https://github.com/vaavaa/hippograph-pro/commit/b72e81915f1e538c6bfe598fcf224e72db608806) at 00:28:38Z. GH Archive `2026-03-25-0`: `PushEvent` 00:28:39Z head b72e819, `public: true`. b72e819 is in both forks ([vaavaa](https://github.com/vaavaa/hippograph-pro/commit/b72e81915f1e538c6bfe598fcf224e72db608806), [iflow-mcp](https://github.com/iflow-mcp/artemmprokhorov-hippograph-pro/commit/b72e81915f1e538c6bfe598fcf224e72db608806)) |
| 2026-03-28 23:29 | hippograph-pro | [17a3c18](https://github.com/artemMprokhorov/hippograph-pro/commit/17a3c188d045351db2e2a34b30b70faa6f9bfb85) | BENCHMARK/README document the production config with bge-reranker-v2-m3 | originally [b2f2f10](https://github.com/vaavaa/hippograph-pro/commit/b2f2f1028ce5ed9f5a7d1849c3277e680208ed69), pushed 23:29:59Z; in forks |

**Note on hippograph-pro hashes.** The history from 2026-03-24 22:36Z on was rewritten on 2026-04-10. The trees of ca15c87 and b72e819 differ only in one configuration line; `docker-compose.yml` is identical.
GitHub still serves the original hash b72e819. The fork vaavaa/hippograph-pro (created 2026-03-26) and the Wayback [2026-04-06](https://web.archive.org/web/20260406082821/https://github.com/artemMprokhorov/hippograph-pro/blob/main/AGENT_PROMPT.md) snapshot (head 0ef8eee, a descendant of b72e819) are independent copies.

## Variants disclosed

- **Models:** `cross-encoder/ms-marco-MiniLM-L-6-v2` as the default (7502438), or `BAAI/bge-reranker-v2-m3` in deployment (ca15c87). The model is configurable through `RERANK_MODEL`. The 2026-02-12 enterprise roadmap also names "BGE-m3".
- **Mixing:** a linear mix of the normalized cross-encoder score with the blend score, with `RERANK_WEIGHT` 0.3 (default) or 0.5 (deployment) and `RERANK_TOP_N` = 20.
- **Position:** after the weighted blend or RRF and after the hub penalty, before the final sort, in both hippograph and hippograph-pro (78945a6).
- **Off by default** with a passthrough fallback (7502438). From 2026-03-25 it is enabled in deployment (ca15c87).
- **Not disclosed:** the 2026-02-04 idea of multi-factor or learning-to-rank reranking was never implemented in these repositories.

Later developed in An Airy Spirit (github.com/Seventhwalllab/anairyspirit).

## Prior work and our contribution

Builds on: cross-encoder re-ranking (Nogueira & Cho 2019); cross-encoder re-ranking in agent-memory retrieval (Zep,
2025; Hindsight/TEMPR, 2025).
No distinct contribution claimed.

## Citation

Prokhorov, Artem (Artëm). *HippoGraph*, MIT License. https://github.com/artemMprokhorov/hippograph, commit `75024382806b6254c52d11a77949c325870bb220` ("feat: cross-encoder reranking pass (optional)"), publicly pushed 2026-02-12T17:00:32Z.
Prokhorov, Artem (Artëm). *HippoGraph Pro*, MIT License. https://github.com/artemMprokhorov/hippograph-pro, commit `ca15c87a56d36216b7de3a6b09c3d3f7079e3da8`. The original hash `b72e81915f1e538c6bfe598fcf224e72db608806` was pushed 2026-03-25T00:28:38Z.
Software Heritage (first archived 2026-10-31; preserves the content but does not prove the earlier dates):
- https://archive.softwareheritage.org/browse/origin/?origin_url=https://github.com/artemMprokhorov/hippograph
- revisions `swh:1:rev:75024382806b6254c52d11a77949c325870bb220` and `swh:1:rev:b72e81915f1e538c6bfe598fcf224e72db608806` (the latter archived via the fork origin https://github.com/vaavaa/hippograph-pro)
