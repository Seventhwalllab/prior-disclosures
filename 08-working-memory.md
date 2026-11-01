# Working memory inside the memory graph: overwrite note, then a time-linked journal

**First public disclosure:** 2026-03-17 — `update_working_memory` tool, hippograph-pro [`7f45070`](https://github.com/artemMprokhorov/hippograph-pro/commit/7f45070) (journal mode 2026-04-07, [`82f7b50`](https://github.com/artemMprokhorov/hippograph-pro/commit/82f7b50); short-term → long-term promotion as a roadmap idea 2026-01-26, hippograph [`0ccd4f2`](https://github.com/artemMprokhorov/hippograph/commit/0ccd4f2))
**Status of disclosure:** working code (MIT) for the working-memory note and journal; short-term → long-term promotion only as an idea plus an explicit markup convention

All times are UTC; see [README.md](README.md#how-to-verify-dates).

## What it is

The AI agent keeps a short "where we are now" record in the same memory graph as its long-term notes, as a note of
category `working-memory`. The agent itself writes it as part of its normal reasoning, after significant events, and
reads it at the start of the next session. First it was one note overwritten on every update; later every update became
a new note linked to the previous one by a `TEMPORAL_AFTER` edge, so the working memory turned into a chronological
journal, and the session start returns the latest entry plus a short history.

## How it worked as disclosed

**1. Idea: short-term vs long-term tiers** — hippograph [`0ccd4f2`](https://github.com/artemMprokhorov/hippograph/commit/0ccd4f2),
[`ROADMAP.md`](https://github.com/artemMprokhorov/hippograph/blob/0ccd4f2/ROADMAP.md) ("Memory Hierarchy & Lifecycle", enterprise phase): session-based
short-term / working memory with 24 h retention, a cache of recently accessed nodes, and "Access-based Promotion —
Frequently used short-term → long-term". Repeated in [`2b4e5b8`](https://github.com/artemMprokhorov/hippograph/commit/2b4e5b8) (2026-02-12). No code.

**2. Working-memory note (overwrite)** — hippograph-pro [`7f45070`](https://github.com/artemMprokhorov/hippograph-pro/commit/7f45070), files
[`src/mcp_sse_handler.py`](https://github.com/artemMprokhorov/hippograph-pro/blob/7f45070/src/mcp_sse_handler.py) (`tool_update_working_memory`),
[`AGENT_PROMPT.md`](https://github.com/artemMprokhorov/hippograph-pro/blob/7f45070/AGENT_PROMPT.md); tests [`f307352`](https://github.com/artemMprokhorov/hippograph-pro/commit/f307352).
- MCP tool `update_working_memory(content, session_id)`: finds the newest note of category `working-memory`; if it
  exists, replaces its content, re-embeds it and updates the vector and BM25 indexes; otherwise creates it;
- the agent prompt makes the update an "inference trigger": call it after a decision, a finished task, a change of
  direction or important new context — "Keep it concise (5-10 lines)", and not on every exchange;
- the same commit adds a `tags` field written by the AI at save time and indexed by BM25 together with the content.
- From [`e745cca`](https://github.com/artemMprokhorov/hippograph-pro/commit/e745cca) (pushed as [`1b03c95`](https://github.com/artemMprokhorov/hippograph-pro/commit/1b03c95), 2026-03-28) the session-start script in
  `AGENT_PROMPT.md` reads it explicitly: `search_memory("working memory", category="working-memory")`.

**3. Working memory as a journal** — [`82f7b50`](https://github.com/artemMprokhorov/hippograph-pro/commit/82f7b50) (pushed as [`fe3af56`](https://github.com/artemMprokhorov/hippograph-pro/commit/fe3af56), re-pushed as
[`873b4d7`](https://github.com/artemMprokhorov/hippograph-pro/commit/873b4d7)), file [`src/mcp_sse_handler.py`](https://github.com/artemMprokhorov/hippograph-pro/blob/82f7b50/src/mcp_sse_handler.py); prompt note
[`063d5f4`](https://github.com/artemMprokhorov/hippograph-pro/commit/063d5f4) (pushed as [`afa065b`](https://github.com/artemMprokhorov/hippograph-pro/commit/afa065b)).
- every call inserts a new `working-memory` note through the normal write path (embedding, entities, links);
- if a previous entry exists, an edge `TEMPORAL_AFTER` (weight 0.9) is added from the new entry to it;
- "current" working memory = newest entry (`ORDER BY timestamp DESC LIMIT 1`); older entries stay searchable.
- Prompt text: "update_working_memory() creates a NEW node each call (temporal journal)."

**4. Session-context service and explicit promotion to long-term memory** — [`47ce223`](https://github.com/artemMprokhorov/hippograph-pro/commit/47ce223) (pushed as
[`e13f263`](https://github.com/artemMprokhorov/hippograph-pro/commit/e13f263)), file [`session_context_mcp.py`](https://github.com/artemMprokhorov/hippograph-pro/blob/47ce223/session_context_mcp.py); README
[`cd535e9`](https://github.com/artemMprokhorov/hippograph-pro/commit/cd535e9), [`CHANGELOG.md`](https://github.com/artemMprokhorov/hippograph-pro/blob/97690be/CHANGELOG.md) ([`97690be`](https://github.com/artemMprokhorov/hippograph-pro/commit/97690be)).
- a small separate MCP server: `get_session_context()` returns the newest journal entry as the working memory plus
  the two previous entries (timestamp + first 150 characters) as history; `update_session_context(content)` appends a
  journal entry through the memory API;
- the tool description carries a standing instruction: while answering, the model marks content worth long-term
  memory as `[REMEMBER: importance=…, intensity=…, category=…, tags=…] … [/REMEMBER]`; `extract_remember_blocks(text)`
  turns those blocks into ready parameters for a normal long-term write; "Temporary session context" goes to the
  working memory instead. This is the disclosed short-term → long-term path: an explicit, model-made decision, not an
  automatic promotion.

## Disclosure timeline

| Date (UTC) | Repo | Commit | What was added | Evidence |
|---|---|---|---|---|
| 2026-01-26 | hippograph | [`0ccd4f2`](https://github.com/artemMprokhorov/hippograph/commit/0ccd4f2) | idea: short-term / long-term tiers, access-based promotion | push 21:22:14Z ([activity](https://github.com/artemMprokhorov/hippograph/activity)); GH Archive: no event for that hour, contained in [`bfc9310`](https://github.com/artemMprokhorov/hippograph/commit/bfc9310) ([2026-02-04-18](https://data.gharchive.org/2026-02-04-18.json.gz), `public: true`) |
| 2026-02-12 | hippograph | [`2b4e5b8`](https://github.com/artemMprokhorov/hippograph/commit/2b4e5b8) | idea restated (24 h TTL, promotion) | push 16:24:58Z; GH Archive [2026-02-12-16](https://data.gharchive.org/2026-02-12-16.json.gz), `public: true` |
| 2026-03-17 | hippograph-pro | [`7f45070`](https://github.com/artemMprokhorov/hippograph-pro/commit/7f45070), [`f307352`](https://github.com/artemMprokhorov/hippograph-pro/commit/f307352) | `update_working_memory` (overwrite), inference trigger in prompt, tags | push 18:46:07Z ([activity](https://github.com/artemMprokhorov/hippograph-pro/activity)); GH Archive has no event for that hour, first captured descendant [`b8e96f7`](https://github.com/artemMprokhorov/hippograph-pro/commit/b8e96f7) ([2026-03-21-2](https://data.gharchive.org/2026-03-21-2.json.gz)); forks [vaavaa](https://github.com/vaavaa/hippograph-pro/commit/7f45070), [iflow-mcp](https://github.com/iflow-mcp/artemmprokhorov-hippograph-pro/commit/7f45070) |
| 2026-03-28 | hippograph-pro | [`e745cca`](https://github.com/artemMprokhorov/hippograph-pro/commit/e745cca) (pushed as [`1b03c95`](https://github.com/artemMprokhorov/hippograph-pro/commit/1b03c95)) | session-start read of working memory | push 02:53:40Z; fork [vaavaa](https://github.com/vaavaa/hippograph-pro/commit/1b03c95); Wayback [2026-04-06 `AGENT_PROMPT.md`](https://web.archive.org/web/20260406082821/https://github.com/artemMprokhorov/hippograph-pro/blob/main/AGENT_PROMPT.md) |
| 2026-04-07 | hippograph-pro | [`82f7b50`](https://github.com/artemMprokhorov/hippograph-pro/commit/82f7b50) (pushed as [`fe3af56`](https://github.com/artemMprokhorov/hippograph-pro/commit/fe3af56) / [`873b4d7`](https://github.com/artemMprokhorov/hippograph-pro/commit/873b4d7)) | journal: INSERT + `TEMPORAL_AFTER` | push 17:33:13Z, re-push 17:46:10Z; GH Archive [2026-04-07-17](https://data.gharchive.org/2026-04-07-17.json.gz): push 17:56:08Z with `before = 873b4d7` |
| 2026-04-07 | hippograph-pro | [`063d5f4`](https://github.com/artemMprokhorov/hippograph-pro/commit/063d5f4) (pushed as [`afa065b`](https://github.com/artemMprokhorov/hippograph-pro/commit/afa065b)) | prompt: journal semantics | push 17:56:08Z; GH Archive same file, `public: true` |
| 2026-04-09 | hippograph-pro | [`47ce223`](https://github.com/artemMprokhorov/hippograph-pro/commit/47ce223) (pushed as [`e13f263`](https://github.com/artemMprokhorov/hippograph-pro/commit/e13f263)) | session-context MCP, last-3 entries, `[REMEMBER]` blocks | push 01:04:16Z; GH Archive [2026-04-09-1](https://data.gharchive.org/2026-04-09-1.json.gz), `public: true` |
| 2026-04-09 | hippograph-pro | [`97690be`](https://github.com/artemMprokhorov/hippograph-pro/commit/97690be) (pushed as [`067acee`](https://github.com/artemMprokhorov/hippograph-pro/commit/067acee)) | CHANGELOG: "Working Memory Journal — … INSERT instead of overwrite" | push 01:31:15Z; GH Archive [2026-04-09-1](https://data.gharchive.org/2026-04-09-1.json.gz), `public: true` |

## Variants disclosed

- **Overwrite** (one current note, 2026-03-17) and **append-only journal** (chain of notes, 2026-04-07).
- **Reading**: by category search at session start (prompt), or by a dedicated tool returning newest entry + history.
- **Exclusion from analytics**: working-memory notes are left out of the embedding pools used by sleep-time topic
  clustering ([`a22cc4b`](https://github.com/artemMprokhorov/hippograph-pro/commit/a22cc4b), 2026-04-08), so the journal does not distort topics.
- **Short-term → long-term**: as an idea, access-based automatic promotion (2026-01-26); as working text, explicit
  `[REMEMBER …]` markup by the model (2026-04-09). Nothing else in the old repos implements promotion.

## Prior work and our contribution

Builds on: multi-store and working-memory models (Atkinson & Shiffrin 1968; Baddeley & Hitch 1974); an editable
working context for LLM agents (MemGPT/Letta, 2023); short-term to long-term promotion by access or "heat"
(MemoryOS, 2025; CerebroCortex, 2026-02-05).
No distinct contribution claimed. The working-memory journal (each update is a new node chained by a temporal edge;
2026-04-07, hippograph-pro 82f7b50) is disclosed as implementation.

## Citation

Prokhorov, Artem. *HippoGraph Pro*, `update_working_memory` in `src/mcp_sse_handler.py`. GitHub,
https://github.com/artemMprokhorov/hippograph-pro, commit `7f45070d3e49ae1e82bf9efc4729d6a2b184d7db` (2026-03-17); journal
mode `82f7b5095d1d29c5d520a2c1c75091fc834a0b78` (first pushed as `fe3af5615633dc5ac400c2e2d4d34a29cc76a5a4`, 2026-04-07);
`session_context_mcp.py` `47ce22331bdde6f7b6455d85df1326c58e332185` (first pushed as
`e13f263248f59c82d4ffff2d4f5fb81de41e49a6`, 2026-04-09). MIT License. Software Heritage:
https://archive.softwareheritage.org/browse/origin/?origin_url=https://github.com/artemMprokhorov/hippograph-pro
