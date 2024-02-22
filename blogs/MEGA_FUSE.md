---
# Must present
author: Ruoqing He
title: MEGA FUSE
# Optional
date: 2024-02-22
summary: A work record of implementing Fuse module of Mega.
---

## FUSE

Generally specking, Filesystem in Userspace (FUSE) is a filesystem in which data and metadata are provided by an ordinary userspace process. The filesystem can be accessed normally through the kernel interface.

i.e. it is a software interface for Unix and Unix-like computer OSs that lets non-privileged users create their own file systems without editing kernel code. This is achieved by running file system code in user space while the FUSE module provides only a bridge to the actual kernel interfaces.

Specifically, FUSE is actually a userspace filesystem framework. It consists of a kernel module (`fuse.ko`), a userspace library (`libfuse.*`) and a mount utility (`fusermount`).

FUSE is particularly useful for writing *virtual file systems*. Unlike traditional file systems that essentially work with data on mass storage, virtual filesystems don't actually store data themselves. They act as a view or translation of an existing file system or storage device.

To get a FUSE working, there are things required:
- Filesystem daemon: Process(es) providing the data and metadata of the filesystem.
- Non-privileged mount (user mount): A userspace filesystem mounted by a non-privileged user. The filesystem daemon is running with the privileges of the mounting user.
- Filesystem connection: A connection between filesystem daemon and the kernel. The connection exists until either the daemon dies, or the filesystem is unmounted. Note that detaching (or lazy umounting) the filesystem does not break the connection, in this case it will exist until the last reference to the filesystem is released.

### Filesystem Daemon

To implement a new file system, a handler program linked to the supplied `libfuse` library needs to be written. The main purpose of this program is to psecify how the file system ts to respond to read/write/stat requests. The program is also used to mount the new file system. At the time the file system is mounted, the handler is registeered with the kernel. If a user now issues read/write/stat requests for this newly mounted file system, the kernel forwards these IO-requests to the handler and then send the handler's response back to the user.