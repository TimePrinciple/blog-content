---
# Must present
author: Ruoqing He
title: Import Xline to openEuler
# Optional
date: 2024-02-26
summary: A process record of importing new packages/softwares to openEuler.
---

To introduce a new software to openEuler, the process can be divided into the following major steps:

- Create the repository used for distribution in [src-openEuler](https://gitee.com/organizations/src-openeuler/projects).
- Prepare the source (source code, patches and etc.) needed for packaging.
- Write a `.spec` and test it with `rpmbuild` to make sure it works properly.
- OBS related (to be expanded later on).

## Create Repository

Repositories could not be cerated directly under *src-openEuler*. Most of the cases, it is done by rasing a PR which describes metadata of the repository and purpose of importing in [openEuler/Community](https://gitee.com/openeuler/community). The PR I raised to create `Xline` project can be found [here](https://gitee.com/openeuler/community/pulls/5400).

Simply put, you will need to figure out which *SIG* (Special Interest Group) the repository to be imported belongs to, and consult the members of that *SIG* whether they are willing to accept it.

After the *SIG* which is about to manage the repo has been determined, navigate to `sig/` folder to find out that *SIG*. In my case, that *SIG* is `sig/sig-Rust`. Therefore, the `sig-info.yaml` under that directory should be appended with `src-openeuler/<repo_name>`. And under `sig/sig-Rust/src-openeuler/`, create a folder with the **starting letter** lower cased of the repository to be imported (this serves as a manually created index, I guess). Again in my case, the project is `Xline` (repository name starts with an upper case is rare, but not forbidden), so the folder is `x/` not `X/`, then add a `<repo_name>.yaml` file there. The files and modifications I made can be referenced [here](https://gitee.com/openeuler/community/pulls/5400/files).

Cooperate with the maintainers/reviewers to get the PR merged, then the repository will be created under `src-openEuler` automatically. And that's where the actual work begins.

## Prepare Source

The term `source` here represents all files needed for the `.spec` and `rpmbuild` to work. To my understanding, the `.spec` documents the **steps taken to build the package**. Therefore I navigated to the `README.md` (or anywhere build related docs available) of `Xline`. If you find that document, the `supported architectures` (and the like) requires attention. If the architecture you working on is listed, there would be a detailed description (most likely) of building the software from source. For me, I plan to support `Xline` on *openEuler-x86_64* first, and advance to *riscv64* after it is completed.

Generally, the source code could be found in *Release* page, and shares a `<repo_name>.tar.gz` pattern on Github. But be advised, repositories using *submodules* is a little different. The source tarball does not contain the *submodules*, if you were to package these repositories, additional steps are needed.

placeholder: git-archive-all related content here.
