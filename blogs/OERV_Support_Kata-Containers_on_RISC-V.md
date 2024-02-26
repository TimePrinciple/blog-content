---
# Must present
author: Ruoqing He
title: Support Kata-Containers on RISC-V
# Optional
date: 2024-02-26
summary: A work record (or report) for supporting kata-containers on RISC-V.
---

## Preparation

Navigate to [RISC-V Mainline](https://build.openeuler.openatom.cn/project/show/openEuler:Mainline:RISC-V) page, search keyword `kata`, two packages could be found there: [kata-containers](https://build.openeuler.openatom.cn/package/show/openEuler:Mainline:RISC-V/kata-containers) and [kata-integration](https://build.openeuler.openatom.cn/package/show/openEuler:Mainline:RISC-V/kata_integration).

Click into the *kata-containers* page, an error emerged immediately with message:

```bash
service tar_scm failed:
Detected cached repository...
Traceback (most recent call last):
File "/usr/lib/obs/service/tar_scm", line 30, in
main()
File "/usr/lib/obs/service/tar_scm", line 26, in main
TarSCM.run()
File "/usr/lib/obs/service/TarSCM/__init__.py", line 35, in run
task_list.process_list()
File "/usr/lib/obs/service/TarSCM/tasks.py", line 118, in process_list
self.process_single_task(task)
File "/usr/lib/obs/service/TarSCM/tasks.py", line 216, in process_single_task
args.outdir)
File "/usr/lib/obs/service/TarSCM/archive.py", line 45, in extract_from_archive
shutil.copy2(src, outdir)
File "/usr/lib64/python3.6/shutil.py", line 257, in copy2
copyfile(src, dst, follow_symlinks=follow_symlinks)
File "/usr/lib64/python3.6/shutil.py", line 120, in copyfile
with open(src, 'rb') as fsrc:
IsADirectoryError: [Errno 21] Is a directory: '/srv/obs/service/11294/out/kata_integration-1693203328.0e47ca9/patches'
```

Put it aside, check where the source of this package is pulled from. In `_service` file, the line `<param name="url">git@gitee.com:src-openeuler/kata_integration.git</param>` reveals its origin.

So far, the target to work with has been determined, since the changes are to be committed to this repo afterwards, `fork` this repo on Gitee before going.

Let's move to debugging phase.

## First Tryout (2024-02-26)

! From this point on, all commands are executed in *openEuler 23.09* on *RISC-V* architecture.

Clone the forked `kata-containers` project hosted on *src-openEuler* with command: `git clone https://gitee.com/<gitee_username>/kata-containers.git`.

My principle is to solve whatever errors emerged along building process, so let's build this package directly to see what happens.

Consult `kata-containers.spec` file, it suggests that `apply-patches` script is need to build the package (just search the pattern in `.spec` file). And in `apply-patches` file, `series.conf` is referenced. So basically, `kata-containers-3.2.0.tar.gz`, `apply-patches` and `series.conf` are to be placed under `~/rpmbuild/SOURCES` folder.

Run `rpmbuild -ba kata-containers.spec`, the command complains about missing build dependencies. Then install the packages required with `yum`.

Run build again, it returns with an error `error: bad date in %changelog:` which I've never met. Comparing the change logs, it seems to be a missing `day` field in the date section. Add the two missing `day`, the error disappears.

This time, `rpmbuild` returns with a new error `File /home/openeuler/rpmbuild/SOURCES/kata_integration-openeuler.tar.gz: No such file or directory`. But there is no such file in this repo. Eventually, I found this file in [_service](https://build.tarsier-infra.com/package/view_file/home:misaka00251:Fix2303/kata-containers/_service?expand=1). It seems that the missing `kata_integration-openeuler.tar.gz` file is the repository compressed.

After examine the previously distributed [repo](https://gitee.com/misaka00251/kata-containers), it turns out the `kata-containers` is using the Go version `runtime` instead of its Rust version counterpart `runtime-rs`. The reason the previous build requires `cargo`, `rust` and the like dependencies is that the `agent` is implemented by Rust. Although both implementation of `runtime` doesn't support *RISC-V* architecture.

I plan to directly support `kata-containers v3.2.0` while referencing the works done in `v2.5.1` next time.
