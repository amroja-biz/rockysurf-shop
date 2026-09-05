# Rocky Surf Shop

<p align="center">
  <picture>  
    <img src="rocky-surf-shop-optimized.webp" alt="Rocky Surf Shop" width="400">
  </picture>
</p>

Registry for [Rocky Surf](https://github.com/amroja-biz/rockysurf) **Surge Packs** and **Providers** created by community members. This shop is visible within Rocky Surf.

## A Note On Security

DO NOT TRUST THE CONTENTS OF THIS SHOP! 

This repo is the primary means for Rocky Surf users to share what they've made with others. That said, don't assume that community-contributed Surge Packs and Providers are safe. While we govern this shop via pull requests and basic security checks, we can't ensure your safety. If a Surge Pack includes a tool that is itself malicious, it won't be picked up in a YAML scan.

If you find something that looks suspicious, report it on the [Rocky Surf Discord](https://discord.gg/AbPsjNEmbh).

With that out of the way, let's get on to the good stuff!

## Surge Packs
A Surge Pack a bag of software defined in a YAML file. It's a core part of Rocky Surf and makes it dead easy to spin up cloud servers on the Provider of your choice, pre-installed with your favorite agentic coding harnesses.

There are three types of Surge Packs:

- Official. These are bundled into Rocky Surf.
- Personal. Surge Packs that you create using the create-surge-pack skill from Rocky Surf.
- Community. What you'll find here.

### Contributing a pack

Read [CONTRIBUTING.md](CONTRIBUTING.md). In short:

#### Your pack defines its own tools

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

#### Reusing the plumbing, if you want it

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

`index.json` is generated, and **your pull request should carry its own regeneration** —
`npx rockysurf pack index --source packs --out index.json`. It is the listing every control plane
reads, so a pack missing from it reaches nobody. CI checks the committed index against `packs/`
and names that command if the two disagree. Do not hand-edit it; regenerate it.

### Naming Surge Packs

Name your pack for what it does. `rust-dev`, `data-science`, `elixir-phoenix` — all fine. Do not
name it in a way that reads as official, and do not use the Rocky Surf name or logo as your own
branding; see the main repository's
[TRADEMARK.md](https://github.com/amroja-biz/rockysurf/blob/main/TRADEMARK.md), which governs
this repository too. Accurate descriptive references — "a Surge Pack for Rocky Surf" — are
explicitly fine and always will be.

## Providers

A **provider** is the code that talks to a cloud — how Rocky Surf creates, stops, describes and
terminates a machine there. Five ship inside Rocky Surf (AWS, Azure, GCP, Hetzner and
bring-your-own); anyone can write another against the published provider SDK, and this registry is
how one reaches other people's installations.

Providers are listed in [`providers.json`](providers.json), a separate file from `index.json`.
Rocky Surf fetches it only when an operator opens the Providers tab of their Shop page, and shows,
for every entry, what the provider will ask them to configure and what its machines can and cannot
do — before anything is installed.

Unlike Surge Packs, a provider is not defined in a YAML file. It is a package that runs **inside your Rocky Surf installation**,
with everything that process can reach: its database, its master key, and every cloud credential in
its environment. Rocky Surf states this clearly in the interface:

> a provider runs with Rocky Surf's full access — install ones you trust.

The onus is on you to test that there's no sneaky business going on with community-authored Providers distributed by this Shop.

Two things follow for anyone contributing an entry:

- **The artifact must be self-contained.** Rocky Surf never runs `npm`, never runs a lifecycle
  script, and never executes anything from the package at install time. So a tarball whose
  `dependencies` are not already present on the operator's machine is refused, naming them. Bundle
  what you import.
- **The `sha256` is checked**, by this repository's CI and again by every control plane that
  installs it. Update it whenever you publish a new version.

To add one, read [CONTRIBUTING.md](CONTRIBUTING.md#contributing-a-provider) and the authoring
guide, [`docs/writing-a-provider.md`](https://github.com/amroja-biz/rockysurf/blob/main/docs/writing-a-provider.md),
whose "Publishing to the shop" section has the `pnpm pack` recipe and the full entry format.

## Discord

Come and discuss community Surge Packs and Providers on the [Rocky Surf Discord](https://discord.gg/AbPsjNEmbh).

## License

MIT, matching [the main repository](https://github.com/amroja-biz/rockysurf/blob/main/LICENSE).
Pack files are configuration people are meant to read, copy, adapt and redistribute, and nothing
about them argues for terms different from the software that runs them — a contributor who has
already agreed to the main repository's terms should not have to read a second set here.

The Rocky Surf name and logo are **not** covered by that license; see
[TRADEMARK.md](https://github.com/amroja-biz/rockysurf/blob/main/TRADEMARK.md).
