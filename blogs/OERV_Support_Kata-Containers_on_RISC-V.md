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

## Second Tryout

Currently, the RISC-V sig is adapting the Go version `runtime` and Rust version `agent`. So the supporting job is around them.

### `runtime`

Kata-Containers originally is implemented in Golang, even if the emerge of Rust made them start to migrate this entire project to Rust (`runtime-rs`). With the historical reason, many users are still using the Go version Kata-Containers, so as openEuler community.

But a direct build, which is also issued in `SPEC`, would trigger an error message complaining:

```
Makefile:36: *** "ERROR: invalid architecture: 'riscv64'".  Stop.
```

After looking into the `Makefile` in `runtime`, the supported architectures are under the `arch` directory, and each with a file suffix `-option.mk`. Taking `x86_64` for example, the arch file would be named `amd64-option.mk`. For the Make file to run on `riscv64`, the according modification on this `Makefile` to support detection of `riscv64` and addition of `riscv64-option.mk` are required before preceding.

#### `*-option.`

Taking `amd64-option.mk` for example, it has primarily the following sections.

```
# Copyright (c) 2018-2019 Intel Corporation
#
# SPDX-License-Identifier: Apache-2.0
#

# Intel x86-64 settings

MACHINETYPE := q35
KERNELPARAMS :=
MACHINEACCELERATORS :=
CPUFEATURES := pmu=off

QEMUCMD := qemu-system-x86_64
QEMUTDXCMD := qemu-system-x86_64-tdx-experimental
QEMUSNPCMD := qemu-system-x86_64-snp-experimental
TDXCPUFEATURES := -vmx-rdseed-exit,pmu=off

# Firecracker binary name
FCCMD := firecracker
# Firecracker's jailer binary name
FCJAILERCMD := jailer

#ACRN binary name
ACRNCMD := acrn-dm
ACRNCTLCMD := acrnctl

# cloud-hypervisor binary name
CLHCMD := cloud-hypervisor

DEFSTATICRESOURCEMGMT_CLH := false

# stratovirt binary name
STRATOVIRTCMD := stratovirt
```

by observing this file, we can induce an according answer:

```
# Copyright (c) 2018-2019 Intel Corporation
#
# SPDX-License-Identifier: Apache-2.0
#

# Architecture settings

MACHINETYPE := virt
KERNELPARAMS :=
MACHINEACCELERATORS :=
CPUFEATURES := 

# QEMU
QEMUCMD := qemu-system-riscv64

# Below hypervisors are not supported, ignoring for now

## Firecracker related settings
### Firecracker binary name
#FCCMD := firecracker
### Firecracker's jailer binary name
#FCJAILERCMD := jailer
#
## ACRN related settings
### ACRN binary name
#ACRNCMD := acrn-dm
#ACRNCTLCMD := acrnctl
#
## cloud-hypervisor related settings
### cloud-hypervisor binary name
#CLHCMD := cloud-hypervisor
#
#DEFSTATICRESOURCEMGMT_CLH := false
#
## stratovirt related settings
### stratovirt binary name
#STRATOVIRTCMD := stratovirt
```

#### `runtime/pkg/govmm`

This time `make` successfully starts without complaining, but stopped at 

```
imports github.com/kata-containers/kata-containers/src/runtime/pkg/govmm: build constraints exclude all Go files in /home/openeuler/steady/kata-containers/src/runtime/pkg/govmm
```

which suggest a related file in this condition (on `riscv64`) would be needed. There files in `govmm` is telling the maximum VCPUs supported on its according architecture.

Fortunately, the `MaxCPUs` could be found in QEMU's [documentation](https://www.qemu.org/docs/master/system/riscv/virt.html), thus, the file `vmm_riscv64.go` to be added can be inferred from other architectures.

```Go
//
// Copyright (c) 2018 Intel Corporation
//
// SPDX-License-Identifier: Apache-2.0
//

package govmm

// MaxVCPUs returns the maximum number of vCPUs supported
func MaxVCPUs() uint32 {
    return uint32(512)
}
```

#### `runtime/virtcontainers`

After resolving `virtcontainers`, `make` continues to complain:

```
     BUILD    /home/openeuler/steady/kata-containers/src/runtime/kata-runtime
# github.com/kata-containers/kata-containers/src/runtime/virtcontainers
../../virtcontainers/clh.go:420:21: undefined: availableGuestProtection
../../virtcontainers/hypervisor.go:1079:13: undefined: availableGuestProtection
../../virtcontainers/qemu.go:251:16: undefined: newQemuArch
../../virtcontainers/qemu.go:2332:21: undefined: qmpMigrationWaitTimeout
../../virtcontainers/qemu.go:2347:79: undefined: qmpMigrationWaitTimeout
../../virtcontainers/qemu.go:2792:16: undefined: newQemuArch
make: *** [Makefile:776: /home/openeuler/steady/kata-containers/src/runtime/kata-runtime] Error 1
```

To define these missing symbols (functions or variables), the below files are required:

- For `availableGuestProtection`, file `hypervisor_riscv64.go` is needed, with content:
```Go
// Copyright (c) 2021 Arm Ltd.
//
// SPDX-License-Identifier: Apache-2.0

package virtcontainers

// Guest protection is not supported on RISCV64.
func availableGuestProtection() (guestProtection, error) {
    return noneProtection, nil
}

```
- For symbols in `qemu.go`, file `qemu_riscv64.go` is needed, with content:
```Go
// Copyright (c) 2023 Xin Liu
//
// SPDX-License-Identifier: Apache-2.0
//

package virtcontainers

import (
    "fmt"
    "time"

    govmmQemu "github.com/kata-containers/kata-containers/src/runtime/pkg/govmm/qemu"
)

type qemuRiscv64 struct {
    // inherit from qemuArchBase, overwrite methods if needed
    qemuArchBase
}

const defaultQemuPath = "/usr/bin/qemu-system-riscv64"

const defaultQemuMachineType = QemuVirt

const qmpMigrationWaitTimeout = 10 * time.Second

const defaultQemuMachineOptions = ""

var defaultGICVersion = uint32(3)

var kernelParams = []Param{
    {"numa", "off"},
}

var supportedQemuMachine = govmmQemu.Machine{
    Type:    QemuVirt,
    Options: defaultQemuMachineOptions,
}

// MaxQemuVCPUs returns the maximum number of vCPUs supported
func MaxQemuVCPUs() uint32 {
    return uint32(512)
}

func newQemuArch(config HypervisorConfig) (qemuArch, error) {
    machineType := config.HypervisorMachineType
    if machineType == "" {
        machineType = defaultQemuMachineType
    }

    if machineType != defaultQemuMachineType {
        return nil, fmt.Errorf("unrecognised machinetype: %v", machineType)
    }

    q := &qemuRiscv64{
        qemuArchBase{
            qemuMachine:          supportedQemuMachine,
            qemuExePath:          defaultQemuPath,
            memoryOffset:         config.MemOffset,
            kernelParamsNonDebug: kernelParamsNonDebug,
            kernelParamsDebug:    kernelParamsDebug,
            kernelParams:         kernelParams,
        },
    }

    q.handleImagePath(config)

    return q, nil
}
```

The contents of two files are inferred from other architectures.

And `make` still complains:

```
# github.com/kata-containers/kata-containers/src/runtime/virtcontainers/factory/template
../../virtcontainers/factory/template/template_linux.go:106:71: undefined: templateDeviceStateSize
make: *** [Makefile:776: /home/openeuler/steady/kata-containers/src/runtime/kata-runtime] Error 1
```

Then add a `template_riscv64.go` file:

```Go
// Copyright (c) 2019 HyperHQ Inc.
//
// SPDX-License-Identifier: Apache-2.0
//
// template implements base vm factory with vm templating.

package template

// templateDeviceStateSize denotes device state size when
// mount tmpfs.
// when bypass-shared-memory is not support like arm64,
// creating template will occupy more space. That's why we
// put it here.
const templateDeviceStateSize = 8
```

#### `runtime/cmd/kata-runtime`

The errors emitted from `virtcontainers` are now resolved. New errors emerged this time:

```Go
// Copyright (c) 2018 Intel Corporation
//
// SPDX-License-Identifier: Apache-2.0
//

package main

import (
    "fmt"
    "strings"

    vc "github.com/kata-containers/kata-containers/src/runtime/virtcontainers"
    "github.com/sirupsen/logrus"
)

const (
    cpuFlagsTag        = genericCPUFlagsTag
    archCPUVendorField = "uarch"
    archCPUModelField  = "isa"
)

// archRequiredCPUFlags maps a CPU flag value to search for and a
// human-readable description of that value.
var archRequiredCPUFlags = map[string]string{}

// archRequiredCPUAttribs maps a CPU (non-CPU flag) attribute value to search for
// and a human-readable description of that value.
var archRequiredCPUAttribs = map[string]string{}

// archRequiredKernelModules maps a required module name to a human-readable
// description of the modules functionality and an optional list of
// required module parameters.
var archRequiredKernelModules = map[string]kernelModule{
    "kvm": {
            desc:     "Kernel-based Virtual Machine",
            required: true,
    },
    "vhost_vsock": {
            desc:     "Host Support for Linux VM Sockets",
            required: false,
    },
}

func setCPUtype(hypervisorType vc.HypervisorType) error {
    return nil
}

// kvmIsUsable determines if it will be possible to create a full virtual machine
// by creating a minimal VM and then deleting it.
func kvmIsUsable() error {
    return genericKvmIsUsable()
}

func archHostCanCreateVMContainer(hypervisorType vc.HypervisorType) error {
    return kvmIsUsable()
}

// hostIsVMContainerCapable checks to see if the host is theoretically capable
// of creating a VM container.
func hostIsVMContainerCapable(details vmContainerCapableDetails) error {

    _, err := getCPUInfo(details.cpuInfoFile)
    if err != nil {
            return err
    }

    count, err := checkKernelModules(details.requiredKernelModules, archKernelParamHandler)
    if err != nil {
            return err
    }

    if count == 0 {
            return nil
    }

    return fmt.Errorf("ERROR: %s", failMessage)

}

func archKernelParamHandler(onVMM bool, fields logrus.Fields, msg string) bool {
    return genericArchKernelParamHandler(onVMM, fields, msg)
}

func getRiscv64CPUDetails() (vendor, model string, err error) {
    prefixModel := "processor"
    cpuinfo, err := getCPUInfo(procCPUInfo)
    if err != nil {
            return "", "", err
    }

    lines := strings.Split(cpuinfo, "\n")

    for _, line := range lines {
            if archCPUVendorField != "" {
                    if strings.HasPrefix(line, archCPUVendorField) {
                            fields := strings.Split(line, ":")
                            if len(fields) > 1 {
                                    vendor = strings.TrimSpace(fields[1])
                            }
                    }
            } else {
                    vendor = "Unknown"
            }
            if archCPUModelField != "" {
                    if strings.HasPrefix(line, prefixModel) {
                            fields := strings.Split(line, ":")
                            if len(fields) > 1 {
                                    model = strings.TrimSpace(fields[1])
                            }
                    }
            }
    }

    if vendor == "" {
            return "", "", fmt.Errorf("cannot find vendor field in file %v", procCPUInfo)
    }

    if model == "" {
            return "", "", fmt.Errorf("Error in parsing cpu model from %v", procCPUInfo)
    }

    return vendor, model, nil
}

func getCPUDetails() (string, string, error) {
    if vendor, model, err := genericGetCPUDetails(); err == nil {
            return vendor, model, nil
    }
    return getRiscv64CPUDetails()
}
```

Up unitl now, the KataContainers 3.3.0 Go version runtime compiles without complaining.
