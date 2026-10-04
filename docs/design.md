# Design overview

This page describes how VSM works inside Basalt OS, at the level of what
the public code and the released packages contain. The code is in
[basalt-os/basalt-os](https://github.com/basalt-os/basalt-os) under
`packages/basalt-assistant`.

## The questions it answers

The Basalt assistant has a decision layer with a small set of typed
questions. VSM answers four of them:

| Question | Possible answers |
|---|---|
| `unit.cause`: why did this service fail | configuration error, SELinux denial, port conflict, failed dependency, full disk, missing file, crash, unknown |
| `avc.class`: what kind of SELinux denial is this | mislabeled file, missing file context, port, boolean, suspicious, unknown |
| `disk.cause`: what holds most of the reclaimable space | snapshots, journal, package cache, other data |
| `dnf.next`: after a failed package transaction, roll back or look first | roll back, investigate |

The assistant's rules answer every other question, and also answer these
four whenever VSM is not available. Severity, notifications and routing
stay with the rules.

## Pieces

| Piece | Package | Code |
|---|---|---|
| Knowledge index | `basalt-knowledge` (`/usr/share/basalt/knowledge/<fedora>`) | `internal/knowledge` |
| Planner | `basalt-vsm-planner` (`/usr/share/basalt/vsm-planner`) | `internal/vsm/planner` |
| Runtime, guard, binder | part of `basalt-assistant` | `internal/vsm` |
| Decision backend | part of `basalt-assistant` | `internal/decide/vsm.go` |

### Knowledge index

A knowledge case is a known problem written down as data: the evidence
pattern it needs (codes that must be present, codes that must be absent),
the answer it gives to each question, a diagnosis text, the typed actions
of the fix from the assistant's closed action set, the checks that confirm
the fix, the Fedora releases where it applies, its license and where it
came from. `cases.jsonl` in the package is readable as is; `index.bin` is
a compact table built from it; `manifest.json` names the format, the
version of the evidence code set (the DSL) and the SHA-256 of the cases.

For a question, the index ranks the cases by the required codes they
match, minus contradictions, then by how specific they are, and returns
the best three.

### Planner

The planner is a gated linear recurrent network with dimension 128 and
about 65,000 parameters (256 KiB of float32 weights). It reads a short
character sequence: the goal, the evidence codes, and for each candidate
how many of its required codes matched, how many are missing and whether
any contradiction was found. It never sees names, paths or ports. It then
emits one decision: accept a candidate, pick another one or abstain. The
probabilities of that decision become the confidence of the answer.

The Go implementation in the assistant reproduces the reference runtime
operation by operation, and a parity test replays recorded decisions from
the reference: on the released weights every recorded decision is
identical.

### Deterministic guard

The guard is ordinary code, outside the model, and is always on when VSM
serves an answer:

- a pick whose required evidence is not all present, or whose excluded
  evidence is present, is refused and becomes the cautious answer;
- when the SELinux policy could not be queried, the answer is cautious;
- a denial's fix is proposed only when the confident answer is the class
  that fix was built for, never for "unknown" or "suspicious";
- a case known to have no verified fix is shown as a hint, never as an
  automatic proposal.

The guard cannot make VSM more confident, only less.

### Action binding

A case's actions are templates. The binder fills their parameters from
the values the diagnosers extracted (a unit name, a path, a port) and runs
them through the same validators every other assistant proposal goes
through. A missing value or a refused action turns the case into a hint.
Actions that need checks only root can make are held as hints when the
confined daemon is the one asking.

### Proposals and confirmation

What VSM decides becomes, at most, a proposal. The person sees the exact
commands and confirms them (`basalt apply`) before anything changes, the
same as for proposals from the rules. Every decision is written to the
local audit log with the picked case, the candidates, the evidence and
the knowledge and planner versions.

## Failure handling

VSM is the default decision backend. The assistant checks the knowledge
index each time it opens it: the detached OpenPGP signature of the
manifest by the OpenBasalt knowledge signing subkey (the release key and
the subkey fingerprints are pinned in the code, and the check uses only
Go's standard library), the checksums of the cases and of the index
table, the table layout, the Fedora release and the DSL version shared
with the planner. If the packages are missing, damaged, unsigned or for
another DSL, or a question takes more than 2 seconds, the rules answer,
marked as a fallback, and the reason is recorded.

## Confinement

The assistant runs in its own SELinux domain. The weights and the
knowledge files carry their own type, `basalt_knowledge_t`, which that
domain may list and read but not write or map. The daemon has no network
access.

## How it improves

Most improvements are knowledge releases: a new case for a problem seen in
the field, with its evidence pattern and verified fix. That is a data
update shipped with the system updates and needs no retraining. A new
kind of evidence (a new code in the DSL) needs new planner weights, which
are released as a new `basalt-vsm-planner` package together with a
matching knowledge release.
