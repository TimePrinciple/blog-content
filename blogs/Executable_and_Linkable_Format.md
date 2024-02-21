---
date: 2024-02-18
title: Executable and Linkable Format
author: Ruoqing He
summary: An introduction to `ELF` and `readelf` command.
cover: https://raw.githubusercontent.com/TimePrinciple/resources/master/images/covers/7a755c8e2d3cf72a54db26fa7cb3122b412d2352.png
---

> **Executable and Linkable Format** (ELF, formerly named **Extensible Linking Format**), is a common standard file format for executable files, object code, shared libraries, and core dumps. By design the ELF format is flexible, extensible, and cross-platform. For instance, it supports different endiannesses and address sizes so it does not exclude any particular CPU or instruction set architecture. This has allowed it to be adopted by many different operating systems on many different hardware platforms.

## Object File

Object files are created by the assembler and link editor. Object files are binary representations of programs intended to *execute directly on a processor*.

Object files participate in program linking (building a program) and program execution (running a program). For efficiency, the object file format provides parallel views of a file's contents, reflecting the differing needs of these activities.

```
//            Linking View                         Execution View
// +=================================+   +=================================+
// |           ELF Header            |   |           ELF Header            |
// +=================================+   +=================================+
// | Program Header Table (optional) |   | Program Header Table            |
// +---------------------------------+   +---------------------------------+
// | Section 1                       |   | Segment 1                       |
// +---------------------------------+   +---------------------------------+
// | ...                             |   | Segment 2                       |
// +---------------------------------+   +---------------------------------+
// | Section n                       |   | ...                             |
// +---------------------------------+   +---------------------------------+
// | ...                             |   | Section Header Table (optional) |
// +---------------------------------+   +---------------------------------+
// | ...                             |
// +---------------------------------+
// | Section Header Table            |
// +---------------------------------+
```

### Types of Object File

- *relocatable file*: holds code and data suitable for linking with other object files to create an executable or a shared object file.
- *executable file*: holds a program suitable for execution.
- *shared object file*: holds code and data suitable for linking in two contexts. First, the link editor may process it with other relocatable and shared object files to create another object file. Second, the dynamic linker combines it with an executable file and other shared objects to create a process image.

#### Shared Object, Dynamic Library

The *shared object* (.so) file is also of ELF format under Linux systems. The symbols inside could be examine by command `readelf -Ws --dyn-syms <path/to/lib.so>`.

##### Rust

Accordingly, to compile a `.so`, `.dll` or `dylib` in Rust, navigate to the `Cargo.toml` file under that package, add `"dylib"` (or `"cdylib"`) to the `crate-type` field in `[lib]` section. Then compile. **Don't forget to add `#[no_mangle]` under the symbols which are designated to be exported (`libloading`ed specifically in Rust).

### View, Layout

Examples below uses a `Hello World` executable written in Rust, named `exe` , which could simply be created through `cargo new <binary_name>`.

#### ELF Header

Position Fixed.

An *ELF header* resides at the beginning and holds a "road map" describing the file's organization. *Section*s hold the bulk of object file information for the linking view: instructions, data, symbol table, relocation information, and os on.

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

#### Program Header Table

If present, the table tells the system how to create a *process image*. Files used to build a process image (execute a program) must have a program header table; while relocatable files do not need one.

#### Section Header Table

If present, the table contains information describing the file's sections. Every section has an entry in the table; each entry gives information such as the section name, the section size, and so on. Files used during linking must have a section header table; other object files may or may not have one.