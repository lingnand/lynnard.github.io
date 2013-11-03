---
layout: post
date: 2013-11-03 00:37
title: "Implementing the missing drafting and queueing function in Octopress"
#uncomment the next line to use the creation date
#date: 2013-09-21 10:40
comments: true
published: true
categories: 
---

It's been a while since I first started using Octopress. Its simple and elegant way of writing has now become my *de facto* standard of writing, but there are still things to be desired. For one thing, the workflow consisting of *writing, commiting, compiling and deploying* is still not *lazy* enough; ideally the only thing that should be required of the writer is *to write*. Inspired by Dennis Wegner in his [Synced and scheduled blogging with Octopress](http://instant-thinking.de/2012/08/03/synced-and-scheduled-blogging-with-octopress/), I worked on a bash script to allow for friction-free, painless deployment process that goes totally automatic underhood.

### A short summary of the tinkering

1. Like in Dennis'es original post, two new folders called `_drafts` and `_queue` are added to the `source` directory to hold drafts and queued posts respectively.
2. Again, just like in the original post, you can put any markdown files within the `_drafts` folder; as soon as there's the `published: true` property within the YAML front matters like this

        ---
        published: true
        ---

    Then the draft will be moved to the `_queue` folder
3. All the files within the `_queue` folder will be checked for its `date` property in the YAML front matters; if it's before the current time, then the file will be moved to the `_posts` folder for publishing

### So what's added / improved?

1. The script automatically checks for the validity of the YAML front matters during each move of any post; among other things, it will add the following properties if absent:
    - date: the last modified date of the file is used; this is useful if you don't want to take the creation date for the draft as the date published on your website (which is the weird default behavior of Octopress) -- in that case just leave this field blank and it will be added automatically when the draft gets moved to the queue (or when you put a file yourself into the queue)
    - layout: `post` is assumed as the default layout, which should be sensible enough
    - title: the file name of the post (subtracting the date prefix, if any) is used

    What all these mean is that the minimal setup you'd need to have for a post is

    **Either**

    a. touch a new file in `_drafts` with its file name as the title
    b. when finished, add `published: true` to the top (don't forget the triple dashes)

    **OR**

    a. touch a new file anywhere with its file name as the title
    b. when finished, move it to `_queue`; the file will then be published during the next update

    *The second approach eliminates the need to have the YAML front matters entirely*
2. The script checks for **all changes** within the `octopress` directory using `git diff` and `git ls-files`. This means compared to the approach in the [original post](http://instant-thinking.de/2012/08/03/synced-and-scheduled-blogging-with-octopress/), now the website will be rebuilt even when files like `_config.yaml` is changed

    Note that this might not be optimal if you've had `_queue` and `_drafts` tracked by git. In that case whenever a draft is changed or something gets moved into the queue then the source will be commited, changes pushed to the remote, website rebuilt and all. I myself have excluded these two directories from git because I don't like the idea of having my drafts posting up to the cloud (I'm using GitHub as my host and I'm *a little* against having my unpolished thoughts shown in public) and I've had Dropbox versioning control these posts in the first place.

    However, if you're pointing your source to a private remote, then it might be a good idea for versioning control your drafts. In that case I'd recommend you change the script to auto-commit the changes in `_drafts` and `_queue` but exclude website re-compiling in such cases by greping the output of `git diff` and `git ls-files` to see if all changes happen in these directories.

### Where to download?

Since it's just a small file, I'm too lazy to put it up to any git repo. Instead I'm just going to show the whole source file here:

{% include_code lang:bash octopress %}
