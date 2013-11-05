---
layout: post
date: 2013-11-03 02:25
title: "Pinboard synchronization with Vimb/Vimprobable bookmarks"
#uncomment the next line to use the creation date
#date: 2013-09-21 10:39
comments: true
published: true
categories: [pinboard, vimb, sh, tech, english]
---

When it comes to workflow, I'm a screen-estate *fascist* who disapproves of any 'extra' GUI elements like buttons and menu-bars. That's why I switched to Vimperator from conventional idioms like Safari and Firefox a long time ago; and that's also the reason that I switched to [Vimb](https://github.com/fanglingsu/vimb)/[Vimprobable](http://www.vimprobable.org) from Vimperator in the end - if all I need is Vim-like features from Vimperator, why bother having Firefox in the first place?

All is well except a few things. One of them is what I'm going to talk about today, bookmark synchronization with popular online services. With light-weight browsers such as Vimb, you can't really expect the developer to write extensions for such things *(well, how many developers does Vimb have? Mhh, just one)* However, this simplicity is also the beauty of such light-weight programs - usually you can just achieve what you want by a little bit of tinkering.

To start with, the social bookmark service I'm using is [Pinboard](http://pinboard.in) and they offer an easy-to-understand, comprehensive api [here](http://pinboard.in/api). On the other hand, Vimb is even more straightforward on the treatment of bookmarks - it simply stores them in plain-text under its config directory.

With knowledge above we can easily build a synchronizer from scratch. The problem of synchronization is really about two smaller sub-problems:

1. When the local storage (Vimb bookmark file) changes, reflect the change on the cloud
2. When the cloud storage (Pinboard) changes, translate the change to local

For point 1 we can keep a backup of the bookmark file and use `diff` on it with the current file each time before the update. This gives us the changes, if any, done to the bookmark file since the last update and all remains is just doing the same thing on the cloud.

For point 2 Pinboard has offered a convenient query method `/posts/update`, which essentially returns the timestamp of the latest change. By retrieving this timestamp before each update cycle we can be sure that we won't miss any updates on the cloud.

Of course this leaves out details such as which should be performed before which, how we should avoid unnecessary pulling from the cloud given that some of the changes on the cloud are done by the script itself; but the main idea is still the same.

For completeness the whole script is [here](downloads/code/pinboard) for download. The main abstract of the code is shown below

```sh
# $1 is the method to query upon, $2 is the file to direct the output, while all the subsequent arguments
# in the format 'name=value' are appended as data arguments to
# the query
#---------------------------------------------------------------
function pinboard_api() {
    local datastring= data=
    for data in "${@:3}"
    do
        # we need to escape the single quote..merr...
        datastring="$datastring --data-urlencode \$'${data//\'/\'}'"
    done
    eval $CURL_CMD$datastring $HTTPS_PREFIX$1$AUTH_TOKEN > $2
	local CURL_EXIT_CODE="$?"
	if [[ $CURL_EXIT_CODE != "0" ]] ; then
		error "Curl failed with exit code $CURL_EXIT_CODE"
	fi
}

#---------------------------------------------------------------
function verify_config() {
	if [[ ! -e "$PINBOARD_CFG" ]]; then
		error "Could not find $PINBOARD_CFG file! \nIt should contain two lines. Line 1 the username. Line 2 the API token. \nYou can create it manually, or use the \"login\" command."
	fi

	USER_NAME=`head -n1 $PINBOARD_CFG`
	API_TOKEN=`tail -n1 $PINBOARD_CFG`
	AUTH_TOKEN="?auth_token=$USER_NAME:$API_TOKEN"
}

# This will compare the current update with the last update and
# return true if there's new update
#---------------------------------------------------------------
function pinboard_has_update() {
	# See if we need to do an update at all
	echo "Retrieving current pinboard update time..."
	pinboard_api "/posts/update" "$CUR_UPDATE_FILE"
	if [[ ! -e "$LAST_UPDATE_FILE" ]]; then
		echo "$LAST_UPDATE_FILE does not exist..."
		return 0
	else
		# Compare current update with last update
		echo "Comparing last update with current update..."
		diff "$CUR_UPDATE_FILE" "$LAST_UPDATE_FILE"
		if [[ ! "$?" == "0" ]]; then 
			return 0
		fi ;
	fi
    return 1
}

#---------------------------------------------------------------
function synchronize_cmd() {
    local to_update_pinboard=false to_update_local=false
    echo "######## Checking changes on the server ########"
    verify_config
    pinboard_has_update && to_update_pinboard=true
    echo "######## Starting local-to-cloud synchronization ########"
    # first compare the two bookmarks file to see if there are any changes
    if [[ -e "$BOOKMARK_BACKUP_FILE" ]]; then
        echo "Detecting changes to the bookmark file..."
        local changes="`diff "$BOOKMARK_BACKUP_FILE" "$BOOKMARK_FILE"`" 
        if [ -n "$changes" ]; then
            to_update_local=true
            # we need to grep for all deletions 
            echo "$changes" | grep '^<' | while read line; do
                delete_pinboard_bookmark_from_vimb_format "${line#< }" 
            done
            # have to exit the script if error occurred
            (( $? != 0 )) && exit "$?"
            echo "$changes" | grep '^>' | while read line; do
                insert_pinboard_bookmark_from_vimb_format "${line#> }" 
            done
            (( $? != 0 )) && exit "$?"
        else
            echo "No local-to-cloud synchronization required. Local bookmark storage is in sync with pinboard account."
        fi
    else
        to_update_local=true
        echo "Bookmark file has not been backed up before. Trying to insert all entries..."
        while read line; do
            insert_pinboard_bookmark_from_vimb_format "$line"
        done < "$BOOKMARK_FILE"
    fi

    if $to_update_local; then
        # we have updated the server so we should refresh the update time
        echo "Refreshing the last update time..."
        pinboard_api "/posts/update" "$CUR_UPDATE_FILE"
    fi

    echo "######## Starting cloud-to-local synchronization ########"
    if $to_update_pinboard; then
        echo "Retrieving bookmarks..."
        pinboard_api "/posts/all" "$CURL_TMP"
        # we need to convert the format into vimb's format
        xml sel -t -m '/posts/post' -c '.' -n "$CURL_TMP" | while read line; do
            # retrieve the attributes
            local href="`echo "$line" | xml sel -t -v '/post/@href'`"
            local desc="`echo "$line" | xml sel -t -v '/post/@description'`"
            local tag="`echo "$line" | xml sel -t -v '/post/@tag'`"
            # just adding the bookmark to tmp file
            insert_vimb_bookmark "$href" "$desc" "$tag"
        done
        # move the tmp file to overwrite the old bookmark file
        # this might be dangerous if there're unsynced changes in the bookmark file 
        # but for the time being we'd assume that the bookmark file is already in sync with
        # the cloud
        echo "Overwriting the old bookmark file..."
        mv -f "$BOOKMARK_TMP" "$BOOKMARK_FILE"
    else
        echo "No cloud-to-local synchronization required. $BOOKMARK_FILE is current."
    fi

    # back up the bookmark file
    if $to_update_local || $to_update_pinboard; then
        echo "Backing up the current bookmark file..."
        cp -f "$BOOKMARK_FILE" "$BOOKMARK_BACKUP_FILE"
    fi

    echo "Moving current update time to last update time..."
    mv -f $CUR_UPDATE_FILE $LAST_UPDATE_FILE
    echo "Done."  
}

#---------------------------------------------------------------
function login_cmd() {
	if [[ ! -d "$PINBOARD_DIR" ]]; then
		mkdir "$PINBOARD_DIR"
	fi

	if [[ ! -d "$PINBOARD_DIR" ]]; then
		echo "$PINBOARD_DIR could not be made."
		exit 1
	fi

	echo "We will attempt to log into pinboard now."
	`which echo` -n "Please enter pinboard username: "
	local USER_NAME=`head -n 1`
	echo $USER_NAME > "$PINBOARD_CFG"

	$CURL_CMD -u $USER_NAME "$HTTPS_PREFIX/user/api_token" | tail -n 1 | sed 's/<[^>]*>//g' >> $PINBOARD_CFG
	local CURL_EXIT_CODE="$?"
	if [[ $CURL_EXIT_CODE != "0" ]]; then
		echo "Curl failed with exit code $CURL_EXIT_CODE"
		exit 1
	fi

	if [[ ! -e "$PINBOARD_CFG" ]]; then
		echo "$PINBOARD_CFG could not be made."
		exit 1
	else 
		chmod og-rw "$PINBOARD_CFG"
	fi

	echo "Created $PINBOARD_CFG."
	echo "Logged in!"
}

#***
# "Main"
#***

if [[ $# -lt 1 ]]; then
    error "Expected a command for the second argument. \nTry \"help\"."
fi

#***
# Process the second commands
#***
case "$1" in
    synchronize) 
        [ "PINBOARD$PPID" = "$LOCKRUNPID" ] || exec lockrun -qna PINBOARD /tmp/lockrun.pinboard "$0" "$@" || exit 1
        synchronize_cmd
        ;;
    login) 
        login_cmd
        ;;
    help) 
        help_cmd 
        ;;
    *)
        error "Unknown command $1. Try \"help\"." 2
esac

# exiting without any errors
exit 0
```
