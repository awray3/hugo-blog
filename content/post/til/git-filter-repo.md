---
title: "How to Alter Git Histories"
date: 2022-11-12T16:36:25-08:00
draft: false
tags:
    - git
    - TIL
description: Remove files from your git history with git-filter-repo
---

I recently cleaned a couple of git repos that had large data files committed early in their history, and in the process I learned about [git-filter-repo][filter_repo_github], a tool for cleanly altering git histories.

There are many reasons you might need to modify your git history. For example, consider a local repo you want to push to Github that at some point in time had a file larger than [their 100MB cap][github_filesize_caps] committed. 
In order to push to Github, you would need to not only remove the file from your repo with `git rm`, but also remove the file from _any commit_ it showed up in.
Another common scenario: you want to purge your git history of any accidentally tracked junk files, such as `__pycache__` folders or `.DS_Store` files.

In both scenarios, the goal becomes to completely rid a file (or directory) from the git history.

## The old way: `git filter-branch`

When you search around for ideas on how to rid files from histories you might find a lot of older stack-exchange posts and tutorial websites with solutions involving `git filter-branch`. However, according to the git filter-repo [readme][filter_repo_github_subsec], `filter-branch` has numerous problems: it is slow, potentially unsafe for your repository, and clunky to use.
For that reason I won't describe how to use it here.


## Enter `git filter-repo`

People have since built simpler, more effective tools for performing git history manipulations, and the best one I've found is [git-filter-repo][filter_repo_github]. I picked it after having been convinced by their [comparisons to other tools][filter_repo_github_subsec] in this area.
They cover many use cases in their [handbook][manpage], which is worth at least glancing over.

## Example: Removing files from the git history

In this post I'll focus on the example of removing a file from the git history. However, this works the same with directories and similarly with glob patterns or regex; see `--path-glob` and `--path-regex`. 

For illustration, I'll initialize an empty git repository and add two files, `file_1.txt` and `file_2.txt`, in a single commit.

```console
$ mkdir /tmp/new-repo && cd /tmp/new-repo
$ git init
$ touch file_1.txt file_2.txt
$ git add .
$ git commit -m "Initial commit"
```

```console
$ git log --name-status --onleline

8fe6ecb (HEAD -> main) Initial commit
A       file_1.txt
A       file_2.txt
```

For this example our plan will be to delete `file_2.txt` from the git history _without_ deleting `file_2.txt` itself.

### Step 1: Decaching

"Decaching" is a word I made up for this step of "remove the file from the current git commit but keep it around in the folder." You can do this with

```console
$ git rm --cached file_2.txt
```

Verify by checking that `ls` still shows both files, and that `git status` shows that `file_2.txt` is no longer tracked.

```console
$ ls

file_1.txt file_2.txt

$ git status

On branch main
Changes to be committed:
  (use "git restore --staged <file>..." to unstage)
        deleted:    file_2.txt

Untracked files:
  (use "git add <file>..." to include in what will be committed)
        file_2.txt
```

### Step 2: `filter-repo`

With `file_2.txt` unstaged, apply `filter-repo` like this to delete `file_2.txt` from the history:

```console
$ git filter-repo --path file_2.txt --invert-paths --force
```

The `--path` specifies the path you're trying to target for removal, and the `--invert-paths` is basically the logical negation of the filtering condition, so when it's applied it will _only delete_ `file_2.txt`. When you leave that flag off, you instead _delete everything except_ `file_2.txt`. You get only the file, or everything but the file.

The `--force` flag is needed because `filter-repo` expects us to follow best practices by only using it on a fresh clone. 
In practice[^fresh_clone], you would commit all your work, get a clean git state, and make a fresh clone of your repo to operate on with `filter-repo`. 

Now check the files in the git log and filesystem:

```console
$ ls

file_1.txt file_2.txt

$ git log --name-status --oneline                         
l82b222

A       file_1.txt
```

The single commit now does not have any information pertaining to `file_2.txt`, and `file_2.txt` is still around on the filesystem.
(If you just wanted to delete it completely, you could just skip the detaching step altogether.)

That's all there is to it!

[filter_repo_github]: https://github.com/newren/git-filter-repo
[filter_repo_github_subsec]: https://github.com/newren/git-filter-repo#why-filter-repo-instead-of-other-alternatives
[manpage]: https://htmlpreview.github.io/?https://github.com/newren/git-filter-repo/blob/docs/html/git-filter-repo.html

[^fresh_clone]: See [this part of the handbook](https://htmlpreview.github.io/?https://github.com/newren/git-filter-repo/blob/docs/html/git-filter-repo.html#FRESHCLONE)

[github_filesize_caps]: https://docs.github.com/en/repositories/working-with-files/managing-large-files/about-large-files-on-github