---
layout: "post"
title: "git submodule tips"
categories: "blog"
tags: ['git-submodule']
published: True
comments: true
script: [post.js]
excerpted: |
    A short description...
---

* TOC
{:toc}

# Add submodule to project

```shell
# git submodule add <submodule project URL>
```

# Clone project with submodules

## Method 1: Step by step

```shell
# git clone <project URL> 
# git submodule init    # add submodule record to .git/config (which is recorded in .gitmodule)
# git submodule update  # based on submodule record in .git/config, clone the submodule project and checkout to specific commit (the commit is maintained internally in git)
```

## Method 2: Shorter

```shell
# git clone <project URL> 
# git submodule update --init --recursive
```

## Method 3: One step

```shell
# git clone --recurse-submodules <project URL>
```

# Pulling in submodule upstream changes

## Method 1: manually pull, one submodule each time

```shell
# cd <submodule dir>
# git pull origin <branch>
```

## Method 2: pull all submodules

```shell
# git submodule update --remote [submodule]
```

This command will by default update to latest **master** branch for each/specified submodule(s). If you want to change the branch, run:

```shell
# git config -f .gitmodules submodule.<submodule project name>.branch <branch name>
```

(NOTE: If you leave off the -f .gitmodules it will only make the change for you, but it probably makes more sense to track that information with the repository so everyone else does as well)

Tips: run `git submodule update` to revert.
