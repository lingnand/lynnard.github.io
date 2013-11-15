---
layout: post
date: 2013-11-15 23:41
title: "Making Vim play nice with your Window Manager"
#uncomment the next line to use the creation date
#date: 2013-11-15 22:00
comments: true
published: true
categories: 
---

Motivation
==========

For people who've used Vim extensively and exclusively, you've probably seen this dreaded message a dozen of times

->![](/images/vim/swap_warning.png)<-

For most cases this glaring warning is there just because you've forgotten having opened the same file in another Vim instance - and it happens for me a lot. Therefore for a long time I've been thinking of an extension to Vim that:

1. whenever a file with an existing swap is asked to be opened, instead of showing the message, jump to that editing session of Vim and switch to the file in question
2. when a file is asked to be opened check whether there is already some Vim instance lying around in the current workspace; if yes, then forward that request to the existing instance

Collecting the pieces
=====================

One can argue that such features should come shipped with any modern editor - but well, this is our good old Vim running in terminals, so we probably shouldn't complain too much. But the good news is, I've finally come to a solution for this based on [wmctrl][wmctrl] (and I don't know why I didn't bump into this little gem before).

To start with, [wmctrl][wmctrl] is a little program that interacts with your window manager on the command line (yes that is possible). The most useful features include listing the workspaces, checking for window information, jumping to a specific window based on title or window ID.

The other piece of the puzzle is Vim's own server-client feature, which I've somehow looked over in the past. It's actually very simple: 

* To start a server-enabled Vim

        vim --servername <name> ARG1 ARG2 ARG3

* To send command to a server-enabled Vim

        vim --servername <name> --remote FILE1 FILE2

    This will connect to the vim by the server name and make it edit the files given in the rest of the arguments.

* To query information regarding the remote vim, you can use

    vim --servername <name> --remote-expr {expr}

    This will connect to the vim server, evalute `{expr}` in it and print the result on stdout.

Another interesting discovery of mine is that Vim actually includes a plugin called `editexisting.vim` for the default installation. This will be the script we build upon.

Taken from `editexisting.vim`:

> 1. On startup, if we were invoked with one file name argument and the file is not modified then try to find another Vim instance that is editing this file.  If there is one then bring it to the foreground and exit.
> 2. When a file is edited and a swap file exists for it, try finding that other Vim and bring it to the foreground.  Requires Vim 7, because it uses the SwapExists autocommand event.

Most of the script works fine, except the part which concerns itself with *bringing (the remote Vim session) to the foreground and exit*. From my testing it doesn't work with XMonad (and I guess it wouldn't work with other lightweight window managers as well under Linux). So our primary aim would be to fix this problem.

Solution
========

Since we've had `wmctrl`, what we need to do is really

1. get the process id of the remote vim that is editing the same file
2. get the process id of the window that actually holds that vim; this is achieved by repeatedly getting the parent pid and checking against the process ids given in `wmctrl -lp`
3. from the table of `wmctrl -lp`, get the title for the window and use `wmctrl -a TITLE` to jump to the window

This is best shown in code

{% include_code sh wtitle %}

The above code will print the title of the window given a process id. To use it in the vimscript:

```vim
let pid = remote_expr(servername, "getpid()")
" execute the wmctrl command
call system("wmctrl -a \"`wtitle " . pid . "`\"")
```

Now the second task I mentioned is to forward all file opening requests to the same vim instance within the workspace. This is a little more complicated than the previous task, but there's nothing especially tricky

1. get the list of vim servers by `vim --serverlist`; for each server get its pid
2. for each pid get its window title using the `wtitle` script shown above
3. check whether this window title corresponds to the same workspace as the current one
4. if yes, we've obtained the vim instance in the current workspace
5. else if there's no vim instance in the current workspace, we should then start a new server-enabled vim

All this can be wrapped up in this tiny script

{% include_code sh xvim %}

[wmctrl]: http://tomas.styblo.name/wmctrl/
