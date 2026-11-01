# Zero-LLM contradiction detection between memory notes (semantic similarity + negation/update cues)

**First public disclosure:** 2026-03-03, hippograph-pro [`c547d78`](https://github.com/artemMprokhorov/hippograph-pro/commit/c547d783fb6d1502053e6e45a9d9fe0e1bee3b86), originally pushed 2026-03-03T23:04:43Z as [`2d691dc`](https://github.com/artemMprokhorov/hippograph-pro/commit/2d691dced94b1ef647c3e238062f76e6cdc975bc).
**Status of disclosure:** working code (MIT License). An earlier one-line idea appeared on 2026-02-20.

All times are UTC. "Pushed" times are from GitHub's server-side push log (`/activity`, saved 2026-10-31).

## What it is

A background ("sleep") step of the memory graph finds pairs of notes that probably contradict each other, without calling a language model. A pair is flagged when two conditions hold:
- the two notes are semantically close (embedding cosine above a threshold), so they are about the same thing;
- at least one of them contains a lexical cue of negation, change or correction, such as "no longer", "used to", "replaced", "actually", "больше не", "теперь".

The older and newer note are identified by timestamp, and a severity score is computed. The pair is written to a review log; the notes themselves are never changed.

Later versions added an identity-aware mode, in which similarity alone is enough for self-identity notes. They also added cue lists for more languages, and graph edges of type `CONTRADICTS` that lower the rank of a contradicted note.

## How it worked as disclosed

**2026-03-03** ([`src/contradiction_detection.py` @ c547d78](https://github.com/artemMprokhorov/hippograph-pro/blob/c547d783fb6d1502053e6e45a9d9fe0e1bee3b86/src/contradiction_detection.py); wired into [`src/sleep_compute.py`](https://github.com/artemMprokhorov/hippograph-pro/blob/c547d783fb6d1502053e6e45a9d9fe0e1bee3b86/src/sleep_compute.py) as "Step 1c"):
1. **Cue lists.** There are 24 English and 21 Russian case-insensitive regular expressions for negation, update and correction. Examples: `\bno longer\b`, `\bnot anymore\b`, `\bchanged\b`, `\bnow\s+(?:is|are|…)\b`, `\bpreviously\b`, `\bused to\b`, `\binstead\b`, `\breplaced\b`, `\bdeprecated\b`, `\bcorrection\b`, `\bactually\b`, `\bwas wrong\b`; `\bбольше не\b`, `\bтеперь\b`, `\bраньше\b`, `\bна самом деле\b`.
2. **Candidates.** All notes with embeddings are loaded and sorted by timestamp. Each note is compared with the next `window_size` = 100 notes, which makes the work O(n·window) instead of O(n²).
3. **Flagging.** A pair is a candidate if cosine ≥ `similarity_threshold`, and if either note matches at least one cue. The threshold was 0.72. It was raised to 0.80 the same evening in [`e6451b3`](https://github.com/artemMprokhorov/hippograph-pro/commit/e6451b3153d93c0fa18197889dc7ad08c91a4cba) (original [`58bcc6c`](https://github.com/artemMprokhorov/hippograph-pro/commit/58bcc6c5d2e5e2d0b241d213e406b127c30670f0)); the commit message reports 1679 false positives marked.
4. **Ordering and severity.** The older note is the one with the earlier timestamp. `severity = cosine × min(total_matched_cues / 3, 1)`, and candidates are sorted by severity.
5. **Output.** New pairs are stored in a `contradiction_log` table (older id, newer id, similarity, severity, matched cues). The status starts as `pending` and can later be `resolved` or `false_positive`. Pairs already logged, other than those marked false positive, are skipped. The module docstring says the step "Does NOT delete anything", and a human decides.

**2026-03-05, identity-aware mode** ([`2b4f84f`](https://github.com/artemMprokhorov/hippograph-pro/commit/2b4f84f0588b7dd783ab929a4c21f5ed7f7953c5), original [`144672a`](https://github.com/artemMprokhorov/hippograph-pro/commit/144672ae0316a2d856206ce0d85cd9e57c31220f)):
- It applies to notes in the categories `self-identity`, `self-reflection`, `consciousness-research`, `critical-lesson` and `anchor`.
- For these notes all pairs are compared, and cosine ≥ 0.50 alone flags a pair; no lexical cue is needed. The rationale given is that statements about the self should be consistent.
- Severity = cosine × 0.8, and the type is `identity_conflict`. A `type` column was added to the log.
- The README feature table was updated the same day ([`0876b25`](https://github.com/artemMprokhorov/hippograph-pro/commit/0876b25090fef268f6189e30eaf4114382bf9278)).

**2026-03-10, more languages** ([`4f8ca35`](https://github.com/artemMprokhorov/hippograph-pro/commit/4f8ca35b90d0820ca46a456264336f3cc75acf10), original [`5aa9d1a`](https://github.com/artemMprokhorov/hippograph-pro/commit/5aa9d1a6ca88311b29077c10f54aec1f1c11c18d)):
- Cue lists were added for German (15), Spanish (14), French (14) and Portuguese (12), written without word-boundary anchors. Examples: `nicht mehr`, `ya no`, `ne plus`, `já não`. The detector then covered 6 languages.
- The commit title reads "multilingual NER and contradiction detection for 50+ languages". The "50+" applies to the entity-extraction part of the same commit, not to the cue lists.
- The README section added the same night ([`d15e073`](https://github.com/artemMprokhorov/hippograph-pro/commit/d15e073cb9de9ab25220089f94283688045c566c)) lists the six languages. It says that for other languages "semantic similarity alone triggers" detection.
- In the code at 4f8ca35, however, a lexical cue is still required outside the identity categories. This doc reports both as disclosed.

**2026-03-16, graph edges** ([`da9d3c4`](https://github.com/artemMprokhorov/hippograph-pro/commit/da9d3c406aac8ec640e9020f4dd79c27a3b63e4b), [`a6b6465`](https://github.com/artemMprokhorov/hippograph-pro/commit/a6b6465827e86876b0af709581752cb6596a0e0c)):
- Every logged pair also creates bidirectional `CONTRADICTS` edges with weight −severity.
- In retrieval, a note is multiplied by 0.5 if a note that contradicts it is itself scored above 0.3. See [04-typed-edges.md](04-typed-edges.md) and [02-blended-scoring.md](02-blended-scoring.md).

**Earlier idea, 2026-02-20** (hippograph [`e265471`](https://github.com/artemMprokhorov/hippograph/commit/e265471db21200f1262fd75405f3675a0db3bdb2), Pro roadmap): an optional LLM sleep task to "Identify contradictions or outdated information". It gives no method.

## Disclosure timeline

| Date (UTC) | Repo | Commit | What was added | Evidence |
|---|---|---|---|---|
| 2026-02-20 12:28 | hippograph | [e265471](https://github.com/artemMprokhorov/hippograph/commit/e265471db21200f1262fd75405f3675a0db3bdb2) | idea only (LLM sleep task) | push log 12:28:49Z. GH Archive `2026-02-20-12`: `PushEvent` 12:28:50Z head e265471, `public: true` |
| 2026-03-03 23:04 | hippograph-pro | [c547d78](https://github.com/artemMprokhorov/hippograph-pro/commit/c547d783fb6d1502053e6e45a9d9fe0e1bee3b86) | detector: similarity + EN/RU cues, window 100, severity, review log | pushed as [2d691dc](https://github.com/artemMprokhorov/hippograph-pro/commit/2d691dced94b1ef647c3e238062f76e6cdc975bc) at 23:04:43Z. GH Archive has no events for hour `2026-03-03-23`. Public by `2026-03-04-0`: `PushEvent` 00:20:09Z head fac0d9d, `public: true`; fac0d9d is 4 commits after 2d691dc (GitHub API parent chain). Wayback [2026-03-04 01:17Z](https://web.archive.org/web/20260304011737/https://github.com/artemMprokhorov/hippograph-pro) shows head fac0d9d (52 commits) |
| 2026-03-03 23:51 | hippograph-pro | [e6451b3](https://github.com/artemMprokhorov/hippograph-pro/commit/e6451b3153d93c0fa18197889dc7ad08c91a4cba) | threshold 0.72 → 0.80 | pushed as 58bcc6c within the push of 2026-03-04 00:01:20Z. Ancestor of fac0d9d (as above) |
| 2026-03-05 18:14 | hippograph-pro | [2b4f84f](https://github.com/artemMprokhorov/hippograph-pro/commit/2b4f84f0588b7dd783ab929a4c21f5ed7f7953c5) | identity-aware mode | pushed as [144672a](https://github.com/artemMprokhorov/hippograph-pro/commit/144672ae0316a2d856206ce0d85cd9e57c31220f) at 18:14:20Z. GH Archive `2026-03-05-18`: `PushEvent` 18:14:21Z head 144672a, `public: true` |
| 2026-03-10 03:45 | hippograph-pro | [4f8ca35](https://github.com/artemMprokhorov/hippograph-pro/commit/4f8ca35b90d0820ca46a456264336f3cc75acf10) | DE/ES/FR/PT cue lists | pushed as [5aa9d1a](https://github.com/artemMprokhorov/hippograph-pro/commit/5aa9d1a6ca88311b29077c10f54aec1f1c11c18d) at 03:45:37Z. GH Archive `2026-03-10-3`: `PushEvent` 03:55:06Z with before = 5aa9d1a, `public: true`. Wayback [2026-03-13](https://web.archive.org/web/20260313061019/https://github.com/artemMprokhorov/hippograph-pro) (head 13a8397, a descendant, 82 commits); the snapshot README contains the contradiction-detection language table |
| 2026-03-16 00:18 / 00:22 | hippograph-pro | [da9d3c4](https://github.com/artemMprokhorov/hippograph-pro/commit/da9d3c406aac8ec640e9020f4dd79c27a3b63e4b), [a6b6465](https://github.com/artemMprokhorov/hippograph-pro/commit/a6b6465827e86876b0af709581752cb6596a0e0c) | `CONTRADICTS` edges; retrieval penalty | push log 00:18:38Z and 00:22:33Z. GH Archive `2026-03-16-22` (a descendant), `public: true`. Same hashes in both forks |

**Note on hippograph-pro hashes.** The history before 2026-03-14 was rewritten on 2026-03-14 12:51Z, a force push. The commits above from 2026-03-03 to 2026-03-10 therefore have new hashes on `main`.
GitHub still serves the original hashes (2d691dc, 58bcc6c, 144672a, 5aa9d1a). The files `src/contradiction_detection.py` and `src/sleep_compute.py` are byte-identical between the original and current commits (checked through the GitHub API on 2026-10-31).
The current hashes have been public since 2026-03-14 12:51Z (GH Archive `2026-03-14-12`). They are also kept in the independent forks [vaavaa/hippograph-pro](https://github.com/vaavaa/hippograph-pro) (created 2026-03-26) and [iflow-mcp/artemmprokhorov-hippograph-pro](https://github.com/iflow-mcp/artemmprokhorov-hippograph-pro) (created 2026-04-01).
The original hashes are reachable from no branch, so Software Heritage does not hold them (checked for 2d691dc); it does hold the current ones.

## Variants disclosed

- **Trigger rule:** semantic similarity above a threshold AND a lexical cue in either note (general mode), or similarity ≥ 0.50 alone for identity categories (identity-aware mode).
- **Thresholds:** 0.72 (initial) and 0.80 (from e6451b3). The scan window is 100 neighbouring notes in time order; identity mode compares all pairs within its categories.
- **Languages of the cue lists:** EN and RU (2026-03-03); DE, ES, FR and PT added (2026-03-10).
- **Severity:** `cos × min(cues/3, 1)` in general mode and `cos × 0.8` in identity mode.
- **Outputs:**
  - a review log with a pending / resolved / false_positive workflow, with no automatic deletion (c547d78);
  - from 2026-03-16 also `CONTRADICTS` edges, first used as negative spread and then as a ×0.5 retrieval penalty (da9d3c4, a6b6465).
- **Not disclosed:** automatic resolution or deletion of contradicted notes. The code explicitly never modifies notes.

Later developed in An Airy Spirit (github.com/Seventhwalllab/anairyspirit).

## Prior work and our contribution

Builds on: negation and contrast cues for contradiction detection (Harabagiu, Hickl & Lacatusu, AAAI 2006; de
Marneffe, Rafferty & Manning, ACL 2008); model-free, negation-based contradicts links in agent memory (CerebroCortex,
2026-02-05); similarity-based conflict flagging (self-improving-memory-mcp, 2025-10-07).
Our contribution (first disclosed 2026-03-03, hippograph-pro c547d78; edges 2026-03-16, da9d3c4): an offline,
model-free detector that pairs notes within a window of temporally adjacent notes by embedding similarity and
requires a change or negation marker ("no longer", "previously", "instead", "replaced", …) in at least one of the
two; the newer note is treated as the update, and the pair is linked by a negative-weight CONTRADICTS edge that acts
as a ranking penalty. Variant (2026-03-05): for self-identity categories, similarity alone flags a conflict. Marker
lists for six languages (2026-03-10).

## Citation

Prokhorov, Artem (Artëm). *HippoGraph Pro*, MIT License. https://github.com/artemMprokhorov/hippograph-pro, commit `c547d783fb6d1502053e6e45a9d9fe0e1bee3b86` ("feat: add contradiction detection (semantic similarity + negation patterns, zero LLM cost)"). It was originally pushed 2026-03-03T23:04:43Z as `2d691dced94b1ef647c3e238062f76e6cdc975bc`.
Related commits: identity mode `2b4f84f0588b7dd783ab929a4c21f5ed7f7953c5` (2026-03-05) and the additional languages `4f8ca35b90d0820ca46a456264336f3cc75acf10` (2026-03-10).
Software Heritage (first archived 2026-10-31; preserves the content but does not prove the earlier dates): https://archive.softwareheritage.org/browse/origin/?origin_url=https://github.com/artemMprokhorov/hippograph-pro, revision `swh:1:rev:c547d783fb6d1502053e6e45a9d9fe0e1bee3b86`.
