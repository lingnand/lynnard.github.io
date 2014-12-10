---
layout: post
title: "More XMonad goodies: 3-dimensional Workspace, Window Sorting and Shelving"
date: 2013-12-30 23:37
comments: true
categories: [xmonad, en]
---

Following up my previous [entry](http://lynnard.me/blog/2013/11/05/building-a-vim-like-xmonad-prompt-task-groups-topical-workspaces-float-styles-and-more/) on some *advanced vim-like* feature addition to XMonad, I've played around with my configuration a bit more and apparently made it more than 2000 lines, marking a new milestone where my config file has even more lines than the core xmonad source.

*Well, that can be a good or bad thing, depending on how you look at it.*

But the good news is that most of the new additions I consider quite useful, and therefore would take some time here to share them with those interested.

3-dimensional Workspace
=======================

Inception
---------

If you've followed my previous blog on XMonad then hopefully you've grasped my idea at building a *vim-splitter-like* layout for workspaces (if you've forgotten by now, [here](http://lynnard.me/blog/2013/11/05/building-a-vim-like-xmonad-prompt-task-groups-topical-workspaces-float-styles-and-more/#splitter)'s the refreshment).

After I wrote that article, I've taken some more serious moments at pondering about this and have gotten a few more insights:

1. For the inner-most layout, **Tabbed** is still by far the most efficient available. The functional equivalent in vim is a *split view*. The rationale here is that you want to divide the whole screen into some non-overlapping rectangles, but at the same time you want to allow some windows to be *viewed* from the same rectangle. For example, imagine you have two rectangles - one for *research*, and the other for taking *notes* - you might want to open multiple browser windows in the *research* rectangle and switch between them while keeping the same *notes* rectangle on the other side.
2. What's the best layout for managing these *rectangles*, you ask? For me, I used the stock **Tall** layout for a *long, long* time but I had to admit it's not very efficient. For one thing, although you have all the rectangles tiled, their position and width are not really controllable - the only thing you can do is shrinking and expanding the master view. On the other hand, the rectangles are essentially tiled in a one-dimensional manner - you navigate through these rectangles using *up's* and *down's* and at each point of time you only have access to this one axis. 
3. Again we turn to vim for inspiration - in vim, you can split each *view*, or *rectangle*, *vertically or horizontally*. You can navigate between these views *vertically or horizontally*. You can also shrink or expand each view *vertically or horizontally*. More importantly, when you perform these actions, other views aren't affected in major ways, and that's a really nice advantage which many people might overlook - a serious drawback of dynamic tiling (which XMonad is based on) is that each new window can disrupt the positions and sizes of existing windows, and that can be very frustrating for the person sitting before the screen trying to remember all of them - *YOU*.

In light of these observation and thoughts, an obvious conclusion is that we should use a layout that allows the same sort of power, as well as the flexibility, afforded by the similar in vim. The problem is that there isn't such a layout available. If you think carefully about this, you'd realize that this wouldn't be an easy one-time pull-off even for those Haskell gurus.

Problem-solving
---------------

*What should we do then?*

Luckily, during my mulling over this, I turned to [XMonad.Layout.Groups](http://xmonad.org/xmonad-docs/xmonad-contrib/XMonad-Layout-Groups.html) and [XMonad.Layout.Groups.Examples](http://xmonad.org/xmonad-docs/xmonad-contrib/XMonad-Layout-Groups-Examples.html) again. The **Rows of columns**, or as it is called in the examples, seems to be the exact answer to my need. Essentially a combination of **ZoomRows**, it allows one to divide a workspace into multiple columns, which each then holds several rows. The size of each *rectangle* within can be adjusted in both dimensions. However, by using this layout we have to forgo **Tabbed** for each rectangle - the **Rows of columns** is already taking advantage of the **Groups** combinator and we have no way to insert another level of group nesting below it.

*Or do we?*

As I played around with the **Groups** source code, it turns out that further nesting of groups is indeed possible, with only minor tweaks and limitations. Simply open the source file for `XMonad.Layout.Groups` and add this function

```haskell
group3 l l2 l3 = Groups g l3 start (U 2 0)
    where g = group l l2
          start = fromJust $ singletonZ $ G (ID (U 2 1) g) emptyZ
```

Then in your `xmonad.hs` you can use a *3-dimensional layout* like this

```haskell
rectTabs = G.group3 (addTabs shrinkText myTabsTheme Simplest) (Mirror (zoomRowWith GroupEQ) ||| Full) (zoomRowWith GroupEQ ||| Full)
```

An additional benefit with using a combinator like **Groups** is that you can switch between different layouts for each level of nesting, as you can see from my code above, which allows me to switch between rows/columns and **Full** layout.

In more practical terms, this allows us to

- group windows into work contexts at the *workspace* level e.g. `xmonad`, `school project`, etc.
- group windows in each work context into functional sub-groups in the form of columns e.g. inside `xmonad`, you can have a column devoted to `research` (many browser and document windows) and another one devoted to `code` (vim windows)

    Note how similar this concept is to the `taskGroup` concept that I spent a lot of words explaining in my previous blog. Later in this blog you'll see how I combine these two concepts together and make *window sorting* possible.

    *Also, just a slight digression:* I saw a lot of tutorials or videos online teaching people how to put all browser windows into one workspace and all text-editors into another - in my opinion such approaches miss the entire point of window tiling: if you can't compare an editor and a browser window side by side then there's no difference from just using tabbed editor and tabbed browser in a normal window manager - if you don't *need to* compare two windows, why bother *tiling them* in the first place? In that case a tabbed layout would be the most efficient one that makes full use of all screen estate.

    My previous setup - that is, **Tabbed** groups embedded in a **Tall** layout - does not allow much functional separation among windows, assuming I'm using a topical assortment of workspaces. This problem is now fully solved: the column dimension, as I explained at the beginning of this bullet point, serves to separate windows by functions. If I switch the layout at the column level to **Full**, then I essentially have `research` and `code` workspaces inside a bigger workspace called `xmonad` (if you'd like to visualize it in that way), and I can switch back to columns and compare these functional groups if I need to.
- group windows in each functional group, or column, into rows (visually, it's probably more straightforward to call them rectangles or cells) e.g. inside the column `code`, you can have a `editor` rectangle holding vim instances and another `debugger` rectangle positioned below it holding xterms
- lastly for windows that do not need to be compared against each other, group them into tabs and fit them into each rectangle e.g. in the `debugger` rectangle you can have `xterm 1`, `xterm 2`, etc.

This is getting a little confusing, so we'd better hurry to the next section which explains how to output these complicated group nestings into human-readable forms.

Piecing it together
-------------------

Here's what my dzen *log bar* typically looks like

->![](/images/xmonad/logbar.png)<-

Some of the terms might require explanation:

* columnLayout: the layout used by the outermost group, or the columns in a workspace
* rectLayout: the layout used by the intermediate group (the one below columns but above the tabs)
* auto sort: `M` means *manual* while `A` means *auto*; will be explained in [a later section](#auto_sort)
* history: just like the history indicator in vim; `+` and `-` mean there are *newer* and *older* windows to navigate to respectively; refer to [window history](http://lynnard.me/blog/2013/11/05/building-a-vim-like-xmonad-prompt-task-groups-topical-workspaces-float-styles-and-more/#window_history) if you are curious about how this is implemented
* ref key: the highlighted letter is the `filterKey` of the respective `taskGroup` (refer to [task group] if you don't understand), which reminds me what filter key to press to perform actions on that group

The main part relevant to our 3-dimensional layout is the part labelled `columns` and `rows` - each column is also referenced by a number (just like workspaces); and each row is contained in a pair of square brackets for clarity.

It took me quite a while to hack the original **Groups** layout to enable proper logging of the full window structure. Those interested can scroll down to the bottom and go to my github link directly to see the source.

Of course, a 3-dimensional layout would be less interesting if you don't have a full set of operations to perform on it. Here I'll just list a few operations that I normally use

* `M-{k,j,h,l}`: navigate up/down/left/right to the rectangle on the corresponding direction
* `M-{p,n}`: navigate to previous/next tab
* `M-S-{p,n}`: swap the current tab with the previous/next tab
* `M-C-{h,l}`: split a window out to the left/right to form a new column
* `M-C-{k,j}`: split a window out to the up/down to form a new row/rectangle within the current column
* `M-S-{k,j,h,l}`: move a window up/down/left/right to the rectangle on the corresponding direction
* `M1-<num>`: go to the `num`'th tab within the current rectangle
* `M1-S-<num>`: swap the current tab with the `num`'th tab
* `C-<num>`: go to the `num`'th column
* `C-S-<num>`: move the current window to the `num`'th column
* `M-S-q`: close all tabs in the current rectangle
* `M-C-q`: close all windows in the current column
* `M-C-S-{k,j}`: swap the current rectangle up/down
* `M-C-S-{h,l}`: swap the current column left/right
* `M-{-,+}`: shrink/expand current rectangle vertically
* `M-{<,>}`: shrink/expand current rectangle horizontally
* `M-{=,\}`: reset the focus

There are *many, many* more. In fact, since there are three levels of nesting within a single workspace, there is such an abundance of operations to invent that I eventually ran out of keyboard shortcuts to assign.

Window Sorting
==============

Like I mentioned in the last section, the new dimension - `column` - allows me to group windows by their functions inside each workspace; and since I've already defined many task groups in my config file, why not combine the two together? The results are what I call *Window Sorting* and *Auto Window Sorting*.

*Note: you'd need to first understand [task group] before you can possibly understand what I'm doing here*

Manual Window Sorting
---------------------

I've assigned `M-s` for this useful function. As soon as I press down this key combination, my XMonad loops through all the windows inside the current workspace and put them into columns, according to my task group definition. For example, if I have two `vimb` windows, one `vim` window and another `zathura` window all jumbled together in a single rectangle inside a workspace, pressing `M-s` gives me back three columns - one holding the two `vimb` windows, another holding the `vim`, and the last one holding the `zathura`. My function also tries to be as smart as possible: it will keep the focus on the same window after sorting; and it also tries to maintain the position of the current rectangle in the final assortment.

On the implementation side, I don't feel there's too much *magic* to talk about. Sorting is just like filtering, which also makes use of `Query Bool`s. If you are familiar with them then that's good - you can do a hell lot of stuff with these predicate functions.

<a id="auto_sort"></a>Automatic Window Sorting
----------------------------------------------

This is more interesting as what I'm trying to achieve is essentially `Manage Hook` within *workspace*. Each workspace starts off in the *manual* mode, as indicated by the `M` in the sample image of my dzen bar. But if I press down `M-S-s`, *automatic* mode is turned on and the `M` changes to `A`. In the *automatic* mode, each new window is inserted into the column which has the highest percentage of windows of the same task group; if such column is not found, a new column is created before the current column for that new window to fit in. An example will better illustrate the use case: 

1. imagine you are in a workspace and you've spawned a `vim` window to write some code in
2. now you want to search for something, so you press `M-[ b` - this triggers the [cycling protocol](http://lynnard.me/blog/2013/11/05/building-a-vim-like-xmonad-prompt-task-groups-topical-workspaces-float-styles-and-more/#task_group) of the `vimb` task group, and since there are no `vimb` windows to cycle about, the construction function of the group is invoked and a new `vimb` window is created
3. since this is the first `vimb` window and there are no columns found in the workspace to contain a window with the matching task group, the `vimb` window is put into a new column and the focus automatically transfers to it
4. all subsequent new `vimb` windows shall be inserted in that new column just created, given that you don't do any manual arrangement

As you can see, this automates some common sorting tasks while at the same time preserves the manual arrangement made so far by the user. I found this useful especially for simple work contexts (workspaces) containing less than 10 windows.

Shelf
=====

Although we can now manage windows with *3 dimensions*, maximise a column or even maximise a rectangle, sometimes we still wish that a window can be stashed somewhere temporarily. In my previous post I talked about how we can use [perworkspace scratchpad](http://lynnard.me/blog/2013/11/05/building-a-vim-like-xmonad-prompt-task-groups-topical-workspaces-float-styles-and-more/#scratchpad) to achieve this aim, but what I realized later is that it still has some major limitation, the biggest of which is that all the scratchpads have to be pre-defined in the config file.

The alternative is to use [XMonad.Layout.Minimize](http://xmonad.org/xmonad-docs/xmonad-contrib/XMonad-Layout-Minimize.html). This allows any window in a workspace to be toggled, but it also has problems:

1. the user cannot see directly how many and which windows are currently minimized
2. the user can only un-minimize windows one by one using the un-minimize shortcut

Fortunately they are both easy to solve:

1. using hacks similar to what I've done to **Groups** for group structure displaying, we can show the minimized windows on our bar, just as in my sample image
2. since I've already invented many ways to navigate to a particular window (by *filterKey cycling*, by *history*, by *title prefix*, etc.) we can easily retrieve a minimized window by one of these means

What's more?
============

This is by no means the end of all possible things one can do with XMonad. In particular, I haven't played around with `UrgencyHook`s and a couple of other seemingly useful modules. The more I customize XMonad, the more I begin to appreciate how powerful and flexible it is. For those interested in the customizations I've talked about in my blogs, I've managed to put all of them into a single repo. [Here](https://github.com/lynnard/xmonad).


[task group]: http://lynnard.me/blog/2013/11/05/building-a-vim-like-xmonad-prompt-task-groups-topical-workspaces-float-styles-and-more/#task_group
