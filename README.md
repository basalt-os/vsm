# VSM: Verified Specialist Model

VSM is a small specialist decision model for diagnosis and repair on
[Basalt OS](https://basalt-os.org), a Fedora-based Linux distribution. It
answers a few precise questions about a broken system (why did this unit
fail, what kind of SELinux denial is this, why is the disk full, what to do
after a package transaction) and, when it knows a verified fix, names it.

This repository is the public overview of VSM: what it is, what is open,
how it was measured, how to verify what we ship and how to report problems.
It holds documentation only. The code that runs VSM is open and lives in
[basalt-os/basalt-os](https://github.com/basalt-os/basalt-os).

## Why it exists

General chat models are good at talking about a problem and bad at being
sure. On a running system, a wrong but confident repair is worse than no
answer. VSM is built the other way around:

- it knows a closed set of problems, written down as cases with the
  evidence each one needs and the fix that was checked for it;
- it says "unknown" when the evidence does not fully support a case, and a
  rule outside the model enforces that;
- it never acts by itself: every change is shown to the person first and
  needs their confirmation;
- it is small enough to run on the machine it is diagnosing, on CPU, in
  about a millisecond per question, with no network access and no
  telemetry.

"Verified" refers to that design: the model only chooses among cases whose
fixes were verified, its choice is checked by deterministic code before it
counts, and its quality is measured against suites and injected faults
(see Results below), not asserted.

## How it works

1. The Basalt assistant's diagnosers read the system (journal, unit state,
   SELinux policy, disk and snapshot data) and turn what they find into
   short evidence codes. Names, paths and port numbers stay out of the
   codes.
2. The knowledge index (package `basalt-knowledge`, one per Fedora
   release) ranks the known cases against those codes and returns the
   three best candidates.
3. The planner (package `basalt-vsm-planner`, about 65,000 parameters,
   256 KiB of weights) reads the goal, the codes and how well each
   candidate matches, and accepts a candidate, picks another or abstains.
4. A deterministic guard outside the model refuses any pick the evidence
   does not fully support and turns it into the cautious answer
   ("unknown", "investigate"). It also answers cautiously when the
   SELinux policy could not be queried.
5. If the picked case has a fix, its typed actions are bound to the real
   names from the findings and validated. The assistant then shows the
   proposal; nothing is applied until the person confirms it.

Every decision goes to the assistant's local audit log with the picked
case, the candidates, the evidence and the knowledge and planner versions.
VSM is the assistant's default decision backend since basalt-assistant
0.8.0. If the data packages are missing, damaged, unsigned or for another
version, or a question takes more than 2 seconds, the assistant falls
back to its rules and records why. New problems arrive as new knowledge cases, a data
update, not as a new model.

A longer description is in [docs/design.md](docs/design.md).

## What is open and what is not

| Part | Status | Where |
|---|---|---|
| Runtime inside the Basalt assistant: knowledge index reader, planner inference in Go, deterministic guard, action binding, decision backend | Open, Apache-2.0 | [basalt-os/basalt-os](https://github.com/basalt-os/basalt-os), `packages/basalt-assistant/internal/knowledge`, `internal/vsm`, `internal/vsm/planner`, `internal/decide/vsm.go` |
| Data package recipes and pinned checksums | Open, Apache-2.0 | `packages/basalt-knowledge` and `packages/basalt-vsm-planner` in the same repository |
| Released planner weights (`basalt-vsm-planner`) | Open, Apache-2.0 | signed RPM at [obpkg.org/basalt](https://obpkg.org/basalt/44/x86_64/) |
| Released knowledge index (`basalt-knowledge`), cases included in readable form | Open, Apache-2.0 | signed RPM at [obpkg.org/basalt](https://obpkg.org/basalt/44/x86_64/) |
| Evaluation command (`basalt-eval`) and its case suite | Open, Apache-2.0 | `packages/basalt-assistant/tools/basalt-eval` and `eval/` in basalt-os/basalt-os |
| Training code | Not public for now | |
| Data generation | Not public for now | |
| Dataset repository | Not public for now | |
| Evaluation harness internals used during training | Not public for now | |
| Research repository | Not public for now | |

Future releases of the weights and of the knowledge index stay under
Apache-2.0.

The parts that are not public are not public yet. We have not set a date
for opening them and do not promise one here; the reasons are in
[docs/faq.md](docs/faq.md). Everything you need to run, inspect, audit and
reuse what ships in Basalt OS is open.

## Results

Measured on 2026-10-04 with basalt-assistant 0.7.0, knowledge index
44.20261004 (39 cases) and planner 3.20261004, against the assistant's
own rules (now the fallback). basalt-assistant 0.8.0 changes only the
default backend; the suite gives the same numbers with it.

Evaluation suite (253 questions over 239 cases: 34 from a lab machine, 30
from the confined daemon on that machine, 189 generated):

| Backend | Accuracy | ECE | Brier | Lab | Lab, confined daemon | Generated |
|---|---|---|---|---|---|---|
| rules | 96.8 % | 0.212 | 0.138 | 32/34 | 29/30 | 184/189 |
| VSM | 98.0 % | 0.020 | 0.042 | 32/34 | 29/30 | 187/189 |

ECE (expected calibration error) and Brier measure how well the stated
confidence matches how often the answer is right; lower is better.

Faults injected one at a time on a lab virtual machine, each diagnosed by
the root command line and by the confined daemon: 35 faults from earlier
runs and 15 new ones, written and committed before any engine ran on them
(a fresh held-out set):

| Set | Backend | Root: answers right | Daemon: answers right | ECE root / daemon | Unsafe proposals |
|---|---|---|---|---|---|
| earlier 35 | rules | 33/35 | 31/35 | 0.27 / 0.19 | 0 |
| earlier 35 | VSM | 33/35 | 31/35 | 0.01 / 0.13 | 0 |
| new 15 | rules | 14/15 | 12/15 | 0.23 / 0.17 | 0 |
| new 15 | VSM | 14/15 | 12/15 | 0.06 / 0.11 | 0 |

The changes proposed were right exactly where the answers were, for both
backends, and the rules never had to stand in for VSM during these runs.

The suite, the evaluation command, the runtime, the weights and the
knowledge index are all public, so the suite numbers can be reproduced on
any Fedora 44 machine; the steps are in
[docs/eval-suite.md](https://github.com/basalt-os/basalt-os/blob/main/docs/eval-suite.md)
in basalt-os/basalt-os. The fault-injection harness runs on our lab
machines and is not packaged for outside use yet.

Cost: about 1 ms per question in the suite (about 4 ms on the lab machine)
and about 0.4 MB of memory for the knowledge index and the planner
together.

Caveats, stated plainly:

- VSM became the default because it now meets the bar we set for that,
  "at least as good as the rules on every source": equal or better on the
  suite, equal answers and equal changes on both live sets in both views,
  no unsafe proposals, and much better calibrated. The rules stay as the
  fallback, and `backend = rules` under `[decision]` in
  `/etc/basalt/assistant.conf` turns VSM off.
- An earlier live run the same day, with assistant 0.6.0, found two unsafe
  proposals. Both came from how the assistant used VSM's answer, not from
  the model's pick, and were fixed with two deterministic checks in 0.6.2
  (a fix is proposed only when the confident answer equals the class the
  fix was built for). The table above is after that fix.
- The samples are small (39 known cases, 50 injected faults) and most of
  the suite is generated. They show the design works on the problems it
  covers; they say nothing about problems no case covers yet, where the
  expected answer is "unknown".
- Calibration is very good on the suite. On the confined daemon's view
  the live ECE is still 0.11 to 0.13: when VSM abstains there, part of the
  probability now goes to the causes the daemon could not see, which
  fixed most of the earlier overconfidence but not all of it.
- One answer on the new set was overconfident: a permissive SELinux
  denial with no known fix, answered "unknown" at 0.995 in the root view.
  The answer was the cautious one and no change was proposed.
- Only Fedora 44 has a knowledge index today.

The same numbers, with more detail on the assistant side, are in the
assistant documentation:
[docs/assistant.md](https://github.com/basalt-os/basalt-os/blob/main/docs/assistant.md)
in basalt-os/basalt-os. New measurements will be added here with their
date when new weights or knowledge are released.

## Verifying a release

Weights and knowledge are released as RPM packages in the Basalt OS
repository at <https://obpkg.org/basalt>. Packages and repository metadata
are signed with the OpenBasalt release key:

- key: <https://obpkg.org/keys/openbasalt-release-key.asc>
- primary key fingerprint: `3601 7348 42BD 4E48 2D19  DE4A E4EE D5EC A395 B302`
- packages signing subkey: `3024 61D2 6520 E077 D07F  FCA9 AA27 C62C 36CC FC4B`
- knowledge signing subkey: `85D6 1430 B704 3868 0F6E  B955 E79E 4020 605A 659A`

On Basalt OS the key is already installed and dnf checks both signatures.
Elsewhere, check the key and a package by hand:

```sh
curl -fsSLO https://obpkg.org/keys/openbasalt-release-key.asc
gpg --show-keys --with-fingerprint openbasalt-release-key.asc
sudo rpm --import openbasalt-release-key.asc
curl -fsSLO https://obpkg.org/basalt/44/x86_64/basalt-vsm-planner-3.20261004-2.fc44.noarch.rpm
rpm -K basalt-vsm-planner-3.20261004-2.fc44.noarch.rpm
```

`rpm -K` must print `digests signatures OK`. Then check the files against
the checksums pinned in the public repository (`sources.manifest` in
`packages/basalt-vsm-planner` and `packages/basalt-knowledge` of
basalt-os/basalt-os):

```sh
rpm2cpio basalt-vsm-planner-3.20261004-2.fc44.noarch.rpm | cpio -idm
sha256sum usr/share/basalt/vsm-planner/*
```

| Package | File | SHA-256 |
|---|---|---|
| basalt-vsm-planner 3.20261004 | weights.bin | `71351ce5144ed98e9b2c45fd6e7990116a315d0e8a7532e5e1acbb30dc0e5290` |
| basalt-vsm-planner 3.20261004 | config.txt | `86060318f9fd8dd270672acda42a9b27bb16d4832c6801e4519324ec8cc832b9` |
| basalt-vsm-planner 3.20261004 | calibration.json | `a67311a1eb5d29c4933debe6dc1d2b1128d0ab03976c7e6c09abfd5f7bc4411c` |
| basalt-knowledge 44.20261004 | cases.jsonl | `ed3e96f53d140fcbb885c5901f1a35fe62b43b5b469bf9dbe86fd8d28965e7eb` |
| basalt-knowledge 44.20261004 | index.bin | `f8bfcad7aa5dbf48230e2a3a799d71190c41107dabfb0457bab94136b278411e` |
| basalt-knowledge 44.20261004 | manifest.json | `fb946ade619cd9ef9bbf1f942da2357184bff0879e94358c84c76c2ac50b8064` |
| basalt-knowledge 44.20261004 | manifest.json.sig | `861408663ae70f4a75c7b6a6f89fae702cef897bea407aa41ff7cb34ba2bdbb9` |

The knowledge index is signed on its own as well. Its `manifest.json`
carries the SHA-256 of `cases.jsonl` and of `index.bin`, and
`manifest.json.sig` is a detached OpenPGP signature of the manifest by the
knowledge signing subkey above. The assistant checks that signature (with
both fingerprints pinned in its code), the checksums, the table layout and
the version each time it opens the index; if anything does not match, the
rules answer and the decision record says why. Anyone can make the same
check with standard tools on an installed system:

```sh
gpg --dearmor </usr/share/basalt/knowledge/openbasalt-release-key.asc >openbasalt-release-key.gpg
gpgv --keyring ./openbasalt-release-key.gpg \
     /usr/share/basalt/knowledge/44/manifest.json.sig /usr/share/basalt/knowledge/44/manifest.json
sha256sum /usr/share/basalt/knowledge/44/cases.jsonl /usr/share/basalt/knowledge/44/index.bin
```

## Privacy

- VSM runs inside the assistant on your machine. The assistant daemon has
  no network access; VSM sends nothing anywhere and has no telemetry.
- The planner sees evidence codes and match statistics, never file names,
  paths, user names or port numbers.
- Decisions are written only to the local audit log under
  `/var/log/basalt-assistant`.
- The weights and the knowledge files have their own SELinux type; the
  confined assistant may read them and cannot change them.

## Reporting problems

- A wrong diagnosis, a missing case or a question about results: open an
  issue in this repository. Please include the assistant version, the
  `basalt-knowledge` and `basalt-vsm-planner` versions and, if you can,
  the decision record from the audit log (check it for anything you do not
  want to share first).
- A bug in the runtime code: open it in
  [basalt-os/basalt-os](https://github.com/basalt-os/basalt-os/issues), or
  here if you are not sure; we move issues between repositories.
- A security problem: do not open a public issue. See
  [SECURITY.md](SECURITY.md).

## License

The documentation in this repository is licensed under the Apache License,
Version 2.0 (see [LICENSE](LICENSE) and [NOTICE](NOTICE)). The runtime
code, the released weights and the released knowledge index are also
Apache-2.0, each with the license file in its own package.
