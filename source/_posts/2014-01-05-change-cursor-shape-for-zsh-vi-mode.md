---
layout: post
title: Change cursor shape for zsh vi-mode
comments: true
date: 2014-01-05 18:36
categories: [en, zsh]
published: true
---

I've been using zsh and its wonderful *vi-mode* line editing keybindings for a long time, but one thing that has always troubled me is the lack of visual clues to the current *mode*.

With [oh-my-zsh](https://github.com/robbyrussell/oh-my-zsh) you can optionally have an indicator to the right of the prompt telling you this, but I feel it's not *native* enough - we need something simpler and more direct, just as in vim I've set my cursor to be an underscore in the insert mode and a solid block in the command mode.

Luckily you can do just that with the powerful zsh shell.

First off the relevant escape sequences for changing the cursor shape are:

* `"\e[4 q"`: solid underscore
* `"\e[2 q"`: solid block

The hook called when your vi mode changes is `zle-keymap-select`. If you want comprehensiveness, also include `zle-line-init` and `zle-line-finish`.

Now just append this to the end of your `zshrc`.

```sh
zle-keymap-select () {
    if [ "$TERM" = "xterm-256color" ]; then
        if [ $KEYMAP = vicmd ]; then
            # the command mode for vi
            echo -ne "\e[2 q"
        else
            # the insert mode for vi
            echo -ne "\e[4 q"
        fi
    fi
}
```

*Boom*, you've just configured your cursor to change shape on the fly according to your editing mode!
