# `packs/`

Community-contributed Surge Packs. One pack per file, named for its `packId`
(`packs/rust-dev.yaml` declares `packId: rust-dev`).

This is where your pull request goes. Read [`../CONTRIBUTING.md`](../CONTRIBUTING.md) first, and
the normative authoring contract it points at.

**Rocky Surf's official packs are not here**, and no file in this directory is official. They
ship inside the Rocky Surf release, in that repository's `packs/`. Everything here is community
work, and the shared base toolchain your pack references by tool id lives over there too — which
is why the checks need `--base-packs` pointed at a checkout of it.

Every file here has passed `rockysurf pack lint` and `rockysurf pack check` — the same checks the
Rocky Surf repository applies to its own packs — and been read by a maintainer. **Neither of
those is a security review.** A pack's install scripts are arbitrary shell run as root on your
box; the checks prove the file is well-formed and survives being resumed, and nothing more. Read
what you install. Rocky Surf shows you every script before you consent to it, which is the moment
to.
