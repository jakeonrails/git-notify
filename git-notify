#!/bin/bash

set -eu

function log {
    if [ $verbose = true ]; then
        (>&2 echo "[$(date)]: $@")
    fi
}

function ps_jobs {
    local curr_comm=$(ps -o command -p "$$" | grep -v COMMAND)
    ps $2 | grep -v "$curr_comm" | grep "$1" | grep -v "grep $1" || true
}

function count {
    echo "$bg_jobs" | sed '/^\s*$/d' | wc -l | tr -d ' '
}

function git-curr-branch {
    git rev-parse --abbrev-ref --symbolic-full-name @{u} 2>/dev/null
}

# Command Line Options

branch=$(git-curr-branch || echo "origin/master")
async=false
refresh_delay=60
verbose=false

while getopts ":b:t:avlk" opt; do
    case "$opt" in
    b)  branch="$OPTARG"
        ;;
    t)  refresh_delay="$OPTARG"
        if ! [[ $refresh_delay =~ ^[0-9]+$ ]]; then
            echo "Refresh delay must be a number (of seconds)"
            exit 1
        fi
        ;;
    a)  async=true
        ;;
    v)  verbose=true
        ;;
    l)  bg_jobs=$(ps_jobs "$0" "-eaf")
        bg_count=$(count "$bg_jobs")
        echo "Running background jobs: $bg_count"
        if [ $bg_count -gt 0 ]; then
            ps -fp 0 | head -n 1 # print ps column names header
            echo "$bg_jobs"
        fi
        exit 0
        ;;
    k)  curr_comm=$(ps -o command -p "$$" | grep -v COMMAND)
        bg_jobs=$(ps_jobs "$0" "-eao pid,command" | cut -f 1 -d ' ')
        bg_count=$(count "$bg_jobs")
        if [ $bg_count -eq 0 ]; then
            echo "No jobs running!"
        else
            echo -n "Killing $bg_count jobs: "
            echo $bg_jobs
            kill $bg_jobs
            echo "Success!"
        fi
        exit 0
        ;;
    :)  echo "Required argument for option $OPTARG"
        exit 1
        ;;
    ?)  echo "Unrecognized option $OPTARG"
        exit 1
        ;;
    esac
done
shift $((OPTIND-1))

# Main watch function

function run {
# how we want to extract the variables from the commit message.
format_name="--format=%cn"
format_when="--format=%cr"
format_summary="--format=%s"
format_body="--format=%b"

latest_revision="none"
echo "Watching branch $branch"

# loop forever, need to kill the process.
while [ 1 ]; do

    # get the latest revision SHA.
    current_revision=`git rev-parse $repository`

    # if we haven't seen that one yet, then we know there's new stuff.
    if [ $latest_revision != $current_revision ]; then
        log "Changed! New revision: $current_revision"
        # mark the newest revision as seen.
        latest_revision=$current_revision

        # extract the details from the log.
        commit_name=`git log -1 $format_name $latest_revision`
        commit_when=`git log -1 $format_when $latest_revision`
        commit_summary=`git log -1 $format_summary $latest_revision`
        commit_body=`git log -1 $format_body $latest_revision`

        # notify the user of the commit.
        summary="$commit_name committed to $repository $commit_when!"
        body="$commit_summary\n\n$commit_body"
    	if [ "`uname`" == "Darwin" ]; then
    		command="osascript -e 'display notification \"$body\" with title \"$summary\"'"
    		eval $command
    	else
            `notify-send "$summary" "$body"`
    	fi
    fi
    sleep "$refresh_delay"
done
}

# Test git repository and start execution

if !(git rev-parse --git-dir > /dev/null 2>&1); then
    echo "Error: not a git repository"
    exit 1
else
    if [ $async = true ]; then
        run &
        echo "Started background job with PID $!"
    else
        (run)
    fi
fi
