+++
title = "Another Site, Another Framework"
date = 2026-01-15
[taxonomies]
tags = ["meta", "web"]
+++

Yes. You read right. Another site.

I've lost count of how many times I've rebuilt this thing. There was the first attempt: raw HTML and CSS, back when I thought that was impressive. Then there was the Semantic UI phase. "Hey, at least I am trying Semantic UI. It's meh, like Bootstrap, has a lot of ready stuff, but worse responsiveness." That's a direct quote from my old README, and I stand by it.

Then there were a couple of false starts with various frameworks that never made it past `localhost:3000`. We don't talk about those.

## Why Zola

I needed something that:

1. **Generates static HTML.** I'm not spinning up a Node server for a personal site. I'm not paying for hosting. GitHub Pages, done.
2. **Doesn't require npm.** I have enough `node_modules` black holes in my life already.
3. **Uses Markdown.** I write in Markdown anyway. Let me just... use it.
4. **Builds fast.** Zola is written in Rust. It builds the entire site in milliseconds. I've waited longer for Maven to resolve dependencies.

Hugo was the other contender, but Go templates make me question my life choices. Zola uses Tera, which is close enough to Jinja2 that my brain doesn't have to context-switch too hard.

## The Theme Situation

I'm using [anemone](https://github.com/Speyll/anemone). It's minimal, it's clean, it supports dark mode, and it doesn't ship 400KB of JavaScript to render text on a screen. That's all I need.

I've added some custom CSS and a template override to handle pages without dates (because Zola really wants everything to have a date, and sometimes things just... don't). There's also a theme toggle with a click sound, because why not.

## Will This One Last?

Probably not. But it'll last longer than the Semantic UI one, and that's progress.
