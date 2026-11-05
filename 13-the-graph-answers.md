# The graph answers: a personal memory that marks every outdated record with its newest replacement and answers from its own edges and times, with no language model writing the answer

**First public disclosure:** 2026-11-05, this document (full description). Earlier public traces: typed
"supersedes" edges between records, see [Earlier public traces](#earlier-public-traces).

**Status of disclosure:** written description (this document). The memory service is distributed in binary form; its
source code is not public.

**Author:** Artem Prokhorov

Documents 01 to 10 record earlier publications. This one, like 11, 12 and 14, is the publication itself, and its date
is the date this file was first published in this repository. All times are UTC; see [README.md](README.md#how-to-verify-dates).

## What it is

A personal memory for an AI agent keeps records as nodes of a graph, with typed edges and times. A person writes the
same thing several times as it changes: a version, a status, a decision taken again. A search then returns the old
record and the new one side by side, often the old one first, and the agent answers from whichever it reads.

In this design the graph itself says which record is newer, and it says it in two ways. First, every record a search
returns that has a later record about the same thing carries the id and the date of the newest one. The search result
is not reordered, filtered or replaced: the pointer travels with the record. Second, the agent can ask the graph,
about a given record, for one of three answers built from the graph's own edges and times: what is current now and
what it replaced; how the thing changed over time, including the same thing happening again; and the different
sides held about it. Each answer is a small subgraph of real records with their dates. No language model writes it.
When the graph holds nothing that fits, the answer says so.

## How it works

### 0. Terms

* **Record:** a node with its text, a creation time, optionally an event time, a category and tags.
* **Freshness pair:** two records about the same thing where the later one replaces the earlier. Found during offline
  maintenance of the memory (its sleep) from the closeness of the two records, their times and their subject, not
  from negation words; stored as a directed edge from the later record to the earlier.
* **Chain:** records joined by freshness pairs, ordered in time. Its **end** is the latest record of the chain.
* **Recurrence edge:** two records about the same thing that happened again (a mistake, a decision, a question). One
  edge per pair of events; the edge carries no counter.
* **Side:** one of several records about one subject that differ in what they say about it.

### 1. The pointer on every search result

1. The search runs as usual: same order, same weights, same number of results.
2. During sleep, for every chain, the end is found and remembered with the pair.
3. Every returned record that has a later record by a freshness pair carries "later record: #id (date)", where #id and
   the date are those of the end of its chain.
4. So an outdated record is never returned without the id and date of its newest replacement, while the result
   itself stays exactly what the search produced. Which record to believe is left to the reader, who now has what it
   needs to decide.

### 2. Three answers, each asked for a given record

The agent chooses which answer to ask for, as it chooses any tool. Each answer is a separate operation that takes the
id of a record (or of a subject), not the text of a question. The edge types each operation follows are fixed by the
operation.

**What is current now.** From record #id, follow freshness pairs forward to the end of the chain. The answer is the
chain in time order: each record with its text, date and id, marked "earlier record" or "later record", ending with
the current one. When two chains about the same thing end in records of the same day, both ends are given as current
rather than one being chosen.

**A line in time.** From record #id or a subject, and optionally from a given date, collect the records joined by
freshness pairs and by recurrence edges, order them by event time when there is one and by creation time otherwise.
The answer is the line: "was A (date), then B (date), then C (date)", with the recurrences on the same line: "the same
again (date), and again (date); not since (date)". The number of changes in a period is read off the line as a measure
of drift: where what the memory knows about the subject is moving, and where it has settled. A sleep can list in its
own report the records that got a later record during that sleep.

**Different sides.** From the records found for a subject, or the records of the subject, select representatives one
at a time: each next one is the record most relevant to the subject and least similar to the representatives already
chosen (maximal marginal relevance). Nothing is clustered at question time. The answer is the set of sides, each with
its date, so that "in July A was held, in October B" can be read from it.

### 3. What an answer is made of

Every answer is a small subgraph: the records chosen, the typed edges between them, their dates, ids and own text. It
is handed to the caller in that form, so a person or an agent can follow it back to the records. Nothing in it is
generated text, and nothing of the answer is written back into the graph. When nothing in the graph fits, the answer
says that the graph does not know, instead of composing something plausible.

## Earlier public traces

* Typed edges between records, including "supersedes", in the public releases of
  [github.com/Seventhwalllab/anairyspirit](https://github.com/Seventhwalllab/anairyspirit), and the README line that
  edges carry similarity, contradiction and supersession.
* A "supersedes" edge built during sleep in the author's earlier public repository (2026-03-22); see document 04.

The pointer of §1, the three answers of §2 and the form of §3 were not published before this document.

## Variants and generalizations

* The pointer can carry only the id, the id and date, or also a short excerpt of the newest record; it can point to
  the end of the chain or to the next record only.
* Freshness pairs and recurrence edges can come from sleep, from the agent's own explicit writes ("this replaces
  #id"), or from both.
* More answers of the same form: who was involved, what depends on a record, what was decided about a subject; each is
  a fixed walk over fixed edge types from a given record.
* The subgraph can be rendered as a list, a timeline, a table or a picture, for an agent, a person or another program.
* The same pointer and answers apply to any store of dated records: notes, tickets, documents, chat history.

## Advantages

* An outdated record never reaches the reader without what replaced it, and the search result itself is untouched.
* "What is current" is answered by the graph's structure, not by whichever of several similar records ranks first.
* Change over time and repetition become visible as such, which a ranked list of similar records hides.
* The answer cannot contain anything the graph does not hold: there is no generation step that could add it.
* It runs on a small machine with no language model at all.

## Prior work and our contribution

Question answering over knowledge graphs by templates and walks is long established, as are temporal knowledge graphs
with validity intervals, "superseded by" pointers in legal citators and document standards, and maximal marginal
relevance. Agent memories that keep a current value and the values it displaced are described in Zep (2025), Engram
(arXiv 2606.09900) and MoM / P-Mem (arXiv 2609.25054); they filter the old value out, replace it with the current one,
or hand both to a language model that writes the answer.

What this document adds, for a personal memory: a pointer from every returned outdated record to the end of its chain,
with the search result itself unchanged and the chain read separately; three answers, each a separate operation from
a given record over fixed edge types (current state, line in time with recurrences and drift, sides), built from edges
the memory found in its own sleep; and the form of every answer as a subgraph of real records with dates, with "the
graph does not know" when nothing fits, and no generated text.

## Reference implementation

An Airy Spirit (github.com/Seventhwalllab/anairyspirit): typed edges, times and sleep are in the public releases; the
pointer of §1 is planned for a coming release; the answers of §2 are described here.

## Citation

> Prokhorov, Artem. *The graph answers: a personal memory that marks every outdated record with its newest replacement
> and answers from its own edges and times, with no language model writing the answer.* Prior disclosure 13,
> github.com/Seventhwalllab/prior-disclosures (2026-11-05). CC BY 4.0.
