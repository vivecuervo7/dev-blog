---
title: "PostCSS: Styling without the Sass"
description: Embrace PostCSS' modular approach to styling and enjoy a codebase with... Less Sass
slug: less-sass-with-postcss
date: 2025-03-28 00:00:00+0000
image: cover.jpg
categories:
  - Development
  - CSS
tags:
  - Styling
  - PostCSS
weight: 1
links:
  - title: PostCSS
    description: A tool for transforming CSS with JavaScript
    website: https://postcss.org/
---

Whenever I decide to do any programming outside of work hours, I like to pick up a bunch of tools that I don't get to touch in my day-to-day, even if it's just so side projects don't _feel_ like work.

Sometimes, I stumble across some tooling that I want to roll into my standard workflow &mdash; [PostCSS](https://postcss.org/) being one that I've stumbled across a couple of times now and decided to cobble together something a little firmer than just a "hey, I tried this and I liked it". More specifically, I have found myself leaning towards PostCSS as a replacement for the default of dragging Sass into every new project I've worked on.

Don't get me wrong &mdash; Sass is and has been a great tool to use over the years, but I feel as though I've only ever really needed a fraction of what it offers. Not to mention that the growth of CSS itself has slowly started eating away at the benefits I've found in Sass, such as nesting or variables.

## What is PostCSS

Or maybe it's easier to talk about what it isn't.

It certainly isn't a pre-processor like Sass or Less. It doesn't use a different syntax the way these pre-processors do, allowing us to instead write standard CSS. Which is good for newcomers who may not be familiar with a particular flavour of pre-processor!

It also isn't strictly a _post_-processor, despite the name alluding to such. The rich plugin ecosystem really allows it to act as both a pre- and post-processor. It's modular, and highly flexible &mdash; you could even keep your desired pre-processor around. This isn't an either-or scenario.

### Ok... so, what _is_ it?

PostCSS is, in their own words, "a tool for transforming CSS with JavaScript".

What PostCSS does is transpile your written CSS into JavaScript, allowing us to process it before writing it back as CSS. By itself, it doesn't actually transform anything &mdash; for that we'll need to tap into the plugins available to PostCSS.

On the topic of plugins, there are a few of them in the wild which are regularly used &mdash; often without the explicit intention to use PostCSS. [Autoprefixer](https://autoprefixer.github.io/), [cssnano](https://cssnano.github.io/cssnano/) and [Stylelint](https://stylelint.io/) are all commonly used PostCSS plugins, meaning there's a good chance you've already used PostCSS!

### Wait, so if it's _not_ a pre-processor...

Yep, that's right. You can stick with your trusty Sass, Less, or Stylus.
Or, you could bring in a few plugins to get your pre-processor-like functionality. This is where a lot of my interest in PostCSS started, looking at it as a potential drop-in replacement for Sass entirely. I'll cover some of [those specific plugins below](#postcss-the-pre-processor).

## Plugins

All the useful things you can do with PostCSS come from plugins. Below we'll have a look at some of the plugins that I've found consistently seem to bubble up as the more useful picks.

If you want to see _all_ of the plugins, [here is a good place to start](https://postcss.org/docs/postcss-plugins).

### The plugins you probably want

**Autoprefixer**

[Autoprefixer](https://github.com/postcss/autoprefixer?tab=readme-ov-file) adds vendor prefixes to your CSS using rules from [Can I Use](https://caniuse.com/). Forget about manually writing vendor prefixes entirely! Stylelint can also be configured to [disallow writing vendor prefixes](https://stylelint.io/user-guide/rules/property-no-vendor-prefix/) too, so you don't even need to remember to forget about vendor prefixes.

**cssnano**

[cssnano](https://github.com/cssnano/cssnano?tab=readme-ov-file) makes your CSS small. It's a minifier that compresses your CSS to reduce your overall bundle size. There are a few presets, the default being a safe option, making it easy enough to drop in that it seems almost silly to _not_ include it. The [full list of optimisations can be found here](https://cssnano.github.io/cssnano/docs/what-are-optimisations/).

**Stylelint**

Ok, ok. You probably won't install this as a plugin the way you will the others, but it's still a PostCSS plugin so I'll include it here. [Stylelint](https://stylelint.io/) is simply put, a linting tool for your CSS. There are a [bunch of useful rules](https://stylelint.io/user-guide/rules) that are worth perusing.

If the rules included with Stylelint aren't quite enough for you, there are a bunch of [plugins, configs and integrations](https://github.com/stylelint/awesome-stylelint) available. Just in case you need plugins for your plugins.

### Tomorrow's CSS, today

This one is in my opinion one of the most powerful and compelling reasons to use PostCSS, so it's getting its own section.

[PostCSS Preset Env](https://github.com/csstools/postcss-plugins/tree/main/plugin-packs/postcss-preset-env) essentially allows us to use modern CSS without worrying about it not being supported in a bunch of browsers (looking at you, ~Internet Explorer~ Safari). By leveraging [CSSDB](https://cssdb.org/) and your specified browser targets, it automatically includes the necessary plugins to ensure consistent behavior across different environments.

You can even start using experimental or proposed CSS features. PostCSS Preset Env's complete feature list can be found [here](https://github.com/csstools/postcss-plugins/blob/main/plugin-packs/postcss-preset-env/FEATURES.md). Some favourites are listed below:

- [custom-media-queries](https://cssdb.org/#custom-media-queries)
- [custom-selectors](https://cssdb.org/#custom-selectors)
- [light-dark-function](https://cssdb.org/#light-dark-function)

### Sass-like functionality

The last time I looked at bringing in a bunch of Sass-like functionality, I found myself installing a bunch of plugins for all the various things. Those individual plugins are still available if you really want to bring them in individually, such as [postcss-mixins](https://github.com/postcss/postcss-mixins).

It looks as though the easiest path to similar functionality might be to just go straight for [postcss-advanced-variables](https://github.com/csstools/postcss-advanced-variables?tab=readme-ov-file). This plugin brings Sass-like `$variables`, `@if`, `@else`, `@for` and `@each` rules, as well as `@mixin` rules. And honestly, with nesting already supported, this covers 90% of the things I really got out of Sass anyway. A counterpart for Sass' maps exists in [postcss-map-get](https://github.com/Scrum/postcss-map-get), and really the last missing piece is support for functions.

To that end, it seems the Sass-like [postcss-define-function](https://github.com/mcattx/postcss-define-function) plugin for this hasn't seen any love in a while &mdash; and I'd probably consider using [postcss-functions](https://github.com/andyjansson/postcss-functions) which is a bit different in that it leans towards defining functions in JavaScript.

If I found myself needing any other features that weren't readily available via existing plugins I'd likely start considering a move to just go back to using Sass, but I think the above would cover just about all the functionality I've actually used over the past few years.

**TL;DR** [postcss-advanced-variables](https://github.com/csstools/postcss-advanced-variables?tab=readme-ov-file) and [postcss-map-get](https://github.com/Scrum/postcss-map-get) is _probably_ going to give you most of what you're using Sass for today.

### Writing custom plugins

While I haven't found the need to do this myself, PostCSS also purports to make it fairly straightforward to [write your own custom plugins](https://postcss.org/docs/writing-a-postcss-plugin), which is a huge leap from being at the mercy of whatever functionality is offered by one of the pre-processor options.

Chances are however that you'll generally find what you're after. There are even existing plugins [to let you use yards, feet and twips](https://github.com/sebdeckers/postcss-imperial)! Don't do that, though.
