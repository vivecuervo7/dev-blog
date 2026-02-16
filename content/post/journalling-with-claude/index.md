---
title: Automated journalling for AI-assisted development
description: Letting Claude track what I'm building, surface blog post topics, and capture reusable patterns
slug: journalling-with-claude
date: 2026-02-17 00:00:00+1000
image: cover.jpg
categories:
  - Development
  - AI
tags:
  - LLM
  - Claude
  - Skills
  - Automation
  - Productivity
weight: 1
links:
  - title: Claude Code
    description: Anthropic's CLI for Claude
    website: https://docs.anthropic.com/en/docs/claude-code
---

## What was that interesting thing I did last week?

I've always leaned towards the idea of writing blog content when it comes to sharing knowledge. It's low-friction, and I can get a lot of info onto a page.

At least, I like the _idea_ of writing blog content.

Looking back over my career to date and there's been a consistent trend with the way I try to write about my work. I can _never_ come up with a good, blog-worthy topic!

The same story keeps repeating itself. I'll do a thing, run onto problems, solve them, and come out the other side with some learnings and just... pick up the next task.

And then, weeks later, I'll mention it to someone and they'll respond with something along the lines of "that's interesting, I'd love to hear more about _that thing that you've half-forgotten about_".

As my work has become increasingly funnelled through Claude, the notion struck me that Claude could probably keep track of the things I was doing better than I could–potentially even flagging work that I'd done that might be interesting to others, or picking up on potentially reusable techniques or patterns.

## Skill: /journal

I landed on the creation of a `/journal` skill that Claude could invoke (or could be triggered manually) to add an entry to a journal file. There were a few considerations to take into account:

- **Append-only** &mdash; entries are only added to a single markdown file, to avoid losing previous entries
- **Structured but flexible** &mdash; each entry has a timestamp and a project, but Claude has latitude in what it includes
- **Context is retained** &mdash; all the relevant context required for a blog-worthy topic would be captured
- **Non-blocking** &mdash; the journal runs as a background task so it doesn't interrupt the flow of work

### What does it do?

<!-- TODO: include the actual skill definition -->

The skill prompts Claude to reflect on the recent work and produce an entry covering:

- **What was done** &mdash; a concise summary of the work completed
- **Decisions made** &mdash; any non-obvious choices and the reasoning behind them
- **Notable observations** &mdash; things that surprised me, patterns worth remembering, or potential blog topics
- **Reusable patterns** &mdash; techniques or approaches that could be extracted into skills, hooks, libraries etc.

I did also get it pop in some Obsidian-style [tags](https://help.obsidian.md/tags), just to try and make it easier to quickly collate journal entries or blog-worthy topics.

### A gentle nudge

To make this work passively, I added an instruction to my project's `CLAUDE.md`:

{{< code-hint "~/.claude/CLAUDE.md" >}}

```markdown {linenos=false}
When you complete a significant piece of work, make a notable decision, solve an interesting problem, or
identify something blog-worthy or reusable, use `/journal` as a background task to record it. Don't
interrupt flow — spawn it in the background and continue working.
```

This means Claude will proactively journal without me having to remember to ask.

I didn't actually see much success in it actually spawning it as a background task, but it ran quickly enough and I started to get used to the frequent "hey, I want to pop this in the journal". It was nice to know it was working, and it served as a reminder that I could go back and check the journal if I needed to.

### Project mapping

To allow for mapping entries to projects, I needed to maintain this mapping somewhere. Fortunately I already had something in place due to some work that I was also doing to help drive some cross-project [ChunkHound](https://chunkhound.github.io/) search capabilities.

While the ChunkHound work I'd done was populating this file on my behalf whenever a new project was indexed, even manually creating the file below would allow for journal entries to be mapped to specific projects rather than all be tagged under `#general`.

{{< code-hint "~/.claude/skills/projects.json" >}}

```json
{
  "project1": {
    "path": "path/to/project1"
  },
  "project2": {
    "path": "path/to/project2"
  }
}
```

### Skill definition

And of course, the meaty part–the skill definition itself!

One gotcha that I ran into that still has some caveats is when running a session that spans multiple days. Initially it was just taking the session start date, but adding a proper date lookup means that _if_ it's running autonomously, the entries should end up under the right day.

Running the skill manually after multiple days work would lump them all into the day it was run, meaning this more accurately reflects the time a journal entry was written rather than when the work actually happened.

{{< code-hint "~/.claude/skills/journal/SKILL.md" >}}

````markdown {linenos=false}
---
name: journal
description: Append a journal entry summarising recent work, decisions, and notable observations
argument-hint: "[optional focus or note]"
allowed-tools: Bash, Read, Write, Glob
---

# Journal

Append a structured journal entry to today's daily file. This should be fast — do not read existing journal files or attempt to deduplicate.

## Step 1: Determine the date and file

**IMPORTANT:** Do not rely on your own sense of the current date — it may be stale in long-running sessions. Always run `date +%Y-%m-%d` and `date
  +%H:%M` via Bash to get the actual current date and time.

The journal lives in `~/.claude/journal/`, one file per day, named `YYYY-MM-DD.md`.

If today's file doesn't exist, create it with a heading:

```markdown
# YYYY-MM-DD
```

## Step 2: Resolve project context

Determine the current project by matching the working directory against `~/.claude/skills/projects.json`. If not inside a registered project, use
"general" as the project name.

## Step 3: Write the entry

Append an entry to the daily file. Each entry should follow this structure:

```markdown
## HH:MM — {project name} #{project name}

{Summary of what was done — keep it concise but capture the key points}

**Decisions:** {any notable decisions made and why, or "None"}

**Topics:** {flag anything blog-worthy, reusable, or worth revisiting — or "None"}
```

Tag the project name in the heading (e.g., `#project1`, `#general`). When flagging topics, prefix with `#blog-worthy`, `#reusable`, or
`#demo-worthy`:

Guidelines:

- Summarise what was accomplished, not every step taken
- Capture the "why" behind decisions — this is what you'll forget
- Flag blog-worthy topics with a brief note on why it's interesting
- Flag reusable patterns, utilities, or approaches worth extracting
- Flag demo-worthy work — things that would make a good presentation, show-and-tell, or live walkthrough
- If the user provided `$ARGUMENTS`, use it to focus or annotate the entry
- Keep entries concise — a few lines per section, not paragraphs

## Step 4: Write detail files (when topics are flagged)

When a topic is flagged as blog-worthy or reusable, create a supporting detail file that captures the context needed to act on it later. Without
this, the journal flags opportunities but loses the detail needed to follow through.

Detail files live in `~/.claude/journal/details/` and are named `YYYY-MM-DD-{slug}.md`.

Link them from the journal entry:

```markdown
**Topics:**

- #blog-worthy Teaching Claude to reflect — [detail](details/2026-02-13-reflect-workflow.md)
- #reusable The skill/hook pattern — [detail](details/2026-02-13-starter-kit.md)
```

Each detail file should include whichever of the following are relevant:

- **Context** — what problem was being solved and why
- **Approach** — what was tried, including dead ends and alternatives considered
- **Key code** — relevant snippets, patterns, or configurations that were created
- **Outcome** — what worked, what didn't, and why
- **Blog angle** — if blog-worthy, what makes it interesting to write about
- **Extraction notes** — if reusable, what could be extracted and how it might be generalised

These files are meant to preserve enough context that someone (including a future Claude session) could flesh out a blog post or extract reusable
code without the original conversation.
````

## What the output looks like

The resulting entries were split over two aspects. The first was a file-per-day journal structure filled with simple entries, grouped under timestamps and project headings. The other was the tracking of detail pages, where context needed to be retained.

### Journal entries

```markdown {linenos=false}
# 2026-02-14

## 09:59 — general #general

Built a comprehensive Claude Code customisation layer: global skills and CLAUDE.md instructions.

- Created `/journal` and `/standup` skills for daily work journalling and standup summaries
- Added continuous improvement and autonomous journalling instructions to CLAUDE.md
- Added two-tier journalling (entries + detail files) with Obsidian-friendly `#tags`
- Fixed date staleness in long sessions by shelling out to `date` command
- Added `#demo-worthy` as a topic tag alongside `#blog-worthy` and `#reusable`

**Decisions:**

- Two-tier journal (entries + detail files) — entries stay scannable, detail preserves context
- Obsidian-style `#tags` for filtering — zero lock-in, just text elsewhere
- Journal skill shells out to `date` — prevents stale dates in long-running sessions
- Autonomous journalling wording made explicit with concrete examples and "err on the side of journalling too much"

**Topics:**

- #blog-worthy The self-improving assistant loop — skills, hooks, /reflect, journalling as a meta-workflow — [detail](details/2026-02-14-self-improving-assistant.md)
- #reusable The skill/hook/CLAUDE.md pattern as a transferable Claude Code starter kit — [detail](details/2026-02-14-claude-code-starter-kit.md)
- #demo-worthy The full workflow end-to-end: `/search` → work → `/journal` → `/standup` → `/reflect` cycle
```

Each entry ends up as a timestamped section in a markdown file. After a few days of use, the journal becomes a rich log of activity.

The only downside of course is that it only tracks what happened via Claude, requiring the mental mapping to any manual work done on the codebase. I expect that as my maturty with the tooling grows, I'll see a growing amount of my work processed through Claude, and these logs should naturally become richer over time.

### Details / context

Under that **Topics** section, each entry has a link. Those links are to co-located details pages, where the relevant context is captured from the conversation history. This surprised me as to how well it worked, honestly, and it's where I went from curious to excited about the potential for this to truly enrich my workflow.

A little too verbose to paste one of the larger examples, the contents were well-structured and covered all the bases I'd have expected them to. The details for potential blog posts even went as far as providing an angle for the blog post.

```markdown {linenos=false}
# Semantic code search with Claude Code and ChunkHound

## Context

Claude Code defaults to grep/glob for code search — pattern-based, requiring you to know what you're looking for. Goal: wire in ChunkHound semantic search so Claude finds code by intent using natural language.

## Approach

**Architecture:**

- Embeddings: local LM Studio (`text-embedding-bge-base-en-v1.5`) — free, fast, private
- Search: ChunkHound CLI `--semantic` against per-project DuckDB indexes
- Research synthesis: `claude-code-cli` provider — routes LLM through Claude
- Index freshness: `Stop` hook runs background reindex after every response

Initially tried local model for research synthesis too, but it ran out of context. Hybrid approach (local embeddings + Claude for LLM) was the fix.

**Skills:** `/search`, `/research`, `/reindex`

**Configuration:** Two JSON files in `~/.claude/skills/` — `chunkhound-config.json` and `projects.json`. Skills read at invocation. Adding a project = editing one file.

**Auto-reindex:** `Stop` hook → shell script → background `chunkhound index`. Chose `Stop` over `PostToolUse` (once per turn vs per-file) and `SessionEnd` (stale during session).

## Outcome

Claude defaults to semantic search via CLAUDE.md instruction. Zero-maintenance via auto-reindex.

## Blog angle

Local-first — embeddings on your hardware, no API costs, code stays on your network. Draft post at `dev-blog/content/post/semantic-code-search/index.md`.

## Extraction notes

- Config pattern (external JSON read at invocation) cleaner than hardcoding
- Project registry + cwd resolution reused across multiple skills and the hook
- `Stop` hook for background maintenance is generic
```

While I haven't yet tried to do anything with the `#reusable` or `#demo-worthy` content, the `#blog-worthy` details make it a very short jump to go from journal entries to draft blog posts.

## Skill: /standup

With the journal entries populating nicely, I didn't really have a way to prove any tangible benefit to tracking them without writing a blog post or using them in some way.

I decided it might be worth the addition of another skill which could make use of them–in this case, a skill that would synthesise a brief summary for a span of days.

### Skill definition

Once again I put Claude to work and it spat out something even more robust than I had expected, deciding to include a project filter.

{{< code-hint "~/.claude/skills/standup/SKILL.md" >}}

```markdown {linenos=false}
---
name: standup
description: Summarise recent journal entries as a standup update
argument-hint: "[days back, default 1] [project name]"
allowed-tools: Bash, Read, Glob
---

# Standup

Summarise recent journal entries.

## Steps

### Step 1: Determine the range

Parse `$ARGUMENTS` for:

- A number (days to look back, default 1)
- A project name (filter to that project only, default all)

Examples:

- `/standup` — yesterday's entries, all projects
- `/standup 3` — last 3 days, all projects
- `/standup project1` — yesterday's entries for project1
- `/standup 7 project2` — last 7 days for project2

### Step 2: Read journal files

Read the daily files from `~/.claude/journal/` for the date range. Files are named `YYYY-MM-DD.md`.

### Step 3: Present the summary

Summarise the entries concisely, grouped by day and project. Highlight:

- What was accomplished
- Key decisions made
- Any flagged blog topics or reusable IP
- Anything left in progress

Keep it brief — this is a standup, not a report.
```

### Output

Here's the output for `/standup last 2 days`.

```
Feb 15 — project1
- Ported the Unity client to Godot 4.x (device-ID auth, main menu)
- Built a background glow pulse shader, iterated with debug heatmaps to tune luminance thresholds
- Built a step-based test harness with Playwright-style traces, CLI composition, and a /godot-test skill (9 files). Smoke tests passing.

Feb 14 — general
- Built out the Claude Code customisation layer: /search, /research, /commit, /reflect, /journal, /standup skills
- Added Stop hook for ChunkHound reindexing, switched research LLM to claude-code-cli
- Drafted blog posts on semantic code search and teaching Claude to reflect
- Added two-tier journalling (entries + detail files) with Obsidian-friendly tags

Flagged topics:
- #blog-worthy Shader debug heatmaps workflow
- #blog-worthy Self-improving assistant loop
- #blog-worthy Local-first semantic code search
- #demo-worthy Unity-to-Godot port side-by-side
- #reusable Godot step-based test runner, device-ID auth pattern, global skill + project opt-in pattern, Claude Code starter kit
```

The ordering could leave a little to be desired - another run displayed the dates in reverse. Overall though, a very handy way to capture the work that was done, and to recap on any blog-worthy content or reusable IP!

## Next steps

I was pleasantly surprised at how well this worked, even in the short span of time I've been playing with it. Considering it was the first time I really started to look at skills it was a testament to how easy they are to set up.

The pattern extraction is still speculative at this point, but capturing the context and the fact that there was even a thing potentially worth extractng is such a huge leap from my current "do the thing and forget about it". I'm looking forward to trying to progress this onto the next step of actually using that information to build the reusable assets etc.

The most notable thing I've found myself lacking is the absence of screenshots. In one case, I went through an iterative process of visually debugging some changes I had Claude making to a shader in Godot, using a temporary heatmap drawn over the image I was applying the shader to, which drastically helped me work with Claude to get the result I was after. By the time it was solved, I'd lost any opportunity to take in-progress screenshots that would have been incredibly valuable for a blog post.

Now to go full circle and find a way to get Claude to start prompting _me_ to proactively capture media if the current work feels like it might reach a `#blog-worthy` level!
