---
layout: post
title: "The best way to use Svelte with Playframework"
description: "sbt-svelte integrates Svelte with Playframework's asset generation process. It's the cleanest way to use Svelte with Playframework."
date: 2023-07-02
category: main
---

When building a web application on Playframework, there is no good way to integrate a JS framework like Svelte.

The only way is to run Play and run the `npm watch` in 2 separate terminals. 
One is to build the backend code, and the other is to build the frontend code. It feels brittle.

[sbt-svelte](https://github.com/tanin47/sbt-svelte) is offering a better and slicker way to develop with Playframework and Svelte.

Playframework already has a mechanism to watch the JS code and hot-reload during `sbt run`. Playframework's asset generation, specifically `com.typesafe.sbt.web.incremental.syncIncremental((Assets / streams).value.cacheDirectory / "run", sources)`, 
enables us to monitor all changes occurring under `/app/assets`.

sbt-svelte utilizes this mechanism to monitor all `*.svelte` file changes and invoke Webpack to compile *only* those changed files.

This means you just run `sbt run`, and that's it. All your Svelte changes will be compiled when you reload the page. There's no separate process to run. This works with `sbt stage` as well.

The mechanism is beautiful.

At the end of the day, what sbt-svelte does is compiling a Svelte component into a JS file. Then, you'd include the JS file into your HTML file. 
You can make the component full-SPA, hybrid, or non-SPA. It's up to you.

As a side note, sbt-svelte is built in the same way [sbt-vuefy](https://github.com/GIVESocialMovement/sbt-vuefy) (for Vue.js) is built. 
sbt-vuefy has been used by [GIVE.asia](https://give.asia) for years now, so we are confident it is robust.
While you visit GIVE.asia, please also consider donating!

I'd love for you to try it out. You can take a look at [the test project](https://github.com/tanin47/sbt-svelte) to see how it works.

### Bonus - Why using Svelte over React / Vue?

If you look through a lot of materials online, 
I've deduced that Svelte is more suitable for mobile web due to a couple reasons:

1. The size is probably smaller. It doesn't require a separate runtime. 
2. It's probably faster for simpler use cases because it compiles to vanilla JS and doesn't operate on virtual DOM.

React / Vue, on the other hand, requires a runtime and operates on virtual DOM. 

If I understand it correctly, React and Vue manipulate virtual DOM and diff against real DOM. For complex use cases, this allows React to batch multiple changes and might result in better performance.
