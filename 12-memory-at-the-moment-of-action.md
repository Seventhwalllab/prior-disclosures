# Memory at the moment of action: agent-host hooks that bring a memory graph to session start, to each prompt, to each answer and to each tool call, with a gate that names the readers of an edit before it is made

**First public disclosure:** 2026-11-02, this document (full description). Earlier public trace: the manual page `docs/hooks.md` shipped inside every public release of
[github.com/Seventhwalllab/anairyspirit](https://github.com/Seventhwalllab/anairyspirit) from v0.1.359 (2026-09-29),
which describes the first three hook points of §1 for users; see [Earlier public traces](#earlier-public-traces)
**Status of disclosure:** written description (this document). The memory service is distributed in binary form; its
source code is not public. The gate of §5.1 is a tool of the author's development setup and is not distributed
**Author:** Artem Prokhorov

Documents 01 to 10 record earlier publications. This one, like 11, is the publication itself, and its date is the date
this file was first published in this repository. All times are UTC; see [README.md](README.md#how-to-verify-dates).

## What it is

An AI agent runs inside a host program (a coding assistant, a chat client, an IDE agent). The host offers hooks: small
commands it runs at fixed points of its own loop. A personal memory graph serves the agent through four such points,
each with its own job: who the agent is and what it was doing (session start), what it already knows about the
current request (before each prompt), what is worth keeping from what it just said (after each answer), and what is
about to change and what tends to go wrong with a step like this (before each tool call).

The fourth point has two parts. A gate examines an edit before it is applied, computes who reads the names the edit
changes, asks the questions that past defects of the same shape have earned, and lets the edit through only when the
same call is issued again with an explicit acknowledgement. And a recall step searches the memory graph with the tool
call itself, so the lessons and preferences stored there come back at the moment of the action they are about,
instead of only when a person happens to mention them.

## How it works

### 0. Terms

* **Host:** the program the agent runs in. It calls a hook with an event on standard input and reads the hook's
  output (text to add to the agent's context, or a refusal).
* **Memory service:** a long-running local process holding the memory graph (records with content, category, tags,
  importance and time; typed edges), with search, intentions (open tasks with a "done" condition) and a working
  memory journal (short notes of "where I am now").
* **Hook snippet:** a few lines of shell that pass the event to the memory service and print the answer. All logic is
  in the service.

### 1. Four points

| Point | Host event (examples) | Question it answers | Direction |
|---|---|---|---|
| Session start | `SessionStart` | who am I, what was I doing, what did I mean to do | memory → agent |
| Before each prompt | `UserPromptSubmit`, `BeforeAgent` | what do I already know about this request | memory → agent |
| After each answer | `Stop`, `AfterAgent` | what from this answer is worth keeping | agent → memory |
| Before each tool call | `PreToolUse` | what is about to be changed, who reads it, what went wrong with steps like this | memory → agent, and a gate |

### 2. Session start

The service builds one text for the start of a session.

* **On a graph with nothing written yet,** the text is a guide for the agent: what the memory is for and how to
  write the past (records), the future (intentions, closed when done) and the present (working memory notes).
* **Afterwards,** three parts: *who you are* (the agent's own records about itself), *what you are doing* (the newest
  working memory notes), *what you meant to do* (the newest open intentions, optionally narrowed to one project by a
  tag). Every record passed on carries its id, so the agent can fetch the full record when it needs it.

### 3. Before each prompt

The text of the person's prompt is used as a search query. The answer carries a small fixed number of records, and
only those clearly relevant; the number and the relevance filter can be set per hook. A prompt on a subject the graph
does not know gets nothing.

### 4. After each answer

The finished answer is sent to the service. Acknowledgements and answers that are mostly code are skipped; secrets
are masked before anything is stored. What remains becomes a short-term trace that never appears in search, and
becomes a long-term record only through the same consolidation as any memory (offline sleep): what is valuable
stays, the rest is forgotten. A surviving record is marked as written by the hook, apart from the records the agent
writes on purpose. The hook's credential can read and append; it cannot edit, delete or write the agent's identity.

### 5. Before each tool call

#### 5.1. A gate that names the readers of an edit

When the agent is about to edit source code, or to run a command that edits it, the host calls the gate with the text
of the call. The gate:

1. **Extracts the names the edit changes** (functions, types, constants, fields, identifier-like strings) from the
   text of the call.
2. **Finds their readers outside the edited files:** through a language server where it can (references, then callers
   of callers up to the first surface where data leaves the program: a command-line output, a network route, a tool
   exposed to an agent, a binding for another language), and through a text search otherwise (code, tests,
   documentation, test protocols, packaging). Each reader is shown with its line.
3. **Asks the edit's own questions:** a list kept with the project of "paired sides" that a search by name does not
   find (a rule changed in one place that must hold in another; a writer changed and its reader left), each with
   where to look and the earlier defect that taught it.
4. **Refuses once.** The call does not run; the agent sees the readers and the questions.
5. **Lets the same call through** when it is issued again carrying acknowledgement lines: for each reader or question,
   the agent's answer (handled in the edit, or why not).

The gate does not decide whether the edit is right. It makes the agent name, at the moment of the edit, who depends on
what it changes, and turns the history of the project's defects into questions asked where they recur.

#### 5.2. Recall at the moment of action

The same point can bring memories. The hook builds a query from the tool's name and the gist of its arguments (for
example: "asking the person a question in a pop-up", "recreating a container", "publishing a release") and searches
the memory graph for lessons and preferences about that kind of step. It shows a small fixed number of results at the
tool call, without adding them to the person's prompt and without merging their score with any other signal. There
is no separate store of lessons: they are ordinary records of the graph, written when something went wrong or when
the person stated a preference, and they age and consolidate like any other record.

Why this point and not the others: the hook before each prompt is driven by the person's words, and a lesson about the
agent's own habit is usually not in the person's words at that moment. It is in the agent's next action.

## Earlier public traces

* `docs/hooks.md` in the public releases of github.com/Seventhwalllab/anairyspirit from v0.1.359 (2026-09-29): the
  session start, before-prompt and after-answer hooks and their events per host program, for users.

The gate of §5.1 and the recall of §5.2 were not published before this document.

## Variants and generalizations

* Any host with hook points: coding assistants, chat clients, IDE agents, browser agents, voice assistants, robots
  with an action loop. Where a host has no tool-call hook, the gate can wrap the tool itself.
* The reader search can use any reference source: a language server, a compiler's index, a call graph, a build
  system's dependency graph, text search, or several of them.
* The paired-side questions can come from a hand-kept list, from the memory graph (records of past defects tagged by
  shape), or from both.
* The acknowledgement can be marker lines in the call, a separate confirmation step, or a signed note.
* The gate can apply to any action with readers: database migrations, configuration changes, infrastructure
  commands, document edits.
* The recall at the moment of action can be keyed by tool name only, by tool name and arguments, by the file or
  resource touched, or by the kind of step; the number of results can be fixed per tool (for example three).
* One memory service can serve the four points for several agents and hosts at once.

## Advantages

* Each kind of memory arrives at the moment it is useful: identity and plans at start, knowledge with the request,
  experience with the action.
* Lessons about the agent's own habits stop depending on the person repeating them.
* Defects that recur in a recognisable shape are asked about where they recur, without the agent having to remember
  to look.
* The hooks need nothing on the agent's side beyond a shell; all logic stays in one service.

## Prior work and our contribution

Host programs offering hooks at session start, prompt submission, response end and tool use are documented by their
vendors. Injecting retrieved memories into a model's context before generation is widely described
(retrieval-augmented generation; agent memory layers). Static analysis tools that list the callers of a changed
function, and code review tools that comment on a change, are long established.

What this document adds is the combination for one agent and one personal memory graph: four hook points with
distinct jobs served by one memory service; an after-answer path that keeps memories only through consolidation; a
gate at the tool call that turns references and a history of defects into questions and refuses once until the agent
acknowledges them; and recall keyed by the agent's own next action rather than by the person's words, drawn from the
same graph as every other memory, without a separate store of lessons.

## Reference implementation

The memory service and its hooks: An Airy Spirit (github.com/Seventhwalllab/anairyspirit), public releases from
v0.1.359, hooks for six host programs. The gate of §5.1: a tool of the author's development setup, in use since
2026-09. The recall of §5.2: described here.

## Citation

> Prokhorov, Artem. *Memory at the moment of action: agent-host hooks that bring a memory graph to session start, to
> each prompt, to each answer and to each tool call, with a gate that names the readers of an edit before it is made.*
> Prior disclosure 12, github.com/Seventhwalllab/prior-disclosures (2026-11-02). CC BY 4.0.
