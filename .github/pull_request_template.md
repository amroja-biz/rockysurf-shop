## What this pack installs

<!-- One or two sentences. What does someone get on their box, and who is it for? -->

## Checks

Run these locally before opening the pull request — CI runs the same commands, and `pack check`
takes a few minutes per architecture.

```bash
git clone --depth 1 https://github.com/amroja-biz/rockysurf /tmp/rockysurf

npx rockysurf pack lint  packs --base-packs /tmp/rockysurf/packs
npx rockysurf pack check packs --base-packs /tmp/rockysurf/packs --pack <your-pack-id> --arch arm64
npx rockysurf pack check packs --base-packs /tmp/rockysurf/packs --pack <your-pack-id> --arch amd64
```

The `--base-packs` clone is where the shared base toolchain lives; it ships with Rocky Surf and is
not in this repository.

- [ ] One file in `packs/`, named for its `packId`.
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
