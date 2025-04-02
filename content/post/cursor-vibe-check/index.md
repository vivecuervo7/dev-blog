---
title: "Cursor: Are the vibes really worth it?"
description: Is the magic of AI coding a wondrous spell or a lurking curse?
slug: cursor-vibe-check
date: 2025-04-02 00:00:00+0000
image: cover.jpg
categories:
  - Development
  - AI
tags:
  - Cursor
  - GitHub Copilot
  - Vibe Coding
weight: 1
links:
  - title: Cursor
    description: The AI code editor
    website: https://www.cursor.com/
---

Alright, I've scoffed enough at the ridiculous notion of [vibe coding](https://en.wikipedia.org/wiki/Vibe_coding). Time to give [Cursor](https://www.cursor.com/) a whirl.

No, I'm not about to become a vibe coder. But I have found the concept of Cursor intriguing, and I can't help but to be curious about what the tool has to offer. GitHub Copilot has only been able to feel like a roided-up Intellisense &mdash; maybe Cursor can actually feel like the magic AI assistant I was promised.

To be completely fair to Cursor, I wanted to see how it performed without going into setting up custom [project rules](https://docs.cursor.com/context/rules-for-ai) and in hindsight I _may_ not have set up up entirely for success.

## A brand new project for a brand new world

I wanted to experience the deep end of the magic first. I wanted to see what the whole "build me an app" felt like, and so I decided to go straight down that path.

My first impression wasn't exactly great. I knew based on what I had read online that Svelte 5 probably wasn't going to be too well supported due to models having been trained on Svelte 4, and indeed it took some coaxing and I even needed to provide a link to the latest documentation to get it to use the new [Svelte CLI](https://svelte.dev/docs/cli/overview).

While impressive that it did get there once I poked and prodded it a bit, it was also quick to go off one some very opinionated tangents, such as jumping straight into using [Drizzle](https://orm.drizzle.team/) for data persistence. Not my first choice, and it was lumped in with a bunch of other changes that wouldn't have been fun to untangle.

I knew ahead of time this might happen, and so I decided to park Svelte for the time being. React it is!

### React

React had the same problem that Svelte did re: not using the Svelte CLI. It immediately tried to use the deprecated [create-react-app](https://create-react-app.dev/docs/getting-started/), and only decided to use Vite after I explicitly told it to. Not a _great_ start, but not the hardest thing to work around.

I could see this being a problem for the true vibe coders out there, although to be fair I didn't really give Cursor a chance to see any deprecation warnings and adjust accordingly. Maybe it would have realised?

As far as setting up components and state management, it wrote pretty close to the sort of code I'd expect to see. I was pretty happy with this, at least while things were in a simple state.

#### Tailwindn't

I asked Cursor to improve the styling across my components, and it jumped enthusiastically into the task. I was watching a bunch of the code it was generating and it was getting _busy_. But I wasn't seeing any styling changes!

Turns out, Cursor ❤️ Tailwind. But Cursor hadn't _installed_ Tailwind. D'oh!

Easy enough fix, but it did highlight that it could go off on and start using tools that you hadn't even asked it to. I'm honestly still not sold on Tailwind anyway, and I probably wouldn't have gone for it had it asked me first. Ah well, embrace the vibe, right?

It took me quite a few round trips of "didn't work, here's my error" and ultimately an "is Tailwind even installed?" before it actually figured out what was wrong.

#### Tailwind'd

With Tailwind _actually_ installed now, I was pleasantly surprised at how the front end looked with a few nudges in the direction I wanted it to go. I had asked for modern, card-based styling, and that's pretty much what I received.

It wasn't perfect, there were some areas where padding was missing or inputs were hard to read, but overall I was quite impressed with what it spat out. I am curious to see if adding [automated accessibility tests](https://playwright.dev/docs/accessibility-testing) or other might help to keep things on track re: the obvious contrast issues in the inputs.

![Cursor didn't quite get the input styling right](images/illegible-inputs.jpg)

### .NET Core

I was pretty impressed with what it did here, as much as I enjoy using .NET Core I find that there's often a lot of code that needs to be written across many files.

Cursor was able to create a bunch of the files that I would have otherwise had to write myself, all with minimal prompting. Awesome! Except, half the files were in the wrong directory. It had gone and created a `backend/` directory, and then dumped my `Program.cs` file outside of that, along with some other files. Another prompt re-created the files in the correct place but didn't clean up the old ones.

Even once it started putting things in the right places, refactorings indicated a similar disdain for cleaning things up. I do really wonder if putting the right tooling around Cursor might help with a lot of these issues, as it showed a good responsiveness to linter errors and warnings in the frontend project. Perhaps exposing similar errors to Cursor would guide it towards making better decisions.

CORS issues came up &mdash; just like a real programmer! &mdash; and took another couple of spins on the merry-go-round before they were resolved. So far, I've been impressed by Cursor's ability to resolve these issues, but somewhat unimpressed by the number of false resolutions.

A minor frustration was a perceived disconnect between the Postgres database credentials and our configuration. Despite the database connection working perfectly, it kept accusing the configuration of being wrong and would keep changing the database credentials and/or the appsettings. I decided to let it do its thing, and while it didn't necessarily break things it did get annoying.

#### Do as I say, not as I do?

The biggest and most glaring issue encountered was in the, you know, not super-duper-important auth service, where it had written some code and whacked a comment above it essentially saying "don't do this in production". Whoops? I can definitely imagine a true vibe coder missing this altogether. If I'm being honest, I missed it until I was reviewing the output many commits later.

_Maybe_ some tooling could have shown an error here? I'm planning to also look into [CodeRabbit](https://www.coderabbit.ai/) for AI code review. It would be curious to watch an AI tool telling another to get it's sh\*t together.

#### EF Core

Par for the course so far, things didn't go perfectly at first although they did work in the end &mdash; which is the most important thing, right? Vibes!

No, it did a pretty good job at setting up and leveraging migrations, and this was pretty hands-off in general. One of the things that I find tedious with .NET development in general is the need to go and write a migration, apply it, update my model, update the DB context, all before I can even go and start using the new or updated entity. Cursor was definitely working hard to save me keystrokes here.

I don't know if my trust issues would start to surface if and when it came time to actually migrate some data around though.

## A brand old\* project for a brand new world

Sweet, Cursor seems to be capable of building a brand spanking new project. But what about an existing project that already has some conventions about how to do things?

Given that Cursor seemed to have some trouble swallowing Svelte earlier, I decided to try loading an existing SvelteKit project. Specifically, this project is using Svelte 5, which has a significantly different syntax to previous versions &mdash; and this has been Cursor's tripping point.

Let's see how it handles some change requests!

### Starting small

I wanted to ask it to specifically make changes to a single file. In the true spirit of side projects I'd been a bit lax in keeping any relevant documentation up to date, and so I asked Cursor to update my root `README.md` file to improve the project's documentation.

Granted, there wasn't a whole lot that needed to be written, but it did a solid job of inspecting the project to determine the tech stack, prerequisites, steps for getting started etc.

The only issue was that for some reason despite the project using `npm`, Cursor decided to recommend `pnpm` and use that for a bunch of examples. A quick prompt sorted that out, but again this could have easily been missed.

### Database migration

Okay, I was pretty impressed with this one. I wasn't expecting it to pick up on my using [graphile-migrate](https://vivecuervo7.github.io/dev-blog/p/database-driven-codegen/#graphile-migrate) in this project, but it had a look at the tooling and it correctly updated my `current.sql` migration file. Nice!

Using a prompt to "create a migration allowing for league member's to assign teams for each round", it created the table I would have expected to see, along with indexes and even a trigger to enforce the team size limit.

It didn't pick up on the need for the migration to be idempotent, but a quick prompt and it was back on track.

### A little more complicated

I went through a series of prompts, from simply showing new information to adding an entire new form.

Cursor picked up on the use of Kysely to build queries, continuing with a repository pattern and implementing database queries as necessary. It proved to be similarly competent when it came to building UI components, inserting the new code and resolving linting errors to come up with a working solution.

Where Cursor ran into a bit of a wall was accessing the database service. I had settled on a usage pattern of initialising the database service and passing it down the chain via [locals](https://svelte.dev/docs/kit/hooks#Server-hooks-locals). Granted, this isn't the most common approach (most tutorials suggest simply importing the database client and using it directly) however I thought there would be enough code there for Cursor to figure out the usage pattern.

Alas, every time Cursor wanted to use the database service, it would attempt to instantiate a new copy of it. Not what I had wanted at all, and despite a prompt fixing it on one occasion I found it faster to just go and clean up after it. I _did_ eventually see Cursor pick up on a linting error and subsequently explain to me that it should be using the `locals.db` instead of creating a new one, so maybe I was just trying to fix Cursor's mistakes a little too eagerly.

## Overall impressions

Honestly, it felt like taking a machine gun to my code. It felt very [Spray n' Pray](https://fallout.fandom.com/wiki/Spray_n%27_Pray), and there was a _lot_ of "yeah, I'm just going to commit here because I'm probably going to throw away the next change a few times before it's right". The problem was that each thing I wanted to do would take multiple prompts, which made me quite hesitant to discard whatever I had so I would keep throwing prompts at it to try and make it right. That led to me feeling too invested (why hello, [sunk cost fallacy](https://thedecisionlab.com/biases/the-sunk-cost-fallacy)) to throw it all away.

A big sticking point for me was the potential for insecure code to make its way into the codebase. I don't think I trust it fully to keep things secure, pending wrapping additional tooling around it. Even then, it would require a close watch and there is just so much code being generated in such a short amount of time that it's hard to pick up on little things.

Cursor did doing a pretty good job of coming up with meaningful commit messages, although they got pretty long-winded which exposes an actual underlying problem in that the changes themselves can easily become quite extensive with a series of iterative prompts causing many files to be touched.

### Cursor turned me into a rubber duck

Cursor felt like speed-running a [pair programming](https://en.wikipedia.org/wiki/Pair_programming) session with a coked-up auctioneer as the driver.

It talks through the decisions it's making, and shows you the code being generated. I found I could _just_ keep up with it, occasionally spotting things that I wanted to ask about or prompt in another direction. It was nice to have that immediate feedback and it mitigated the feeling of blindly trusting Cursor to come up with a good solution, but there was an absolute flood of code to try and parse in very little time.

In particular, I found it curious to watch how it went about debugging issues, or more specifically the way it went about addressing linting errors and warnings. I wasn't too impressed about the nonchalant way it dismissed warnings as essentially "meh, not critical so don't bother" but we could either prompt it to address those concerns or just configure our project to treat warnings as errors etc.

### Cursor's rules for AI

As mentioned at the top of this post I did not make use of Cursor's [rules for AI](https://docs.cursor.com/context/rules-for-ai), which I imagine may have mitigated if not completely avoided many of the issues I ended up facing.

I may do a deep dive into these rules in the future to do a comparison &mdash; especially regarding the use of Svelte 5, as this proved to be something of a sticking point. A very quick dabble in this yielded the ability to specify any particular tooling that I wished to use &mdash; telling it to use Svelte 5, Kysely and `graphile-migrate` did a pretty good job, although it still had trouble with the Svelte 5 syntax.

There are plenty of sample `.cursorrules` files floating around &mdash; these are being phased out per the docs above in favour of `.cursor/rules/` but still provide a good base for building the rules themselves. [Here's an example](https://gist.github.com/aashari/07cc9c1b6c0debbeb4f4d94a3a81339e) of a relatively comprehensive set of general purpose Cursor rules.

And... we can always ask Cursor to generate a set of rules for itself based on the current project!

I do get the sense that with the right rules Cursor could be kept largely in check, but I can't help wondering if constantly tweaking those rules and needing to gauge the output might start eating significantly into whatever time Cursor initially saved.

### Does it have a place in my workflow?

The biggest question of them all, and truth be told, I don't really know.

A day spent playing with it doesn't really give it the time it needs to really show its full potential &mdash; or frustrations, nor does the fact that I largely ignored setting up any rules.

I can certainly see how I might prefer this over GitHub Copilot purely due to the fact that it can use the entire codebase as context for my next operation &mdash; but I think for now I'd prefer to keep the changes scoped to the file I'm working on.

Cursor passes the vibe check, but of me a vibe coder it shall not make.
