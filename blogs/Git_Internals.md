---
date: 2024-02-05
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
