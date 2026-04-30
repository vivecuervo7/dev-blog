---
title: Failing the icebreaker
description: A small letter about the bigger picture
slug: failing-the-icebreaker
date: 2026-05-01 00:00:00+1000
image: cover.png
categories:
  - Reflections
tags:
  - Cognitive Style
  - Software Development
weight: 1
links:
  - title: "Navon figure — Wikipedia"
    description: The classic global-local perception experiment
    website: https://en.wikipedia.org/wiki/Global_precedence
  - title: "Construal Level Theory — Wikipedia"
    description: How psychological distance shapes abstract vs. concrete thinking
    website: https://en.wikipedia.org/wiki/Construal_level_theory
  - title: "Top-down and bottom-up design — Wikipedia"
    description: The two classical approaches to system design
    website: https://en.wikipedia.org/wiki/Top-down_and_bottom-up_design
---

## Wait, you can fail an icebreaker?

At a previous job, every new project kicked off with a round of team introductions. Part of the ritual was a slide deck, one per person, where you'd answer a handful of icebreaker-style questions. One of them asked whether you considered yourself more of a _big-picture thinker_ or more _detail oriented_.

I always picked detail oriented. It felt honest. I like specifics. I like understanding how the small things work before I worry about the shape of the larger thing they belong to.

At some point I got a piece of feedback that I should "try to start seeing the bigger picture." This wasn't in response to some project where I'd missed the forest for the trees. It was a reaction to how I'd _described myself_. I'd ticked the "detail oriented" box, and somewhere along the line that had been interpreted as a gap.

I didn't think much of it at the time. I filed it away as one of those bits of feedback you nod at and quietly disagree with. But recently, something dug it back up and sent me down a bit of a rabbit hole.

## I'd rather stub my toe

I was working on a piece of UI, a shell for an upcoming form. A wrapper, basically. The layout and structure that the form's inputs would eventually live inside. The catch was that none of those inputs existed yet. They were coming in a later ticket. So I was stubbing things out, building a container for something I hadn't seen.

I could do it. I _did_ do it. But the whole time I had this low-level discomfort I couldn't shake. It wasn't a technical problem: the code worked, the layout made sense on paper. It was more that I was making structural decisions without the information I wanted first.

What I _wanted_ to do was start with one of those inputs. Understand its shape, figure out what it needed from the form, and then step back to think about how the form should be built around it. Instead I was going the other direction, building the outside and trusting that the inside would fit later.

The following ticket was to actually add an input. Almost immediately, things shifted. The input had a real shape: props, validation, a layout of its own. And that shape _informed_ the shell. I went back and made changes to the wrapper, and suddenly the whole thing felt right.

It got me thinking about _why_ I work that way. And that old slide came back to mind.

## Big letter; small letter

In 1977, a psychologist named David Navon ran an experiment. He showed people images of large letters composed of smaller letters, a big "H" made up of tiny "S"s for example, and asked them to identify either the large letter or the small ones.

{{< figure src="images/navon.png" alt="A Navon figure: a large H made of small S characters" width="400px" >}}

Most people identify the global shape first. This is called the [global precedence effect](https://en.wikipedia.org/wiki/Global_precedence): we tend to see the forest before the trees. But not everyone. Some people consistently pick up on the local features first, the small letters, before the large shape registers.

This isn't a deficiency. It's a _preference_, a stable and measurable cognitive tendency. People who default to local processing aren't incapable of seeing the global shape, they just don't start there.

When I first saw a Navon figure, I noticed the small letters immediately. The large letter came second. Which, in hindsight, isn't surprising at all.

## Try it yourself

Read the following sentence:

> The project manager&nbsp;&nbsp;said that the team needed to _"start thinking about the bigger picture_," and that we should focus on the overall architecture before diving into the details.

Did you read the sentence, or did you _scan_ it? If your eyes snagged on the double-space, or the quotation mark that's on the wrong side of the italic, before you even processed what the sentence was actually saying, you might be wired the same way.

## It turns out there's a name for this

Construal Level Theory, developed by psychologists Nira Liberman and Yaacov Trope, describes [the relationship between psychological distance and abstract versus concrete thinking](https://pmc.ncbi.nlm.nih.gov/articles/PMC3152826/). The core idea is that the further away something feels, whether in time, space, or just how hypothetical it is, the more abstractly we think about it. The closer it feels, the more concretely we think.

High-level construal is big-picture thinking. You focus on the gist, the "why," the overall shape. Low-level construal is detail thinking: specifics, mechanics, the "how."

This maps onto the form shell problem almost exactly. The inputs didn't exist yet. They were psychologically distant, future work, hypothetical shapes. The task was asking me to think about them abstractly: _imagine_ roughly what they'll look like, and build a structure around that imagination. I wanted the opposite. Low-level construal first, a real input to anchor the structural decisions on.

CLT doesn't frame this as a fixed trait, but the research does suggest people have default tendencies.

## The same tension in software

This isn't just a psychology question. Software engineering has its own version of the same debate.

[Top-down design](https://en.wikipedia.org/wiki/Top-down_and_bottom-up_design) starts with the whole system and decomposes it into smaller parts. You define the architecture, lay out the boundaries, and work inward toward the details. Bottom-up design does the opposite: you build the individual components first and compose them into a larger system.

There's a passage on Wikipedia that describes top-down design in a way that felt uncomfortably familiar:

> Top-down approaches emphasize planning and a complete understanding of the system. It is inherent that no coding can begin until a sufficient level of detail has been reached in the design of at least some part of the system. Top-down approaches are implemented by attaching the stubs in place of the module.

Stubs. That's literally what I was doing, building a form shell with placeholders where the real inputs would go. And the reason it felt wrong is right there in the description of the bottom-up alternative:

> Bottom-up emphasizes coding and early testing, which can begin as soon as the first module has been specified.

That's what I wanted. Give me _one_ real module, one input, and I can start building and testing immediately. The structure reveals itself as the pieces take shape. Working with stubs meant deferring that understanding, and for my particular way of processing things, that deferral was where all the friction was coming from.

## The ISTP connection

Welcome to the horoscope section. I'm an ISTP on the Myers-Briggs framework, and yes, I know MBTI has [well-documented scientific limitations](https://en.wikipedia.org/wiki/Myers%E2%80%93Briggs_Type_Indicator#Criticism). I don't treat it as gospel. But as a lens for thinking about cognitive preferences, the pieces line up in a way that's hard to dismiss.

The ISTP's dominant cognitive function is Introverted Thinking (Ti), a drive to build precise internal frameworks by pulling things apart and understanding how the pieces connect. The auxiliary function is Extraverted Sensing (Se), a preference for present-moment sensory information over abstract or hypothetical data.

In practical terms, this combination means ISTPs tend to [break problems into parts and figure out how they're connected](https://www.psychologyjunkie.com/heres-how-you-solve-problems-based-on-your-personality-type/), starting from facts rather than theories. They want to _see_ and _handle_ the thing before they're comfortable reasoning about the system it belongs to.

That's exactly what I've been describing: start with a real piece, understand it, and then zoom out. I wouldn't go so far as to say MBTI _explains_ all of this, but it does rhyme with it in a way that's hard to ignore.

## The generative AI angle

This took one more turn when I started thinking about how I use generative AI, because there's a tension there too.

A lot of the recommended workflows for AI tools feel very top-down: plan modes, spec-driven prompts, that sort of thing. I've tried working that way, and I bounce off it every time. When AI gives me a high-level scaffold, I get a version of the same unease I felt with the form shell. It looks right at a distance, but I don't trust it until I've built something real inside it.

So I iterate instead. Small prompts, one piece at a time, evaluating each output before thinking about the next layer. Get a draft of one section, or a single component, react to it, adjust, then build outward. Each iteration gives me something real to anchor on, and the bigger picture takes shape as the pieces accumulate.

The pairing might work _because_ the two styles are opposite. The AI is fast at producing the broad shape that I'd otherwise have to build toward slowly from the details up. And the same instinct that catches double-spaces and mismatched quotation marks catches hallucinated details, slightly-off tone, or code that looks right at a glance but has something subtly wrong. AI tends to be weakest in exactly the kind of details that my attention lands on first.

## Now that we've covered the details

Five different angles, from perceptual psychology to AI workflows, all circling the same basic idea: people differ in _where they start_ when making sense of something complex. Some people start with the big shape and fill in the details. Others start with the details and let the big shape emerge. They're different directions through the same problem.

The old project slide didn't really capture this. "Big-picture thinker" versus "detail oriented" sounds like a question about capability, when it's really a question about _sequence_. I can see the big picture fine. I just build my way there from the details first.

If there's a takeaway, it's probably this: teams benefit from having both. The top-down thinker defines the boundaries that keep the system coherent. The bottom-up thinker pressure-tests those boundaries with real pieces. You want both on the team. You just might not learn which one you've got from an icebreaker slide. And that, I think, is the bigger picture.
