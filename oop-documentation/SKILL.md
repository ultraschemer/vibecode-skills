---
name: oop-documentation
description: Use when asked to document a codebase's architecture or object-oriented structure, to evaluate a repository's structure and OOP design, or to produce class, component, sequence, activity and state diagrams with PlantUML and enrich an architecture document. Covers reconnaissance of declared types and their relationships, the four required diagram families, PlantUML authoring and PNG generation, the Software Architecture Document (SAD) template, and hard size caps on the PlantUML sources and on the architecture document. Trigger on "document the architecture", "OOP documentation", "class diagram", "component diagram", "sequence diagram", "activity diagram", "state diagram", "system architecture document", "SAD", "evaluate project structure", or "PlantUML diagrams".
license: MIT
metadata:
  audience: maintainers
  workflow: documentation
---

# OOP and Architecture Documentation with PlantUML

This skill turns a repository into a set of architecture diagrams and a Software Architecture
Document (SAD). The diagrams are the evidence; `docs/architecture.md` is the narrative that
embeds them. The work is reconnaissance first, diagrams second, document last — never the
reverse.

The output is a **presentation of the repository as it is**, not a redesign. Every declared type,
relationship and routine you draw must exist in the code. When a premise cannot be verified,
say so rather than smoothing it over.

## 0. Scope and Precedence

**What this covers.** Evaluating a project's structure and object-oriented design; enumerating
declared types, their functions and their relationships; and producing the four diagram families
below with PlantUML, rendered to PNG and embedded in a Markdown architecture document.

**What this does not cover.** Refactoring the code, fixing design defects, or writing application
tests. This is a documentation task: it changes documentation and diagrams, never production
code. If a design problem is found, record it in the document's prose or leave it out; do not fix
it in passing.

**Precedence.** A project that ships its own agent instructions (`AGENTS.md`, `CLAUDE.md` or
equivalent) governs where the artifacts live, what they are named, and what caps apply. Where
those instructions are silent, the defaults in this skill apply. If the project's convention
differs from the template below, follow the project and note the deviation.

**Language-agnostic.** The procedure applies to any language. The reconnaissance commands below
use `grep` and `find`; adapt the patterns to the language's declaration syntax. The PlantUML is
independent of the source language.

## 1. Deliverables and Hard Caps

Every run produces:

| Artifact | Location | Cap |
|---|---|---|
| PlantUML sources | `docs/diagrams/*.puml` | **32 KB total**, measured with `wc -c` |
| Rendered images | `docs/diagrams/*.png` | no cap |
| Architecture document | `docs/architecture.md` | **12 KB**, measured with `wc -c` |

- **Measure with `wc -c`, never `du`.** `du` rounds up to block size and counts directory
  entries, so it overstates and cannot be compared with the cap.
- **The PlantUML cap binds the text definitions**, i.e. the sum of the `.puml` byte counts, not
  the PNGs: `cat docs/diagrams/*.puml | wc -c`.
- **Both caps bind a finished deliverable.** A document or diagram set over its cap is not done.
- The documents live under the project's `docs/` at the repository root; diagrams under
  `docs/diagrams/`; the document is named exactly `docs/architecture.md`.

If the project's own instructions set different caps, locations or names, those win.

## 2. Phase 1 — Reconnaissance

Before drawing anything, build a complete inventory of the structure. Read the agent/instruction
files first: they often state the module boundaries, the dependency direction and the invariants
the diagrams must respect.

**Inventory checklist:**

1. **Modules / packages.** List every module, its manifest, and its ownership. Find the
   dependency direction from the imports; a diagram that draws a dependency the code does not have
   is wrong.
2. **Declared types.** Every class, struct, interface, enum, record, trait. For each: fields,
   methods/functions, and whether it is a port (interface) or an adapter (implementation).
3. **Relationships.** Composition, inheritance/implementation, embedding, association, and
   foreign keys between data models. Note where a port is declared by the consumer and
   implemented elsewhere.
4. **Entrypoints.** `main`, bootstrap, server start-up, CLI commands.
5. **Interfaces exposed.** REST/HTTP routes, RPC methods, message-queue subjects, CLI, GUI. Note
   the envelope, status codes and authentication for each.
6. **Outbound dependencies.** Databases, brokers, remote services, files, caches, external APIs.
7. **Concurrency.** Background goroutines/threads, scheduled jobs, worker pools, queues, locks,
   and the state machines of long-lived routines.
8. **State.** The lifecycle of the main domain entity: statuses and every transition between them.

Useful commands (adapt the patterns):

```bash
find . -type f \( -name '*.go' -o -name '*.py' -o -name '*.ts' -o -name '*.java' \) | sort
grep -rnE '^(type|func|class|interface|struct|enum|impl|trait|def|public) ' --include='*.go' .
grep -rnE 'router\.(GET|POST|PUT|DELETE|PATCH)|@(Get|Post|Put|Delete|Patch)Mapping' .
grep -rnE 'go func|go [a-zA-Z]|Thread|async |schedule|ticker|@Scheduled' .
grep -rnE 'sql|gorm|jdbc|session|redis|kafka|zmq|amqp|grpc|http\.(Get|Post)' .
```

**Re-derive every fact from a command.** Do not state a count, a route list or a dependency edge
from memory; run the grep and read the result. A number written without its command is a
liability.

## 3. Phase 2 — Design the Diagram Set

Produce all four families. Split a family into several diagrams when one image would be
unreadable — a per-layer class diagram is usually better than one giant canvas.

| # | Family | Diagrams to produce | Must answer |
|---|---|---|---|
| 1 | **Class** | One per layer or package group (e.g. control, domain, persistence, integration) | Which types are declared, what each does, and how they relate |
| 2 | **Sequence** | One generalized call for the main API style (REST, RPC, queue); add one for any distinct transport (e.g. encrypted RPC, registry lookup) | How a call traverses every layer end to end |
| 3 | **Activity / State** | Background routines, each long routine, the main foreground routine, and the domain entity's state machine | What runs concurrently, in what order, and what states the data takes |
| 4 | **Component** | One overview of layers, interfaces, package domains and external systems | Where each layer sits and what crosses each boundary |

A good default set for a layered service:

- `component-overview.puml`
- `class-<layer>.puml` (one per layer)
- `sequence-generalized.puml`, plus `sequence-<transport>.puml`
- `activity-background.puml`, `activity-<long-routine>.puml`, `activity-foreground.puml`
- `state-<entity>-lifecycle.puml`

## 4. Phase 3 — Author the PlantUML Sources

One diagram per `.puml` file, named after the diagram. Keep the syntax conservative so the file
renders on the installed PlantUML version.

**Style:** set `skinparam shadowing false`, a white background, and a default font. Do not use
`!theme` or recent-only features; they fail on older installs. A plain, readable palette beats a
decorated one.

**Correctness rules that cost the most time:**

- **Alias before stereotype.** `class "Name" as Alias <<stereotype>>` is valid;
  `class "Name" <<stereotype>> as Alias` is not.
- **Never put free prose inside a class body.** Body lines are members. A sentence with spaces
  is a parse error. Use a `note` for prose.
- **Quote names with punctuation** (parentheses, commas, dots): `class "func(a, b)" as F`.
- **One relationship direction, stated honestly.** If an implementation does not import the port
  package (e.g. the wiring is done in `main`), say so in a `note` instead of drawing a
  compile-time realization that does not exist.
- **State guards on transitions** as edge labels (`A --> B : trigger (R23)`).
- **Name every actor, participant and component** consistently with the code.

**Verify each source by compiling it before writing the document.** A failed compile can still
leave an old PNG in place, so delete stale images first:

```bash
rm -f docs/diagrams/*.png
plantuml -charset UTF-8 -tpng docs/diagrams/*.puml
```

`plantuml` requires Graphviz `dot`; confirm both are installed. Inspect the rendered PNGs (read
the images) and fix layout problems before proceeding. If a file fails, the error names the line;
fix and recompile until the run is silent.

**Watch the cap as you write.** `cat docs/diagrams/*.puml | wc -c` must stay at or below 32 KB. A
source that exceeds it is cut by trimming notes and redundancy, not by dropping a diagram.

## 5. Phase 4 — Generate the PNG Images

Render all sources to PNG in `docs/diagrams/`:

```bash
plantuml -charset UTF-8 -tpng docs/diagrams/*.puml
```

Confirm every source produced a valid image, and that every image the document references
exists:

```bash
for f in docs/diagrams/*.png; do file "$f"; done
for img in $(grep -oE 'diagrams/[a-z0-9-]+\.png' docs/architecture.md | sort -u); do
  [ -f "docs/$img" ] || echo "MISSING $img"
done
```

## 6. Phase 5 — Write docs/architecture.md

Write the document to the mandatory template in section 7, embedding each PNG with a relative
path (`diagrams/<name>.png`, resolved from `docs/`).

Rules:

- **The template's headings are compulsory and in order.** Add subheadings under them as needed;
  never remove or rename the top-level ones.
- **Every diagram is embedded and then described.** An image with no text description, or a
  description with no image, is incomplete.
- **The Layer & Interface Specifications must be tables**, covering external interfaces (API and
  integration interfaces, and GUI if one exists) and internal ports.
- **Keep every identifier and description; cut prose.** When over the cap, see section 9.
- **State counts as what they are made of**, and re-derive each figure from a command before
  writing it.
- **Do not mention the tool used to produce the document** unless the project asks for it; the
  document is about the system, not the process.

## 7. The Mandatory Template

```markdown
# Software Architecture Document (SAD)

## Executive Summary & System Overview

Put the system architecture summarization here.

---

## 1. Component Diagram

Put the component diagrams here, followed by the description of each component and their relationships.

### Layer & Interface Specifications

Table representations of external interfaces, including API and integration interfaces, and GUI, if it exists.

---

## 2. Class Diagram

Put the class diagrams here, for applicable types, then a text description of the diagram elements follow.

---

## 3. Generalized REST Call Sequence Diagram

Sequence diagram of the generalized REST/ZMQ/Any-other-available-API calls. A proper description of the elements of the diagrams need to follow.

---

## 4. Activity Diagram: Parallel Background Routines

This activity and state diagram models for background operations, following their descriptions, and some information about concurrency and reliability guarantees.
```

The Executive Summary carries the dependency direction and any load-bearing invariants, because
those are what a reader needs before reading the diagrams.

## 8. Diagram Content Requirements

**Class diagrams.** For each type: its role (port vs adapter, DTO vs entity), its fields, and its
functions/methods. Draw implementation (`..|>`), composition (a field of that type), and foreign
keys. Mark a port's implementation as such even when the language wires it at start-up.

**Sequence diagrams.** Include every hop: client, transport/edge, auth/decorator layer, handler,
service/domain, ports, adapter, external system. Use `alt` for failure paths and `group` for a
nested transport (encryption, retries, fan-out). Number the messages. Show where the reply
unwinds.

**Activity and state diagrams.** Model the foreground and the background as concurrent activities
(a swimlane or `fork`). For each background routine, show its loop, its wait, its per-item guard,
and its terminal paths. For the domain entity, draw a state diagram with one transition per
status change and the guard that fires it, then tabulate the transitions.

**Component diagrams.** Show layers as packages, ports as `interface`, adapters as components,
and external systems (database, broker, remote service, file) outside the boundary. Label every
edge with what crosses it. Add a note wherever a realization is done by wiring rather than by
import.

## 9. Size Discipline and Trimming Order

The caps are hard. When `architecture.md` or the PlantUML total is over:

**Cut in this order:**

1. Repeated rationale and prose that restates a diagram.
2. Worked examples and history.
3. Content duplicated between the summary and a later section.
4. Optional diagrams or optional rows.

**Never cut:** a required heading, a diagram that a required heading needs, an interface table,
a type or relationship, a state transition, or the dependency direction and invariants.

**Prefer compression to deletion.** Shorten cells and merge rows (e.g. combine sibling models
into one row) before removing a table. A document at the cap must still read as complete.

## 10. Verification Gates

Run all of these before reporting the work done:

```bash
# 1. PlantUML total within cap (32 KB)
cat docs/diagrams/*.puml | wc -c

# 2. Architecture document within cap (12 KB)
wc -c docs/architecture.md

# 3. Every source compiles and every image is valid
rm -f docs/diagrams/*.png && plantuml -charset UTF-8 -tpng docs/diagrams/*.puml
for f in docs/diagrams/*.png; do file "$f"; done

# 4. Every referenced image exists
for img in $(grep -oE 'diagrams/[a-z0-9-]+\.png' docs/architecture.md | sort -u); do
  [ -f "docs/$img" ] || echo "MISSING $img"
done

# 5. Required headings present, in order
grep -nE '^## ' docs/architecture.md

# 6. No production code changed
git status --short
```

Gate 6 must show only `docs/architecture.md` and `docs/diagrams/`. If a source file appears,
revert it: this task does not modify code.

## 11. Completion Report

Report, concisely:

- The number of PlantUML sources and PNGs, and the measured PlantUML total against its cap.
- The measured size of `docs/architecture.md` against its cap.
- The list of diagrams, grouped by the four families.
- Any design finding or unverified premise, stated as such.
- That no production code was changed, and the files created or modified.
