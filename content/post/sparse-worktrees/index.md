---
title: Sparse worktrees on a 6 GB repo
description: What I tried, what worked, what didn't, and whether it's worth doing
slug: sparse-worktrees
date: 2026-06-11 00:00:00+1000
image: cover.png
categories:
  - Development
  - AI
tags:
  - Git
  - Worktrees
  - Sparse Checkout
  - Claude
  - Developer Experience
weight: 1
links:
  - title: "git-worktree"
    description: The git docs for worktree
    website: https://git-scm.com/docs/git-worktree
  - title: "git-sparse-checkout"
    description: The git docs for sparse-checkout (cone mode)
    website: https://git-scm.com/docs/git-sparse-checkout
---

## The problem

The pattern was familiar. A long-running Claude session would be mid-task on my feature branch when review feedback landed on an open PR. The options were either to wait for Claude to finish before switching, or to stash, switch branches, address the feedback, switch back, unstash, and try to recreate the context Claude had been holding. Doing it once was fine. Three times in an afternoon was the kind of friction worktrees exist to solve.

Except worktrees were off the table on this project. The `CLAUDE.md` is explicit:

```markdown
## Worktrees

- Prefer working directly on a branch — large file count makes worktree checkout slow
- Never build inside a worktree — solution is too large, will exhaust disk space
```

For most of the team that's the right default. A fresh clone alone is around 6.2 GB: 3.6 GB of working-tree files plus 2.6 GB of `.git` history. `npm ci` adds 2.1 GB more for the React frontend's `node_modules`, and a NuGet restore plus backend build adds several more GB:

{{< stacked-bar title="Disk cost of one fully-provisioned worktree" total="15.3" total-label="≈ 15+ GB before any actual work happens" >}}
{{< segment label="Working tree · 3.6 GB" value="3.6" class="good" >}}
{{< segment label=".git · 2.6 GB" value="2.6" class="soft" >}}
{{< segment label="node_modules · 2.1 GB" value="2.1" class="mid" >}}
{{< segment label="Backend deps + build · ~7 GB" value="7" class="bad" >}}
{{< /stacked-bar >}}

Three concurrent worktrees, all provisioned at once, would consume more than 30 GB before any actual work happened.

What pushed me to revisit the rule was the desire to run multiple Claude agents in parallel. Branch switching on a single checkout forces those agents into serial execution, which is the opposite of why you'd spawn multiple. The `CLAUDE.md` rule existed for good reasons, but it was now also the thing blocking one of the more interesting use cases I had for the project.

## Sparse checkouts

The obvious first lever was git's sparse-checkout in cone mode. Restricting the checked-out tree to just the directories I actually touch (for the frontend role, that's roughly the React app and its build tooling) takes the 3.6 GB of working-tree files down to about 1.8 GB. The `.git` directory and the dependency installs are untouched; sparse-checkout only changes which files get materialised on disk.

{{< bar-chart title="Working tree size for the frontend role" max="3.6" >}}
{{< bar label="Full checkout" value="3.6" annotation="3.6 GB" class="fail" >}}
{{< bar label="Sparse checkout (cone)" value="1.8" annotation="1.8 GB · 50% less" class="pass" >}}
{{< /bar-chart >}}

This _partially_ solves the problem. The clone is still ~6 GB, but at least the working tree shrinks. For this repo that's still meaningful; on a different repo where the clone itself isn't oversized, the source-tree savings might not matter much.

What sparse-checkout _doesn't_ touch is the expensive part: a new worktree's first run still pays a full `npm ci` (~50 seconds, ~2 GB) and a NuGet restore plus first build (multiple minutes). The source-tree savings were real, but a rounding error against the dependency cost.

## Copy-on-write

I was about to file sparse-checkout away as a half-win when someone I work with poked at it from a different direction. Their suggestion was roughly this: combine the sparse worktree with symlinks, so that for the projects you weren't actively touching in a given worktree, the `bin/obj` directories pointed back at the primary checkout's build output instead of being rebuilt from scratch.

Symlinks alone wouldn't work. A write through the symlink modifies the primary's `bin/obj` and corrupts your primary checkout. What I actually wanted was _share-by-default-but-diverge-on-write_, which is exactly what copy-on-write filesystems do. macOS's APFS exposes it as `clonefile(2)`, Windows ReFS calls it `block-clone`, Linux's btrfs and xfs have `reflink`. All produce a copy that shares storage blocks with the source until either side writes, when fresh blocks get allocated transparently.

For Node this was easy to reason about. If the worktree's `package-lock.json` matches the primary's, the dependencies are byte-identical, so `node_modules` can be cloned in one syscall. For .NET it was messier (restored packages, generated code, multiple `bin/obj` paths, projects that are inside the cone versus borrowed from outside) and required several iterations to get right.

But the shape of the idea was clear. Sparse-checkout addresses the source-tree size. CoW addresses the dependency and build-output cost. Together they make worktree creation feel near-instant.

## Wiring it together

I ended up with a bash script wrapping three steps:

1. `git worktree add` the new branch into a sibling directory.
2. `git sparse-checkout init --cone` and set the cone to the relevant role's directories.
3. If the worktree's lockfile matches the primary's, CoW-clone `node_modules`. Otherwise fall back to `npm ci`. Same pattern for the .NET dep and build dirs.

One implementation wrinkle worth noting: `cp -c -R` produces CoW blocks on macOS but walks the tree file-by-file, taking ~22 seconds for `node_modules`. Calling `clonefile(2)` directly on the directory does it in 2–4 seconds.

{{< bar-chart title="Cloning node_modules: clonefile syscall vs cp -c -R walk" max="22" >}}
{{< bar label="cp -c -R (walks tree, ~166k syscalls)" value="22" annotation="~22 s" class="fail" >}}
{{< bar label="clonefile(2) (one syscall on the directory)" value="4" annotation="~4 s · ~6× faster" class="pass" >}}
{{< /bar-chart >}}

On Windows, ReFS `block-clone` only fires through specific APIs. `cp -r` in Git Bash silently bypasses it and produces a real copy, which is how the first Windows run produced 22 GB of physical disk per worktree.

The script also grows the cone organically: when an agent runs into a missing import during testing, it broadens the cone and tries again.

## How it went

### On paper

{{< stat-grid >}}
{{< stat label="Sparse + CoW" value="~4" unit="s" sub="when lockfile matches primary" >}}
{{< stat label="Full + npm ci" value="~50" unit="s" sub="the prior baseline" >}}
{{< stat label="Per-worktree disk" value="~170" unit="MB" sub="vs ~3.9 GB full" >}}
{{< stat label="Speedup" value="~12" unit="×" sub="time, default fast path" >}}
{{< /stat-grid >}}

{{< bar-chart title="Cost to create one ready-to-use worktree" max="50" >}}
{{< bar label="Full worktree + npm ci" value="50" annotation="~50 s · 3.9 GB" class="fail" >}}
{{< bar label="Sparse + npm ci fallback" value="16" annotation="~16 s · 2.2 GB" class="pass" >}}
{{< bar label="Sparse + CoW (cp -c -R)" value="22" annotation="~22 s · 170 MB" class="pass" >}}
{{< bar label="Sparse + CoW (clonefile)" value="4" annotation="~4 s · 170 MB" class="pass" >}}
{{< /bar-chart >}}

The top row is the baseline I'd have paid without any of this. Everything below it is an improvement, with the bottom row being the fast path: CoW via `clonefile`, which fires when the lockfile matches primary — most of the time.

### In practice

The more honest test was whether the workflow held up under actual use. The most relevant case was a Friday afternoon when three open PRs had review comments waiting and I was mid-feature on an unrelated branch. Claude handled the whole loop end to end: spinning up three sparse worktrees (one per PR), addressing the feedback in each, and tearing the worktrees down once I'd reviewed the changes.

{{< bar-chart title="Wall-clock: three PR-feedback agents, parallel vs sequential" max="18" >}}
{{< bar label="Sequential (one agent at a time)" value="18" annotation="~18 min" class="fail" >}}
{{< bar label="Parallel (3 worktrees, 3 agents)" value="7.5" annotation="~7.5 min · 2.4× speedup" class="pass" >}}
{{< /bar-chart >}}

Worktree creation totalled 25.7 seconds across all three. Seven and a half minutes of wall-clock later, three agents had clean commits with full test suites passing. Sequentially this would have been roughly 18 minutes of agent time plus a stash dance between each one. The primary checkout stayed parked on its feature branch the whole time.

### The .NET side

The .NET path was lower-traffic for me personally (most of my day-to-day work is the frontend), but the validation numbers held up. A clean build of the runnable web app's role-scoped solution took around 53 seconds, against 274 seconds for the full solution. Three concurrent .NET worktrees occupied 68 GB of apparent disk while consuming only ~3.76 GB of actually-allocated space, thanks to ReFS `block-clone`. About 18× compression on the borrowed `bin/obj` files.

{{< bar-chart title="Backend build (clean + rebuild)" max="274" >}}
{{< bar label="Full solution (162 projects)" value="274" annotation="~274 s" class="fail" >}}
{{< bar label="Role-scoped .slnf (45 projects)" value="53" annotation="~53 s · ~5× faster" class="pass" >}}
{{< /bar-chart >}}

The wrapper also generates a Visual Studio Solution Filter (a `.slnf`) as a byproduct of the cone, which slotted in cleanly. Pointed at the `.slnf`, Visual Studio only loads the role's 45 projects instead of the full 162. The startup project built and ran fine from it, but ran into some issues at runtime. I figured this was likely due to some of the projects missing, but didn't dig too much further as the majority of my pressing work was all front-end related.

The takeaway here is that making changes from a narrow cone is cheap, running and verifying them isn't.

## Where it gets rough

The big one is that the cone is correct for editing but reliably too narrow for running tests. In the .NET case, sometimes too narrow for running the solution at all. Frontend test runners walk real imports across the source tree, and a diff-derived cone only contains the files in the PR. The first version of the script left agents bouncing off "module not found" errors until they manually broadened the cone.

The fix was a hybrid strategy: organic cone growth that broadens on resolution failures, combined with role-based "always include" presets so the common cases avoid the dance entirely. The frontend cone, for instance, always pulls in `webpack/`, `config/`, and a handful of others.

This helps, but doesn't eliminate the cost. There's still a meaningful gap between "small enough cone to be fast" and "broad enough cone to validate the change locally." In practice, for many quick PR-feedback rounds, the path of least friction became: make the change, push, let CI validate. _The cone makes making changes cheap and makes validating them awkward._

The .NET path works, but the implementation isn't as clean as the frontend's. Where the frontend story is a couple of hundred lines of mostly-generic bash, the .NET equivalent is several times that, and most of the extra exists to handle this repo's particular shape: code-gen targets that hardcode output paths, vendor DLL references via `<HintPath>`, `packages.config`-style restoration, and a few locally-modified project files that need to be carried into each worktree to produce binaries that match the primary checkout.

The tooling on that side is closer to a project-specific wrapper than a portable library.

## Was it worth it?

A few weeks on, the picture's more mixed than I expected. The experiment did what it set out to do: parallel Claude agents on a repo that had been explicitly deemed unsuitable for worktrees. But folding the workflow into daily work was more involved than I'd hoped. I reach for sparse worktrees for specific cases now (PR-feedback rounds, parallel small features), not as the default.

When I do reach for it, the frontend path is the cleaner of the two: per-worktree cost is low enough that spinning one up is a non-decision. The .NET path works but it's a brittle, repo-specific win; I wouldn't expect that side to lift to another .NET codebase without similar investigative work upfront. The cone-versus-validation tension still bites too – it's a real ceiling, not a minor footnote, and it shows up whenever a test runner reaches further than a diff would suggest.

Would I recommend this approach on a more reasonably-sized repo?

![Nope](images/season-3-no-gif-by-the-office.gif)

If worktrees are already cheap to spin up, there's little reason to layer this on top, especially with a package manager that deduplicates dependencies the way `pnpm` does.

The bigger takeaway is less about this specific tooling than how a codebase gets set up in the first place. Most of the friction I kept running into wasn't from sparse-checkout or copy-on-write. Those did their jobs. It came from the codebase not having been designed with worktrees in mind: a port assumption here, an environment-file shortcut there, build state baked into the wrong place. None of that is AI-specific; a human dev wanting to work on two branches in parallel would have hit the same edges. But in a world where running multiple Claude agents against the same repo in parallel is a meaningful unlock, the value of considering worktrees from the outset is higher than it's ever been.
