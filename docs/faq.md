# FAQ

## Is VSM a chatbot or a large language model?

No. It is a small decision model (about 65,000 parameters) that picks
among known cases for a few typed questions, plus a knowledge index and
deterministic checks around it. It does not write free text and does not
hold a conversation. The Basalt assistant may use a separate language
model for conversation; VSM does not depend on one.

## Does it need a GPU or a network connection?

No. It runs on CPU inside the assistant, in about a millisecond per
question, with about 0.4 MB of memory for the weights and the knowledge.
The assistant daemon has no network access, and VSM has no telemetry.

## Is it on by default?

Yes, since basalt-assistant 0.8.0. We switched the default once VSM was
at least as good as the rules on every source we measure (see the README).
The assistant recommends `basalt-knowledge` and `basalt-vsm-planner`, so a
default install has them, and the rules stay as the fallback. To use the
rules only, set `backend = rules` under `[decision]` in
`/etc/basalt/assistant.conf`.

## Can I use the weights outside Basalt?

Yes. The planner weights and the knowledge index are released under the
Apache License, Version 2.0, and future releases keep that license. You
may use, modify and redistribute them, commercially or not, under its
terms. Keep in mind that they were made for the Basalt assistant's
evidence codes and questions; outside that setting they are unlikely to
be useful without the matching runtime, which is also Apache-2.0.

## Why not open the training code and data yet?

Because they are not in a state we can support in public. The training
code, the data generator, the dataset repository and the internals of the
evaluation harness are research tools: they change daily, carry
experiments that are not about Basalt OS, and were not written for
outside use. Publishing them now would mean either publishing something
misleading or spending the time on cleanup instead of on the system
itself.

What ships is what we wanted open first: the code that runs on your
machine, the exact weights and knowledge it loads, the suite the numbers
come from and the signatures to check all of it. That is enough to audit
what VSM does on a system, rerun the suite and reuse the weights.

We have not set a date for opening the rest, and we would rather not
promise one we might miss. When that changes, it will be announced here.

## How do I know the weights I have are the ones you measured?

Check the package signature with the OpenBasalt release key, then compare
the file checksums with the ones in the README and in the `sources.manifest`
files of basalt-os/basalt-os. The knowledge index also carries its own
signature by the OpenBasalt knowledge signing subkey, which the assistant
checks every time it loads the index. The results in the README name the exact
package versions they were measured with.

## What happens with a problem VSM has never seen?

If no known case fully matches the evidence, the guard turns the answer
into "unknown" (or the cautious answer for that question), and the
assistant says so instead of guessing. Problems seen in the field become
new knowledge cases in a later release.

## How do I suggest a new case?

Open an issue in this repository describing the problem, what the
assistant reported and, if you can, the decision record from the audit
log. Check the record first for anything you do not want to share.
