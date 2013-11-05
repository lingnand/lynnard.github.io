---
layout: post
date: 2013-11-05 16:45
title: "Building a Vim-like XMonad -- Prompt, Task Groups, Topical Workspaces, Float styles and more"
#uncomment the next line to use the creation date
#date: 2013-09-21 10:38
comments: true
published: true
categories: [xmonad, vim, haskell, tech, en]
---

About a year ago, a friend of mine reading Computer Science introduced me to XMonad. At that time the window manager I was using - or rather stuck with - is Aqua from Mac OS X, and I was fanatically resorting to tmux as a temporary replacement for all purposes. This might be one of the most important moments for my working environment, since from that time I was enchanted by the simplicity of a Linux system and switched to Arch Linux + XMonad permanently. 

Now one year later I've had enough experience (I can't say I'm a Haskell master though) and it's a good time to review my customizations and share my tips.

As the title indicates, my starting point is with Vim in mind. For me Vim is such an enlightening piece of software that it does not only count as one of the best editors, but also offers a lot of metaphors on improving efficiency, especially for a keyboard-driven workflow.

What I miss from the start
==========================

1. `Mod-o` and `Mod-i` to navigate through the window history 

    [jump to discussion](#window_history)
2. better splitter and buffer *(window)* integration; allow each split window to show multiple buffers *(windows)* 

    [jump to discussion](#splitter)
3. improved command line control through [XMonad.Prompt](http://xmonad.org/xmonad-docs/xmonad-contrib/XMonad-Prompt.html); especially integrate frequently-used commands into the prompt to make a dynamic system like [Alfred](http://alfredapp.com) 

    [jump to discussion](#prompt)
4. topical workspaces with terminals starting inside with custom directories (there are implementations for this in Contrib but I don't think the approach is elegant enough) 

    [jump to discussion](#topical_workspace)
5. dynamic renaming, reorganization, removement of workspaces; but at the same time each workspace should still be linked to a shortcut i.e. `Mod-1` to `Mod-9`, `Mod--` and `Mod-=` 

    [jump to discussion](#dynamic_workspace)
6. `Mod-/` to search the windows; `Mod-m` and `Mod-'` to mark windows and jump to them 

    [jump to discussion](#window_jumping)
7. manage windows in groups according to their different roles/tasks; I call them *task groups*. More specifically
    * `Mod-] <specifier>` and `Mod-[ <specifier>` cycles through the particular group
    * `Mod-d <specifier>` quits all windows in that group
    * `Mod-S-]` and `Mod-S-[` switches between groups
    * `Mod-S-o <specifier>` and `Mod-S-i <specifier>` steps through the window history for that group

    Here `<specifer>` is a single or a sequence of keystrokes to represent a particular task group. As an example I have the following in my `xmonad.hs`

        b   --> Vimb
        v   --> Vim
        z   --> zathura
        m   --> mutt
        f   --> finch
        r   --> ranger
        t   --> idle xterms
        S-t --> all other xterms

    Note that as a convenience a special specifier is designed for each action. For example, `Mod-] <specifier>` and `Mod-[ <specifier>` have specifiers `[` and `]` that will cycle through the *current group* (the group of the currently focused window); similarly pressing `Mod-d d` will delete windows in the current group. 

    [jump to discussion](#task_group)
8. expanding on the concept of task group in the previous point, each group has its particular *float styles* and pressing `Mod-t` and `Mod-S-t` shall cycle forward/backward through these styles. For example, I have defined for my `ranger` instances such that pressing `M-t` once will make it float and occupy the lower half of the screen; pressing it again makes it occupy the upper half of the screen; a third time sinks it back into the layout. 

    [jump to discussion](#float_style)
9. able to toggle a few windows on and off (floating). Think of it as the dock in the Mac OS system; sometimes you need it but sometimes you don't; and the best way to make it readily available without clustering the workspace is to have it hidden in the background and activated via a key. This is essentially [XMonad.Util.Scratchpad](http://xmonad.org/xmonad-docs/xmonad-contrib/XMonad-Util-Scratchpad.html) from Contrib; however, the original scratchpad does not provide workspace-specific scratchpads, so all workspaces have to share the same scratchpad. My extension will work around this limitation and allow a different scratchpad in each workspace, toggled with the same key sequence. 

    [jump to discussion](#scratchpad)
10. a wallpaper system that allows easy previewing and changing of the wallpaper. Press `M-x` to make all windows half-transparent to show the wallpaper (I call this *gallery mode*); press it again to go back to normal mode. Press `M-S-x` to switch to the next random wallpaper - if it's in normal mode, turn on the gallery mode for a few seconds (to show the newly changed wallpaper) and go back to normal; if it's already in gallery mode, do nothing more. 

    [jump to discussion](#wallpaper)

OK, enough said. I think I might have left out a few features but there is already too much to talk about. 

Implementing the missing features
=================================

## <a id="window_history"></a>Window history navigation

As a starting point, [XMonad.Group.Navigation](http://xmonad.org/xmonad-docs/xmonad-contrib/XMonad-Actions-GroupNavigation.html) provides a function `nextMatch` that can navigate back in history like this `nextMatch History (return True)` (shown as example in the doc). Here `(return True)` is the predicate to match all windows in history. However this is pretty useless since as soon as you go back to the last visited window the previous window becomes the new last visited one and essentially all this does is to toggle between two windows.

With a little thought though we can devise this simple barebone algorithm to improve upon that and allow for two-way navigation like in Vim

* before we go backwards in history we **mark** the current window (in some storage) and go to the next **unmarked** window in the history. So now pressing `M-o` multiple times would skip windows previously navigated to since they've already been marked.
* before we go forward in history we **unmark** the current window and go to the next **marked** window in the history. This requires a little more thought to bend the head around but think about this: when you go forward in history you are revisiting the windows that have previously been visited via `M-o`, so these windows *must* have been marked. At the same time you are annulling the effect of `M-o`s so that's the reason for unmarking. 

The algorithm is fairly simple so the details of the implementation are left to you to work out.

## <a id="splitter"></a>Window groups

To have *multiple buffers in the same splitter*, or in other words, *multiple windows occupying the same tile*, all one needs is a tabbed layout embedded in each tile. Luckily this has already been implemented by other people. Check out the *tiled tab groups* in [XMonad.Layout.Groups.Examples](http://xmonad.org/xmonad-docs/xmonad-contrib/XMonad-Layout-Groups-Examples.html)

*Note: the original tiled tab groups seem to have a few bugs to prevent consistent redrawing of the tablines. If you encounter any such problems you can contact me for a fix*

A preview of the effect:

->![](images/xmonad/splitter.png)<-

An implication of this change is that now *EVERY WINDOW* becomes a tab. Additionally, new windows won't change the outer layout in anyway - they just start as a new tab in the current group. This essentially eliminates any need for application-level window tabbing: now you don't need [suckless tabbed](http://tools.suckless.org/tabbed/) for tabbing your xterm windows, or Firefox/Chrome for their ostentatious but aesthetically un-unifiable tabs. The responsibility of window management truely falls to XMonad and it does it tons better (I actually hope that Vim can delegate window management to XMonad too but apparently many functionalities can't be shared across in this way). Just a few reasons why this is so much better:

* one single keystroke to move the current tab to another group, in an entirely visual way. Still remember the 'groups' from Firefox? You have to click the zoom button, drag the tab to the desired group and all. Also, now you can mix all sorts of 'tabs' together - you can put a xterm tab with a browser tab - anyway you want.
* shifting tabs with ease - again, single strokes to move the tab left/right in the group
* create new groups with ease - move any tab out of the current group to form a new group
* unified look and keyboard shortcuts

This is precisely the reason why I've changed almost all my applications to those without window tabbing e.g. Vimb for browsing. This makes everything so much simpler - every window can be treated in the same manner as if it's just a buffer in Vim.

## <a id="prompt"></a>All the glories about prompts

Prompts are an interesting addition to XMonad as it allows manual tasks to be performed anywhere and anytime in an unintrusive way. However, I'm not entirely satisfied by the line of prompt systems included in Contrib - most of them only allow input, and have neglected another important feature of prompt - showing real-time output of the query. This can be best illustrated by Spotlight from Mac OS X or better yet, Alfred. Both apps attempt to show the user search results from the query and allow the user to easily go to any result. I feel that XMonad's Prompt should do the same thing.

Here I'll list a few prompt systems I've designed myself. Some are more complicated than others (might span a couple hundred of lines of code) and I won't indulge into the details. To better understand how a prompt system should be written it's better to consult the documentation directly at [XMonad.Prompt](http://xmonad.org/xmonad-docs/xmonad-contrib/XMonad-Prompt.html). The main purpose here is more about showing what we can do rather than how it can be done.

### Information system 

The information system consists of a calculator (`calc` underneath) and a bunch of dictionaries (`sdcv` underneath). Press `Mod-c <character>` will activate the corresponding prompt i.e. `Mod-c <digit>` activates the calculator and puts that digit into the prompt whereas `Mod-c <letter>` activates the dictionaries and again puts that first letter into the prompt. In addition `Mod-c <Return>` will activate the prompt taking the words from the clipboard as the input.

When checking definitions for words one can press `~` to switch between different dictionaries e.g. WordNet, Thesaurus, etc.

Pressing `<Return>` for the calculator will copy the result into the clipboard whereas for the dictionaries will pronounce the word using `espeak`.

A screenshot of the calculator:

->![](images/xmonad/prompt_calc.png)<-

And an example of using Chinese dictionaries:

->![](images/xmonad/prompt_dict.png)<-

### Vimb prompt

Vimb stores history and bookmarks in plain-text and that makes building a prompt for it (think about Omnibox for Firefox) a breeze. My implementation just repeatedly greps the words on the prompt in Vimb's bookmark and history file and outputs the result in formatted columns; and when the input is empty shows the last 10 visited URLs.

->![](images/xmonad/prompt_vimb.png)<-

### Taskwarrior prompt

I was once attracted by [Taskwarrior](http://taskwarrior.org) and had since written a complete prompt system for it. Pressing `<tab>` and `<S-tab>` will autocomplete tasks and it also shows the real-time output for the filters used in the command. In addition, it autocompletes projects, due times, commands, etc. I've also integrated `taskopen` into the system such that pressing `<Return>` on any focused task automatically opens the notes file for it. 

->![](images/xmonad/prompt_task.png)<-

## <a id="topical_workspace"></a>Topical workspaces

The implementations in Contrib on topical workspaces demand the user to put all the configurations into `xmonad.hs`. I found this too rigid and static - what if I just have created a new directory (a context) and want to start the workspace inside?

As a solution to my quagmire, I've conjured up an entirely different approach - managing topics by tags. Every tag is represented by a directory, and a file/directory with multiple tags will be hard-linked in all those respective directories. 

An example tag structure:

```
$ tree -d ./dev
./dev
├── algorithm
│   └── web_crawler
├── applescript
├── c
│   └── va_list
├── c++
├── graphics
│   └── opengl
├── java
│   └── swing
├── keyboard
├── keyword
│   └── static
├── latex
├── matlab
├── mmd
├── obj_c
│   ├── arc
│   │   └── weak
│   ├── block
│   └── category
├── os
│   ├── android
│   ├── blackberry
│   ├── ios
│   │   ├── coredata
│   │   └── scrollview
│   └── osx
│       └── uti
├── python
├── regex
├── unix
│   └── bash
│       ├── condition
│       ├── find
│       ├── grep
│       ├── osascript
│       ├── sort
│       ├── string
│       └── xargs
└── web
    ├── ip
    ├── js
    ├── php
    ├── xml_rpc
    └── zend
```

Then the task of switching to a particular topic is as simple as searching in the tag database for the tag, creating a new workspace, and storing the path of that tag directory for this workspace. After that load the path for any newly spawned xterms, etc.

The searching of tags, again, is achieved via a prompt.

->![](images/xmonad/prompt_ws_view.png)<-

## <a id="dynamic_workspace"></a>Dynamic workspaces

There is already [XMonad.Actions.DynamicWorkspaces](http://xmonad.org/xmonad-docs/xmonad-contrib/XMonad-Actions-DynamicWorkspaces.html) but I feel like wanting a little more.

What I've added upon it:

### Assign each workspace to a symbol

This is achieved via a customly defined *symbol stream*. It is similar to the Enum stream of `Char` in Haskell but slightly different - the order of the symbols is adjusted to maximize keyboard efficiency. For example, I've defined the symbol stream for my workspaces as follow

    `123457890-=...

The `...` refers to any other symbols e.g. obtained from `enumFrom '='`. You might notice that `6` is left out of the symbol stream and that's right - I've specifically assigned `Mod-6` to toggling between last visited workspaces, as a tribute to `<Ctrl-6>` in Vim. The first symbol, `` ` `` denotes the temporary workspace and cannot be removed.

Why is binding each workspace to a symbol useful? Because then one can perform many tasks with these symbols efficiently.

For example

* `Mod-<symbol>` switches to the workspace with that symbol
* `Mod-S-<symbol>` shifts the focused window to the workspace with that symbol
* `Mod-C-<symbol>` swaps the current workspace with the workspace with that symbol

The symbol stream is also helpful for maintaining a consistent ordering of the workspaces in the `dynamicLog`.

### Bind renaming/adding workspaces to the topic selection prompt I explained in the previous section

The original DynamicWorkspaces implementation simply allows you to type any string as the name of the current/new workspace. What I've done is to replace this prompt with my prompt for topic selection - so renaming/adding workspace would at the same time switch the workspace to the given tag/context (and create the new tag directory if necessary).

## <a id="window_jumping"></a>Window jumping

This is the easiest task of all. Simply use [XMonad.Prompt.Window](http://xmonad.org/xmonad-docs/xmonad-contrib/XMonad-Prompt-Window.html) for `Mod-/` window searching and [XMonad.Actions.TagWindows](http://xmonad.org/xmonad-docs/xmonad-contrib/XMonad-Actions-TagWindows.html) for `Mod-m` and `Mod-'` window marking.

## <a id="task_group"></a>Task groups

The key to task groups is carefully modelled `Query Bool`s (see [manageHook](http://xmonad.org/xmonad-docs/xmonad/XMonad-ManageHook.html)). 

Basically what I've done is defining the `Query Bool`s that match the windows in each task group.

Example:

```haskell
windowGroups = siftedWindowGroups
    [ -- vimb instances
      def { filterKey = "b"
          , filterPredicate = className =? "Vimb"
          , onAbsence = vbPrompt
          }
      -- zathura instances
    , def { filterKey = "z"
          , filterPredicate = className =? "Zathura"
          , onAbsence = spawn "zathura"
          }
    ]
```

Using `ifWindows` from [XMonad.Actions.WindowGo](http://hackage.haskell.org/package/xmonad-contrib-0.11.2/docs/XMonad-Actions-WindowGo.html) you can select the windows using the `Query Bool`; after that, you can pretty much imagine how to extend from it.

To ensure mutual exclusion among groups I've also made it such that the task group of a window will be the first matched group in the definition list i.e. so in the above example, a vimb instance will never be considered a zathura instance.

Some interesting `Query Bool`s I've made:

* `isInCurrentWorkspace`: matches windows in the current workspace
* `currentGroupQuery`: returns the `Query Bool` of the task group for the currently focused window

What all remains is to use list comprehension to generate all the possible key-chords for the actions. So for the above example we will probably generate `Mod-[ b` `Mod-] b` `Mod-d b` `Mod-[ z` `Mod-] z` `Mod-d z`, etc.

## <a id="float_style"></a>Float styles

This should be relatively straightforward, given that the concept of *task groups* is already established in the previous section. Use `currentGroupQuery` to get the task group of the current window and cycle through the pre-defined styles (just plain `ManageHook`s) accordingly. Of course you'd need to store the index of the current style for each window so on next invocation of style-switching it will jump to the correct style, but it should be trivial to implement. 

## <a id="scratchpad"></a>Per workspace scratchpads

The main technique for getting a per workspace scratchpad is through exploiting the original `namedScratchpadAction` function from [XMonad.Util.NamedScratchpad](http://hackage.haskell.org/package/xmonad-contrib-0.11.2/docs/XMonad-Util-NamedScratchpad.html) like this

```haskell
mkPerWSScratchpad cmd = do
    curr <- gets (W.currentTag . windowset)
    con <- perWSScratchpadContext curr
    dir <- getCurrentWorkspaceDirectory
    let csterm = UniqueTerm con cmd "" in
        namedScratchpadAction [ NS "cs" (uniqueTermFullCmd dir csterm) (isUniqueTerm csterm) idHook ] "cs"
```

So that each time the function is triggered the `Query Bool` associated for selecting the scratchpad changes according to the workspace. The code above might be a little abstruse to understand, but basically

1. `perWSScratchpadContext` gets a particular context for the given workspace (a unique string for the workspace)
2. `UniqueTerm` constructs a shell command that will spawn a xterm given a command to run and write the context obtained from the previous step into its own `appName`
3. `isUniqueTerm` checks whether a window is the particular xterm window obtained before by checking its `appName` field

There are more subtle details to consider - for example, what happens when a workspace's name changes? that will change the context obtained from 1 and subsequently lose the scratchpad associated with the old name. To solve this problem we have to introduce another level of indirection: create a map that associates each workspace with a unique handle, and use that handle to generate the context string. When the workspace's name changes it should still be made to point at the same handle so the old scratchpad is still valid for use.

## <a id="wallpaper"></a>Dynamic wallpaper system

This last part is actually not that necessary in terms of functionality; but I still made it for the sake of my aesthetics. [XMonad.Hooks.FadeInactive](http://hackage.haskell.org/package/xmonad-contrib-0.11.2/docs/XMonad-Hooks-FadeInactive.html) provides the function `fadeIf` to paint arbitrary transparencies to windows matching a `Query Bool` and if you've followed through this long post then you might immediately get the idea.

The described *gallery mode* is nothing more than painting transparencies to all windows; whereas the *normal mode* only excludes the focused window.

The trick for going into *gallery mode* for a few seconds and then going back is to use [XMonad.Util.Timer](http://hackage.haskell.org/package/xmonad-contrib-0.11.2/docs/XMonad-Util-Timer.html), which provides a simple interface to construct a timer and do something when the time is up.

Of course, you still need to write your own script or install certain applications to actually switch the wallpaper, but that should be a piece of cake.

Conclusion
==========

I have imagined that this would be a very long post and indeed it is. During writing I myself had to pause for a few times to try to remember what I was doing with *that* particular part of the code and what I was trying to achieve. To be honest I haven't talked much about the implementation details in most cases. This post is more about summarizing my current understanding and imagery of a modern *- or rather, geeky -* window manager and what I have done to approach my ideals. Maybe in the future my ideals will change again and then I should redo/rewrite everything. But as always, *joy lies in endless tinkering / 折腾就是快乐*.
