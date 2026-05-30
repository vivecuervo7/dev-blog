---
title: Sparse worktrees on a 6 GB repo
description: What I tried, what worked, what didn't, and whether it's worth doing
slug: sparse-worktrees
date: 2026-05-30 00:00:00+1000
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
  - title: "Partial clone"
    description: How --filter=blob:none changes what you download
    website: https://git-scm.com/docs/partial-clone
---

## The problem

The pattern that pushed me into this was a familiar one. A long-running Claude session would be mid-task on my feature branch, and partway through, review feedback would land on an open PR I'd raised a day or two earlier. The options were either to wait for Claude to finish before switching, or to stash, switch branches, address the feedback, switch back, unstash, and try to pick up where Claude had left off. Doing this once was fine. Doing it three times in an afternoon, across an agent that's holding context I now have to recreate, was the kind of friction worktrees exist to solve.

Except worktrees were off the table on this project. The `CLAUDE.md` is explicit:

```markdown
## Worktrees

- Prefer working directly on a branch — large file count makes worktree checkout slow
- Never build inside a worktree — solution is too large, will exhaust disk space
```

For most of the team that's the right default. A fresh clone is around 6.2 GB, `npm ci` adds another 2.1 GB to the React frontend's `node_modules`, and a full backend build piles several more GB on top across `packages/`, `bin/`, and `obj/`. Three concurrent worktrees, fully provisioned, would consume more than 30 GB before any actual work happened.

What pushed me to revisit the rule was the desire to run multiple Claude agents in parallel. Branch switching on a single checkout forces those agents into serial execution, which is the opposite of why you'd spawn multiple. The `CLAUDE.md` rule existed for good reasons, but it was now also the thing blocking one of the more interesting use cases I had for the project.

## A first instinct: sparse checkouts

The obvious first lever was git's sparse-checkout in cone mode. Restricting the checked-out tree to just the directories I actually touch in a given session (for the frontend role, that's roughly the React app and its build tooling) drops the working tree from 3.6 GB to about 1.8 GB.

{{< figure src="images/sparse-checkout-working-tree.png" alt="Bar chart: working tree size before and after sparse-checkout for the frontend role" >}}

This _partially_ solves the problem. The clone is still ~6 GB, but at least the working tree shrinks. For this repo that's still meaningful; on a different repo where the clone itself isn't oversized, the source-tree savings might not matter much.

What sparse-checkout _doesn't_ help with is the expensive part of "ready to use": the `node_modules` directory you need to run the frontend, and the `packages/`, `bin/`, and `obj/` directories MSBuild needs to build the backend. Even with a tight cone, a new worktree's first run still pays a full `npm ci` (~50 seconds, ~2 GB) and a full NuGet restore plus first build (multiple minutes). The disk savings on source files were real, but they were the rounding error against the dependency and build artifact cost. I was about to file sparse-checkout away as a half-win, when a coworker poked at it from a different direction.

## The question that changed it

The framing was roughly: _"I wonder what would happen if you did the sparse worktree combined with symlink the `obj/bin` folders of all the projects you aren't touching back to the build output of the main checkout."_

Symlinks alone wouldn't work — a write through the symlink modifies the primary's bin/obj, which corrupts your primary checkout. What I actually wanted was _share-by-default-but-diverge-on-write_. Which is exactly what copy-on-write filesystems give you for free. macOS's APFS exposes it as `clonefile(2)`; Windows ReFS calls it block-clone; Linux's btrfs and xfs have `reflink`. All produce a copy that initially shares the underlying storage blocks with the source, then transparently allocates fresh blocks when either side writes.

For Node this was easy to reason about. If the worktree's `package-lock.json` matches the primary's, the dependencies are byte-identical, so `node_modules` can be cloned in one syscall. For .NET it was messier (restored packages, generated code, multiple `bin/obj` paths, projects that are inside the cone versus borrowed from outside) and required several iterations to get right. The journal entries and validation reports have the gory detail.

But the shape of the idea was clear. Sparse-checkout addresses the source-tree size. CoW addresses the dependency and build-output cost. Together they make worktree creation feel near-instant. _This was the win we'd identified._

## The shape of the solution

I ended up with a bash script wrapping three steps:

1. `git worktree add` the new branch into a sibling directory.
2. `git sparse-checkout init --cone` and set the cone to the relevant role's directories.
3. If the worktree's lockfile matches the primary's, CoW-clone `node_modules`. Otherwise fall back to `npm ci`. Same pattern for the .NET dep and build dirs.

One implementation wrinkle worth noting: `cp -c -R` produces CoW blocks on macOS but walks the tree file-by-file, taking ~22 seconds for `node_modules`. Calling `clonefile(2)` directly on the directory does it in 2–4 seconds.

{{< figure src="images/clonefile-vs-cp.png" alt="Bar chart: clonefile syscall vs cp -c -R walk for cloning node_modules" >}}

On Windows, ReFS block-clone only fires through specific APIs. `cp -r` in Git Bash silently bypasses it and produces a real copy, which is how the first Windows run produced 22 GB of physical disk per worktree.

The script also grows the cone organically: when an agent runs into a missing import during testing, it broadens the cone and tries again. More on that under caveats.

## Findings

{{< figure src="images/single-worktree-cost.png" alt="Bar chart: cost to create one ready-to-use worktree across four approaches" >}}

The fast row (Sparse + CoW via `clonefile`) is the path that fires when the lockfile matches the primary's, which is most of the time. The two middle rows are slower fallbacks. The full-worktree row at the bottom is what I'd have paid without any of this.

The numbers are nice on their own. The more honest test was whether the workflow held up under actual use. The most relevant case was a Friday afternoon when three open PRs had review comments waiting and I was mid-feature on an unrelated branch. I spun up three sparse worktrees, one per PR, and ran a Claude agent in each.

{{< figure src="images/parallel-pr-feedback.png" alt="Bar chart: wall-clock for three PR-feedback agents in parallel versus the equivalent sequential time" >}}

Worktree creation totalled 25.7 seconds across all three. Seven and a half minutes of wall-clock later, three agents had clean commits with full test suites passing (730/730, 743/743, 726/726). Sequentially this would have been roughly 18 minutes of agent time plus a stash dance between each one. The primary checkout stayed parked on its feature branch the whole time.

A separate run a few days later did something similar with three small features I'd been batching: three independent branches, three worktrees, parallel work, primary untouched. Same shape of result.

## Caveats

The big one is that the cone is correct for editing but reliably too narrow for running tests. Frontend test runners walk real imports across the source tree, and a diff-derived cone only contains the files in the PR. The first version of the script left agents bouncing off "module not found" errors until they manually broadened the cone.

The fix was a hybrid strategy: organic cone growth that broadens on resolution failures, combined with role-based "always include" presets so the common cases avoid the dance entirely. The frontend cone, for instance, always pulls in `webpack/`, `config/`, and a handful of others.

This helps, but doesn't eliminate the cost. There's still a meaningful gap between "small enough cone to be fast" and "broad enough cone to validate the change locally." In practice, for many quick PR-feedback rounds, the path of least friction became: make the change, push, let CI validate. _The cone makes making changes cheap and makes validating them awkward._

The .NET path works, but the implementation isn't as clean as the frontend's. Where the frontend story is a couple of hundred lines of mostly-generic bash, the .NET equivalent is several times that, and most of the extra exists to handle this repo's particular shape: code-gen targets that hardcode output paths, vendor DLL references via `<HintPath>`, `packages.config`-style restoration, and a few locally-modified project files that need to be carried into each worktree to produce binaries that match the primary checkout.

The tooling on that side is closer to a project-specific wrapper than a portable library. The win is real, but it doesn't lift cleanly — anyone applying the same idea to a different .NET codebase would need their own investigative pass to find the equivalent set of quirks.

## Conclusion

A week of daily use in, sparse worktrees have stuck on this repo. The original goal — running multiple Claude agents in parallel against the same project without serialising them on a single working tree — is now part of how I work, not a thought experiment. The interesting outcome isn't that parallel worktrees are possible (they always were), it's that they're possible on a repo that had been explicitly deemed unsuitable for them. _That reframing was the actual unlock._

The frontend story is the cleanest part of the win. Per-worktree cost is low enough that I reach for one without thinking, and the workflow scales to however many parallel agents I want to spawn.

The .NET story is also a win, but a brittle one. It works because this codebase's project structure happens to allow a cone that builds independently, and the tooling around it is shaped specifically to this repo. I wouldn't expect this part to port to another .NET codebase without similar investigative work upfront.

Would I recommend this approach on a more reasonably-sized repo? No. If worktrees are already cheap to spin up, there's little reason to layer any of this on top, especially if your package manager already deduplicates dependencies across projects the way `pnpm` does.

The piece I'd flag hardest is the cone-versus-validation tension. It's a real ceiling, not a minor footnote, and it shows up whenever a test runner reaches further than a diff would suggest.
