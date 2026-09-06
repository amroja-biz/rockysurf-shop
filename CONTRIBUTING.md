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

If you would rather not run these by hand, the
[`contribute-surge-pack`](https://github.com/amroja-biz/rockysurf/tree/main/.agents/skills/contribute-surge-pack)
agent skill in the main repository runs exactly these commands, in this order, and stops at the
first one that fails.

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

## Contributing a provider

A provider is a different kind of contribution from a pack. A pack is a YAML file that describes
scripts to run on a server the operator creates. A provider is **a package of code that runs
inside the operator's Rocky Surf process**, with that process's database, master key and every
cloud credential in its environment. Rocky Surf shows one sentence wherever a provider appears,
*a provider runs with Rocky Surf's full access — install ones you trust*, and nothing in this
repository changes that.

This section is the complete procedure, from a built provider to a merged listing. It walks one
path in full: the provider's source lives in a GitHub repository you own, the built tarball is
attached to a GitHub release in that repository, and this repository's `providers.json` points
at that release. Other hosting options are at the end.

If your provider was written with an agent, the
[`contribute-provider`](https://github.com/amroja-biz/rockysurf/tree/main/.agents/skills/contribute-provider)
skill in the main repository automates every step below: it packs, checks the tarball, creates the
release, downloads the asset back to compare the digest, generates the listing entry from the
artifact rather than transcribing it, and opens the pull request. It refuses to open one for a
package that declares runtime dependencies or a release whose asset does not match the digest it
published. This section stays the contract; the skill follows it.

### What you are actually submitting

Two things end up in two places:

1. **The tarball** — the built package, produced by `npm pack`. It lives in **your** GitHub
   repository, as a release asset. This repository never holds it.
2. **The listing entry** — one JSON object in [`providers.json`](providers.json) in **this**
   repository, added by a pull request. It names your package, the URL of the tarball, the
   tarball's SHA-256 digest, the settings your provider asks for, and its capability answers.

An operator reads the entry, downloads the tarball from your release, checks the digest, extracts
it under their data directory, names it in their config file and restarts. Nothing in Rocky Surf
fetches the listing or installs the package; the install is the operator's own hands, following
[Installing one](README.md#installing-one).

### Prerequisites

- A GitHub account and a **public** repository containing your provider's source. Rocky Surf
  operators are being asked to run your code with full access; they need to be able to read it.
- The [`gh` CLI](https://cli.github.com/) installed and logged in (`gh auth status`). Every step
  below has a browser equivalent, noted where it differs.
- Node 24 or newer, and a provider that builds and passes the acceptance suite,
  `@rockysurf/provider-conformance`. The authoring contract is
  [`docs/writing-a-provider.md`](https://github.com/amroja-biz/rockysurf/blob/main/docs/writing-a-provider.md);
  the `add-provider` agent skill in the main repository's `.agents/skills/` walks an agent through
  it.
- Your `package.json` declares **no runtime `dependencies`**. The documented install is `tar -xzf`
  and nothing else: no `npm install`, no lifecycle script, nothing resolves a dependency for the
  operator. Anything your manifest lists under `dependencies` is an import that throws at their
  next start. Bundle what you use into `dist/` instead. The worked example is
  [`@rockysurf/provider-digitalocean`](https://github.com/amroja-biz/rockysurf/tree/main/packages/provider-digitalocean),
  which compiles the SDK helpers it uses into its own output with `esbuild` and keeps the SDK as a
  `devDependency`.

The rest of this section uses a provider named `mycloud` in a repository `you/rockysurf-provider-mycloud`
with a package name `@you/rockysurf-provider-mycloud` at version `1.0.0`. Substitute yours.

### Step 1 — build, pack, and check the tarball

In your provider's repository:

```bash
npm run build
npm pack
```

`npm pack` writes `you-rockysurf-provider-mycloud-1.0.0.tgz` into the current directory (a scoped
name has its `@` dropped and the `/` turned into `-`). Check it before going any further:

```bash
tar -tzf you-rockysurf-provider-mycloud-1.0.0.tgz
```

Every path is under `package/`. Confirm that the file your manifest's `exports` (or `main`) points
at is in the list, normally `package/dist/index.js`. A tarball that carries `package.json` and no
`dist/` is the most common way a publish goes wrong: Rocky Surf refuses it at the operator's next
start with *is the package built?*

Then confirm the manifest inside the tarball declares no runtime dependencies:

```bash
tar -xzOf you-rockysurf-provider-mycloud-1.0.0.tgz package/package.json | grep -A3 '"dependencies"'
```

No output is the right answer. You do not need to hash the file yourself: Step 3 digests the
file it reads, which is the only digest worth publishing.

### Step 2 — publish the tarball as a GitHub release

A GitHub release is a tag in your repository with files attached to it. Each attached file gets a
permanent `https://` download URL, which is what the listing needs.

Commit everything, then create the tag and the release with the tarball attached:

```bash
# In a repository that holds only this provider. In a monorepo, use a package-scoped tag such
# as provider-mycloud-v1.0.0 on all three lines, so a bare version tag stays free for the repo.
git tag v1.0.0
git push origin v1.0.0
gh release create v1.0.0 ./you-rockysurf-provider-mycloud-1.0.0.tgz \
  --title "@you/rockysurf-provider-mycloud 1.0.0" \
  --notes "First release. Install per https://github.com/amroja-biz/rockysurf-shop#installing-one"
```

In the browser instead: open your repository, **Releases** → **Draft a new release**, choose or
create the tag `v1.0.0`, attach the `.tgz` under **Attach binaries**, and **Publish release**.

The download URL for an attached file always has this shape:

```
https://github.com/<owner>/<repo>/releases/download/<tag>/<file name>
```

so for this example it is:

```
https://github.com/you/rockysurf-provider-mycloud/releases/download/v1.0.0/you-rockysurf-provider-mycloud-1.0.0.tgz
```

Prove the URL serves the exact bytes you hashed. Download it the way an operator will and compare
the digest with Step 1's:

```bash
curl -fLO https://github.com/you/rockysurf-provider-mycloud/releases/download/v1.0.0/you-rockysurf-provider-mycloud-1.0.0.tgz
shasum -a 256 you-rockysurf-provider-mycloud-1.0.0.tgz
```

If the digests differ you attached a different file than you hashed; fix the release before
touching the listing. Do not delete or re-upload a release asset after the listing is merged:
operators compare the digest, and a changed file fails their install. A new version is a new tag
(Step 5).

### Step 3 — generate the listing entry

Fork this repository and clone your fork:

```bash
gh repo fork amroja-biz/rockysurf-shop --clone
cd rockysurf-shop
git checkout -b providers/mycloud
```

**Do not write the entry by hand.** Nine fields, and seven of them are already inside the tarball
you built in Step 1 — a settings summary copied field by field out of a declaration, and a
capability struct copied out of a source file, are two transcriptions that go stale silently.
`@rockysurf/provider-sdk` ships a command that reads them:

```bash
npx rockysurf-shop-entry ~/src/mycloud/you-rockysurf-provider-mycloud-1.0.0.tgz \
  --tarball-url https://github.com/you/rockysurf-provider-mycloud/releases/download/v1.0.0/you-rockysurf-provider-mycloud-1.0.0.tgz \
  --description "MyCloud compute, one API token, four regions."
```

Point it at the **same `.tgz` you attached to the release** — the digest it prints is of the bytes
it read. It writes the entry to stdout and nothing else, so it pipes into `pbcopy`, into `jq`, or
straight into your editor:

```json
{
  "providerId": "mycloud",
  "name": "MyCloud",
  "description": "MyCloud compute, one API token, four regions.",
  "version": "1.0.0",
  "package": "@you/rockysurf-provider-mycloud",
  "tarball": "https://github.com/you/rockysurf-provider-mycloud/releases/download/v1.0.0/you-rockysurf-provider-mycloud-1.0.0.tgz",
  "sha256": "227011c38b5a4033cfafbf7797692d763ba81c25ef5e6141f90d03705236723d",
  "settings": [
    { "name": "token", "label": "API token variable", "kind": "secret" },
    { "name": "region", "label": "Region", "kind": "string" }
  ],
  "capabilities": {
    "stop": true,
    "ipStableAcrossStop": false,
    "canInjectHostKeys": false,
    "generatesUserData": false,
    "userDataMaxBytes": 0,
    "managesSshAccess": true,
    "billsWhileStopped": true
  }
}
```

Paste that object into the `providers` array in `providers.json`.

**Only the two options are yours to write.** Everything else was read out of the artifact, and the
table below says from where — you are still the one signing the pull request, so it is worth
knowing what each value is a reading of:

| Field | Where its value comes from |
|---|---|
| `providerId` | Your factory's `id`. It becomes the operator's `providers.<id>:` config section key. |
| `name` | Your factory's `settings.title` — the heading an operator sees over these very fields once it is installed — falling back to `displayName`. |
| `description` | **You**, in `--description`. One line, and the first thing an operator reads. |
| `version` | The `version` in the manifest inside the tarball, which is the tag you released. |
| `package` | The `name` in that manifest. The operator writes it on the `package:` line of their config. |
| `tarball` | **You**, in `--tarball-url`. `https` only; `http` is refused before you can open the pull request. |
| `sha256` | The digest of the bytes the command read. Hash nothing yourself. |
| `settings` | Your factory's declared `settings.fields`, in declared order, reduced to `name`, `label` and `kind`. `enabled`, `package` and `sizes` are left out; every panel gets those. |
| `capabilities` | The capabilities of the provider your factory's `createProvider()` returns. The required five, plus `managesSshAccess`, `billsWhileStopped` and `simulatedInstances` when your factory sets them. `billsWhileStopped` is how somebody learns, before installing, that a stopped machine on your cloud still costs money. |

The command refuses two things here rather than letting this repository's CI find them: a package
whose manifest declares runtime `dependencies`, naming them, and a `--tarball-url` that is not
`https`.

Do not add any other field. There is no `trust`, `tier` or `verified`: a registry never publishes a
trust label, and the validator rejects one.

Set `generatedAt` at the top of the file to today's date, in the same ISO-8601 form it already has.

Run the validator. It is the first thing CI runs and it needs no install:

```bash
node scripts/validate-providers.mjs providers.json
```

It prints `providers.json: N provider entries, all valid` or lists every problem with the field it
is in.

> If `npx rockysurf-shop-entry` cannot resolve — Rocky Surf has not published to npm yet — get
> `@rockysurf/provider-sdk` from a checkout of the main repository (`pnpm pack` in
> `packages/provider-sdk`) and install that tarball; it is the same artifact the release will
> publish.

### Step 4 — open the pull request

```bash
git add providers.json
git commit -m "providers: add mycloud 1.0.0"
git push -u origin providers/mycloud
gh pr create --repo amroja-biz/rockysurf-shop --base main \
  --title "providers: add mycloud 1.0.0" \
  --body "Source: https://github.com/you/rockysurf-provider-mycloud. Conformance suite passes at <commit>."
```

Put the link to your source repository in the pull request body. A reviewer will read the code,
because reading it is the only review a provider gets; the validator checks the shape of a
description and can say nothing about what the description points at.

CI runs the `provider listing` workflow on the pull request. It runs the validator, then downloads
every tarball named in `providers.json` and compares its digest with the `sha256` beside it. A
stale digest is therefore caught here, in your pull request, rather than by an operator.

When a maintainer merges, your entry is live. There is no build and no index step for providers:
the README's [Providers](README.md#providers) section points operators at `providers.json`, and
they take the `package`, `tarball` and `sha256` from your entry and follow
[Installing one](README.md#installing-one).

### Step 5 — publishing a new version

Repeat Steps 1 and 2 with the new version number and a new tag (`v1.1.0`, a new release, the new
tarball attached). Then re-run the Step 3 command on the new tarball and replace your whole entry with what it
prints, rather than editing three fields: a new version can change the settings summary and the
capabilities too, and regenerating is the only way those stay in step. Bump `generatedAt`. Leave the old release in place: an operator who already installed
1.0.0 keeps working from it.

An operator updates by extracting the new tarball over the installed directory and restarting,
which deletes nothing. Say in your release notes if a file has moved or gone.

### Other places to host the tarball

The listing needs a URL that serves the exact bytes you hashed, over `https`, indefinitely.
Anything that does is acceptable. The two other common choices:

- **npm.** Publish the package (`npm publish --access public` under a scope you own). The
  registry serves the tarball at
  `https://registry.npmjs.org/@you/rockysurf-provider-mycloud/-/rockysurf-provider-mycloud-1.0.0.tgz`
  (scope in the first path segment, no scope in the file name). Download it and hash it exactly as
  in Step 2; the digest is of the bytes the registry serves, not of your local `npm pack`
  output, and those are usually but not always identical.
- **Any static host** you control that serves the file over `https` with a stable URL. Plain
  `http` fails the validator: the artifact and the digest meant to catch a change to it would
  travel over the same rewritable connection.

### What the checks do not prove

The validator checks the shape of a description. The digest check proves the tarball at the URL
is the one you described. Neither says anything about what the code does when it runs, and
nothing here can: that code runs with an operator's full access. Review is a person reading your
repository, and the control that matters most is the operator's own decision, made at their own
command line with that one sentence in front of them.

## License

By contributing you agree your pack is licensed under the [MIT license](LICENSE), the same terms
as the main repository. The Rocky Surf name and logo are not covered by it. A provider you list
here stays under whatever licence you publish it with — this repository holds only the entry that
describes it.
