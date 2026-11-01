# Intentions (prospective memory) as non-decaying nodes of the memory graph

**First public disclosure:** 2026-04-08, working code, hippograph-pro [`b05871f`](https://github.com/artemMprokhorov/hippograph-pro/commit/b05871f) (pushed as [`3a2938c`](https://github.com/artemMprokhorov/hippograph-pro/commit/3a2938c))
**Status of disclosure:** working code (MIT), with unit tests

All times are UTC; see [README.md](README.md#how-to-verify-dates).

## What it is

Besides facts about the past, the memory holds plans: things the agent intends to do. An intention is stored as an
ordinary note of category `prospective` with a status tag (`pending`, later `done` or `cancelled`) and an optional due
date. While it is pending it is kept at the highest importance and so does not fade; once it is completed or cancelled
it drops to low importance and ages like any other note. The sleep job keeps statuses and importance consistent and
marks overdue intentions.

## How it worked as disclosed

**Code**: [`b05871f`](https://github.com/artemMprokhorov/hippograph-pro/commit/b05871f), files
[`src/mcp_sse_handler.py`](https://github.com/artemMprokhorov/hippograph-pro/blob/b05871f/src/mcp_sse_handler.py), [`src/sleep_compute.py`](https://github.com/artemMprokhorov/hippograph-pro/blob/b05871f/src/sleep_compute.py),
[`src/graph_engine.py`](https://github.com/artemMprokhorov/hippograph-pro/blob/b05871f/src/graph_engine.py); tests
[`tests/test_prospective_memory.py`](https://github.com/artemMprokhorov/hippograph-pro/blob/6d86974/tests/test_prospective_memory.py) ([`6d86974`](https://github.com/artemMprokhorov/hippograph-pro/commit/6d86974)).

1. **`add_intention(content, due?, tags?)`** (MCP tool and CLI `--add-intention`, `--due YYYY-MM-DD`): writes a note
   through the normal write path (embedding, entities, links) with content `[INTENTION] <text>`, category
   `prospective`, importance `critical`, tags `pending [due:YYYY-MM-DD] [extra tags]`.
2. **`complete_intention(id, status = done | cancelled, note?)`** (MCP tool and CLI `--complete-intention`): replaces
   `pending` by the status in the tags, optionally appends `[DONE] …` / `[CANCELLED] …` to the content, and sets
   importance `low`, "After completion, the note switches to low importance and can decay normally."
3. **Sleep step "3b: Prospective Memory"** (`step_prospective_memory`): for every `prospective` note, if `pending`,
   force importance `critical`; if a `due:` tag is in the past, prefix the content with `[OVERDUE]` once; if `done` or
   `cancelled`, set importance `low`. Supports dry run. CLI `--list-intentions` lists them.
4. **Why pending intentions do not fade**: importance `critical` places the category under the anchor protection
   described in [07-anchors-and-importance.md](07-anchors-and-importance.md) (categories holding critical notes are
   auto-protected from stale-edge decay); the CHANGELOG states it as "pending intentions are auto-protected from stale
   edge decay".
5. **Retrieval**: pending intentions receive a fixed boost in retrieval (`PROSPECTIVE_BOOST`, README entry
   [`6d86974`](https://github.com/artemMprokhorov/hippograph-pro/commit/6d86974)).

## Disclosure timeline

| Date (UTC) | Repo | Commit | What was added | Evidence |
|---|---|---|---|---|
| 2026-04-08 | hippograph-pro | [`b05871f`](https://github.com/artemMprokhorov/hippograph-pro/commit/b05871f) (pushed as [`3a2938c`](https://github.com/artemMprokhorov/hippograph-pro/commit/3a2938c)) | `add_intention` / `complete_intention`, sleep step, CLI | push 23:27:11Z (activity); GH Archive has no event for that hour; first captured descendant: push 2026-04-09T01:04:17Z with `before = b47c8bf` ([2026-04-09-1](https://data.gharchive.org/2026-04-09-1.json.gz)), and `b47c8bf` descends from `3a2938c` ([compare](https://github.com/artemMprokhorov/hippograph-pro/compare/3a2938c...b47c8bf)) |
| 2026-04-08 | hippograph-pro | [`6d86974`](https://github.com/artemMprokhorov/hippograph-pro/commit/6d86974) (pushed as [`a00cf48`](https://github.com/artemMprokhorov/hippograph-pro/commit/a00cf48)) | README entry + 7 unit tests | push 23:37:51Z (activity) |
| 2026-04-09 | hippograph-pro | [`97690be`](https://github.com/artemMprokhorov/hippograph-pro/commit/97690be) (pushed as [`067acee`](https://github.com/artemMprokhorov/hippograph-pro/commit/067acee)) | CHANGELOG entry "Prospective Memory" | push 01:31:15Z; GH Archive [2026-04-09-1](https://data.gharchive.org/2026-04-09-1.json.gz), `public: true` |

No fork or Wayback copy covers 2026-04-07…04-10 (the forks stop at 2026-04-05, the last Wayback capture is 2026-04-06).
For these dates the evidence is GitHub's push log, GH Archive and Software Heritage (which also holds the first-pushed
revision, e.g. [`swh:1:rev:3a2938c4…`](https://archive.softwareheritage.org/swh:1:rev:3a2938c4d5b657dd5d351005eb54c5466199fdc9)).

## Variants disclosed

- Statuses: `pending`, `done`, `cancelled`; optional due date with automatic `[OVERDUE]` marking.
- Two interfaces to the same store: MCP tools for the agent, command-line flags for the user.
- Completion either keeps the outcome text in the note (`note=` argument) or only changes the status.

## Prior work and our contribution

Builds on: prospective memory and the persistence of unfinished intentions (Zeigarnik 1927; Goschke & Kuhl 1993;
Einstein & McDaniel 1990); prospective "TODO" memories with high salience and slow decay in an agent memory server
(CerebroCortex, 2026-02-05).
Our contribution (first disclosed 2026-04-08, hippograph-pro b05871f): pending intentions are ordinary memory nodes
kept at critical (non-decaying) importance while open; at retrieval each one receives a fixed additive bonus to its
initial activation, and one outside the candidate set is admitted at a near-zero similarity floor (0.05), so it
surfaces for almost any related query until it is marked done or cancelled, after which it decays normally.

## Citation

Prokhorov, Artem. *HippoGraph Pro*, "Prospective Memory" (`add_intention`, `complete_intention`,
`step_prospective_memory`). GitHub, https://github.com/artemMprokhorov/hippograph-pro, commit
`b05871fad4f338912c98a691007504fc8abba8ab` (first pushed 2026-04-08T23:27Z as `3a2938c4d5b657dd5d351005eb54c5466199fdc9`). MIT License. Software Heritage:
https://archive.softwareheritage.org/browse/origin/?origin_url=https://github.com/artemMprokhorov/hippograph-pro
