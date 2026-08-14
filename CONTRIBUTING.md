# Contributing a Surge Pack

Thanks for writing one. This document says what this repository expects; the **authoring
contract** — what a pack file may contain and how its scripts must behave — lives in the main
repository and is normative:

- [`docs/writing-a-pack.md`](https://github.com/amroja-biz/rockysurf/blob/main/docs/writing-a-pack.md)

Read that first. Everything here assumes it.

## Where your file goes

One pack, one file, in `packs/`, named for its `packId`:

```
packs/rust-dev.yaml     →  pack.packId must be "rust-dev"
```

Do not edit `index.json`. It is regenerated on every merge; an edit to it will be overwritten,
and until it is, it will describe files that do not match it.

## Reference the base toolchain; do not redefine it

Rocky Surf's own `packs/ai-coding-agents.yaml` — in the
[main repository](https://github.com/amroja-biz/rockysurf/tree/main/packs), not here — defines
the shared base tools: `build-essential`, `curl`, `git`, `gh`, `tmux`, `unzip`, `nodejs`, the
Python bits, and so on. List their **ids** in your `pack.tools`; do not copy their definitions
into your file.

This is not a style preference. A `toolId` defined twice is rejected outright, and because a
control plane loads its whole catalog together, a pack that redefines `git` does not merely fail
for itself — it can break the catalog for anyone who has both installed. The checks below catch
it.

## Run the checks before you open the pull request

Both come from the published `rockysurf` package, so what you run locally is what CI runs. Both
need a checkout of the main repository, because that is where the base toolchain lives:

```bash
git clone --depth 1 https://github.com/amroja-biz/rockysurf /tmp/rockysurf

# static — a second, no Docker. Run it constantly.
npx rockysurf pack lint packs --base-packs /tmp/rockysurf/packs

# behavioural — a few minutes, needs Docker. Run it before you open the PR.
npx rockysurf pack check packs --base-packs /tmp/rockysurf/packs --pack my-pack --arch arm64
npx rockysurf pack check packs --base-packs /tmp/rockysurf/packs --pack my-pack --arch amd64
```

`--base-packs` is how your references to the base toolchain resolve. Without it every one of them
is reported as an unknown tool. Nothing from that checkout is copied into this repository — it is
an input to the check, never an artifact.

Exit codes, if you are scripting this: **0** clean, **1** your pack failed the check, **2** the
check could not be run at all (no Docker, wrong directory, a `--pack` id matching nothing).

### What `pack check` actually does, and why it is stricter than your laptop

It starts a **stock `ubuntu:24.04` container** — no preinstalled convenience packages, empty apt
lists, no `sudo` — creates an unprivileged `rocky` user, runs your pack's real install plan,
then **deletes the resume journal and runs the whole thing again in the same container**.

That deletion is the entire test. Rocky Surf's on-box agent resumes an interrupted install by
skipping every step already marked done, so re-invoking it without clearing the journal proves
nothing. With the journal gone, every script runs again for real, and the second run must:

- exit 0;
- leave `/home/rocky/.bashrc`, `/root/.bashrc` and `/etc/apt/sources.list.d/` **byte-identical**.

That last one is what catches the single commonest pack bug: an unguarded `>>` that appends the
same `PATH` line twice.

A pack that passes on one architecture and fails on the other fails the check. There is no
amd64-only pack.

## Review

A maintainer reads every community pull request before it merges. Expect questions about:

- **what your scripts download, and from where.** Pin a version or a checksum. `@latest` and a
  `main`-branch install script mean the thing CI tested is not the thing users get.
- **which steps need root.** Declare `runAs` honestly. A `runAs: rocky` step that reaches for
  `sudo` fails here and would fail the same way on a real box.
- **the `guide` field.** Say what a user has to do by hand once the box is theirs — how to
  authenticate whatever you installed. It is shown to them verbatim.

## Naming

Name your pack for what it does. Do not name it so that it reads as an official Rocky Surf pack,
and do not use the Rocky Surf name or logo as your own branding. Nothing in this repository is an
official Rocky Surf pack; those ship inside the release. The main repository's
[TRADEMARK.md](https://github.com/amroja-biz/rockysurf/blob/main/TRADEMARK.md) governs this
repository too, and it is short and reasonable — accurate descriptive references like "a Surge
Pack for Rocky Surf" are explicitly fine.

## What the checks do not prove

Worth being plain about, because it shapes what review is for. Your install script is arbitrary
shell that runs as **root** on other people's machines. No schema check and no pattern match can
decide whether it is benign, and nothing in this repository claims to. `pack lint` and
`pack check` prove that a pack is well-formed and survives a resume — nothing more.

What carries the rest is maintainer review, the label an operator's own config puts on this
registry, and the fact that a control plane shows them every script verbatim before they consent
to run it.
If you would be uncomfortable with a stranger reading your `installScript` line by line, it is
not ready.

## License

By contributing you agree your pack is licensed under the [MIT license](LICENSE), the same terms
as the main repository. The Rocky Surf name and logo are not covered by it.
