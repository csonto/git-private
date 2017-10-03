# git-private-push

Create a hidden branch in remote repositories.

Use `git private-push` to backup your work-in-progress branches, to keep
history when frequently rebasing a branch without cluttering everyone's view,
or to let others know the branch may get rebased at any time.

`gitk` will display these refs in a nice gray color.

**WARNING:** Hidden branches offer absolutely no security or privacy.
They are just neither displayed nor fetched by default.

**Features**

* multiple private spaces
* pushing multiple configured refs by single command (`private.*.push`)
* automatic timestamping (`private.*.tsfmt`)
* automatic force push (`private.*.force`)
* customizable remote refs (`$USER`, `$EMAIL`, `$TS` and `$DST` are recognized)

**Missing features**

* man page!
  Sorry. `git private-push --help` will not work until a man page is added.
  Use `git-private-push --help` instead.

**Internals**

Git by default fetches from `refs/heads/`, `refs/tags/`. Pushing to a
non-default subdirectory in `refs` will create a "hidden" branch which is not
fetched unless asked for.

## Installation

### Requirements

* git
* bash

### How to install

1. Get the the `git-private-push` and `git-private-add-fetch` scripts, by either:
    * Cloning this repository:  
      `git clone https://github.com/csonto/git-private.git`
    * Downloading [the zip archive](https://github.com/csonto/git-private/archive/master.zip)
      and extracting it locally:  
      `wget https://github.com/csonto/git-private/archive/master.zip && unzip master.zip`
2. Make the script callable form anywhere on your system, by either:
    * Adding the script files to a directory in your `$PATH`, e.g.:  
      `cp <path/to/git-private>/git-private-* /usr/local/bin/`
    * Creating a link to the script files from such a directory, e.g.:  
      `ln -s <path/to/git-private>/git-private-push /usr/local/bin/git-private-push`
    * Adding the directory where you downloaded the files to your `$PATH`, e.g.:  
      `echo 'export PATH="<path/to/git-private>:$PATH"' >> ~/.bashrc && source ~/.bashrc`

## Usage

### Pushing

The `git private-push` is used as follows:

    git private-push [--id ID] [--force|--no-force] [BRANCH [DESTINATION]]

Push to the default private space, named "junk":

    git private-push master

Push a ref from remote to a private space named "wip", with custom destination:

    git private-push --id wip remotes/local/master local

### Fetching

Add fetch config options to private "remote":

    git config --add remote.$remote.fetch "+refs/$path/*:refs/$path/$remote/*"

e.g. for the private space named `wip` at the `origin` remote:

    git config --add remote.origin.fetch "+refs/wip/*:refs/wip/origin/*"

Or use `git-private-add-fetch [ID]`.

### Concrete examples

By default `.git/config` contains something like this:

    [remote "origin"]
        url = git@github.com:csonto/git-private.git
        # This is the default:
        fetch = +refs/heads/*:refs/remotes/origin/*

The directory structure is up to you. In general you will likely want to follow
up git's `refs/$PATH/$REMOTE` pattern. To fetch all refs from the remote add a
line like this:

        fetch = +refs/*:refs/all/origin/*

To fetch a subdirectory from the remote, add a line like this:

        fetch = +refs/wip/csonto/*:refs/wip/csonto/*

Common options should go to `~/.gitconfig`, and project-specific options to `$PROJECT/.git/config`.

Let's see something more useful...

#### Backup

I want a copy of my work stored on the server. I do not require history, single
copy is good enough for me.

    git wip
    git wip BRANCH

Eventually I might want to push uncommited changes:

    git stash
    git wip "stash@{0}" stashed
    git stash pop

If you want this to work for all your git repos, add a common configuration to
`~/.gitconfig`:

    [private "wip"]
        # anything will do, timestamps are not used:
        tsfmt = "-"
        # ignoring TS, wip is not tracking history, only latest change is pushed:
        fmt = "$EMAIL/$DST"
        # the wip refs will be rewritten so push has to be forced:
        force = 1

    [alias]
        wip = private-push --id wip

Add a project specific part - remote, list of branches - to project's `.git/config`.

For each branch to store run:

    git config --add private.wip.push BRANCH

To use different remote for this project add:

    git config private.wip.remote REMOTE

The configuration will look like this (without comments):

    [private "wip"]
        # you can use your own git server for backups:
        #remote = private

        # refs to backup with git wip:
        push = master
        push = gh-pages
        # add more here...

Now you could do `git wip` and it will push the listed refs (`master` and
`gh-pages`) to `refs/wip/user@example.com/BRANCH`
(`refs/wip/user@example.com/master` and `refs/wip/user@example.com/gh-pages`.)

Anything what is in your `refs` can be pushed, if you have another remote
(`local`) set up, remote heads can be pushed like this:

    git config --add private.wip.push remotes/local/master

If you want to keep the refs short use different name:

    git config --add private.wip.push "remotes/local/master local"

To automatically push the most recent stashed changes run:

    git config --add private.wip.push "stash@{0} stashed"

To fetch them all:

    git config --add remote.origin.fetch "+refs/wip/*:refs/wip/*"

To fetch only yours:

    git config --add remote.origin.fetch "fetch = +refs/wip/user@example.com/*:refs/wip/*"

Well, that's it. Except, backup without history, is not a real backup. Let's add history...

#### Backup with history

I am working on a feature branch and I need to rebase frequently.

I want a copy of my work stored on the server and I want to have full history of
changes if I need to go back after a bad rebase.

    git backup BRANCH
    git backup

Let's add the configuration:

    [private "backup"]
        tsfmt = "%Y%m%d-%H%M"
        fmt = "$EMAIL/$DST/$TS"

    [alias]
        backup = private-push --id backup

Add refs to backup by default:

    git config --add private.wip.push wip

Now running `git backup` will push `wip` branch as
`refs/backup/user@example.com/wip/20160310-1135`.

Hooray!

## Configuration

- `private.$ID.*` - configuration for particular private space
- `private.*` - default options

It may be useful to define global defaults:

    git config --global private.tsfmt "%Y/%m/%d/%H%M/"

The configuration options are:

- `private.id` - default ID.
- `*.fmt`    - path format. Defaults to `$TS$DST`.
  Strings `$TS`, `$USER`, `$EMAIL` and `SDST` are replaced with corresponding
  values. These strings are replaced in `$TS` but not in any other of the
  values.
- `*.tsfmt`  - timestamp format. This defaults to `%Y/%m/%d/%H/`.
- `*.path`   - path within remote. This defaults to `$ID/`.
- `*.remote` - remote where private refs are kept. Defaults to `$ID`.
- `*.push`   - `SRC [DST]` couple to use when not given on command line. There
  may be multiple of them.
- `*.force`  - if not empty force push if remote ref already exists. This may
  be overridden on command line by `--no-force` option.

For example to define origin for "WIP" space one can use:

    git config private.wip.remote origin

Add a default push "pair" (using default destination):

    git config --add private.wip.push master

Add a default push "pair" (using different destination):

    git config --add private.wip.push "stash@{0} stashed"

In the .git/config it looks like:

    [private "wip"]
    remote = origin
    path = wip/nb1/
    tsfmt = 0/ # Use constant. Default would be used if empty.
    push = master
    push = "stash@{0} stashed"

    [remote "origin"]
    fetch = +refs/heads/*:refs/remotes/origin/*
    fetch = +refs/wip/*:refs/wip/origin/*
