---
date: 2024-02-05
title: Git Internals
author: Ruoqing He
summary: Inner logic of Git server-side, noted for `mega`'s `fuse` submodule.
---

# Git Internals

A content-addressable filesystem. Which implies that the core of Git is a simple key-value data store. Which enables you to insert any kind of content into a Git repository, for which Git will hand you back a unique key can be used later to retrieve that content.

After `git init` executed, Git creates the `.git` directory, which is where almost everything that Git stores and manipulates is located. A newly-initialized `.git` directory typically looks like:

```sh
$ ls -F1
config
HEAD
hooks/
info/
objects/
refs/
# index
```

- `config` file contains the project-specific configuration options
- `info` directory keeps a global exclude file for ignored patterns you don't want to track in a `.gitignore`
- `hooks` directory contains your client- or server- side hook scripts
- `objects` directory stores all the content for your database, the *object database*
- `refs` directory stores points into commit objects in that data (branches, tags, remotes and more)
- `HEAD` file points to the branch you currently have checked out
- `index` file stores your staging area information

## Storage Strategy

All the content is stored as `tree` and `blob` objects, with `tree`s corresponding to UNIX directory entries and `blob`s corresponding more or less to `inode`s or file contents. Git normally creates a tree by taking the state of the `staging area` or `index` and writing a series of tree objects from it.

Common modes for `blob`s:
- `100644`: A normal file
- `100755`: An executable file
- `120000`: A symbolic link

