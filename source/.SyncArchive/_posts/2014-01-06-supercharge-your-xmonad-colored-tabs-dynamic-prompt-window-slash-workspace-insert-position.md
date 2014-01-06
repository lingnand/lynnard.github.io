---
layout: post
title: Supercharge your XMonad! Colored Tabs, Dynamic Prompt, Window/Workspace Insert Position
comments: true
date: 2014-01-06 17:57
categories: [xmonad, en]
published: true
---

Here's another post to show some of my latest hacks on XMonad.

Colored Tabs
============

->![](/images/xmonad/color_tabs)<-

One of the things that annoyed me for the Tabbed layout is that I couldn't make out which program is in which tab from the look of the tabs alone easily. Having a different theme for different [task groups](http://lynnard.tk/blog/2013/11/05/building-a-vim-like-xmonad-prompt-task-groups-topical-workspaces-float-styles-and-more/#task_group) thus becomes a natural choice for me.

To apply this mod the key is to modify the `updateDeco` method in `XMonad.Layout.Decoration`.

A little thanks goes to *OODavo* from *#xmonad*.

Dynamic Prompt
==============

The stock `XMonad.Prompt.Shell` is fine but I can't think of any other things it can do aside from launching programs.

My dynamic prompt, in contrast, serves to

* launch apps with shell completion (of course)
* render output directly in the autocompletion window for *some* commands *e.g. man head*
* opens the file or directory on prompt directly using [rifle](http://ranger.nongnu.org/)

Now I don't have to launch a ranger instance in each and every workspace anymore.

Window/Workspace Insert Position
================================

This one is not totally necessary (I think), but I consider it useful at times.

Window Insert Position
----------------------

Using `XMonad.Hooks.InsertPosition` you can easily decide whether the focus should stay with the old window or transfer to the new window when it is created; the module also claims to support changing the insert position of the new window but I haven't been able to make that work (probably it's a compatibility problem with my complicated custom layout). I've further extended this module to allow dynamic toggling of the *focus-stay-with-old-or-new-window* feature (let's call it *window insert position* feature) via a keyboard shortcut.

Why would this be useful? Imagine you are using a browser application like `vimb`; it does not manage windows itself so new web pages opened by it will always take the focus in XMonad. Now with this mod you can decide that behavior. In fact I've defined `g;t` in my `vimb` (the shortcut responsible for entering the hint mode and continue opening the activated links until the mode is quit via `Esc`) such that it automatically opens the tab in the background and when it finishes recovers the old *window insert position* configuration.

Workspace Insert Position
-------------------------

Continuing from my [Dynamic Workspace](http://lynnard.tk/blog/2013/11/05/building-a-vim-like-xmonad-prompt-task-groups-topical-workspaces-float-styles-and-more/#task_group), the following keybinding are added:

* `M-a {a,M-a}`: launches the workspace creation prompt and on completion insert the workspace *just after the current one*
* `M-a {i,M-i}`: same as before but on completion insert the workspace *just before the current one*
* `M-a {I,M-I}`: you've guessed it; insert at the beginning
* `M-a {A,M-A}`: insert at the end

There is another set of key bindings beginning with `M-S-a` which creates the new workspace in the same way but also at the same time move the current window to it.
