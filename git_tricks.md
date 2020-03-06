## git tricks

### using `upstream` instead of `origin`

Origin is the base name that denotes the starting position of git. But I prefer to use `upstream` because its more descriptive.

Hence my alias:
```
[alias]
    upstream = remote rename origin upstream
```

Where running `git upstream` renames `origin` for me.

### pruning merged

I found this gem Googling, but is really useful for cleaning my local branches up. I dropped the following in my `~/.gitconfig`:
```
[alias]
    prune-merged = "!f() { git fetch -p; git branch --merged; git branch --merged | awk '/!(master|staging)/{system(\"git branch -D \"$2)}'; }; f;"
```

### recommiting

The standard method of updating a commit during PR reviews is to use `--amend`. But I like something more flexible:

```
[alias]
    recom = !sh -c 'git commit --date=\"$(date)\" --amend -a -m \"$(git log -1 --pretty=%B)\"'
    recommit = !sh -c 'git commit --date=\"$(date)\" --amend -a'
```

When I run `git recom` it uses the previous commit message, while `git recommit` allows me update the message. The best part is even though the `-a` (all) is there, using `git recom <file>` will only change the file in that commit.

And unlike `--amend`'s behavior using the previous date, these commands use the amended date stamp.

_Here be dragons_: The danger to these commands is that you can accidentally update an commit you are not intending to change.

### the long weekend

After a long weekend, time off, or night of indulgent potions, or even losing track of which branch, finding out what branch was last worked on can be a pain. This little spell I found thanks to divination on Google (my apologies for not including attribution):

```
[alias]
    branch-date = "!f() { git for-each-ref --sort=committerdate refs/heads/ --format='%(committerdate:short) %(refname:short)'; }; f;"
```

Running `git branch-date` gives me something like this:
```
2020-03-04 master
2020-02-28 pr/you_got_to_be_kidding
2020-02-01 feature/demogorgans
2020-01-29 feature/hawkins_il
```

### git-clean-master

Perhaps my favorite command is what I call `git-clean-master`. I fleshed this out because often times I'll hack on something and then want to revert to pristine. With few exceptions, this command works to reset to master, in a pristine state, without having to delete everything.

```

#!/bin/bash
git remote rename origin upstream 2> /dev/null
head_ref=$(git symbolic-ref refs/remotes/upstream/HEAD)
if [ "${head_ref}x" == "x" ]; then
    head_ref="refs/remote/upstream/master"
fi
ref_name=$(echo $head_ref | awk '-F/' '{$1=$2=$3=""; print$0}' | tr -d '[:space:]')
remote_name=$(echo $head_ref | awk '-F/' '{print$3}')

# clean up
git reset --hard
git clean -ff -d

# fetch newest
git fetch upstream

# detatch branch
git checkout --detach "${remote_name}/${ref_name}"

# delete the local copy
git branch -D "${ref_name}"  >> /dev/null

# checkout a local copy
git checkout -b "${ref_name}" "${remote_name}/${ref_name}"

# cleanup
git prune-merged

# submodules
git pull --recurse-submodules

# add some info
cat <<EOM
Submodules
===============================================
$(git submodule status)

Remotes:
===============================================
$(git remote -v)

Last Log:
===============================================
EOM
git log -1

echo ""
```
