# Content of Ruoqing He's Blog

> Noted in `markdown`, for Blog page generation.

My blog is designed to be published on `github.io`, but striped with dependencies of `npm` and its packages, built purely in Rust. I hate the felling that when dependencies becomes huge, which leads to unreliable build might fail some other day, especially when the related fields I'm not familiar with, in this case `js` and `np*`.

## Overall Design

The whole system is consisted of two parts: the `blog-content` (public repo) and `blog-generator` (private repo):
- `blog-generator` is literally a renderer, which renders the contents publish in `blog-content` and generates pages to be deployed according to templates.
- `blog-content` holds all the notes in `markdown` format, and provides content to `blog-generator`.

### Blogs

Blogs must be written with metadata specified, or they would not be recognized by the generator. Here is a template:

```yaml
---
# Must present
author:
title:
# Optional
cover:
date:
summary:
---
```


### Posts

### Records

### Thoughts

A directory to be ignored by `blog-generator` other than this repository. This directory is meant to record the above types of contents which is still at primitive stage, and may or may not be completed then migrated to published folders.
