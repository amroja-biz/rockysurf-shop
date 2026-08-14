---
name: A pack does not work
about: A pack in this registry fails to install, or installs something broken
title: '[pack] '
labels: bug
---

**Which pack** (and, if it is not from this registry, which registry it came from — the pack shop
shows the source next to every pack):

**What happened on the box** — the failing step's output if you have it. Rocky Surf keeps a
per-step log at `/var/lib/rockysurf/steps/<step>.log`:

**Architecture** (`amd64` / `arm64`) and where the box is hosted:

**Rocky Surf version** (`rockysurf --version`):

---

If the problem is that a pack does something you did not expect it to do — rather than failing —
please say so plainly. Pack install scripts run as root, this registry's automated checks do not
and cannot prove one is benign, and a report like that is the thing maintainers most want to
receive quickly. If you would rather not say it in public, use the private reporting route in the
main repository's SECURITY.md.
