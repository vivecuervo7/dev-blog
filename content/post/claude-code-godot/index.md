---
title: Claude Code as a Godot Editor
description: How a composable test runner, DevGrid, and file-based IPC let me build game UI without opening the Godot editor
slug: claude-code-godot
date: 2026-03-04 00:00:00+1000
image: cover.jpg
categories:
  - Development
  - Game Development
tags:
  - Claude Code
  - Godot
  - Testing
  - AI
weight: 1
links:
  - title: Claude Code
    description: Anthropic's CLI for Claude
    website: https://claude.ai/claude-code
  - title: Godot Engine
    description: Free and open source 2D and 3D game engine
    website: https://godotengine.org/
---

## I've never opened the editor

I'm building a card game in Godot. It has a deck builder with drag-and-drop, a battle arena, crafting screens, shaders I never could have written by hand &mdash; and I've never opened the Godot editor. Not once.

The entire UI has been built through Claude Code: editing scene files and scripts in the terminal, verifying changes through a composable test runner, and iterating with a live command queue that talks to the running game. No GUI editor required.

The game started in Unity, and the friction there was what pushed me to try something different. Unity's editor is resource-heavy and needs to be running for most workflows. Every time I tabbed back into it, the domain reload would kick in &mdash; already annoying on its own, but constant when you're bouncing between an AI assistant and the editor. I kept interrupting my own flow just by interacting with it. Getting the MCP server set up was yet another required step before Claude could even talk to the engine.

Godot changed that equation. Text-based scene files, a lightweight runtime, and no editor dependency meant I could skip the GUI entirely and let Claude do the editing.

It helps that the game is server-authoritative &mdash; the client is purely a presentation layer. There's no sensitive logic or secrets to worry about on the UI side, which makes it a good fit for giving Claude a long leash.

<!-- TODO: screenshot of the game running (deck screen or battle screen) -->

## Why Godot is AI-friendly

Godot is CLI-friendly, and that's what makes it AI-friendly. The engine runs the game directly from the command line &mdash; no editor required. A single `godot --path <project> --main-scene <scene>` launches the game windowed, ready to interact with. In Unity, you can't run the game without loading the editor first. That distinction matters when your workflow is entirely terminal-based.

The file formats help too. `.tscn` scene files are text-based and surprisingly readable &mdash; even as someone who'd never seen one before, I could follow what was going on:

```ini {linenos=false}
[node name="StartButton" type="Button" parent="MainMenu/VBox"]
layout_mode = 2
size_flags_horizontal = 4
disabled = true
text = "Start Game"
```

GDScript is the same story &mdash; clean, Python-like, easy to read and easy to edit. Nothing about the toolchain demands a GUI to work with it.

<!-- TODO: side-by-side of a .tscn file in a text editor vs the rendered scene in Godot -->

## The test runner

Being able to edit scenes as text is only half the story. The other half is knowing whether your changes actually work. Godot's runtime errors only surface when you navigate to the affected scene &mdash; a broken crafting screen won't tell you anything until you click through to it.

What I wanted was something like end-to-end testing with Playwright &mdash; the ability to programmatically navigate through the application, interact with elements, and verify that things work. The implementation ended up as a step-based test runner: each step is a standalone GDScript file with a simple contract.

{{< code-hint "tests/steps/open_crafting.gd" >}}

```gdscript {linenos=false}
extends RefCounted

func run(ctx: TestContext, args: Dictionary) -> String:
    # return "" for pass, error message for fail
    ctx.press_button("CraftingButton")
    await ctx.wait_for_node("CraftingScreen")
    return ""
```

Steps compose into chains via the CLI:

```bash {linenos=false}
godot --path src/client/godot \
  --main-scene res://tests/runner.tscn \
  -- --steps=auth,crafting,select --stay-open
```

A JSON manifest maps aliases to step scripts and defines macros &mdash; `setup:deck` expands to `auth, deck, set_formation, fill_deck, save_deck`. Common sequences get a short name, and you can mix and match steps freely.

### Surviving scene transitions

One design challenge was keeping the runner alive across scene changes. When Godot calls `change_scene_to_file()`, the current scene's children get freed. The runner handles this by reparenting itself to `/root` on startup:

{{< code-hint "tests/runner.gd" >}}

```gdscript {linenos=false}
func _ready():
    var parent := get_parent()
    if parent != get_tree().root:
        parent.remove_child.call_deferred(self)
        get_tree().root.add_child.call_deferred(self)
```

This lets the runner persist through any number of scene transitions, executing steps across the entire application.

### Dual-speed steps

Not every test needs to drive the real UI. Setting up a full deck through drag-and-drop takes 10&ndash;15 seconds; doing it through API shortcuts takes 2. So there are two flavours of most steps:

- **`fast_*` steps** &mdash; API shortcuts that set up state directly, skipping the UI
- **`ui_*` steps** &mdash; real clicks and drags, simulating actual player input

The manifest descriptions and macro naming make the intent clear enough that Claude picks the right variant from context. If I ask it to verify a UI flow, it reaches for `ui_fill_deck`. If the deck is just setup for testing something else, it uses `fast_fill_deck`. The macros reflect this too &mdash; `setup:deck` uses API shortcuts, `test:battle_e2e` drives the full UI.

Output goes to both JSONL (for Claude to parse) and Markdown (for human review), with timestamped screenshots at each step. Claude reads the trace output and examines the screenshots to diagnose failures &mdash; it's not just running the tests, it's analysing the results.

<!-- TODO: terminal output of a test run (step names, pass/fail, trace path) -->

## DevGrid &mdash; talking about pixels

Here's a problem that doesn't come up in traditional development: how do you tell an AI where things are on screen?

Early on, this was genuinely frustrating. I'd say "nudge that panel a bit to the left", then "no, a bit more to the right", then "make it a little bigger" &mdash; and never quite land on what I actually wanted. "Set the anchor to 0.3" is more precise, but requires you to already know the current value and do the spatial maths. What I needed was a shared coordinate system &mdash; a way for both me and Claude to point at the same spot on screen and agree on what we're talking about.

### An interactive grid overlay

DevGrid is a dev-only overlay that draws a coordinate grid over the running game. The backtick key cycles through four modes: OFF &rarr; LARGE (64px) &rarr; MEDIUM (32px) &rarr; SMALL (16px) &rarr; OFF.

When the grid is active, it becomes a measurement tool:

- **Hover** shows a coordinate tooltip (e.g. `MEDIUM (12, 5)`)
- **Click** copies a single cell reference to the clipboard
- **Drag** copies a range (e.g. `MEDIUM (2,7):(79,35)`)

The grid consumes all mouse input when visible &mdash; it's intentionally a modal measurement mode, not a passive overlay.

<!-- TODO: screenshot of the game with DevGrid overlay active -->

### Spatial communication

This is where it gets useful. Instead of vague descriptions, I can tell Claude exactly what I want:

> Reposition the board area to MEDIUM (2,7):(79,35)

And Claude knows precisely what region of the screen I'm talking about. It can translate that grid reference into the right anchor and margin values in the `.tscn` file. No guessing, no "a bit more to the left" iterations.

This turned out to be the piece that made CLI-driven layout work genuinely viable. Without a shared spatial language, I'd have been constantly fighting imprecise descriptions. With DevGrid, positioning conversations are as concrete as code reviews.

<!-- TODO: Claude Code conversation snippet showing a DevGrid coordinate being used in a prompt -->

## The edit-test-verify loop

With editable scene files and a composable test runner, the workflow becomes a tight loop:

1. **Edit** &mdash; Claude modifies a `.tscn` or `.gd` file
2. **Run** &mdash; launch the test runner with the relevant steps
3. **Analyse** &mdash; Claude reads the trace output and examines the screenshots to check whether the change achieved what was asked for
4. **Fix** &mdash; adjust based on what it sees
5. **Re-run** &mdash; verify the fix

The screenshot analysis is what makes this self-correcting. Claude isn't just checking pass/fail &mdash; it's looking at the rendered UI and comparing it against the intent. If a panel is misaligned or a label is truncated, it can see that and iterate without me having to point it out.

A single `/client` skill handles both launching and testing. `/client arena` launches the game navigated to the arena screen for manual inspection. `/client verify crafting works` triggers the test runner instead &mdash; composing steps, running them, and reporting the trace. Same skill, different mode based on intent.

Because the skill maps natural language to step chains, you can also just describe what you want in plain English:

> Launch the game as a new user. Redeem the code "MATERIALS". Navigate to the crafting screen and craft one copy of each card. Then go to the deck editor, switch the formation to mage. Fill the deck and then go to the arena and battle one of the opponents.

Claude reads the manifest, maps that description to the available steps, and composes the chain. In practice, a chain this long doesn't always execute cleanly end-to-end &mdash; but portions of it do, and those portions are what get used in verification loops. The ability to describe intent rather than spell out step names lowers the friction of running tests considerably.

Once Claude is satisfied that the changes are working, it'll often relaunch the client with `--stay-open` so I can interactively verify the results myself. It handles the automated loop, then hands it back to me for a final look.

<!-- TODO: screenshot sequence showing before/after of a UI change caught by the test runner -->

## Live reload

The test runner is great for verification, but for small tweaks &mdash; nudging element positions, adjusting margins &mdash; relaunching the client every time adds unnecessary friction. This was the one point where I was genuinely tempted to just open the Godot editor. Hot reload removed that temptation: make a change, see it immediately in the running game.

A `LiveReload` autoload handles this. It polls for a `.reload` sentinel file, and when it detects a change, it reloads the current scene's script and resources from disk and swaps them into the running game. Touch the file, and the scene updates in place.

Right now the touch is manual, but Claude Code's [hooks](https://docs.anthropic.com/en/docs/claude-code/hooks) feature could automate this with a `PostToolUse` hook that fires on every `Edit` or `Write` tool call. The result would be fully automatic: Claude edits a file, the running game updates.

## The live command queue

Sometimes I want the game running interactively &mdash; but I don't want to click through everything myself. If I'm testing different formations against different opponents, it gets tedious fast: switch formation, fill the deck, save, navigate to the arena, pick an opponent, fight. I just wanted to be able to tell Claude "switch to mage, fill the deck, and go fight that opponent again" while the game was already running.

### File-based IPC

The mechanism is simple: two files.

- **`.cmd-queue`** &mdash; written by Claude, contains the step string to execute
- **`.cmd-result`** &mdash; written by the game after execution, contains the results

When the `/client` skill detects a running session, it writes directly to `.cmd-queue` instead of launching a new instance. A `CommandWatcher` autoload in the game polls for that file every 300ms:

1. Claude writes `navigate:deck,picker` to `.cmd-queue`
2. CommandWatcher picks it up and deletes the file
3. Steps execute via the shared `StepRunner` class
4. Results (with screenshots and trace paths) get written to `.cmd-result`
5. Claude reads the result and deletes the file

Why file-based instead of sockets or HTTP? No port management, no networking code, no dependencies. Godot's `FileAccess` makes it trivial from GDScript, and debugging is just `cat .cmd-result`. It's not glamorous, but it's reliable and easy to reason about.

The game stays running, and I can issue commands to it through Claude without touching the mouse.

<!-- TODO: diagram or terminal showing the file-based IPC flow -->

## Case study: Economy Wave 1

To show how all of this comes together, here's a real session. The task was implementing an economy system &mdash; formation fragments, XP, levelling, and a battle reward screen. It touched 33 files across 6 projects: domain entities, database migrations, API endpoints, and the Godot client UI.

The implementation used parallel background agents, each given front-loaded context and conventions from `.claude/rules/` files. At peak, five agents were writing code simultaneously. The result: 0 build warnings on first compile, 214/214 tests passing after one boundary fix.

But the interesting part was what happened after the code was written. Claude drove the deployment and verification start to finish, using four skills in sequence:

1. **`/migrate apply`** &mdash; built the database migrator, ran it through Aspire, confirmed the new tables and columns were created
2. **`/server restart`** &mdash; restarted the API, verified it came up healthy with the new endpoints
3. **`/client verify battle results`** &mdash; ran the test steps (setup deck, navigate arena, fight battle, assert results), then read the trace screenshots to verify the result

On that last step, 8 of 9 steps passed. The one failure was a redemption code that had already been used &mdash; not a bug, just existing game state. Claude recognised this as expected behaviour and didn't flag it as an issue. From the final screenshot, it confirmed three reward sections were rendering correctly: materials, fragments, and formation XP with a progress bar.

Then it went further:

4. **`/client new profile`** &mdash; launched with a fresh device ID to verify the feature from a brand new player's perspective. It confirmed only the Standard formation was owned, Mage and Swarm were locked at 0 fragments, and left the client running for me to explore.

That last step stood out. Verifying with a fresh player is the kind of thing that usually requires someone to think "what about the new user path?" and write a separate test for it. Claude reached for it on its own.

These skills weren't designed as a pipeline. They were built independently for different purposes. But they composed naturally because each one is self-contained and they communicate through system state &mdash; `/migrate` writes to the database, `/server` picks up new code, `/client` hits the running API. No orchestration layer, no skill-to-skill wiring. The running system is the shared state.

The contrast is the point: 33 files across 6 projects is a complex change. Deploying and verifying it was four commands.

<!-- TODO: screenshot of the battle reward screen showing the three reward sections -->

## Wrapping up

Beyond the iteration speed, this workflow expanded what I could do within a game engine. I've never written a shader, but through this setup Claude wrote frosted glass blurs, Voronoi shard patterns, glow pulses, and vignette effects that I could iterate on and refine just like any other UI element. That was never in my skillset &mdash; the tight feedback loop made it accessible.

The pattern here isn't specific to Godot or game development. A composable step runner, a way to send commands to a running application, and a verification loop that analyses screenshots &mdash; that translates directly to web development with Playwright or similar tooling.

The picture I keep coming back to is a demo. A stakeholder asks "what happens if X?" and instead of manually clicking through screens to get the data into the right state just to hit the right button, I type that scenario into Claude and it executes right there. The stakeholder doesn't care about the setup &mdash; they care about the result. This workflow lets you skip straight to it. And that demo isn't a one-off &mdash; it becomes part of the workflow itself, reusable as a verification step the next time something changes.
