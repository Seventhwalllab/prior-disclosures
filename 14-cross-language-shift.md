# Cross-language search in a personal memory, learned from the person's own mixed-language records: a shift along directions taken from the two halves of the same record, checked on held-out halves before it is used

**First public disclosure:** 2026-11-05, this document (full description). Earlier public trace: the manual page
`docs/search.md` in the public releases of [github.com/Seventhwalllab/anairyspirit](https://github.com/Seventhwalllab/anairyspirit)
from 2026-09-29, which describes the effect for users; see [Earlier public traces](#earlier-public-traces).

**Status of disclosure:** written description (this document). The memory service is distributed in binary form; its
source code is not public.

**Author:** Artem Prokhorov

Documents 01 to 10 record earlier publications. This one, like 11 and 12, is the publication itself, and its date is
the date this file was first published in this repository. All times are UTC; see [README.md](README.md#how-to-verify-dates).

## What it is

A personal memory for an AI agent stores records as vectors from one multilingual embedding model. People who live in
two languages write in both, and the model does not bring the two languages equally close: a question in one language
often misses a record in the other, even when they say the same thing. The usual remedies are a translation service, a
second model, or training on translated pairs.

This design uses none of them. It learns the difference between two writing systems from the person's own records
that already mix them. Such a record, split by writing system, gives two pieces about the same topic in two scripts.
The difference of their vectors is mostly the difference of script, because the topic is the same on both sides. A few
leading directions of these differences, together with the average position of each script, let the system move a
vector from one script's region to the other's at search time. Whether to move anything at all is decided by a check
on the same kind of pairs, held out from the ones that built the directions. If the move does not help on the person's
own data, the search stays as it was.

## How it works

### 0. Terms

* **Record:** a stored memory with its text and its vector.
* **Script tag:** the writing system of a word or a text (Latin, Cyrillic, Greek, Arabic, Hebrew, Hangul, Kana, Han),
  found from Unicode ranges of its letters.
* **Mixed record:** a record whose text has words in two or more scripts.
* **Pair:** the two vectors of the two script parts of one mixed record.
* **Centre:** the mean of the stored vectors of the records whose majority script is a given script.
* **Basis:** a few unit directions along which pairs differ most.

### 1. Script tags

Each word gets the majority script of its letters. A text gets the majority script of its letters when it has enough
letters to judge; otherwise it has no tag. No dictionary, no model and no training are involved. Languages that share
a script (for example English and German) are not separated by this step; see [Variants](#variants-and-generalizations).
Tags are opaque labels: nothing downstream depends on which script a tag names.

### 2. Pairs from the person's own records

The system walks the stored records. A record whose words fall into two scripts is split into its script parts. Parts
that are too short are ignored. The two parts are encoded again, separately, with the same embedding model, and the
two vectors form a pair. Pairs come only from text the person wrote or kept; nothing is translated to make them.

### 3. Centres and directions

For each script with enough records, the centre is the mean of the stored vectors of the records of that script.
Mixed records are kept in the centres on purpose.

For a route from script A to script B, each pair gives a difference vector (the A part minus the B part), normalised to
unit length. The leading directions of these differences are found by power iteration with deflation from a fixed
start, so the result is the same on every run. These directions are the basis of the route. With too few pairs there
is no basis and no shift for that route.

### 4. The move

To move a vector x from script A towards script B, its coordinate along each basis direction b is replaced by the
coordinate of B's centre along b:

x' = x + Σ (⟨c_B, b⟩ − ⟨x, b⟩) · b

Along the basis, x now sits where script B sits on average; in every other direction it is unchanged. Topic,
which lives in the other directions, is kept.

At search time the query's script is found (§1) and routed to another script that has a centre. Three forms are used:

* **Query side.** The query q is moved into the target script, q'. A record's score is the larger of cos(q, v) and
  cos(q', v).
* **Record side.** Every record of another script is moved into the query's script, each with the basis of its own
  route, v'. The score is the larger of cos(q, v) and cos(q, v').
* **Both.** The larger of the two scores above.

The score is always a maximum with the unmoved score, so the move can raise a record and never lowers one. Only the
first, vector stage of the search pipeline changes. Later stages (lexical search, the graph walk, fusion, a
cross-encoder reranker) see the shift only through which candidates come in and with what first score.

### 5. The self-check

For each route the system decides whether to shift, which form to use and how many basis directions to keep, on the
person's own data:

1. The pairs of the route are split. Most build the basis; every few pairs are held out.
2. The task: one part of a mixed record is the question, the other part of the same record is the answer, and a pool
   of other stored records are the distractors. The record the pair came from is excluded.
3. Candidates: no shift, the query side and both sides, each with several numbers of basis directions.
4. Each candidate is scored by how high the answer ranks (mean reciprocal rank).
5. The best candidate is kept only if it also beats "no shift" on the held-out pairs. Otherwise the route is not
   shifted.

The verdict is kept per embedding model, source script and target script.

### 6. When it is computed

Centres, bases and verdicts are computed when the service starts, again after the search pipeline is reloaded, and
again when the memory has grown enough since the last computation (both in absolute number of records and as a share
of the memory). A memory that started empty therefore gains the shift once it has enough mixed records, without a
restart. All of it is keyed by the embedding model, so a change of model rebuilds it. The cost is a few vector means,
a small eigen-problem per route and the self-check; nothing is stored in the records.

## Earlier public traces

* `docs/search.md` ("Asking in another language") in the public releases from 2026-09-29: a question in one language
  finds notes in another, with no translation service and no second model.
* The release notes of v0.1.445 (2026-11-04): cross-language search starts without a restart once a memory filled in
  two languages has grown.

The mechanism of §1 to §6 was not published before this document.

## Variants and generalizations

* **Languages that share a script.** Any language identifier can replace the script tag of §1 (a character n-gram
  classifier, a stop-word table, a small language-identification model), so the same steps separate English from
  German or Spanish from Portuguese.
* **Where pairs come from.** The whole record rather than one stored piece of it; quotations, code comments, titles
  and their translations kept by the person; any text the person keeps in two languages side by side. Translated pairs
  can be added where they exist; the steps do not change.
* **How the move is made.** A fraction of the replacement instead of all of it; removal of the coordinates instead of
  replacement; a rotation fitted on the pairs (orthogonal Procrustes) instead of a coordinate replacement; principal
  components or singular vectors instead of power iteration.
* **Routes.** One target script per query, every script that has a centre, or the script with the most records.
* **Edges instead of a moved copy.** The record-side move of §4 can be used once, offline, to link records across
  scripts: a record of script A and a record of script B are linked when each is among the other's nearest neighbours
  in the moved space. The edges are calibrated on the halves of mixed records (the two halves must find each other).
  The search then reaches across languages through the graph walk, and no moved copy of the vectors is kept in memory.
* **Carrying the gain to later stages.** The gain of the moved score over the unmoved one can be passed to the fusion
  stage or added after the reranker to the final score, with one formula for every query and no branch by language,
  so a cross-language answer found by the vector stage is not lost later in the pipeline.
* **When to compute.** During offline consolidation of the memory (a sleep phase), with a flag set when records of a
  new script arrive.
* **Any multilingual embedding model**, and any memory made of vectors: a notes app, a document store, a chat history,
  a code base with comments in two languages.

## Advantages

* No translation service, no second model and no training: a vector mean, a small eigen-problem and a check.
* It learns from the person's own way of mixing languages, so it fits the languages and topics the person actually
  writes in.
* It is never worse than no shift on the person's own held-out data: the self-check either finds a gain or leaves
  search as it was, and the maximum of §4 never lowers a score.
* It grows with the memory: an empty memory starts without it and gains it when the data appears.
* It helps most where the embedding model bridges two languages poorly (in our measurements, English and Chinese) and
  stays out of the way where the model already bridges them well.

## Prior work and our contribution

Moving a vector by the difference of language means (Pires, Schlinger and Garrette, 2019), centring by language
(Libovický, Rosa and Fraser, 2020), removing the first principal components (all-but-the-top; Google US 11,769,011 B2,
claim 16), projecting language out by null-space projection (LSAR; Xie et al., EMNLP 2022) and subtracting the language
mean with scaling or rotation (Microsoft US 2025/0111173 A1) are known. Recent work learns a language direction or
subspace from translation pairs or parallel data and removes it or steers the query with it (SHIFT, arXiv 2606.18801;
PRSR, arXiv 2610.23807; arXiv 2610.18526). Natural code-switching has been used as a cross-language signal (CoSwitch,
MRL 2023). Cross-language edges in a retrieval graph (MaGiX, Findings of EMNLP 2025) and mutual nearest neighbours
across languages (Artetxe and Schwenk, 2019) are known.

What this document adds is the combination inside one person's memory: pairs mined from the person's own mixed-language
records by splitting each record by script; a basis from the differences of the two halves of the same record, where
the topic cancels; a move that replaces the coordinate along that basis with the target script's centre and keeps the
maximum with the unmoved score; a self-check on held-out halves of the same records that decides, per route and per
embedding model, whether to shift at all and how; recomputation as the memory grows; and the variant that turns the
move into cross-language edges calibrated on the same halves.

## Reference implementation

An Airy Spirit (github.com/Seventhwalllab/anairyspirit), the first, vector stage of its search pipeline: on by default
in internal builds since 2026-08-21, in the public releases, and recomputed without a restart since v0.1.445
(2026-11-04). The edge variant and the later-stage variant are described here and are not part of the product.

## Citation

> Prokhorov, Artem. *Cross-language search in a personal memory, learned from the person's own mixed-language records:
> a shift along directions taken from the two halves of the same record, checked on held-out halves before it is used.*
> Prior disclosure 14, github.com/Seventhwalllab/prior-disclosures (2026-11-05). CC BY 4.0.
