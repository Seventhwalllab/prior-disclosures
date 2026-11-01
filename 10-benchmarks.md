# Evaluation methods for a personal memory graph: LoCoMo retrieval adapter, Personal Continuity Benchmark, regression set

**First public disclosure:** 2026-02-11, reference-query precision baseline (P@5 / Top-1), hippograph [`19c08ce`](https://github.com/artemMprokhorov/hippograph/commit/19c08ce); LoCoMo adapter 2026-02-12 ([`3dbd353`](https://github.com/artemMprokhorov/hippograph/commit/3dbd353)); Personal Continuity Benchmark 2026-03-16 (hippograph-pro [`cf5afff`](https://github.com/artemMprokhorov/hippograph-pro/commit/cf5afff))
**Status of disclosure:** working code (MIT) and written methodology (`BENCHMARK.md`, `benchmark/README.md`)

All times are UTC; see [README.md](README.md#how-to-verify-dates). This document describes *what was measured and how*;
result numbers are the ones published in the old repos at the time and are given only where they identify a run.

## What it is

Three ways of measuring a long-term memory for an AI agent without needing an LLM in the retrieval loop:
(1) a public conversational dataset (LoCoMo) adapted into a retrieval-only test: load the dialogues as notes, ask the
dataset questions, check whether the evidence turns are in the top-k; (2) a *Personal Continuity Benchmark* (PCB) that
asks questions about the agent's own long-running relationship and project history and checks the real memory store;
(3) a small regression set of known queries with expected notes, run after every deploy. Around them: baselines on the
same data, an end-to-end variant with answer generation, a fixed seed and an isolated benchmark container.

## How it worked as disclosed

**1. Reference queries and regression test** (hippograph)
- [`19c08ce`](https://github.com/artemMprokhorov/hippograph/commit/19c08ce) ([`ROADMAP_PERSONAL.md`](https://github.com/artemMprokhorov/hippograph/blob/19c08ce/ROADMAP_PERSONAL.md)): "Precision@5 Baseline (Feb 11)" on 10 test queries, P@5 and Top-1.
- [`214dc6f`](https://github.com/artemMprokhorov/hippograph/commit/214dc6f) [`scripts/benchmark_search.py`](https://github.com/artemMprokhorov/hippograph/blob/214dc6f/scripts/benchmark_search.py): 10 queries (mixed
  Russian / English) each with expected note ids; per query `P@5 = hits / min(len(expected), 5)`, `Top-1 = 1` if the
  first hit is expected; averages reported.
- [`8c67296`](https://github.com/artemMprokhorov/hippograph/commit/8c67296) [`tests/regression_search.py`](https://github.com/artemMprokhorov/hippograph/blob/8c67296/tests/regression_search.py): 12 known queries, each
  with expected ids in top-5 and one `critical_id`; the run fails (exit code 1) if mean P@5 < 60 % or any critical note
  is missing; latency per query; JSON export; intended to run after every deploy ([`9123a41`](https://github.com/artemMprokhorov/hippograph/commit/9123a41)).

**2. LoCoMo as a retrieval-only benchmark** (hippograph, [`benchmark/locomo_adapter.py`](https://github.com/artemMprokhorov/hippograph/blob/3dbd353/benchmark/locomo_adapter.py))
- [`6c86e54`](https://github.com/artemMprokhorov/hippograph/commit/6c86e54) skeleton, [`3dbd353`](https://github.com/artemMprokhorov/hippograph/commit/3dbd353) full adapter: parse `locomo10.json` (10 conversations); load each
  session as one note (category `locomo-conv<N>`), keeping a map from note to the dialogue ids (`dia_id`) it contains;
- for each question: search, take top-k, *hit* if any returned note contains a ground-truth evidence `dia_id`;
  metrics Recall@k (k = 5) and MRR, broken down by single-hop / multi-hop / temporal / open-domain; adversarial
  questions (no evidence) are skipped: 1,540 questions; "Retrieval-only evaluation approach (no LLM API calls needed)";
- [`13d95ef`](https://github.com/artemMprokhorov/hippograph/commit/13d95ef): a separate benchmark container with its own empty database and REST endpoints, so the
  personal memory is never mixed with benchmark data;
- granularity as a parameter: session (≈272 notes), turn (≈5,882 notes, [`bb1ca1a`](https://github.com/artemMprokhorov/hippograph/commit/bb1ca1a)), hybrid 3-turn chunks
  ([`d42bd87`](https://github.com/artemMprokhorov/hippograph/commit/d42bd87)), optionally with cross-encoder reranking;
- [`581797a`](https://github.com/artemMprokhorov/hippograph/commit/581797a) [`BENCHMARK.md`](https://github.com/artemMprokhorov/hippograph/blob/581797a/BENCHMARK.md): results per granularity with an explicit
  comparability note, a retrieval metric, "not comparable to end-to-end F1" reported by other systems. First run:
  session level, [`9749d91`](https://github.com/artemMprokhorov/hippograph/commit/9749d91); the figures are in BENCHMARK.md at that commit.

**3. Baselines, end-to-end QA, reproducibility** (hippograph-pro)
- [`7a27e29`](https://github.com/artemMprokhorov/hippograph-pro/commit/7a27e29): baseline servers with the same REST API: pure BM25 and pure cosine (same embeddings,
  no graph), and a runner that evaluates all systems on the same dataset, queries and metric (Recall@5, MRR per
  category, latency P50/P95), granularity `turn | session | hybrid`;
- [`40fd2ea`](https://github.com/artemMprokhorov/hippograph-pro/commit/40fd2ea), [`80a3712`](https://github.com/artemMprokhorov/hippograph-pro/commit/80a3712): QA pairs generated from the own notes, by a small LLM, or rule-based without any
  LLM (entity-based, keyword and first-sentence questions, English / Russian);
- [`dd25b64`](https://github.com/artemMprokhorov/hippograph-pro/commit/dd25b64) ([`benchmark/e2e_eval.py`](https://github.com/artemMprokhorov/hippograph-pro/blob/dd25b64/benchmark/e2e_eval.py)): end-to-end variant: question → top-5 retrieval → a small
  LLM answers from that context only ("If the context doesn't contain the answer, say Unknown") → token F1, ROUGE-1
  and exact match against the reference; retrieval itself stays LLM-free;
- [`dbad43e`](https://github.com/artemMprokhorov/hippograph-pro/commit/dbad43e) ([`benchmark/README.md`](https://github.com/artemMprokhorov/hippograph-pro/blob/dbad43e/benchmark/README.md)): fixed seeds (`random.seed(42)`, `numpy.seed(42)`),
  step-by-step reproduction guide; the dataset is not redistributed (CC BY-NC 4.0), only linked and cited;
- [`9cdb624`](https://github.com/artemMprokhorov/hippograph-pro/commit/9cdb624): configuration-sensitivity section; benchmark and production configurations are kept
  intentionally different (e.g. per-category decay switched off for the external dataset).

**4. Personal Continuity Benchmark (PCB)** (hippograph-pro, `benchmark/personal_continuity*.py`, `benchmark/pcb_v5.py`)
- v1 [`cf5afff`](https://github.com/artemMprokhorov/hippograph-pro/commit/cf5afff) ([file](https://github.com/artemMprokhorov/hippograph-pro/blob/cf5afff/benchmark/personal_continuity.py)): 25 natural-language questions about the agent's
  own history with the user (identity, project history, technical decisions, architecture, security, dated sessions,
  relational context), each with ground-truth note ids; metric Recall@5 (hit if any expected note is in the top 5),
  per category; run against the production memory, not a synthetic set;
- v2 [`16ea48e`](https://github.com/artemMprokhorov/hippograph-pro/commit/16ea48e): ground truth by content instead of ids: a hit if a top-5 result contains any of a
  list of keywords (OR logic, bilingual): so the test survives re-imports and id changes; 27 questions, 7 categories;
- v3 [`4480c7b`](https://github.com/artemMprokhorov/hippograph-pro/commit/4480c7b) (26 q.) and v4 [`c36d9c4`](https://github.com/artemMprokhorov/hippograph-pro/commit/c36d9c4) (32 q.): questions refreshed from recent sessions;
  methodology written up in `BENCHMARK.md` (setup table: questions, metric, data, config);
- atomic facts [`b3d177c`](https://github.com/artemMprokhorov/hippograph-pro/commit/b3d177c) (2026-03-24): 15 numeric / factual questions with known parent notes;
- [`8b8f995`](https://github.com/artemMprokhorov/hippograph-pro/commit/8b8f995) (pushed as [`e772acf`](https://github.com/artemMprokhorov/hippograph-pro/commit/e772acf), 2026-03-27): A/B protocol: atomic-fact set + PCB set run
  in an isolated container on a copy of the production database, configurations compared row by row, with the average
  of both sets as the deciding number;
- v5 [`17a3c18`](https://github.com/artemMprokhorov/hippograph-pro/commit/17a3c18) (pushed as [`b2f2f10`](https://github.com/artemMprokhorov/hippograph-pro/commit/b2f2f10), 2026-03-28),
  [`benchmark/pcb_v5.py`](https://github.com/artemMprokhorov/hippograph-pro/blob/17a3c18/benchmark/pcb_v5.py): 35 questions = 15 atomic facts + 20 semantic questions, keyword-OR hit in
  the content of the top-5 results, run through the public search API.

## Disclosure timeline

| Date (UTC) | Repo | Commit | What was added | Evidence |
|---|---|---|---|---|
| 2026-02-11 | hippograph | [`19c08ce`](https://github.com/artemMprokhorov/hippograph/commit/19c08ce) | P@5 / Top-1 baseline on 10 reference queries | push 02:47:34Z ([activity](https://github.com/artemMprokhorov/hippograph/activity)) |
| 2026-02-12 | hippograph | [`214dc6f`](https://github.com/artemMprokhorov/hippograph/commit/214dc6f) | reference-query script | push 12:34:09Z; GH Archive [2026-02-12-16](https://data.gharchive.org/2026-02-12-16.json.gz): push 16:24:59Z with `before = 214dc6f`, `public: true` |
| 2026-02-12 | hippograph | [`6c86e54`](https://github.com/artemMprokhorov/hippograph/commit/6c86e54), [`3dbd353`](https://github.com/artemMprokhorov/hippograph/commit/3dbd353), [`13d95ef`](https://github.com/artemMprokhorov/hippograph/commit/13d95ef), [`9749d91`](https://github.com/artemMprokhorov/hippograph/commit/9749d91), [`bb1ca1a`](https://github.com/artemMprokhorov/hippograph/commit/bb1ca1a) | LoCoMo retrieval adapter, isolated container, first run, turn level | push 22:22:50Z…23:23:44Z; GH Archive [2026-02-12-22](https://data.gharchive.org/2026-02-12-22.json.gz): `3dbd353` at 22:27:17Z, `13d95ef` at 22:57:13Z, `public: true` |
| 2026-02-13 | hippograph | [`581797a`](https://github.com/artemMprokhorov/hippograph/commit/581797a), [`d42bd87`](https://github.com/artemMprokhorov/hippograph/commit/d42bd87) | `BENCHMARK.md` with comparability notes; hybrid granularity | push 03:49:35Z / 13:11:22Z; GH Archive [2026-02-13-3](https://data.gharchive.org/2026-02-13-3.json.gz), `public: true`; Wayback [2026-03-18](https://web.archive.org/web/20260318170705/https://github.com/artemMprokhorov/hippograph) README "LOCOMO benchmark" |
| 2026-02-18 | hippograph | [`8c67296`](https://github.com/artemMprokhorov/hippograph/commit/8c67296) | regression test (12 queries, critical notes) | push 19:28:42Z; first descendant captured by GH Archive: [`ca0392a`](https://github.com/artemMprokhorov/hippograph/commit/ca0392a) ([2026-02-18-23](https://data.gharchive.org/2026-02-18-23.json.gz), 23:33:00Z, `public: true`) |
| 2026-02-26 | hippograph-pro | [`7a27e29`](https://github.com/artemMprokhorov/hippograph-pro/commit/7a27e29) (first pushed as `3a13bdd`), [`40fd2ea`](https://github.com/artemMprokhorov/hippograph-pro/commit/40fd2ea) (pushed as [`61e8ebd`](https://github.com/artemMprokhorov/hippograph-pro/commit/61e8ebd)), [`80a3712`](https://github.com/artemMprokhorov/hippograph-pro/commit/80a3712) (pushed as [`eb51dfb`](https://github.com/artemMprokhorov/hippograph-pro/commit/eb51dfb)) | BM25 / cosine baselines, comparison runner, generated QA | push 13:13:49Z, 15:15:47Z, 15:19:37Z; GH Archive [2026-02-26-13](https://data.gharchive.org/2026-02-26-13.json.gz), [2026-02-26-15](https://data.gharchive.org/2026-02-26-15.json.gz), `public: true` |
| 2026-02-27 | hippograph-pro | [`dd25b64`](https://github.com/artemMprokhorov/hippograph-pro/commit/dd25b64) (pushed as [`cc9f058`](https://github.com/artemMprokhorov/hippograph-pro/commit/cc9f058)) | end-to-end QA (F1 / ROUGE-1 / EM) | push 21:14:11Z; GH Archive [2026-02-27-21](https://data.gharchive.org/2026-02-27-21.json.gz), `public: true`; ancestor of the Wayback [2026-03-04](https://web.archive.org/web/20260304011737/https://github.com/artemMprokhorov/hippograph-pro) head |
| 2026-03-06 | hippograph-pro | [`dbad43e`](https://github.com/artemMprokhorov/hippograph-pro/commit/dbad43e) (pushed as [`8bdc4db`](https://github.com/artemMprokhorov/hippograph-pro/commit/8bdc4db)) | fixed seed, reproduction guide | push 17:13:23Z; GH Archive [2026-03-06-17](https://data.gharchive.org/2026-03-06-17.json.gz), `public: true`; ancestor of the Wayback [2026-03-13](https://web.archive.org/web/20260313061019/https://github.com/artemMprokhorov/hippograph-pro) head |
| 2026-03-07 | hippograph-pro | [`9cdb624`](https://github.com/artemMprokhorov/hippograph-pro/commit/9cdb624) (pushed as [`59cdc9b`](https://github.com/artemMprokhorov/hippograph-pro/commit/59cdc9b)) | configuration sensitivity; benchmark ≠ production config | push 21:12:18Z; ancestor of the Wayback 2026-03-13 head |
| 2026-03-16 | hippograph-pro | [`cf5afff`](https://github.com/artemMprokhorov/hippograph-pro/commit/cf5afff) | PCB v1 (25 q., note ids, Recall@5) | push 22:52:13Z; GH Archive [2026-03-16-22](https://data.gharchive.org/2026-03-16-22.json.gz), `public: true`; forks [vaavaa](https://github.com/vaavaa/hippograph-pro/commit/cf5afff), [iflow-mcp](https://github.com/iflow-mcp/artemmprokhorov-hippograph-pro/commit/cf5afff) |
| 2026-03-20 | hippograph-pro | [`16ea48e`](https://github.com/artemMprokhorov/hippograph-pro/commit/16ea48e) | PCB v2: keyword-based ground truth | push 19:14:07Z; forks (as above) |
| 2026-03-23 / 24 | hippograph-pro | [`4480c7b`](https://github.com/artemMprokhorov/hippograph-pro/commit/4480c7b), [`c36d9c4`](https://github.com/artemMprokhorov/hippograph-pro/commit/c36d9c4), [`b3d177c`](https://github.com/artemMprokhorov/hippograph-pro/commit/b3d177c) | PCB v3, v4; atomic-facts set | push 03-23T02:53:19Z, 03-24T12:28:14Z, 03-24T17:09:53Z; GH Archive [2026-03-24-12](https://data.gharchive.org/2026-03-24-12.json.gz) (`c36d9c4`), `public: true`; forks |
| 2026-03-27 | hippograph-pro | [`8b8f995`](https://github.com/artemMprokhorov/hippograph-pro/commit/8b8f995) (pushed as [`e772acf`](https://github.com/artemMprokhorov/hippograph-pro/commit/e772acf)) | A/B protocol: atomic facts + PCB on a production copy | push 22:27:15Z; forks [vaavaa](https://github.com/vaavaa/hippograph-pro/commit/e772acf), [iflow-mcp](https://github.com/iflow-mcp/artemmprokhorov-hippograph-pro/commit/e772acf) |
| 2026-03-28 | hippograph-pro | [`17a3c18`](https://github.com/artemMprokhorov/hippograph-pro/commit/17a3c18) (pushed as [`b2f2f10`](https://github.com/artemMprokhorov/hippograph-pro/commit/b2f2f10)) | PCB v5 (15 atomic + 20 semantic) | push 23:29:59Z; forks [vaavaa](https://github.com/vaavaa/hippograph-pro/commit/b2f2f10), [iflow-mcp](https://github.com/iflow-mcp/artemmprokhorov-hippograph-pro/commit/b2f2f10) |

## Variants disclosed

- Ground truth by **note id** (PCB v1, regression set) or by **content keywords** (PCB v2-v5).
- LoCoMo **granularity**: session, turn, hybrid 3-turn chunks.
- **Retrieval-only** (Recall@k, MRR) vs **end-to-end** (generated answer, F1 / ROUGE-1 / EM).
- **Baselines** on identical data: BM25-only, cosine-only, full graph system.
- **QA generation** from one's own notes: LLM-generated or rule-based.

## Prior work and our contribution

Builds on: test collections with known relevant items (Cleverdon, Cranfield, 1967); re-finding evaluation on
personal collections (Dumais et al., SIGIR 2003; Elsweiler & Ruthven, SIGIR 2007); LoCoMo (Maharana et al., arXiv
2402.17753, 2024) and LongMemEval (Wu et al., arXiv 2410.10813, 2024) retrieval evaluation; template-based question
generation (Heilman & Smith, 2010).
Our contribution (first disclosed 2026-03-16, hippograph-pro cf5afff; content-matched version 2026-03-20, 16ea48e): a
personal continuity benchmark, hand-written questions about the agent's own production memory, grouped into
identity, history, decisions, architecture, security, temporal and emotional categories, scored as Recall@5 against
expected notes matched by content keywords rather than note IDs, so the test survives re-ingestion. The LLM-free,
retrieval-only LoCoMo protocol (2026-02-12): no distinct contribution claimed.

## Citation

Prokhorov, Artem. *HippoGraph* (formerly *neural-memory-graph*), `benchmark/locomo_adapter.py`, `BENCHMARK.md`,
`tests/regression_search.py`. GitHub, https://github.com/artemMprokhorov/hippograph, commits
`3dbd35373440a005b05c30fa1a1f035c46ebec72` (2026-02-12), `581797aade12a8fbe4c505b63425a8967f082636` (2026-02-13),
`8c672960e1c3c9f93a422232fccc466cc7baf4ec` (2026-02-18). MIT License. Software Heritage:
https://archive.softwareheritage.org/browse/origin/?origin_url=https://github.com/artemMprokhorov/hippograph

Prokhorov, Artem. *HippoGraph Pro*, Personal Continuity Benchmark, `benchmark/personal_continuity.py` at
`cf5afff0e68aac5230d6008ef887c102092b1b2c` (2026-03-16), v2 `16ea48e483cfd658241c359c661bad728fa84114` (2026-03-20), v5
`benchmark/pcb_v5.py` at `17a3c188d045351db2e2a34b30b70faa6f9bfb85` (first pushed as
`b2f2f1028ce5ed9f5a7d1849c3277e680208ed69`, 2026-03-28). GitHub, https://github.com/artemMprokhorov/hippograph-pro. MIT
License. Software Heritage: https://archive.softwareheritage.org/browse/origin/?origin_url=https://github.com/artemMprokhorov/hippograph-pro

LoCoMo dataset: Maharana et al., "Evaluating Very Long-Term Conversational Memory of LLM Agents", 2024 (used, not
redistributed).
