# content — Agent Guide

## What this is

Data only. Machine-readable facts that more than one Fundaments.work product
must agree on. No build, no dependencies, no site, no framework.

```
support.json    canonical support channels, including the Buy Me Momo URL
```

This is a **source repo**, not a product. It ships nothing.

## The one rule

**One direction. Content pushes; consumers never write back.**

A consumer that can write here becomes a second source of truth. Two sources of
truth in different repositories is not a sync problem, it is a merge problem with
no resolution story. So: change a value here first, push it, then sync the
consumers. Never hand-edit a consumer's copy to "just fix it".

## Format

JSON, and it stays JSON. Consumers include Astro builds, a Vitest/vitest-free
Node script, a compiled Tauri app and an Obsidian plugin manifest.
`JSON.parse` is the only parser all of them already have; YAML would add a
dependency to every one of them.

Keys use lower camelCase (`buyMeMomo`) because Hugo lowercases param keys and
two spellings across repos is drift waiting to happen.

## What belongs here

| Belongs | Why |
| :--- | :--- |
| Values that must be identical across repos | One place to change, one place to review |
| Values with one obvious owner | There is no second opinion to gather |
| Public URLs | Never put a secret, token or private endpoint in this repo |

| Does not belong | Why |
| :--- | :--- |
| Prose belonging to one product | Each site owns its own voice |
| Anything countable that can be computed | Generate it in the consumer; a committed number goes stale |
| Anything a git tag already tells you | Read the tag, don't restate it |

## Consumers

| Repo | How it consumes | Gate |
| :--- | :--- | :--- |
| `fundaments-landing` | `pnpm sync:content` → `data/generated/support-channels.json` | `pnpm sync:content:check` |
| `theorem-landing` | `pnpm sync:content` → `data/generated/support-channels.json` | `pnpm sync:content:check` |
| `Theorem` | keeps the literal in `support-prompt.ts` + `.github/FUNDING.yml` | `pnpm check:content` |
| `Lemma` | keeps the literal in `manifest.json`, `.github/FUNDING.yml`, README, settings tab | `pnpm check:content` |

Consumers are read-only. If one of them looks wrong, the fix belongs here.

## Adding a field

1. Add it to the file, with a `$comment` explaining *why* the field exists and
   what would break if it were dropped.
2. Empty values are meaningful. A channel with `"url": ""` is a channel that is
   known but not yet configured; consumers hide it rather than dropping it, which
   is what keeps ordering stable across syncs.
3. Bump nothing. There is no version field. Consumers record a hash of this
   file's contents, and optionally the commit they read it at, in
   `content.lock.json`.
4. Update the consumer table above if you add a consumer.

## Changing something

```bash
# 1. edit here, commit, push
# 2. in each consuming repo
pnpm sync:content        # landing sites: regenerates the committed copy
pnpm check:content       # apps: fails until you update the literal
```

Push here **first**. A consumer that syncs from a sibling checkout records only
the content hash (`commit` stays `null`); one that reads from
`raw.githubusercontent` records the commit it read. Either way, syncing against an
unpushed edit locks in bytes that do not exist on the remote yet, and CI will
disagree with you.

Note that `commit: null` is not a weakness. What a site actually serves is its
own **committed** generated copy, so a change here cannot alter a deployed site
without a sync and a commit. What `null` buys is that CI reads `main` and
raises the alarm the moment this file changes — which is the drift signal you
want, rather than a pin that would never notice.