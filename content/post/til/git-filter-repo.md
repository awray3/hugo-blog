---
title: "TIL: Cleaning git histories with git filter-repo"
date: 2022-11-12T16:36:25-08:00
draft: true
tags:
    - git
---

I recently had to clean some git repos that had large data files committed early in their history.
(You shouldn't commit data files to git for several reasons, mainly speed of indexing and Github filesize limits.
This is better handled by a tool like DVC or git large-file storage.)

So let's say in a haste one day, you committed some `.csv` files to your git history to track the state of the data along with the code. After a few months of doing more and more analysis, you might have lots of such data files and other stuff in your git history (I'm looking at you, `__pycache__` and `.DS_Store`). So what can you do to remove these meaningless files from your git history?

## The old way: `git filter-branch`

When you search this question on google, you'll find a lot of older stack-exchange posts and tutorial websites with solutions involving `git filter-branch`. However, according to the git filter-repo [readme](https://github.com/newren/git-filter-repo#why-filter-repo-instead-of-other-alternatives), `filter-branch` has numerous problems: it is slow, potentially unsafe for your repository, and clunky to use.


## Enter `git filter-repo`

People have since built other tools for performing git history manipulations, but the best one I've found is [`git filter-repo`](https://github.com/newren/git-filter-repo). Why this one? See the comparisons [here](https://github.com/newren/git-filter-repo#why-filter-repo-instead-of-other-alternatives); they convinced me.
They also cover many use cases in their [handbook](https://htmlpreview.github.io/?https://github.com/newren/git-filter-repo/blob/docs/html/git-filter-repo.html), but I'll just cover the two that I learned about.


## Migrating or removing files from the git history

For illustration, I'll initialize an empty git repository and add two files, `file_1.txt` and `file_2.txt`, in a single commit.

```bash
$ mkdir /tmp/new-repo && cd /tmp/new-repo
$ git init
$ touch file_1.txt file_2.txt
$ git add .
$ git commit -m "Initial commit"
```

```bash
$ git log --name-status --onleline

8fe6ecb (HEAD -> main) Initial commit
A       file_1.txt
A       file_2.txt
```

For this example our plan will be to delete `file_2.txt` from the git history _without_ deleting `file_2.txt` itself.

### Step 1: Decaching

"Decaching" is a word I made up for this step of "remove the file from the current git commit but keep it around in the folder." You can do this with

```bash
$ git rm --cached file_2.txt
```

Verify by checking that `ls` still shows both files, and that `git status` shows that `file_2.txt` is no longer tracked.

### Step 2: `filter-repo`

With `file_2.txt` unstaged, apply `filter-repo` like this to delete `file_2.txt` from the history:

```bash
$ git filter-repo --path file_2.txt --invert-paths --force
```

The `--path` specifies the path you're trying to target for removal, and the `--invert-paths` is basically the logical negation of the filtering condition, so when it's applied it will _only delete_ `file_2.txt`. When you leave that flag off, you instead _delete everything except_`file_2.txt`. You get only the file, or everything but the file.

The `--force` flag is needed because `filter-repo` expects us to follow best practices by only using it on a fresh clone. 
In practice, you would commit all your work, get a clean git state, and make a fresh clone of your repo to operate on.
<!-- todo: find the link I'm thinking of -->

Now check the files in the git log again:

```bash
$ git log --name-status --oneline                         
382b222 (HEAD -> main) Initial commit
A       file_1.txt
```

That's it! You might get an error about the repo not being a fresh clone in these examples. If that happens, you can always add `--force` to the commands to ignore that warning (which should be fine for this example,  but in general it's recommended to only `filter-repo` on a fresh clone of a repo; see link )
<!-- todo: get link -->

## Bonus: Renaming a directory

I also learned from their manual that `filter-repo` can rename directories in the git history. To rename a folder `foo/` in your repo to `bar/` throughout the whole git commit history, you would use `git filter-repo --path-rename foo:bar`.

```bash
# here's what my commit history looks like:
$ git log --name-status --oneline
7c0f12d (HEAD -> main) Initial commit
A       file_1.txt
A       foo/baz_1.txt
A       foo/baz_2.txt

# Then run the filter-repo path rename
$ git filter-repo --path-rename foo:bar --force

$ git log --name-status --oneline
c4770f2 (HEAD -> main) Initial commit
A       bar/baz_1.txt
A       bar/baz_2.txt
A       file_1.txt
```

