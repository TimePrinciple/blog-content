---
# Must present
author: Ruoqing
title: OERV Start RISCV VM
# Optional
date: 2024-03-26
summary: Documentation of starting A riscv64 QEMU virtual machine.
---

## Preface

This documentation is written for *Arch Linux x86_64*, as of other linux distributions or architectures, the steps may vary.

## Preparation

### Install QEMU

```bash
$ sudo pacman -S qemu-full
```

### Download Scripts and Images

To run a basic *openEuler riscv64* virtual machine the following files are needed on [riscv-sig mirror site](https://mirror.iscas.ac.cn/openeuler-sig-riscv/openEuler-RISC-V/preview/). Taking *openEuler 23.09* for example:

- A [firmware](https://mirror.iscas.ac.cn/openeuler-sig-riscv/openEuler-RISC-V/preview/openEuler-23.09-V1-riscv64/QEMU/fw_payload_oe_uboot_2304.bin)
- An [image](https://mirror.iscas.ac.cn/openeuler-sig-riscv/openEuler-RISC-V/preview/openEuler-23.09-V1-riscv64/QEMU/openEuler-23.09-V1-base-qemu-preview.qcow2.zst)
- A [script](https://mirror.iscas.ac.cn/openeuler-sig-riscv/openEuler-RISC-V/preview/openEuler-23.09-V1-riscv64/QEMU/start_vm.sh).

After these files are downloaded, run `unzstd <*qcow2.zst>` to extract the *qcow2* image.

## Start VM

Run `bash start_vm.sh` to boot the VM, the default users are `root` or `openeuler`, and the password for them is `openEuler12#$`.
