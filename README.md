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
A Surge Pack a bag of software defined in a YAML file. It's a core part of Rocky Surf and makes it dead easy to spin up cloud servers on the Provider of your choice, pre-installed with your favorite agent harness.

There are three types of Surge Packs:

- Official. These are bundled into Rocky Surf.
- Personal. Surge Packs that you create using the [create-surge-pack](https://github.com/amroja-biz/rockysurf/tree/main/.agents/skills/create-surge-pack) skill from Rocky Surf.
- Community. What you'll find here.

### Contributing a pack

Read [CONTRIBUTING.md](CONTRIBUTING.md). In short:

#### Your pack defines its own tools

**A pack can install anything.** You are not picking from a list, and nothing has to be added to
Rocky Surf first. A tool is just an id, a description, and an install script. For example:

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

Surge Packs contributed to this repo are tested for validity so use the skill instead of rolling your own. This will save you time.

#### Reusable core

*Do not* redeclare tools that Rocky Surf automatically installs on every box:
`curl`, `git`, `gh`, `nodejs`, `tmux`, `build-essential`, `jq` and maybe others over time. 

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

The authoring contract itself is
[`docs/writing-a-pack.md`](https://github.com/amroja-biz/rockysurf/blob/main/docs/writing-a-pack.md)
in the main repository. It is normative, it has worked examples of the right and wrong way to
write each rule, and it is the document to read first.

`index.json` is generated, and **your pull request should carry its own regeneration** —
`npx rockysurf pack index --source packs --out index.json`. It is the listing every control plane
reads, so a pack missing from it reaches nobody. CI checks the committed index against `packs/`
and names that command if the two disagree. Do not hand-edit it; regenerate it.

Again, you should be fine if you use the provided skill.

### Naming Surge Packs

Name your pack for what it does. `rust-dev`, `data-science`, `elixir-phoenix` — all fine. Please do not
name it in a way that reads as official, and do not use the Rocky Surf name or logo as your own
branding. See the main repository's
[TRADEMARK.md](https://github.com/amroja-biz/rockysurf/blob/main/TRADEMARK.md), which governs
this repository too. 

## Providers

A **Provider** is the code that talks to a cloud — how Rocky Surf creates, stops, describes and
terminates a machine there. Five ship inside Rocky Surf (AWS, Azure, GCP, Hetzner and
bring-your-own); anyone can write another against the published provider SDK, and this registry is
how one reaches other people's installations.

As with Surge Packs, there's a handy [add-provider]() agent skill to do this for you. 

Community-contributed Providers are listed in [`providers.json`](providers.json) that
Rocky Surf fetches when an operator opens the Providers tab of their Shop page. This shows the 
user details about what configuration requirements and relevant information.

Unlike Surge Packs, a provider is not defined in a YAML file. It is a package that runs **inside your Rocky Surf installation**,
with everything that process can reach: its database, its master key, and every cloud credential in
its environment. Rocky Surf states this clearly in the interface:

> a provider runs with Rocky Surf's full access — install ones you trust.

The onus is on you to test that there's no sneaky business going on with community-authored Providers distributed by this Shop.

For contributors:

- **The artifact must be self-contained.** Rocky Surf doesn't run anything from the package at install time. So a tarball whose
  `dependencies` are not already present on the operator's machine is refused. Hence, you need to bundle imports.
- **The `sha256` is checked**, by this repository's CI and again by the Rocky Surf control plane. That means the sha256 needs 
  to be updated whenever you publish a new version.

To submit a Provider to this shop, read [CONTRIBUTING.md](CONTRIBUTING.md#contributing-a-provider) and the authoring
guide, [`docs/writing-a-provider.md`](https://github.com/amroja-biz/rockysurf/blob/main/docs/writing-a-provider.md).

## Discord

Come and discuss community Surge Packs and Providers on the [Rocky Surf Discord](https://discord.gg/AbPsjNEmbh).

## License

MIT, matching [the main repository](https://github.com/amroja-biz/rockysurf/blob/main/LICENSE).
Pack files are configuration people are meant to read, copy, adapt and redistribute, and nothing
about them argues for terms different from the software that runs them — a contributor who has
already agreed to the main repository's terms should not have to read a second set here.

The Rocky Surf name and logo are **not** covered by that license; see
[TRADEMARK.md](https://github.com/amroja-biz/rockysurf/blob/main/TRADEMARK.md).
