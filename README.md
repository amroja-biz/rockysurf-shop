# Rocky Surf Pack Shop

The community registry for **Surge Packs** — the software bundles a
[Rocky Surf](https://github.com/amroja-biz/rockysurf) server is created with.

A pack is one YAML file describing the tools to install on a fresh Ubuntu box and the shell that
installs them. This repository is where community packs live so that people can find them, read
them, and send pull requests against them. A Rocky Surf control plane reads
[`index.json`](index.json) from here and can install any pack in it without restarting.

## This registry is community packs only

Rocky Surf's **official** packs are not here and will never be. They ship inside the Rocky Surf
release itself, in that repository's `packs/` directory, and arrive on your machine with the
software. Everything in this repository is contributed by the community.

That split is the point, and it is what makes the labels mean anything:

- **official** = it came in the Rocky Surf tarball. No registry can supply one, including this
  one.
- **community** = it came from a registry, and the label is the one the *operator* wrote next to
  that registry in their own config file.

So the trust label is never something this repository asserts about itself. There is no `tier`
field in `index.json`, deliberately — a trust label published by a registry is a claim about
trustworthiness written by the party being trusted, and it could only ever be as good as the
document containing it. Your control plane labels our packs `community` because *you* configured
this URL as a community registry, and you can label an internal registry of your own whatever is
true of it.

## What the checks do and do not prove

Every pull request runs two things, both from the published `rockysurf` package pinned to a
version, so a pack is held to exactly the contract the control plane enforces:

- **`rockysurf pack lint`** — the frozen file format, every tool id resolving, no id defined
  twice, and the mechanical half of the four author rules: no hardcoded architecture without an
  `$ARCH` branch, no `apt-get install` without `-y`, no `sudo` in a `runAs: rocky` script, no
  unguarded append, nothing assuming cloud credentials or a metadata service.
- **`rockysurf pack check`** — your pack, installed twice in one stock `ubuntu:24.04` container
  with the resume journal discarded in between, on both `amd64` and `arm64`. The second run must
  exit 0 and change nothing. This is the only thing that *proves* idempotency rather than
  inspecting for it, and it is why a pack that works on your laptop can still fail here.

**Neither is a security scan, and no wording in this repository should suggest otherwise.** An
install script is arbitrary shell that runs as **root** on somebody else's machine. No schema
check and no pattern match can decide whether it is benign. What those checks prove is that a
pack is *well-formed and survives being resumed*.

What protects the person installing a pack is three other things:

1. a maintainer reads every pull request before it merges;
2. the label is shown wherever a pack appears, and a community pack is never presented as
   reviewed by anyone beyond that;
3. the control plane shows an operator **every script a pack will run, verbatim**, along with
   which steps run as root and every URL they fetch, *before* they consent to install it.

`index.json` pins each pack by SHA-256, so a file swapped without regenerating the index is
refused by the client. But whoever can write the index can write both halves — it is a pin, not
a signature. Trust here rests on this repository's `main` branch and GitHub's account controls.
That is a real limit, stated rather than dressed up; detached signatures are planned and are not
in place today.

## Contributing a pack

Read [CONTRIBUTING.md](CONTRIBUTING.md). The short version, and the thing worth knowing first:

### Your pack defines its own tools

**A pack can install anything.** You are not picking from a list, and nothing has to be added to
Rocky Surf first. A tool is just an id, a description, and a shell script you wrote — declare it
in your own file and it exists:

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

That pack passes every check in this repository with no involvement from a Rocky Surf maintainer,
and there is a test in the main repository that proves it. New tools are the normal case — this
registry exists so that the set of installable software is not gated on anyone's release cycle.

### Reusing the plumbing, if you want it

The one thing you should *not* redeclare is the shared plumbing every box already installs:
`curl`, `git`, `gh`, `nodejs`, `tmux`, `build-essential` and friends. Those ship with Rocky Surf,
and you reference them by id instead of copy-pasting their scripts into your file:

```yaml
  tools: [curl, git, deepseek-cli]   # two borrowed, one your own
```

The checks refuse a pack that *redefines* one of those ids — not to restrict you, but because a
reviewer reading your pull request should never have to work out whether your `curl` is the real
one. Want it to behave differently? Give it your own id.

```bash
git clone https://github.com/amroja-biz/rockysurf-shop && cd rockysurf-shop
$EDITOR packs/my-pack.yaml                 # filename must match the packId

# Until Rocky Surf v0.1.0 is on npm, build the harness from source once:
git clone --depth 1 https://github.com/amroja-biz/rockysurf /tmp/rockysurf
(cd /tmp/rockysurf && pnpm install && pnpm --filter 'rockysurf...' build)
rs() { node /tmp/rockysurf/packages/rockysurf/dist/bin.js "$@"; }

rs pack lint  packs
rs pack check packs --pack my-pack
```

**The build step is temporary and it is not you.** Rocky Surf has not published to npm yet, so
`npx rockysurf` does not get you the harness — it gets a placeholder with no `pack` command.
Once v0.1.0 ships, these become `npx rockysurf@<version> pack …` and the clone goes away.

You will notice there is no `--base-packs` flag. A built harness carries the packs its own
release ships, so the shared plumbing your pack references resolves out of the binary itself.
The flag exists for pointing the checks at some *other* toolchain, which is not the normal case.

The authoring contract itself is
[`docs/writing-a-pack.md`](https://github.com/amroja-biz/rockysurf/blob/main/docs/writing-a-pack.md)
in the main repository. It is normative, it has worked examples of the right and wrong way to
write each rule, and it is the document to read first.

`index.json` is generated by CI on every merge. Do not edit it by hand; your change will be
overwritten on the next merge, and until then it will disagree with the files it claims to
describe.

## Naming

Name your pack for what it does. `rust-dev`, `data-science`, `elixir-phoenix` — all fine. Do not
name it in a way that reads as official, and do not use the Rocky Surf name or logo as your own
branding; see the main repository's
[TRADEMARK.md](https://github.com/amroja-biz/rockysurf/blob/main/TRADEMARK.md), which governs
this repository too. Accurate descriptive references — "a Surge Pack for Rocky Surf" — are
explicitly fine and always will be.

## License

MIT, matching [the main repository](https://github.com/amroja-biz/rockysurf/blob/main/LICENSE).
Pack files are configuration people are meant to read, copy, adapt and redistribute, and nothing
about them argues for terms different from the software that runs them — a contributor who has
already agreed to the main repository's terms should not have to read a second set here.

The Rocky Surf name and logo are **not** covered by that license; see
[TRADEMARK.md](https://github.com/amroja-biz/rockysurf/blob/main/TRADEMARK.md).
