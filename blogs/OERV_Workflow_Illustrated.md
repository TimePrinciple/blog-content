---
date: 2024-02-08
---


# OERV Workflow Illustrated

OERV team is currently working on improving the ecosystem of *openEuler* on *RISC V* architecture as the time of writing. But be advised, the primary objective is to improve the ecosystem and impact of *RISC V*, *openEuler* here serves as a breeding ground and should not be a constraint.

## Things Should Know About

keywords: [rpm, SPEC, pkg, PKGBUILD, OBS]

Basic concepts:

- Packaging
- RPM
- SPEC

By importing and packaging softwares on other architectures (e.g., x64, arm), *openEuler* end users on *RISC V* are provided with more options while looking for softwares to satisfy their needs.

### Packaging

*Software packaging* is the process of building an "installer" that includes all the applications' resources and can be used to distribute them on to end users.

Easy works that can be found in OERV team commonly demands an upgrade or fix on imported packages. Ensuring packages works properly is an essential task for better ecosystem.

### RPM

*RPM* here refers to the `.rpm` file format. *RPM* packages could be chiefly categorized into two classes:

- *BRPM*: Binary RPM, containing the compiled version fo some software. Generally follows a naming convention `<name>-<version>-<release>.<architecture>.rpm`.
- *SRPM*: Source RPM, containing the "source code" used to build a binary package. Generally named `<name>-<version>-<release>.src.rpm`.

Simply put, the daily work is to verify that `.rpm`s of this particular architecture (*RISC V*) builds, installs, and behaves as expected.

### SPEC

A "script" describing by how the RPM package is created and used by `rpmbuild` (or something else) to build to actual package.

In another word, it's like the documentations could be found for softwares, which notes the steps required to prepare the dependencies, configurations, and build, install, post install, all the steps would be performed by users whom trying to install the software from source code, but in an automated "script".

Recommended reading: [SPEC detailed illustration](https://rpm-packaging-guide.github.io/#what-is-a-spec-file)

## Sites Should Know About

### [openEuler](https://gitee.com/organizations/openeuler/projects)

Projects raised by openEuler community, most of them have a corresponding repository in *src-openEuler* for distribution of these softwares.

### [src-openEuler](https://gitee.com/organizations/src-openeuler/projects)

Softwares have a supported `.rpm` package would be listed here. Generally, these repositories stores the files needed to build `.rpm` package (software source code, SPEC, patches and etc.). If the assigned task dictates a fix on some package, the works should be done are most likely in the repository which builds that package (they share the same name in most cases).

### [OBS-openEuler](https://build.openeuler.openatom.cn/)

Open Build Service of openEuler, it can be used to manage package distribution development process. It is also the recommended tool to finish the tasks since it has been repeatedly mentioned in pretask.

Recommended reading: [OSC-Documentation](https://en.opensuse.org/openSUSE:OSC)

### [openEuler/Community](https://gitee.com/openeuler/community)

Repositories in `src-openEuler` and `openEuler` are managed here. These repos are not created manually, but a corresponding update of the `.yaml`s of the right *SIG*. It's recommended to reference the merged PRs to get insights on how to write a proper PR for needs. It would be needed and a prerequisite if trying to import new softwares/packages.

## Workflow

Since the changes to complete a task eventually go to the repository under `src-openEuler`, the adopted approach is to directly clone the corresponding repo on target architecture and work on it. The following steps may provide insights on what's going wrong:

#### Building phase:

1. Clone the packaging repository of software which is reported to have errors.
2. Extract the actual source code used to build the software, usually post-fixed by `.tar.gz`.
3. Make sure packages listed under `BuildRequires` section are properly installed.
4. Check `%prep` section to determine which patches are applied, then apply them manually with `git apply <patch>` in source code directory. If the patches declared are not listed, consult [RPM-autopatch](https://rpm-software-management.github.io/rpm/manual/autosetup.html).
5. Follow the steps in `%build` section to inspect what goes wrong while building and make changes to fix them (mostly, here is the phase to address packaging errors, but be advised, chances are that errors might emerge in other phases).
6. Generate a patch with `git diff`, then modify `.spec` accordingly.
7. Verify the packaging repository (which contains `.spec` and `.tar.gz`, not source repository) works with `rpmbuild`.

#### Verifying phase:

Verification of a certain software does not have a relatively concrete steps, but there indeed are some places which provides hints:

1. Check `test/` directory for unit tests and scenario tests.
2. Read `README.md` and docs of usage and use cases in `docs/`.

#### Submitting phase:

When changes are ready for reviewing:

1. Make sure the changes happens in the right branch which describes the works of the pull request by executing: `git branch`.
2. Double check the rules and formats before raising a PR, i.e. some projects/communities might need to raise a corresponding issue which mentions the PR to be pushed, and link the issue raised in the commit message and PR description (optional).
3. PRs are encouraged to be signed at least with name and email by: `git commit -s`, while some projects might need a *GPG* sign with additional `-S` flag.
4. Push changes to the branch of the forked repository, then create a pull request there.
5. If there are modifications to be made, `git commit --amend` would allow to modify the latest commit, and force push by `git push -f`.
