## What this pack installs

<!-- One or two sentences. What does someone get on their box, and who is it for? -->

## Checks

Run these locally before opening the pull request — CI runs the same commands, and `pack check`
takes a few minutes per architecture.

```bash
# Until Rocky Surf v0.1.0 is on npm, build the harness from source once:
git clone --depth 1 https://github.com/amroja-biz/rockysurf /tmp/rockysurf
(cd /tmp/rockysurf && pnpm install && pnpm --filter 'rockysurf...' build)
rs() { node /tmp/rockysurf/packages/rockysurf/dist/bin.js "$@"; }

rs pack lint  packs
rs pack check packs --pack <your-pack-id> --arch arm64
rs pack check packs --pack <your-pack-id> --arch amd64
```

The build step is temporary: Rocky Surf is not on npm until v0.1.0, so `npx rockysurf` fetches a
placeholder rather than the harness. CI builds it the same way. See CONTRIBUTING.md.

- [ ] One file in `packs/`, named for its `packId`.
- [ ] `index.json` regenerated and committed — `rs pack index --source packs --out index.json`.
      It is the listing every control plane reads; a pack missing from it reaches nobody, and CI
      checks the two agree.
- [ ] `pack lint` is clean.
- [ ] `pack check` passes on **both** `amd64` and `arm64`.
- [ ] Base tools are referenced by id, not redefined (see Rocky Surf's `packs/ai-coding-agents.yaml`).
- [ ] Downloads are pinned to a version or a checksum — not `@latest`, not a `main` branch.
- [ ] `runAs` is honest: every step declares the privilege it actually needs.
- [ ] `guide` tells the user what they must do by hand once the box is theirs.

## What your scripts fetch

<!-- List the URLs your install and setup scripts download from. A reviewer will read them, and
     writing them out here is faster for both of us than working them out from the diff. -->

## Anything you are unsure about

<!-- Genuinely useful. "I could not make step X idempotent without Y" gets a better answer here
     than after merge. -->
