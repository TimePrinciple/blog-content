---
# Must present
author: Ruoqing He
title: KVM
# Optional
date: 2024-03-18
summary: A introduction to Kernel-base Virtual Machine.
---

## KVM API

Documented [here](https://www.kernel.org/doc/Documentation/virtual/kvm/api.txt).

The API is a set of `ioctl`s that are issued to control various aspects of a virtual machine. Include:

- System `ioctl`s: These query and set global attributes which affect the whole kvm subsystem. In addition a system `ioctl` is used to create virtual machines.
- VM `ioctl`s: These query and set attributes that affect an entire virtual machine, for example memory layout. In addition a VM `ioctl` is used to create virtual cpus (vcpus) and devices. VM `ioctl`s must be issued from the same process (address space) that was used to create the VM.
- vcpu `ioctl`s: These query and set attributes that control the operation of a single virtual cpu. vcpu `ioctl`s should be issued from the same thread that was used to create the vcpu, except for asynchronous vcpu `ioctl`. Otherwise, the first `ioctl` after switching threads could see a performance impact.
- device `ioctl`s: These query and set attributes that control the operation of a single device. device `ioctl`s must be issued form the same process (address space) that was used to create the VM.

### FDs

The kvm API is centered around file descriptors. An initial `open("/dev/kvm")` obtains a handle to the kvm subsystem: this handle can be used to issue system `ioctl`s. A `KVM_CREATE_VM` `ioctl` on this handle will create a VM file descriptor which can be used to issue VM `ioctl`s. A `KVM_CREATE_VCPU` or `KVM_CREATE_DEVICE` `ioctl` on a VM fd will create a virtual cpu or device and return a file descriptor pointing to the new resource. Finally, `ioctl`s on a vcpu or device fd can be sued to control the vcpu or device. For vcpus, this includes the important task of actually running guest code.

It is important to note that although VM `ioctl`s may only be issued from the process that created the VM, a VM's lifecycle is associated with its file descriptor, not its creator (process). In other words, the VM and its resources, **including the associated address space**, are not freed until the last reference to the VM's file descriptor has been released (due to the nature of file descriptors).

### `KVM_CREATE_VM`

- Capability: basic
- Architectures: all
- Type: system `ioctl`
- Parameters: machine type identifier (KVM_VM_*)
- Returns: a VM fd that can be used to control the new virtual machine.

The new VM has no virtual cpus and no memory. 

### `KVM_CREATE_VCPU`

- Capability: basic
- Architectures: all
- Type: vm `ioctl`
- Parameters: vcpu id (apic id on x86) (in range [0, max_vcpu_id))
- Returns: vcpu fd on success, -1 on error

This API adds a vcpu to a virtual machine. No more than `max_vcpus` may be added (pre-configured in VM's fd). 

### `KVM_CREATE_DEVICE`

- Capability: KVM_CAP_DEVICE_CTRL
- Type: vm ioctl
- Parameters: struct kvm_create_device (in/out)
- Returns: 0 on success, -1 on error

Creates an emulated device in the kernel. The file descriptor returned in fd can be used with `KVM_SET/GET/HAS_DEVICE_ATTR`.

### `KVM_CREATE_IRQCHIP`

- Capability: KVM_CAP_IRQCHIP, KVM_CAP_S390_IRQCHIP (s390)
- Architectures: x86, ARM, arm64, s390
- Type: vm ioctl
- Parameters: none
- Returns: 0 on success, -1 on error

Creates an interrupt controller model in the kernel. On x86, creates a virtual `ioapic`, a virtual PIC, and sets up future vcpus to have a local APIC.

### `kvm_run` structure

Application code obtains a pointer to the `kvm_run` structure by `mmap()`ing a vcpu fd. From that point, application code can control execution by changing fields in `kvm_run` prior to calling the `KVM_RUN` `ioctl`, and obtain information about the reason `KVM_RUN` returned by looking up structure members.

```c
struct kvm_run {
	/* in */
    /// Request that KVM_RUN return when it becomes possible to inject external
    /// interrupts into the guest.  Useful in conjunction with KVM_INTERRUPT.
	__u8 request_interrupt_window;

    /// This field is polled once when KVM_RUN starts; if non-zero, KVM_RUN
    /// exits immediately, returning -EINTR.  In the common scenario where a
    /// signal is used to "kick" a VCPU out of KVM_RUN, this field can be used
    /// to avoid usage of KVM_SET_SIGNAL_MASK, which has worse scalability.
    /// Rather than blocking the signal outside KVM_RUN, userspace can set up
    /// a signal handler that sets run->immediate_exit to a non-zero value.
	__u8 immediate_exit;

    /// This field is ignored if KVM_CAP_IMMEDIATE_EXIT is not available.
	__u8 padding1[6];

	/* out */
    /// When KVM_RUN has returned successfully (return value 0), this informs
    /// application code why KVM_RUN has returned.  Allowable values for this
    /// field are detailed below.
	__u32 exit_reason;

    /// If request_interrupt_window has been specified, this field indicates
    /// an interrupt can be injected now with KVM_INTERRUPT.
	__u8 ready_for_interrupt_injection;

    /// The value of the current interrupt flag.  Only valid if in-kernel
    /// local APIC is not used.
	__u8 if_flag;

    /// More architecture-specific flags detailing state of the VCPU that may
    /// affect the device's behavior.  The only currently defined flag is
    /// KVM_RUN_X86_SMM, which is valid on x86 machines and is set if the
    /// VCPU is in system management mode.
	__u16 flags;

	/* in (pre_kvm_run), out (post_kvm_run) */
    /// The value of the cr8 register.  Only valid if in-kernel local APIC is
    /// not used.  Both input and output.
	__u64 cr8;

    /// The value of the APIC BASE msr.  Only valid if in-kernel local
    /// APIC is not used.  Both input and output.
	__u64 apic_base;

	union {
		/* KVM_EXIT_UNKNOWN */
        /// If exit_reason is KVM_EXIT_UNKNOWN, the vcpu has exited due to unknown
        /// reasons.  Further architecture-specific information is available in
        /// hardware_exit_reason.
		struct {
			__u64 hardware_exit_reason;
		} hw;

		/* KVM_EXIT_FAIL_ENTRY */
        /// If exit_reason is KVM_EXIT_FAIL_ENTRY, the vcpu could not be run due
        /// to unknown reasons.  Further architecture-specific information is
        /// available in hardware_entry_failure_reason.
		struct {
			__u64 hardware_entry_failure_reason;
		} fail_entry;

		/* KVM_EXIT_EXCEPTION */
        /// Unused.
		struct {
			__u32 exception;
			__u32 error_code;
		} ex;

		/* KVM_EXIT_IO */
        /// If exit_reason is KVM_EXIT_IO, then the vcpu has
        /// executed a port I/O instruction which could not be satisfied by kvm.
        /// data_offset describes where the data is located (KVM_EXIT_IO_OUT) or
        /// where kvm expects application code to place the data for the next
        /// KVM_RUN invocation (KVM_EXIT_IO_IN).  Data format is a packed array.
		struct {
#define KVM_EXIT_IO_IN  0
#define KVM_EXIT_IO_OUT 1
			__u8 direction;
			__u8 size; /* bytes */
			__u16 port;
			__u32 count;
			__u64 data_offset; /* relative to kvm_run start */
		} io;

		/* KVM_EXIT_DEBUG */
        /// If the exit_reason is KVM_EXIT_DEBUG, then a vcpu is processing a debug event
        /// for which architecture specific information is returned.
		struct {
			struct kvm_debug_exit_arch arch;
		} debug;

		/* KVM_EXIT_MMIO */
        /// If exit_reason is KVM_EXIT_MMIO, then the vcpu has
        /// executed a memory-mapped I/O instruction which could not be satisfied
        /// by kvm.  The 'data' member contains the written data if 'is_write' is
        /// true, and should be filled by application code otherwise.

        /// The 'data' member contains, in its first 'len' bytes, the value as it would
        /// appear if the VCPU performed a load or store of the appropriate width directly
        /// to the byte array.

        /// NOTE: For KVM_EXIT_IO, KVM_EXIT_MMIO, KVM_EXIT_OSI, KVM_EXIT_PAPR and
        ///     KVM_EXIT_EPR the corresponding
        /// operations are complete (and guest state is consistent) only after userspace
        /// has re-entered the kernel with KVM_RUN.  The kernel side will first finish
        /// incomplete operations and then check for pending signals.  Userspace
        /// can re-enter the guest with an unmasked signal pending to complete
        /// pending operations.
		struct {
			__u64 phys_addr;
			__u8  data[8];
			__u32 len;
			__u8  is_write;
		} mmio;

		/* KVM_EXIT_HYPERCALL */
        /// Unused.  This was once used for 'hypercall to userspace'.  To implement
        /// such functionality, use KVM_EXIT_IO (x86) or KVM_EXIT_MMIO (all except s390).
        /// Note KVM_EXIT_IO is significantly faster than KVM_EXIT_MMIO.
		struct {
			__u64 nr;
			__u64 args[6];
			__u64 ret;
			__u32 longmode;
			__u32 pad;
		} hypercall;

		/* KVM_EXIT_TPR_ACCESS */
        /// To be documented (KVM_TPR_ACCESS_REPORTING).
		struct {
			__u64 rip;
			__u32 is_write;
			__u32 pad;
		} tpr_access;

		/* KVM_EXIT_S390_SIEIC */
        /// s390 specific.
		struct {
			__u8 icptcode;
			__u64 mask; /* psw upper half */
			__u64 addr; /* psw lower half */
			__u16 ipa;
			__u32 ipb;
		} s390_sieic;

		/* KVM_EXIT_S390_RESET */
        /// s390 specific.
#define KVM_S390_RESET_POR       1
#define KVM_S390_RESET_CLEAR     2
#define KVM_S390_RESET_SUBSYSTEM 4
#define KVM_S390_RESET_CPU_INIT  8
#define KVM_S390_RESET_IPL       16
		__u64 s390_reset_flags;

		/* KVM_EXIT_S390_UCONTROL */
        /// s390 specific. A page fault has occurred for a user controlled virtual
        /// machine (KVM_VM_S390_UNCONTROL) on it's host page table that cannot be
        /// resolved by the kernel.
        /// The program code and the translation exception code that were placed
        /// in the cpu's lowcore are presented here as defined by the z Architecture
        /// Principles of Operation Book in the Chapter for Dynamic Address Translation
        /// (DAT)
		struct {
			__u64 trans_exc_code;
			__u32 pgm_code;
		} s390_ucontrol;

		/* KVM_EXIT_DCR */
        /// Deprecated - was used for 440 KVM.
		struct {
			__u32 dcrn;
			__u32 data;
			__u8  is_write;
		} dcr;

		/* KVM_EXIT_OSI */
        /// MOL uses a special hypercall interface it calls 'OSI'. To enable it, we catch
        /// hypercalls and exit with this exit struct that contains all the guest gprs.

        /// If exit_reason is KVM_EXIT_OSI, then the vcpu has triggered such a hypercall.
        /// Userspace can now handle the hypercall and when it's done modify the gprs as
        /// necessary. Upon guest entry all guest GPRs will then be replaced by the values
        /// in this struct.
		struct {
			__u64 gprs[32];
		} osi;

		/* KVM_EXIT_PAPR_HCALL */
        /// This is used on 64-bit PowerPC when emulating a pSeries partition,
        /// e.g. with the 'pseries' machine type in qemu.  It occurs when the
        /// guest does a hypercall using the 'sc 1' instruction.  The 'nr' field
        /// contains the hypercall number (from the guest R3), and 'args' contains
        /// the arguments (from the guest R4 - R12).  Userspace should put the
        /// return code in 'ret' and any extra returned values in args[].
        /// The possible hypercalls are defined in the Power Architecture Platform
        /// Requirements (PAPR) document available from www.power.org (free
        /// developer registration required to access it).
		struct {
			__u64 nr;
			__u64 ret;
			__u64 args[9];
		} papr_hcall;

		/* KVM_EXIT_S390_TSCH */
        /// s390 specific. This exit occurs when KVM_CAP_S390_CSS_SUPPORT has been enabled
        /// and TEST SUBCHANNEL was intercepted. If dequeued is set, a pending I/O
        /// interrupt for the target subchannel has been dequeued and subchannel_id,
        /// subchannel_nr, io_int_parm and io_int_word contain the parameters for that
        /// interrupt. ipb is needed for instruction parameter decoding.
		struct {
			__u16 subchannel_id;
			__u16 subchannel_nr;
			__u32 io_int_parm;
			__u32 io_int_word;
			__u32 ipb;
			__u8 dequeued;
		} s390_tsch;

		/* KVM_EXIT_EPR */
        /// On FSL BookE PowerPC chips, the interrupt controller has a fast patch
        /// interrupt acknowledge path to the core. When the core successfully
        /// delivers an interrupt, it automatically populates the EPR register with
        /// the interrupt vector number and acknowledges the interrupt inside
        /// the interrupt controller.

        /// In case the interrupt controller lives in user space, we need to do
        /// the interrupt acknowledge cycle through it to fetch the next to be
        /// delivered interrupt vector using this exit.

        /// It gets triggered whenever both KVM_CAP_PPC_EPR are enabled and an
        /// external interrupt has just been delivered into the guest. User space
        /// should put the acknowledged interrupt vector into the 'epr' field.
		struct {
			__u32 epr;
		} epr;

		/* KVM_EXIT_SYSTEM_EVENT */
        /// If exit_reason is KVM_EXIT_SYSTEM_EVENT then the vcpu has triggered
        /// a system-level event using some architecture specific mechanism (hypercall
        /// or some special instruction). In case of ARM/ARM64, this is triggered using
        /// HVC instruction based PSCI call from the vcpu. The 'type' field describes
        /// the system-level event type. The 'flags' field describes architecture
        /// specific flags for the system-level event.

        /// Valid values for 'type' are:
        /// KVM_SYSTEM_EVENT_SHUTDOWN -- the guest has requested a shutdown of the
        /// VM. Userspace is not obliged to honour this, and if it does honour
        /// this does not need to destroy the VM synchronously (ie it may call
        /// KVM_RUN again before shutdown finally occurs).
        /// KVM_SYSTEM_EVENT_RESET -- the guest has requested a reset of the VM.
        /// As with SHUTDOWN, userspace can choose to ignore the request, or
        /// to schedule the reset to occur in the future and may call KVM_RUN again.
        /// KVM_SYSTEM_EVENT_CRASH -- the guest crash occurred and the guest
        /// has requested a crash condition maintenance. Userspace can choose
        /// to ignore the request, or to gather VM memory core dump and/or
        /// reset/shutdown of the VM.
		struct {
#define KVM_SYSTEM_EVENT_SHUTDOWN       1
#define KVM_SYSTEM_EVENT_RESET          2
#define KVM_SYSTEM_EVENT_CRASH          3
			__u32 type;
			__u64 flags;
		} system_event;

		/* KVM_EXIT_IOAPIC_EOI */
        /// Indicates that the VCPU's in-kernel local APIC received an EOI for a
        /// level-triggered IOAPIC interrupt.  This exit only triggers when the
        /// IOAPIC is implemented in userspace (i.e. KVM_CAP_SPLIT_IRQCHIP is enabled);
        /// the userspace IOAPIC should process the EOI and retrigger the interrupt if
        /// it is still asserted.  Vector is the LAPIC interrupt vector for which the
        /// EOI was received.
		struct {
			__u8 vector;
		} eoi;

		struct kvm_hyperv_exit {
#define KVM_EXIT_HYPERV_SYNIC          1
#define KVM_EXIT_HYPERV_HCALL          2
			__u32 type;
			union {
				struct {
					__u32 msr;
					__u64 control;
					__u64 evt_page;
					__u64 msg_page;
				} synic;
				struct {
					__u64 input;
					__u64 result;
					__u64 params[2];
				} hcall;
			} u;
		};
		/* KVM_EXIT_HYPERV */
        /// Indicates that the VCPU exits into userspace to process some tasks
        /// related to Hyper-V emulation.
        /// Valid values for 'type' are:
        ///     KVM_EXIT_HYPERV_SYNIC -- synchronously notify user-space about
        /// Hyper-V SynIC state change. Notification is used to remap SynIC
        /// event/message pages and to enable/disable SynIC messages/events processing
        /// in userspace.
        struct kvm_hyperv_exit hyperv;

		/* Fix the size of the union. */
		char padding[256];
	};

	/*
	 * shared registers between kvm and userspace.
	 * kvm_valid_regs specifies the register classes set by the host
	 * kvm_dirty_regs specified the register classes dirtied by userspace
	 * struct kvm_sync_regs is architecture specific, as well as the
	 * bits for kvm_valid_regs and kvm_dirty_regs
	 */
	__u64 kvm_valid_regs;
	__u64 kvm_dirty_regs;
	union {
		struct kvm_sync_regs regs;
		char padding[SYNC_REGS_SIZE_BYTES];
	} s;

    /// If KVM_CAP_SYNC_REGS is defined, these fields allow userspace to access
    /// certain guest registers without having to call SET/GET_*REGS. Thus we can
    /// avoid some system call overhead if userspace has to handle the exit.
    /// Userspace can query the validity of the structure by checking
    /// kvm_valid_regs for specific bits. These bits are architecture specific
    /// and usually define the validity of a groups of registers. (e.g. one bit
    /// for general purpose registers)

    /// Please note that the kernel is allowed to use the kvm_run structure as the
    /// primary storage for certain register types. Therefore, the kernel may use the
    /// values in kvm_run even if the corresponding bit in kvm_dirty_regs is not set.
};
```

## KVM CPUID

A guest running on a kvm host, can check some of its features using `cpuid`. This is not always guaranteed to work, since userspace can mask-out some, or even all KVM-related `cpuid` features before launching a guest.