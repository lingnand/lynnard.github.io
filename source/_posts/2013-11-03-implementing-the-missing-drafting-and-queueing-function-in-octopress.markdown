---
layout: post
date: 2013-11-03 00:37
title: "Implementing the missing drafting and queueing function in Octopress"
#uncomment the next line to use the creation date
#date: 2013-09-21 10:40
comments: true
published: true
categories: [octopress, sh]
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

You can download the whole source file [here](downloads/code/octopress). Below shows a preview of the code:

```bash
case $1 in
    rake) 
        cd "$OCTOPRESS_HOME"
        rake "${@:2}"
        ;;
    update)
        # lock the instance
        [ "OCTOUPDATE$PPID" = "$LOCKRUNPID" ] || exec lockrun -qna OCTOUPDATE /tmp/lockrun.octoupdate "$0" "$@" || exit 1

        OCTOPRESS_STATUS_FILE="$HOME/.octopress_update"
        export LC_ALL=en_US.UTF-8
        export LANG=en_US.UTF-8

        # first move the files in the draft to queue if the draft property is set to true
        echo "######## Checking for finished drafts ########"
        # try to find all the markdown and md files
        find "$OCTOPRESS_DRAFTS_DIR" -type f \( -name '*.markdown' -o -name '*.md' \) | while read draft
        do
            # get the published property; if no published property is found then default to false
            if [ "`valueForProperty "$(YAMLFrontMatterOfPost "$draft" false)" 'published'`" = 'true' ]; then
                new_post_name="`fullNameOfPost "$draft"`"
                echo "$draft: Published is true, moving to queue folder (renaming to $new_post_name)..."
                mv -n "$draft" "$OCTOPRESS_QUEUE_DIR/$new_post_name"
            fi
        done
        # second move the files in the queue to the post directory if the date specified is in the past
        echo "######## Checking for publishable posts in the queue ########"
        find  "$OCTOPRESS_QUEUE_DIR" -type f \( -name '*.markdown' -o -name '*.md' \) | while read post
        do
            currDate="`date '+%s'`"

            # get the post name to ensure that date etc are properly initiated
            new_post_name="`fullNameOfPost "$post"`"
            dateString="`valueForProperty "$(YAMLFrontMatterOfPost "$post")" 'date'`"
            dateString="${dateString% *}"

            postDate="`date -d "$dateString" '+%s'`"

            if ((postDate < currDate)); then
                echo "$post: Post date is in the past, moving to post folder (renaming to $new_post_name)..."
                mv -n "$post" "$OCTOPRESS_POSTS_DIR/$new_post_name"
            fi
        done

        echo "######## Checking for changes in the source directory ########"
        cd "$OCTOPRESS_HOME"
        to_update=false
        # first check if there is any change in the source branch; if there's then commit and update
        # diff --quiet checks whether there's change in the tracked files; the second command checks whether there's untracked file
        # the draft and queue folders are UNTRACKED; so you can basically do anything you want for the drafts 
        if ! git --no-pager diff --exit-code || [ -n "$(git ls-files --others --exclude-standard)" ]; then
            echo "Changes to the octopress directory detected."
            to_update=true
        elif ! [ -e "$OCTOPRESS_STATUS_FILE" ]; then
            echo "No last update status found." 
            to_update=true
        elif [ "`cat "$OCTOPRESS_STATUS_FILE"`" = 1 ]; then
            echo "Last update failed."
            to_update=true
        fi

        echo "######## Beginning update ########"
        if $to_update; then
            echo "Commiting changes in the source branch..." 
            # the add commit phase don't really count towards update result as it can give back error code if there's nothing to commit
            git add -A 
            git commit -m "Source updated" 
            echo "Pushing changes to remote..." \
            && git push origin source \
            && echo "Generating and deploying the website..." \
            && rake gen_deploy \
            && echo 0 > "$OCTOPRESS_STATUS_FILE" \
            || echo 1 > "$OCTOPRESS_STATUS_FILE"
        else
            echo "Nothing to update."
        fi
        ;;
    *)
        usage
        ;;
esac
```
