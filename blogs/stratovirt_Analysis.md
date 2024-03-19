---
# Must present
author: Ruoqing He
title: StratoVirt Analysis
# Optional
date: 2024-03-17
summary: An analysis of `StratoVirt` developed by openEuler community.
---

## Preface

This blog is a research for adding support for *riscv64* architecture to `StratoVirt` project.

`StratoVirt` at some level is similar to `QEMU`, which uses `kvm` as its hypervisor. `StratoVirt` begins by directing the hypervisor to create a hardware VM; after that, I/O requests are funneled by the hypervisor to `StratoVirt` and responded to in much same way any kind of network daemon might exist to handle application requests and provide responses. KVM passes I/O requests to the `StratoVirt` process which asked for the VM to be created, which also effectively supervises the VM. (Purely for purposes of illustration, there's really no reason you couldn't make a VMM which proxies all I/O requests from the hypervisor as HTTP requests to some endpoint. This is obviously a terrible idea and would perform awfully, but illustrates that, after initialization, the VM largely just functions as a server processing requests.)

## Source Code Layout

By searching `#[cfg(target_arch = "x86_64")]` keyword, I located most places in `StratoVirt` which are expect to be architecture dependent, and most likely where works for support riscv64 to be done. The places are marked by `*` below.

```bash
StratoVirt
├── acpi *
├── address_space
├── block_backend
│   └── qcow2
├── boot_loader *
│   ├── aarch64
│   └── x86_64
│       ├── direct_boot
│       └── standard_boot
├── chardev_backend
├── cpu *
│   ├── aarch64
│   └── x86_64
├── devices
│   ├── acpi *
│   ├── camera_backend
│   ├── interrupt_controller
│   │   └── aarch64
│   ├── legacy *
│   ├── misc *
│   │   └── scream
│   ├── pci *
│   │   └── demo_device
│   ├── scsi
│   ├── smbios
│   ├── sysbus *
│   └── usb *
│       ├── usbhost
│       └── xhci
├── hypervisor *
│   └── kvm
│       ├── aarch64
│       └── x86_64
├── image
├── machine *
│   ├── aarch64
│   ├── micro_common
│   ├── standard_common
│   └── x86_64
├── machine_manager *
│   ├── config
│   └── qmp
├── migration *
│   └── migration_derive
├── ozone
├── src
├── trace
│   ├── trace_event
│   └── trace_generator
├── ui
│   ├── gtk
│   └── vnc
│       └── encoding
├── util *
│   └── aio
├── vfio *
└── virtio *
    ├── device
    ├── queue
    ├── transport
    └── vhost
        ├── kernel
        └── user
```

## Dependencies

Before make `StratoVirt` to work on riscv64, it must be able to compile in advance. Most dependencies found in this project are architecture independent, except for `kvm_bindings` crate. This crate currently supports `x86_64`, `arm64` and `arm`. Though it is possible to generate the FFI bindings using `bindgen` manually.

Through `kvm_bindings` crate, the C structures could be directly used in Rust code. And these structures include necessary fields of virtualized vms, vcpus, devices and interrupt-chips.

## Project Layout

## Execution Logic

### Establish Connection to KVM

`StratoVirt` starts using KVM by opening the device `/dev/kvm` when it is constructing a `KvmHypervisor`, through the `new()` method defined in module `Kvm` of crate `kvm-ioctls`. `KvmHypervisor` must get a handle (a file descriptor) to the kernel module before being able to create any VM. Once the device open (by the FD of `/dev/kvm`), the opening process can talk to KVM module using `ioctl`s.

The interaction between `StratoVirt` and `KVM` would look like:

```
StratoVirt                                      
┌────────────────────┐                           
│ KVM FD (/dev/kvm)  ├──► Instance of StrartoVirt
│  ┌───────────────┐ │                           
│  │ KVM VM FD     ├─┼──► Instance of VM         
│  │ ┌───────────┐ │ │                           
│  │ │KVM VCPU FD├─┼─┼──► Instance of VCPU       
│  │ └───────────┘ │ │                           
│  │  ...          │ │                           
│  └───────────────┘ │                           
│   ...              │                           
└────────────────────┘                           
```

### Create a VM

Before creating a VM, `StratoVirt` detects the VM's type to be created from config, possible values are:

- `microvm` => `LightMachine`
- `q35`     => `StdMachine` on x86_64
- `virt`    => `StdMachine` on aarch64

here, the definition of `LightMachine` is more close to *container*. It then maintains structures like `MachineBase` and the like built from configuration and command line input, prepare sockets speaks *QEMU Machine Protocol*.

A succeeding `create_vm()` method on `Kvm` will as its name suggests, create a VM and giving the rights to the process to control that VM. `create_vm()` in its nature is an `ioctl` with `KVM_CREATE_VM` on previously created `kvm_fd`. And this method, returns a `vm_fd`, which represents the VM just created.

The connection would be:

```
StrartoVirt    ┌───────┬───────┬───────┬───────────────┐                               
┌──────────────┼───────┼───────┼───────┼──────────┐    │        ┌─────────────────────┐
│              │       │       │       │          │    │        │ KVM FD (/dev/kvm)   │
│ ┌───────┐ ┌──┴──┐ ┌──┴──┐ ┌──┴──┐ ┌──┴──┐       │    │        │ ┌─────────────────┐ │
│ │ state │ │ CPU │ │ CPU │ │ CPU │ │ CPU │ ...   │    │   ┌────┼─► KVM VM FD       │ │
│ └───────┘ └─────┘ └─────┘ └─────┘ └─────┘       │    │   │    │ │ ┌─────────────┐ │ │
│ ┌─────────────────────────────────────────────┐ │    ├───┼────┼─┼─► KVM VCPU FD │ │ │ 
│ │ RAM                                         │ │    │   │    │ │ └─────────────┘ │ │
│ │ ┌───────────────────┐ ┌───────────────────┐ │ │    └───┼────┼─┼─►...            │ │
│ │ │ Region            │ │ Region            │ │ │        │    │ └─────────────────┘ │
│ │ │ ┌──────┐ ┌──────┐ │ │ ┌──────┐ ┌──────┐ │ │ │        │    │  ...                │
│ │ │ │ type │ │ size │ │ │ │ type │ │ size │ │ │ │        │    └─────────────────────┘
│ │ │ └──────┘ └──────┘ │ │ └──────┘ └──────┘ │ │ │        │                           
│ │ │              ...  │ │              ...  │ │ │        │                           
│ │ └───────────────────┘ └───────────────────┘ │ │        │                           
│ │                                        ...  │ │        │                           
│ └──────────────────────▲──────────────────────┘ │        │                           
│ ┌──────────────────────┴──────────────────────┐ │        │                           
│ │                     BUS                     │ │        │                           
│ └─────────┬───────────────────────┬───────────┘ │        │                           
│ ┌─────────▼─────────┐ ┌───────────▼───────────┐ │        │                           
│ │ IO                │ │ Devices               │ │        │                           
│ │ ┌──────┐ ┌──────┐ │ │ ┌────────┐ ┌────────┐ │ │        │                           
│ │ │ type │ │ size │ │ │ │ Device │ │ Device │ │ │        │                           
│ │ └──────┘ └──────┘ │ │ └────────┘ └────────┘ │ │        │                           
│ │              ...  │ │                  ...  │ │        │                           
│ └───────────────────┘ └───────────────────────┘ │        │                           
│ ┌───────────┐ ┌──────────┐       ┌────────────┐ │        │                           
│ │ FwCfg Dev │ │ irq_chip │  ...  │ hypervisor ├─┼────────┘                           
│ └───────────┘ └──────────┘       └────────────┘ │                                    
└─────────────────────────────────────────────────┘                                    
```

the arrows above does not stand for the flow of data, but a pointer connection between these structures.

### Create vCPUs

The CPUs are determined in a `MachineBase` by field `cpus`, which are created during the preparation stage. Each call to `create_vcpu` on the `vm_fd` through `ioctl` with `KVM_CREATE_VCPU` and `id`, will return a `vcpu_fd` which represents the CPU with index `id` available to that VM on `vm_fd`.

But VMs will not function until they have their vCPUs attached and been invoked with `KVM_RUN`.

The interaction between `CPU` in `StratoVirt` would be:

```
StratoVirt                                                 
┌─────────────────────────┐         ┌─────────────────────┐
│ CPU                     │         │ KVM FD (/dev/kvm)   │
│ ┌────┐ ┌───────┐ ┌────┐ │         │ ┌─────────────────┐ │
│ │ id │ │ state │ │ vm ├─┼─────────┼─► KVM VM FD       │ │
│ └────┘ └───────┘ └────┘ │         │ │ ┌─────────────┐ │ │
│    ┌───────────────┐    │    ┌────┼─┼─► KVM VCPU FD │ │ │
│    │ thread_handle │    │    │    │ │ └─────────────┘ │ │
│    │    ┌─────┐    │    │    │    │ │  ...            │ │
│    │    │ tid │    │    │    │    │ └─────────────────┘ │
│    │    └─────┘    │    │    │    │  ...                │
│    └───────────▲───┘    │    │    └─────────────────────┘
│                │        │    │
│ ┌──────┐ ┌─────▼──────┐ │    │
│ │ arch │ │ hypervisor ◄─┼────┘
│ └──────┘ └────────────┘ │
│                   ...   │
└─────────────────────────┘                                
 ...
```

