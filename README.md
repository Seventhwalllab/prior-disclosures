# Prior public disclosures — Artem Prokhorov / Seventh Wall Lab

This repository collects short records of memory-system mechanisms that **Artem Prokhorov** (Artëm Prokhorov) had
already published, as open-source code and design notes, in his personal GitHub repositories between January and
April 2026. Each record says what the mechanism is, where and when it first became public (with links that let anyone
check the date), and how it worked *as disclosed at that time*. The purpose is to make these earlier public
disclosures easy to find and to cite as prior art. This is not product documentation.

The mechanisms were later developed further in An Airy Spirit (github.com/Seventhwalllab/anairyspirit); nothing here
describes that later work.

## The three source repositories

| Repository | Created (GitHub server time) | Last push in the period | Commits | License |
|---|---|---|---|---|
| [semantic-memory-mcp](https://github.com/artemMprokhorov/semantic-memory-mcp) | 2026-01-21T22:08:54Z | 2026-02-06T02:16:56Z | 8 | MIT |
| [hippograph](https://github.com/artemMprokhorov/hippograph) (created as `neural-memory-graph`; the old URL redirects) | 2026-01-23T00:24:06Z | 2026-02-28T22:52:42Z | 146 | MIT, © 2026 Artëm Prokhorov |
| [hippograph-pro](https://github.com/artemMprokhorov/hippograph-pro) | 2026-02-20T12:46:53Z | 2026-04-10T03:36:17Z | 229 (+ branch `feat/supersedes-edge`) | MIT, © 2026 Artëm Prokhorov |

Independent records of these repositories:

| Repository | GH Archive (public events) | Wayback Machine | Software Heritage | Independent forks |
|---|---|---|---|---|
| semantic-memory-mcp | `CreateEvent` 2026-01-21T22:08:55Z, `public: true` ([2026-01-21-22](https://data.gharchive.org/2026-01-21-22.json.gz)) | no capture found | [origin](https://archive.softwareheritage.org/browse/origin/?origin_url=https://github.com/artemMprokhorov/semantic-memory-mcp) | — |
| hippograph | `CreateEvent` 2026-01-23T00:24:08Z as `neural-memory-graph`, `public: true` ([2026-01-23-0](https://data.gharchive.org/2026-01-23-0.json.gz)); push events cited per document | [2026-03-18](https://web.archive.org/web/20260318170705/https://github.com/artemMprokhorov/hippograph) ("146 Commits", head `169fa4f`) | [origin](https://archive.softwareheritage.org/browse/origin/?origin_url=https://github.com/artemMprokhorov/hippograph) | — |
| hippograph-pro | push events cited per document | [2026-03-04](https://web.archive.org/web/20260304011737/https://github.com/artemMprokhorov/hippograph-pro) ("52 Commits", head `fac0d9d`), [2026-03-13](https://web.archive.org/web/20260313061019/https://github.com/artemMprokhorov/hippograph-pro) ("82 Commits", head `13a8397`), [2026-04-06](https://web.archive.org/web/20260406082821/https://github.com/artemMprokhorov/hippograph-pro/blob/main/AGENT_PROMPT.md) (`AGENT_PROMPT.md`, head `0ef8eee`) | [origin](https://archive.softwareheritage.org/browse/origin/?origin_url=https://github.com/artemMprokhorov/hippograph-pro) | [vaavaa/hippograph-pro](https://github.com/vaavaa/hippograph-pro) (created 2026-03-26T04:48Z, last push 2026-04-06T08:09Z); [iflow-mcp/artemmprokhorov-hippograph-pro](https://github.com/iflow-mcp/artemmprokhorov-hippograph-pro) (created 2026-04-01T06:26Z, copy of the state of 2026-03-31T12:10Z). Both also in Software Heritage |

Notes. GitHub's licence detector shows "Other" for hippograph and hippograph-pro because the MIT text in `LICENSE` is
wrapped in a Markdown code fence; the text itself is the MIT License. Software Heritage first archived all five origins on 2026-10-31.

## Documents

| # | Document | First public disclosure (UTC) |
|---|---|---|
| 01 | [Spreading activation over the memory graph, seeded from vector search](01-spreading-activation.md) | 2026-01-23 (description); 2026-01-26 (working code) |
| 02 | [Blended scoring: semantic + spreading + BM25 + temporal, RRF, hub penalty, lateral inhibition](02-blended-scoring.md) | 2026-02-11 |
| 03 | [Cross-encoder reranking in the memory retrieval pipeline](03-reranker.md) | 2026-02-12 |
| 04 | [Typed edges in an agent memory graph](04-typed-edges.md) | 2026-01-23 (description); 2026-01-26 (code) |
| 05 | [Negation-based contradiction detection](05-contradiction-detector.md) | 2026-03-03 |
| 06 | [Sleep-time consolidation without an LLM](06-sleep-consolidation.md) | 2026-01-28 (consolidation); recency half-life 2026-01-26 |
| 07 | [Importance, anchor categories, anchor policies, keyword anchors, entity merging](07-anchors-and-importance.md) | 2026-01-26 (importance); anchors 2026-02-27; policies 2026-03-05; keyword anchors 2026-04-07 |
| 08 | [Working memory: overwrite note, then time-linked journal](08-working-memory.md) | 2026-03-17 (journal 2026-04-07; short-term → long-term idea 2026-01-26) |
| 09 | [Intentions (prospective memory)](09-intentions.md) | 2026-04-08 |
| 10 | [Evaluation methods: LoCoMo retrieval adapter, Personal Continuity Benchmark, regression set](10-benchmarks.md) | 2026-02-11 (reference queries), 2026-02-12 (LoCoMo), 2026-03-16 (PCB) |

## How to verify dates

All dates and times in these documents are **UTC**. The author worked in UTC−3, so the date shown on a GitHub commit
page can be one day earlier than the UTC date given here.

1. **Commit page** — `https://github.com/artemMprokhorov/<repo>/commit/<sha>` shows the content and the author /
   committer dates. These dates are set by the author's machine and are not proof on their own.
2. **GitHub push log (server time)** — https://github.com/artemMprokhorov/hippograph/activity, …/hippograph-pro/activity,
   …/semantic-memory-mcp/activity, or `gh api repos/artemMprokhorov/<repo>/activity`. Each push lists `before` /
   `after` hashes and a server timestamp. A push of `after = X` proves that commit X and all its ancestors were on
   GitHub at that time.
3. **GH Archive** — hourly files `https://data.gharchive.org/YYYY-MM-DD-H.json.gz` record public GitHub events with
   `"public": true`. Example:
   `curl -s https://data.gharchive.org/2026-02-18-23.json.gz | gunzip | grep artemMprokhorov/`. Some hours have no
   captured events for these repos; in that case we cite the first captured push whose history contains the commit.
4. **Wayback Machine** — list captures with
   `http://web.archive.org/cdx/search/cdx?url=github.com/artemMprokhorov/<repo>&matchType=prefix&fl=timestamp,original`.
   The captured page shows the commit count and the head commit (`currentOid` in the page source); everything that is
   an ancestor of that head was public at the capture time.
5. **Forks** — the forks above are copies held by third parties; a commit present in a fork was public no later than
   the fork's last push.
6. **Software Heritage** — keeps the commits (with their dates) under permanent identifiers, e.g.
   `swh:1:rev:<sha>` resolves at `https://archive.softwareheritage.org/swh:1:rev:<sha>`. The archive visit is dated
   2026-10-31, so it preserves content rather than proving an early date.
7. **Rewritten history (hippograph-pro).** Force-pushes on 2026-03-14T12:51Z (rewriting commits of 2026-02-26…03-14),
   2026-04-07T17:46Z and 2026-04-10T03:36Z (rewriting commits from 2026-03-24T22:36Z on) edited configuration values
   from old commits. The content of the mechanisms did not change. For every such commit we give the hash that is on
   `main` today **and** the hash under which it was first pushed ("pushed as …"); GitHub still serves both. The push
   log and GH Archive refer to the first-pushed hashes; the forks hold whatever was on `main` when they were copied
   (current hashes for 2026-02-26…03-14, first-pushed hashes for 2026-03-24…04-05). Two further force-pushes
   (2026-03-10, 2026-03-23) only dropped tip commits. A first-pushed hash is shown without a link where that
   original commit still contains a configuration value. In hippograph and semantic-memory-mcp all cited hashes
   are on `main` today (their force-pushes on 2026-01-21, 2026-02-04 and 2026-02-20 replaced only unreferenced tip
   commits).

## Citing

Cite the source repository, the commit hash and the date, plus the Software Heritage origin. Each document ends with a
ready citation. General form:

> Prokhorov, Artem. *HippoGraph Pro*. GitHub, https://github.com/artemMprokhorov/hippograph-pro, commit `<sha>`
> (YYYY-MM-DD). MIT License. Software Heritage: https://archive.softwareheritage.org/browse/origin/?origin_url=https://github.com/artemMprokhorov/hippograph-pro

## License of these records

The text of these records is licensed under [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/). The source
repositories they describe are under the MIT License.
