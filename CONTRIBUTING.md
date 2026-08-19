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

**Regenerate `index.json` and commit it with your pack.** It is the listing every Rocky Surf
control plane reads, and a pack that is not in it reaches nobody:

```bash
npx rockysurf pack index --source packs --out index.json
```

CI checks that the committed index matches `packs/` and tells you this command if it does not.
Do not hand-edit it — regenerate it, or it will describe files it no longer matches.

## Your pack defines its own tools

Start here, because it is the thing people get backwards on first contact.

**A pack can install anything.** There is no list to pick from and no registry of approved
software. A tool is an id you choose, a description, and a shell script you wrote — declare it in
your own file and it exists. Nothing needs adding to Rocky Surf first, and no maintainer here has
to know what it is.

```yaml
version: 1
pack:
  packId: deepseek-cli
  name: DeepSeek CLI
  tools: [deepseek-cli]
  displayOrder: 90
  enabled: true
tools:
  - toolId: deepseek-cli          # a name nothing in Rocky Surf has ever heard of
    name: DeepSeek CLI
    description: The DeepSeek coding agent
    category: agent
    url: https://example.com/deepseek
    installScript: |
      set -euo pipefail
      curl -fsSL "https://example.com/deepseek/install.sh" | sh
      deepseek --version
    enabled: true
    installOrder: 40
    bootstrap: false
    runAs: root
```

That file passes `pack lint` and `pack check` exactly as written. The main repository has a test
whose entire job is to prove it — a pack defining a tool id nothing has ever seen, linting clean
and installing twice in a container, with no core involvement at any point.

Brand-new tools are the **normal case**. This registry exists so that what you can install on a
box is not gated on somebody else's release cycle.

## Reusing the shared plumbing, if your pack wants it

The one category you should not redeclare is the plumbing every box already installs:
`build-essential`, `curl`, `git`, `gh`, `tmux`, `unzip`, `nodejs`, the Python bits. Those are
defined in Rocky Surf's own `packs/ai-coding-agents.yaml` — in the
[main repository](https://github.com/amroja-biz/rockysurf/tree/main/packs), not here — and you
reference them by **id** rather than copying their scripts into your file:

```yaml
  tools: [curl, git, deepseek-cli]   # two borrowed, one your own
```

Redefining one of those ids is refused, and the reason is about review rather than restriction: a
maintainer reading your pull request should never have to work out whether your `curl` is the
real one. There is a mechanical reason too — a control plane loads its whole catalog together, so
a pack that redefines `git` does not merely fail for itself, it can break the catalog for anyone
who has both installed.

If you need one of those tools to behave differently, give it your own id and define it. That is
allowed, and it is honest about what it is.

## Run the checks before you open the pull request

Both come from the published `rockysurf` package, so what you run locally is what CI runs. Both
need a checkout of the main repository, because that is where the base toolchain lives:

```bash
# Until Rocky Surf v0.1.0 is on npm, build the harness from source once:
git clone --depth 1 https://github.com/amroja-biz/rockysurf /tmp/rockysurf
(cd /tmp/rockysurf && pnpm install && pnpm --filter 'rockysurf...' build)
rs() { node /tmp/rockysurf/packages/rockysurf/dist/bin.js "$@"; }

# static — a second, no Docker. Run it constantly.
rs pack lint packs

# behavioural — a few minutes, needs Docker. Run it before you open the PR.
rs pack check packs --pack my-pack --arch arm64
rs pack check packs --pack my-pack --arch amd64

# the index — instant, and the one people forget. Run it after EVERY edit to a pack file.
rs pack index --source packs --out index.json
```

**`pack index` is not optional, and it is the easy one to miss.** `index.json` records a
`sha256` per pack file, and the pull request is expected to carry its own index update rather
than leaving `main` stale between runs — so editing a pack without regenerating the index leaves
a committed digest describing the *previous* version of your file. `pack lint` and `pack check`
both pass in that state; CI's "The committed index matches the packs" step is what fails, and it
compares with `generatedAt` stripped, so a moved timestamp alone is not drift. Regenerate, commit
`index.json` alongside the pack, and it is a non-event.

**Why you are building it.** Rocky Surf has not published to npm yet — the release is gated
behind v0.1.0 — so `npx rockysurf` fetches a placeholder with no `pack` command rather than the
harness. This is the pre-release form and it has an expiry date: once v0.1.0 ships, every command
above becomes `npx rockysurf@<version> pack …` and the clone disappears. CI does exactly the same
thing today, in `.github/actions/pack-harness`.

**There is no `--base-packs` here.** A built harness carries the packs its own release ships, so
the shared plumbing your pack references resolves out of the binary. The flag exists for pointing
the checks at a *different* toolchain, which is not the normal case. Nothing from that clone is
copied into this repository — it is an input to the check, never an artifact.

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

## What CI posts on your pull request

A comment appears saying what your pack **does**: how many steps it runs, how many of them run as
root, every URL its scripts fetch, and the scripts it introduces in full. It is the same
derivation a Rocky Surf control plane shows an operator before they consent to install — so you
see what they will see, before a reviewer does.

It is not a verdict, and it is not a check you can fail. It is there so that "this downloads from
somewhere I did not expect" gets noticed by you rather than in review, or in review rather than
by a user.

The URL list carries a caveat, and it is true: the list is read out of your scripts, so one that
builds a URL from a variable will not appear in it. **The scripts are the ground truth.**

## Review

A maintainer reads every community pull request before it merges. Expect questions about:

- **what your scripts download, and from where.** A tool served by a quota-free registry (npm,
  PyPI via `pipx`) installs **unversioned** — users expect the current agent, and a bare name
  takes the registry's stable channel rather than a prerelease. A tool that ships only as a
  GitHub release asset stays **pinned to a tag and checked against a `sha256`**, because the only
  endpoint that answers "what is latest" there is rate-limited per source IP. Either way, do not
  pipe a vendor's `install.sh` to `bash`, and do not cache-bust a download URL. The reasoning is
  in the main repository's
  [writing-a-pack.md](https://github.com/amroja-biz/rockysurf/blob/main/docs/writing-a-pack.md#which-version-to-install).
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
