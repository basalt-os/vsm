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
If the data packages are missing, damaged or for another version, or a
question takes more than 2 seconds, the assistant falls back to its rules
and records why. New problems arrive as new knowledge cases, a data
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

Measured on 2026-10-04 with basalt-assistant 0.6.2, knowledge index
44.20261003 (37 cases) and planner 2.20261003, against the assistant's
own rules (the default backend).

Evaluation suite (253 questions over 239 cases: 34 from a lab machine, 30
from the confined daemon on that machine, 189 generated):

| Backend | Accuracy | ECE | Brier | Lab | Lab, confined daemon | Generated |
|---|---|---|---|---|---|---|
| rules | 96.8 % | 0.212 | 0.138 | 32/34 | 29/30 | 184/189 |
| VSM | 97.6 % | 0.020 | 0.050 | 32/34 | 28/30 | 187/189 |

ECE (expected calibration error) and Brier measure how well the stated
confidence matches how often the answer is right; lower is better.

35 faults injected one at a time on a lab virtual machine, each diagnosed
by the root command line and by the confined daemon (11 of the 35 come
from fault families held out of the knowledge cases):

| Backend | Root: answers right | Root: changes right | Daemon: answers right | Daemon: changes right | Unsafe proposals |
|---|---|---|---|---|---|
| rules | 33/35 | 34/35 | 31/35 | 31/35 | 0 |
| VSM | 33/35 | 34/35 | 30/35 | 30/35 | 0 |

The suite, the evaluation command, the runtime, the weights and the
knowledge index are all public, so the suite numbers can be reproduced on
any Fedora 44 machine; the steps are in
[docs/eval-suite.md](https://github.com/basalt-os/basalt-os/blob/main/docs/eval-suite.md)
in basalt-os/basalt-os. The fault-injection harness runs on our lab
machines and is not packaged for outside use yet.

Cost: about 1 ms per question and about 0.4 MB of memory for the knowledge
index and the planner together.

Caveats, stated plainly:

- VSM is optional and not the default. The rules stay the default because
  VSM is one answer behind them on the confined daemon's view in both
  measurements, and the bar for switching is "at least as good on every
  source".
- An earlier live run the same day, with assistant 0.6.0, found two unsafe
  proposals. Both came from how the assistant used VSM's answer, not from
  the model's pick, and were fixed with two deterministic checks in 0.6.2
  (a fix is proposed only when the confident answer equals the class the
  fix was built for). The table above is after that fix.
- The samples are small (37 known cases, 35 injected faults) and most of
  the suite is generated. They show the design works on the problems it
  covers; they say nothing about problems no case covers yet, where the
  expected answer is "unknown".
- Calibration is very good on the suite. Live, when the confined daemon
  cannot see enough and the right answer is the cautious one, VSM is
  overconfident in that cautious answer. That is a known limit we are
  working on.
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

On Basalt OS the key is already installed and dnf checks both signatures.
Elsewhere, check the key and a package by hand:

```sh
curl -fsSLO https://obpkg.org/keys/openbasalt-release-key.asc
gpg --show-keys --with-fingerprint openbasalt-release-key.asc
sudo rpm --import openbasalt-release-key.asc
curl -fsSLO https://obpkg.org/basalt/44/x86_64/basalt-vsm-planner-2.20261003-1.fc44.noarch.rpm
rpm -K basalt-vsm-planner-2.20261003-1.fc44.noarch.rpm
```

`rpm -K` must print `digests signatures OK`. Then check the files against
the checksums pinned in the public repository (`sources.manifest` in
`packages/basalt-vsm-planner` and `packages/basalt-knowledge` of
basalt-os/basalt-os):

```sh
rpm2cpio basalt-vsm-planner-2.20261003-1.fc44.noarch.rpm | cpio -idm
sha256sum usr/share/basalt/vsm-planner/weights.bin usr/share/basalt/vsm-planner/config.txt
```

| Package | File | SHA-256 |
|---|---|---|
| basalt-vsm-planner 2.20261003 | weights.bin | `41e8c37852375eaf70bf16230c9ded8a2d59f1371be37e6b932961ed2e6e6523` |
| basalt-vsm-planner 2.20261003 | config.txt | `0377aa2e5d047ddc97b36ddc345edc305702195e94d62702fd348519c50155ea` |
| basalt-knowledge 44.20261003 | cases.jsonl | `437fcc78e9aa6e55e5567b77b54f32bf58af9843c21a7ae97acb2af278636473` |
| basalt-knowledge 44.20261003 | index.bin | `c3817c412f4069db1a01143da09302799f426f797ebca92d282777115ff699a5` |
| basalt-knowledge 44.20261003 | manifest.json | `98914caf2b0972404eb8888835a7be0733430e8f5929191ba3e50973479e6e61` |

The knowledge `manifest.json` also carries the SHA-256 of `cases.jsonl`,
and the assistant checks it, the table layout and the version each time it
opens the index. A detached signature on the manifest itself is planned;
today the integrity chain is the package signature plus these checksums.

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
