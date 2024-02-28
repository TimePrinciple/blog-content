---
# Must present
author: Ruoqing He
title: MEGA FUSE
# Optional
date: 2024-02-22
summary: A work record of implementing Fuse module of Mega.
---

## FUSE

FUSE (Filesystem in Userspace) is an interface for userspace programs to export a filesystem to the Linux kernel. It consists of two components: the *fuse* kernel module (`fuse.ko`, maintained in the regular kernel repository) and the *libfuse* userspace library. A FUSE file system is typically implemented as a standalone application that links with *libfuse*.

Specifically, FUSE is actually a userspace filesystem framework. It consists of a kernel module (`fuse.ko`), a userspace library (`libfuse.*`) and a mount utility (`fusermount`).

FUSE is particularly useful for writing *virtual file systems*. Unlike traditional file systems that essentially work with data on mass storage, virtual filesystems don't actually store data themselves. They act as a view or translation of an existing file system or storage device.

To get a FUSE working, there are things required:
- Filesystem daemon: Process(es) providing the data and metadata of the filesystem.
- Non-privileged mount (user mount): A userspace filesystem mounted by a non-privileged user. The filesystem daemon is running with the privileges of the mounting user.
- Filesystem connection: A connection between filesystem daemon and the kernel. The connection exists until either the daemon dies, or the filesystem is unmounted. Note that detaching (or lazy umounting) the filesystem does not break the connection, in this case it will exist until the last reference to the filesystem is released.

### [libfuse](https://github.com/libfuse/libfuse)

*libfuse* provides functions to mount the file system, unmount it, read requests from the kernel, and send responses back. The operation set and the structures involved in FUSE communication is described in the `fuse_kernel.h` file.

#### FUSE Protocol

The FUSE protocol follows the client/server scheme. In existing implementations, *kernel* plays the cline role and the *filesystem daemon* plays the server role.

Communications goes as follows there a fixed set of operations. These can be grouped into two sets: operations for which messaging follows a 

```
client --[query]--> server --[answer]--> client
```

scheme, and operations for which messaging follows a 

```
client --[query]--> server
```

scheme. As of writing this, almost all operations follow the former scheme; AFAIK FUSE_FORGET is the only one which uses the second one.

Queries and answers have a message ID attribute. The server is expected to preserve the *messages ID* when answering a query (Message ID generation is up to the client. Usually the client will want to archive that it can uniquely identify answers. In practice, this implies that message IDs should be **unique among ongoing operations**, but need not be globally unique.)

Each query and answer is self contained, there is **no packet fragmentation** like in TCP/IP. The whole FUSE session starts with a *FUSE_INIT* operation which (among other purposes) is used for negotiating the size limits of messages (in earlier versions of the protocol these limits were constants but now they can be effectively negotiated).

Each query starts with a `fuse_in_header` structure, and each answer starts with a `fuse_out_structure`.

### Filesystem Daemon

To implement a new file system, a handler program linked to the supplied `libfuse` library needs to be written. The main purpose of this program is to specify how the file system ts to respond to read/write/stat requests. The program is also used to mount the new file system. At the time the file system is mounted, the handler is registered with the kernel. If a user now issues read/write/stat requests for this newly mounted file system, the kernel forwards these IO-requests to the handler and then send the handler's response back to the user.

The daemon program will be written in Rust since it is a sub-module (or sub-project) of Mega, and Mega is written in Rust.

## Fuser

The *fuser* crate is the base crate of writing a FUSE daemon process in Rust. It could be understood as a Rust version of the *interface reference* of *libfuse*. Of course, since *fuser* depends on *libc* instead of *nix*, there are certainly `unsafe` code.

## Design

The *fuse* sub-project is ought to be a installable program on user's device. I'll name it `mega-fuse` from this point on. `mega-fuse` by design plays two different roles: client and server. It is the server for *fuse kernel module* and the client of `mega` code servers. When a user tries to access the content of a file in a repository hosted on *mega server*, the contents are fetched gradually (i.e. by simply connecting to that repo, the whole content is not pulled down, unless you somehow accessed all files/directories in that repository).

```
# Query process
fuse.ko --[query]--> mega-fuse --[query]--> mega
# Answer process
mega --[answer]--> mega-fuse --[answer]--> fuse.ko
```

### Usage

Normally, when a user try to mount a repository hosted on a mega server, he/she will issue:

```bash
$ mega-fuse --mount-point <path/to/mount-point> --cache-dir <path/to/cache> --log-dir <path/to/log> --mega-url <url> connect <repo_name>
```

this command will start a process does the preparation, mounting, and starts another process which is to be *re-parent*ed to `init` (daemonize the process), then exit. After the command returns, the `<path/to/mount-point>` will be available, and content of that repo is accessible thereafter.

If the user want to unmount the repository, the proper way to do it is by issuing:

```bash
$ mega-fuse disconnect <repo_name>
```

thus additional procedures, like cleaning and etc. can take place.

### Implementation

`mega-fuse` uses `mega`'s [*git objects retrieval API*](https://github.com/web3infra-foundation/mega/blob/main/docs/api.md#git-objects-retrieval-api). Typically, these APIs support the basic retrieval of directories and files in repositories.

For a repository to be available locally, despite the content, the tree (i.e. the directory) is to be fetched first.

