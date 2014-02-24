---
title: "More prompt stuff: colored prompt, dynamic prompt/preview widget"
categories: [xmonad, en]
---

It's been a long time since I last wrote something on XMonad. It's not that I don't have anything to share in the last couple of weeks, but rather, I felt it a little bit troublesome to explain some of my most recent tweaks. But enough with my laziness, I finally decided to spend some quality time to write on these nifty new things.

Colored prompt
==============

Ever thought about having a colorful prompt system within XMonad? Now it is possible. I first got this idea when I was working on the taskwarrior prompt system - the taskwarrior program supports outputting ANSI colors, but apparently the stock XMonad.Prompt wouldn't do anything for that extra information. 

The functions behind these color renderings aren't terribly complicated - it just looks for terminal color sequences and transform them into hex color codes that can be printed easily using standard X library.

->![](/images/xmonad/colored_prompt.png)<-

However, here's the catch: if you are going with the color, make sure you know something about color encoding and have the time and effort to tweak it to make it look nice. Unfortunately I have neither of those and that's why I ended up not using any color after all.

More on Dynamic Prompt
======================

The concept of dynamic prompt was first introduced in one of my earlier [post](/blog/2014/01/06/supercharge-your-xmonad-colored-tabs-dynamic-prompt-window-slash-workspace-insert-position/).

One major gripe I've always had regarding standard unix tools is that it's not that straightforward to perform certain tasks chained together in a visual and direct way.

For example, say you want to remove a file. You know it's located in a folder with `Prompt` in its name. You also know that the file has the keyword `shell` inside.

What you'd normally do is probably like this:

Step 1
------

~~~
$ pwd
/home/lingnan/DB/indie/XMonad
$ find * -name '*Prompt*'
XMonadContrib/dist/build/XMonad/Prompt
XMonadContrib/XMonad/Prompt
~~~

Now you remember that the folder you are looking for is `XMonadContrib/XMonad/Prompt`.

Step 2
------

~~~
$ grep -R 'shell' XMonadContrib/XMonad/Prompt
XMonadContrib/XMonad/Prompt/Shell.hs:A shell prompt for XMonad
XMonadContrib/XMonad/Prompt/Shell.hs:    , shellPrompt
~~~

Now you've found the file! It's `XMonadContrib/XMonad/Prompt/Shell.hs`. 

Step 3
------

~~~
$ rm XMonadContrib/XMonad/Prompt/Shell.hs
~~~

It's not terribly complicated, but it's certainly nowhere near convenient.

That's where my **dynamic prompt widget** comes in. Basically each of these widgets defines a *keyword* and as long as one such keyword is detected on the prompt line, anything after that *keyword* is passed to the relevant widget, which will then display the appropriate autocompletions for the user to complete against. 

Currently there are 8 search widgets

* `f`: search for recent files using `fasd`
* `a`: search for recent files or directories using `fasd`
* `d`: search for recent directories using `fasd`
* `z`: search for recent directoties using `fasd`; on completion substitute the prompt line command such that it's suitable for changing directory from the prompt
* `l` (locate): search for files recursively in a given directory (or the current one, if not specified) using `find`
* `t` (tag): search for a directory in my tag database using `find`
* `g` (grep): list all files containing the given words
* `gp` (pdfgrep): list all pdf files containing the given words

So for the same problem we discussed before, it can be done with my dynamic prompt system in the following way:

Step 1 
-------

Invoke the prompt. Since you want to remove a file, just type in `rm`.

Step 2
------

Now you realize that you don't know the exact location of the file. You remember that the file contains the word `shell`. That means we should use `g` widget to grep for `shell`. So you type in another `g`. The prompt line is now `rm g`.

Step 3
------

You want to narrow down to a directory having `Prompt` in its name instead of greping blindly in the current directory which might contain 1000 files. That means we need to use `l` widget to locate that directory and pass it back to the `g` widget as its argument. So type in `l Prompt`. Now you'll get some screen like this:


->![](/images/xmonad/widget1.png)<-

Now press tab to autocomplete. The completion algorithm is smart enough to remove the preceding widget keyword - `l` in this case.

->![](/images/xmonad/widget2.png)<-

Step 4
------

Now you've got the directory to grep in. Finish typing by adding the word to grep against.

->![](/images/xmonad/widget3.png)<-

Like before, tab through to the right file to make the final prompt line look like this:

->![](/images/xmonad/widget4.png)<-

Step 5
------

Press *Return* to execute the command.

