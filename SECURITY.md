# Security policy

VSM decides what the Basalt OS assistant proposes to change on a system.
Its promise is that it never makes a confident proposal the evidence does
not support, and that nothing it proposes happens without the person's
confirmation. A way around either is a security problem. Please report it
privately and give us time to fix it before it is disclosed.

## Reporting a vulnerability

Use GitHub's private vulnerability reporting for this repository:
<https://github.com/basalt-os/vsm/security/advisories/new>
(the "Report a vulnerability" button under the Security tab). If you cannot
use GitHub, write to security@openbasalt.org.

Do not open a public issue, pull request or discussion for a security
problem. If the problem is in the runtime code (the assistant, its
knowledge reader, planner inference, guard or binder), you may also report
it in [basalt-os/basalt-os](https://github.com/basalt-os/basalt-os); we
move reports between repositories.

Examples of what we want to hear about:

- input (journal lines, file names, unit names, crafted packages) that
  makes VSM pick a case with a fix the evidence does not support, or gets
  past the deterministic guard;
- a way to make the assistant load weights or knowledge that were not
  shipped in a signed package, knowledge whose manifest signature does
  not verify, or files that fail their checksums;
- action binding that puts attacker-controlled text into a proposed
  command in a way the validation does not catch;
- anything in the released weights or knowledge index that leaks data it
  should not contain.

Please include what you can of:

- the versions of `basalt-assistant`, `basalt-knowledge` and
  `basalt-vsm-planner`, and the distribution;
- whether SELinux was enforcing with the `basalt_assistant` module;
- steps to reproduce, and what an attacker gains (who the attacker is: a
  local user, a service writing to the journal, a remote party, a package);
- whether you want to be credited, and how.

We aim to acknowledge a report within 7 days and to agree on a disclosure
date with you, normally within 90 days of the report or when a fix is
released, whichever comes first. Fixes are published as new packages with
a GitHub security advisory (and a CVE when it applies).

## Supported versions

VSM in Basalt OS is pre-release. Until Basalt OS 1.0, only the latest
released `basalt-knowledge` and `basalt-vsm-planner` packages, with the
latest assistant, receive fixes.

## Verifying releases

Packages in the Basalt OS repository (<https://obpkg.org/basalt>) are signed
with the OpenBasalt release key, published at
<https://obpkg.org/keys/openbasalt-release-key.asc>. Check its fingerprint
before trusting it:

- primary key `3601 7348 42BD 4E48 2D19  DE4A E4EE D5EC A395 B302`
- packages signing subkey `3024 61D2 6520 E077 D07F  FCA9 AA27 C62C 36CC FC4B`
- knowledge signing subkey `85D6 1430 B704 3868 0F6E  B955 E79E 4020 605A 659A`

The file checksums of each release are listed in the
[README](README.md#verifying-a-release).
