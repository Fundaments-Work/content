# Fundaments-Work / content

Machine-readable facts that more than one product needs to agree on. Today:
the support channels. Tomorrow: the product registry and generated stats.

This repo holds **data only**. No build, no dependencies, no site. Every file
here is consumed by another repository.

## The one rule

**One direction. Content pushes; consumers never write back.**

A consumer that can write here becomes a second source of truth, and two
sources of truth in different repositories is not a sync problem, it is a merge
problem with no resolution story. So:

- Change a URL here, then sync the consumers.
- Never hand-edit a consumer's copy to "just fix it". That is the drift this
  repo exists to prevent.

## What belongs here

| Belongs | Why |
| :--- | :--- |
| Values that must be identical across repos | One place to change, one place to review |
| Values with a single obvious owner | There is no second opinion to gather |
| Machine-readable files only | Every consumer must be able to parse it with zero dependencies |

| Does not belong | Why |
| :--- | :--- |
| Prose that belongs to one product | Each site owns its own voice |
| Anything countable that can be computed | Generate it in the consumer; a committed number goes stale |
| Anything a git tag already tells you | Read the tag, don't restate it |

JSON, not YAML, on purpose: consumers include Obsidian plugin manifests, Rust
and TypeScript test suites, and Astro builds. `JSON.parse` is the only parser
all of them already have.

## How a consumer syncs

Consumers must work three ways: in a local dev checkout, on CI, and on
Cloudflare Pages, where sibling directories do not exist. The rule that
satisfies all three:

1. Resolve a source — an explicit override, else a local checkout, else raw
   GitHub (at the commit recorded in the lock, else the default branch).
2. Write the generated copy into the consumer **and commit it**. The committed
   copy is the fallback when the network is unavailable; that is why a failed
   fetch must degrade to the committed file and never to a hardcoded value.
   It is also why this repo cannot change a deployed site on its own.
3. Record the content hash, and the commit when one was read, in
   `content.lock.json`.
4. Re-run the sync in CI and fail on any diff. Without this step the lock is
   decoration.

This repo **must be public**. Consumers fetch it unauthenticated over HTTPS, and
a private repo would return 404 in CI while looking fine locally.

## Consumers

| Repo | How it consumes | Gate |
| :--- | :--- | :--- |
| `fundaments-landing` | `pnpm sync:content` → `data/generated/support-channels.json` | `pnpm sync:content:check` |
| `theorem-landing` | `pnpm sync:content` → `data/generated/support-channels.json` | `pnpm sync:content:check` |
| `Theorem` | keeps the literal in `src/core/lib/support-prompt.ts` + `.github/FUNDING.yml` | `pnpm check:content` |
| `Lemma` | keeps the literal in `manifest.json`, `.github/FUNDING.yml`, README, settings tab | `pnpm check:content` |

The static sites generate a file because they build where this repo is not
present. The apps do not: an Obsidian plugin manifest and a compiled Tauri
bundle cannot take a build step for one URL without adding real fragility to the
release pipeline, so they keep the literal and a check holds it to the canonical
value.

The check is a script, not a vitest case. Comparing against a *remote* source
of truth needs the network; a unit test that needs the network stops being a
unit test, and the day it starts skipping on timeout it stops being a gate. So
it runs in CI (`Theorem/.github/workflows/ci.yml`, and before the Lemma release
build) where network access is a given, and `pnpm test` stays hermetic.

## Changing something

```bash
# 1. edit the file here, commit, push
# 2. in each consuming repo
pnpm sync:content        # landing sites: regenerates the committed copy
pnpm check:content       # apps: fails until you update the literal
```

Order matters: push here first. Syncing a consumer against an unpushed edit locks in bytes that do
not exist on the remote yet, and CI will disagree with you.

After you change a file here, the landing sites' CI gates start reading it and will report their
committed copy as stale until you re-run `pnpm sync:content` and commit the result. That is the
gate working, not a failure.