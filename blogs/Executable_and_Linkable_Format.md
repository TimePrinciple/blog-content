---
date: 2024-02-18
title: Executable and Linkable Format
author: Ruoqing He
summary: An introduction to `ELF` and `readelf` command.
cover: https://raw.githubusercontent.com/TimePrinciple/resources/master/images/covers/7a755c8e2d3cf72a54db26fa7cb3122b412d2352.png
---

> **Executable and Linkable Format** (ELF, formerly named **Extensible Linking Format**), is a common standard file format for executable files, object code, shared libraries, and core dumps. By design the ELF format is flexible, extensible, and cross-platform. For instance, it supports different endiannesses and address sizes so it does not exclude any particular CPU or instruction set architecture. This has allowed it to be adopted by many different operating systems on many different hardware platforms.

## File Layout

Examples below uses a `Hello World` executable written in Rust, named `exe` , which could simply be created through `cargo new <binary_name>`.

### File Header

Examine by `readelf --file-header <executable>`.

```sh
# Sample output
$ readelf --file-header exe
ELF Header:
  Magic:   7f 45 4c 46 02 01 01 00 00 00 00 00 00 00 00 00                          # 0x00
  Class:                             ELF64                                          # 0x04 
  Data:                              2's complement, little endian                  # 0x05
  Version:                           1 (current)                                    # 0x06
  OS/ABI:                            UNIX - System V                                # 0x07
  ABI Version:                       0                                              # 0x08
  # 7 Bytes padding
  Type:                              DYN (Position-Independent Executable file)     # 0x10
  Machine:                           Advanced Micro Devices X86-64                  # 0x12
  Version:                           0x1                                            # 0x14
  Entry point address:               0x87e0                                         # 0x18
  Start of program headers:          64 (bytes into file)                           # 0x20
  Start of section headers:          4415080 (bytes into file)                      # 0x28
  Flags:                             0x0                                            # 0x30
  Size of this header:               64 (bytes)                                     # 0x34
  Size of program headers:           56 (bytes)                                     # 0x36
  Number of program headers:         14                                             # 0x38
  Size of section headers:           64 (bytes)                                     # 0x3A
  Number of section headers:         41                                             # 0x3C
  Section header string table index: 40                                             # 0x3E
                                                                                    # 0x40
```
