---
layout: post
title: "A programmable tooltip on Mac OS"
description: "Universal Tip can save you thousands of hand movements of repetitive micro-workflows."
date: 2020-04-19
---

Several months back, I've started a new job and immediately found myself perform some repetitive micro-workflows like:

* Copying an ID, pasting it on the browser URL's bar, typing some url, and hitting enter
* Copying a millisecond-from-epoch number, visiting a website that can convert it to human-readable date, pasting the number, and hitting enter

I'd do these workflows many times a day. It was tiresome and required to be quite skilled (motor-wise) to perform these flows quickly.

So, I've decided to build [Tip](https://github.com/tanin47/tip) to perform these workflows faster.

Tip is built upon ["Services" (or "System-wide Services")](https://developer.apple.com/design/human-interface-guidelines/macos/extensions/services/), 
which allowed one application to send a selected text to another application. 

With this mechanism, Tip can be used with any Mac OS app! 

Tip also only see the text when user explicitly triggers this feature by hitting the configured shortcut.

Tip gets the selected text, calls the user script, and renders the result in a tooltip as shown below:

![Convert seconds from epoch to time and copy](https://media.giphy.com/media/f952ZuRG9kqCoxGt8v/giphy.gif)

Then, you can select an item to perform the defined action. Currently, Tip supports 2 actions: (1) copying to clipboard and (2) opening a URL, which can be used to trigger a Mac app that supports URL like opening a file in IntelliJ.

With a user script that you write yourself, this means you can customize Tip to do anything you want.

My favourite technical detail on Tip is that it runs in [App Sandbox](https://developer.apple.com/library/archive/documentation/Security/Conceptual/AppSandboxDesignGuide/AboutAppSandbox/AboutAppSandbox.html)
without requesting for additional permissions. Tip can only execute (and read-only) a user script placed in `~/Library/Application\ Scripts/tanin.tip/`. 
Privacy-wise, this is a huge win.

I've been using Tip for 3-4 months now with many use cases. Surprisingly, the use case I use most frequently is:
selecting a file in an error stacktrace and opening that file in IntelliJ/Github. The user script uses the text to search through local file and offers a tooltip item to open matching files on IntelliJ/Github. It looks like below:

![Open file on Github from an error stacktrace line](https://media.giphy.com/media/JSYWptFElQmDJOXzXO/giphy.gif)

I'd love for you to try Tip, and I hope it makes you move faster: [https://github.com/tanin47/tip](https://github.com/tanin47/tip)
