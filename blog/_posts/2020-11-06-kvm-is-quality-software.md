---
layout: post
title: "KVM is Quality Software... lol?"
date: 2020-11-06
---

# Prelude

This article is a bitchy, prolonged rant as the result of weeks of banging my head against tools failing to integrate and code refusing to work as expected; this is not a well written criticism of KVM. KVM (Kernel-based Vritual Machine) is a hypervisor built into the Linux Kernel which can take advantage of hardware assisted virtualisation technologies (including VT-x). The implementation of KVM can be found in the [Linux kernel tree](https://elixir.bootlin.com/linux/latest/source/arch/x86/kvm) under the x86 branch. During a project which involved implementing my own hypervisor, I had to use a hypervisor which implemented nested VT-x. Nested VT-x is a feature for including nested hypervisors within an existing hypervisor (emulate the emulator!) which, as you can imagine, is exceedingly useful for debugging my own hypervisor. In the spirit of FOSS, I chose to use KVM with [QEMU](https://www.qemu.org/) since KVM touts [Nested Guest support](https://www.linux-kvm.org/page/Nested_Guests). What ensued was a nightmare of pain and suffering which has soured my opinions on KVM for the foreseeable future for any kind of low level development.

Now, before you ask, I understand that the best way to develop a low level project like a hypervisor is to have a baremetal host then use serial debugging with [kdb/kgdb](https://www.kernel.org/doc/html/v4.18/dev-tools/kgdb.html#using-kgdb-gdb). However, I didn't have another viable physical host to do remote debugging. If a product is advertised to have support for a specific feature, I expect that feature to work. Although, I will admit my use-case was unorthodox so I will be lenient with issues I believe were the result of my use-case (namely, virtualising an existing system rather than booting a virtualised guest).

## TL;DR

Virtual Box and KVM both suck but VMware is acceptable (though at time of writing it has been consistently broken since Linux Kernel version 5.8). VMware has the best hypervisor implementation I've experienced though it is closed source (this shouldn't matter unless you're an FSF zealot). KVM has awkward nested guest support with several unexpected behaviours which required significant time to debug and diagnose. Some of the bugs were the result of my own misunderstanding and broken assumptions, others are clear bugs.

# Intel VT-x

## Debugging a Type-2 Hypervisor

Initially, I was using [`libvirt`](https://wiki.archlinux.org/index.php/Libvirt) to produce virtual machines using KVM acceleration with a few additions to the XML config for remote debugging (`-s`) and nested VT-x (`-nested-kvm`) support. I had a simple script which would boot the VM with a shared folder containing a fresh build of my hypervisor (it was a [LKM](https://wiki.archlinux.org/index.php/Kernel_module)) then would load the module. My hypervisor consisted of several stages to begin execution, one stage was initialising a CPU structure named the [Virtual Machine Control Structure (VMCS)](https://www.intel.com/content/dam/www/public/us/en/documents/manuals/64-ia-32-architectures-software-developer-vol-3c-part-3-manual.pdf). The initialisation of the VMCS requires setting specific fields to certain values which affect the execution of the guest, any improper values will cause the VM to fail when `vmlaunch`'d with an _unbelievably_ vague indicator so any unexpected behaviour from KVM during this process is significant.

### No XSAVE support

This is a behaviour that only affected my project because I was attacking an existing system. However, it was probably the hardest bug I've ever had the displeasure of finding a workaround for. To provide context, `XSAVE` is an extension to x86 for saving processor state to a memory region. For a guest to use `XSAVE` instructions, the "enable XSAVEs" flag in `VMCS_CTRL_SECONDARY_PROCESSOR_BASED_VM_EXECUTION_CONTROLS` must be set. However, certain fields are fixed depending on the `IA32_VMX_PROCBASED_CTLS2` MSR (hardware may not support a certain extension). I was using the below code to enable specific execution properties of the guest.

```c
static ssize_t setup_secondary_procbased_controls(struct cpu_ctx* cpu) {
    // fixed execution controls
    IA32_VMX_TRUE_CTLS_REGISTER policy = {
        .flags = native_read_msr(IA32_VMX_PROCBASED_CTLS2)};
    IA32_VMX_PROCBASED_CTLS2_REGISTER ctls = {.flags = 0};

    // enabled execution features
    ctls.enable_rdtscp = 1;
    ctls.enable_invpcid = 1;
    ctls.conceal_vmx_from_pt = 1;
    ctls.enable_xsaves = 1;

    // fix controls based on system policy
    ctls.flags &= policy.allowed_1_settings;
    ctls.flags |= policy.allowed_0_settings;

    pr_debug("XSAVE enabled=%d (policy=%x)\n", ctls.enable_xsaves, policy.flags);

    ...
}
```

Within the KVM VM this outputs:

> XSAVE enabled=0 (policy=378ef00000000)

whereas on my host machine it outputs:

> XSAVE enabled=1 (policy=1f7cff00000000)

The KVM guest supports `XSAVE` but the execution control for the nested guest is explicitly disabled! I know, this _can_ happen but I didn't expect it since the virtual CPU _does_ support `XSAVE`. I did not have a convenient debug print the first time I wrote this because all controls are fixed against the system policy so I assumed all controls were valid. If "enable XSAVEs" is disabled within a guest and the guest attempts to execute `XSAVE` (or related functions) then an invalid opcode (`#UD`) exception is raised. If you are unfamiliar with error handling within Linux Kernel it goes like this:
- `#UD` generated
- die.

You are greeted by a kernel panic as the entire kernel locks up.

<p align="center">
<img alt="XSAVE invalid opcode kernel panic" src="/blog/assets/images/xsave_ud.jpg">
</p>

### Debugging

The guest would launch successfully but then immediately trigger a `#UD` with `rip` pointing to `__kmalloc` and `smp_reboot_interrupt`. Disassembling the code listed in the panic gave nothing interesting.

Code dump for `__kmalloc`:
```nasm
0:  47 20 49 8b             rex.RXB and BYTE PTR [r9-0x75],r9b
4:  3f                      (bad)
5:  4c 01 e0                add    rax,r12
8:  48 8b 18                mov    rbx,QWORD PTR [rax]
b:  48 89 c1                mov    rcx,rax
e:  49 33 9f 70 01 00 00    xor    rbx,QWORD PTR [r15+0x170]
...
```

Code dump for `smp_reboot_interrupt`:
```nasm
0:  21 31                   and    DWORD PTR [rcx],esi
2:  ff                      (bad)
3:  e8 43 9e fd ff          call   0xfffffffffffd9e4b
8:  48 8b 44 24 24          mov    rax,QWORD PTR [rsp+0x24]
d:  18 65 40                sbb    BYTE PTR [rbp+0x40],ah
10: 2b 04 25 28 00 00 00    sub    eax,DWORD PTR ds:0x28
17: 00                      .byte 0x0
18: 75                      .byte 0x75
...
```

The call trace didn't give anything particularly exciting so I tried to trace execution by writing to the kernel buffer. However, the exception was non-deterministic (from a single-threaded point of view) since repeated executions would hit a different number of writes. This meant my code was affecting something external which later caused an exception in the SMP section of the kernel, there are **many** possibilities. Memory corruption was the most likely candidate so breakpoints were set at points of interest within my code to view the state of the kernel thread. However, the state of the thread was fine at each breakpoint (though the number of breakpoints hit was varied based on spontaneous `#UD`s). Next, the hypervisor operation post launch was suspected (I'll spare the details here).

After several days of debugging, I began to read the kernel SMP code because none of my code appeared to be incorrect. I recalled that areas of the kernel have routines (e.g. [FPU](https://elixir.bootlin.com/linux/v5.9.1/source/arch/x86/kernel/fpu/core.c#L97), or [signal](https://elixir.bootlin.com/linux/v5.9.1/source/arch/x86/kernel/fpu/signal.c#L131)) which save/restore processor state using `XSAVE` if it is available. Additionally, you can see the kernel probing `XSAVE` during early stages of boot.

```
...
[    0.000000] KERNEL supported cpus:
[    0.000000]   Intel GenuineIntel
[    0.000000]   AMD AuthenticAMD
[    0.000000]   Hygon HygonGenuine
[    0.000000]   Centaur CentaurHauls
[    0.000000]   zhaoxin   Shanghai
[    0.000000] x86/fpu: x87 FPU will use FXSAVE
...
```

To verify all instructions were consistent with the state of the system prior to virtualisation I inspected all of the VM execution controls. It was discovered that the values of `VMCS_CTRL_SECONDARY_PROCESSOR_BASED_VM_EXECUTION_CONTROLS` was different between the guest and the host system - which is to be expected since this is a virtualised guest without full hardware support. However, the VMCS execution controls disabled `XSAVE` for the running environment which had already opted to use `XSAVE` because KVM (and VirtualBox) explicitly disables "enable XSAVE" in the `IA32_VMX_PROCBASED_CTLS2` MSR. Consequently, when a preemptive context switch occured, the kernel then attempted to execute an "invalid" `XSAVE` instruction.

The workaround for this was to include:

```xml
<cpu mode="host-model">
    <feature name="xsave" policy="disable"/>
</cpu>
```

To the `libvirt-manager` VM config so the virtualised guest was unable to use `XSAVE` instructions.

### Additional Details

The "enable XSAVE" wasn't the only control modified by KVM, additional features changed include (no features were falsely enabled):
- Virtualize x2APIC mode
- APIC register virtualization
- Virtual-interrupt delivery
- PAUSE-Loop Exiting
- `ENCLS` VM-exits
- EPT-violations
- Conceal VMX non-root from PT
- Enable `XSAVE/XRSTOR`

I do not believe this is exhaustive so lookout for discrepencies.

### Physical Address Error

During VMCS initialisation I had self-inflicted pain by loading improper into certain fields. To identify what improper values had been loaded, I implemented the same consistency checks as the CPU which validates the state of the VMCS against a wide range of conditions. One of the checks is to verify all physical addresses are within the physical-address width, to get the physical-address width you query `CPUID.80000008H:EAX[7:0]`. The reported value of my host is 39 bits for a maximum physical address width whereas the guest has a value of 40, I believe this may be an off-by-one error.

## Debugging a Type-1 Hypervisor

My type-1 hypervisor requires a different setup since I am building a PE image then loading it as an [EFI DXE driver](https://edk2-docs.gitbook.io/edk-ii-module-writer-s-guide/8_dxe_drivers_non-uefi_drivers/88_dxe_runtime_driver). I have a script which builds my module within [EDK2](https://github.com/tianocore/edk2) then spawns a VM using QEMU with KVM acceleration. A shortened snippet of the script is listed below.

```sh
#!/bin/sh

# ignore
# ...
# build
# ...
# stuff

WORKSPACE=testing
cp ../edk2/Build/SimpleHv/DEBUG_GCC5/X64/SimpleHvDxe.efi $WORKSPACE/virtual_fat/SimpleHvDxe.efi

# KVM has a tendency to lock up the host which will cause recent file changes
# to improperly/fail to flush to disk.
sync

# XSAVE extensions are disabled because nested KVM does not properly provide the
# fixed VMX execution control MSRs (due to missing implementation of nested
# XSAVE).

qemu-system-x86_64 \
    -machine q35,accel=kvm -m 1G \
    -cpu qemu64,+vmx,-xsave -smp cores=8,threads=1 \
    -s \
    -enable-kvm \
    -hda $WORKSPACE/arch.qcow2 \
    -drive if=pflash,unit=0,format=raw,readonly,file=$WORKSPACE/OVMF_CODE.fd \
    -drive if=pflash,unit=1,format=raw,file=$WORKSPACE/OVMF_VARS.fd \
    -drive file=fat:rw:$WORKSPACE/virtual_fat/
```

Setting up a baremetal hypervisor is pretty similar to a type-2 hypervisor but requires more legwork since there aren't many facilites available before `EVT_SIGNAL_EXIT_BOOT_SERVICES` is raised (i.e. until `ExitBootServices` is called).

### SMP

For a core to be virtualised, it must setup its own unique CPU structures then execute `vmxon`. Therefore, to virtualise the entire processor we have to execute a startup routine for each core for which there is an [`EFI_MP_SERVICES_PROTOCOL`](https://uefi.org/sites/default/files/resources/PI_Spec_1_6.pdf) (section 13.4) protocol available. The code to use these services looks like:

```c
VOID
EFIAPI
HvEnvOnEachProcessor(
    VOID *Data,
    EFI_AP_PROCEDURE Procedure,
    EFI_EVENT WaitEvent)
{
    /* Execute callback on BSP. */
    Procedure(Data);
    /* Single-threaded because locking mechanisms are not present for all
     * protocols which causes issues. */
    gMpServiceProtocol->StartupAllAPs(gMpServiceProtocol, Procedure,
            TRUE, WaitEvent, 0, Data, NULL);
}

VOID
Foo(
    VOID)
{
    VOID *gVmmCtx = ...;
    /* Descriptions:     data     AP entry point  blocking */
    HvEnvOnEachProcessor(gVmmCtx, HvCpuInitEntry, NULL);
    ...
}
```

The entry `HvCpuInitEntry` is an assembly stub:

```nasm
.text
ASM_PFX(HvCpuInitEntry):
	pushfq
	STORE_CPU_STATE
	movabs $.VmxGuestResume, %rdx
	mov %rsp, %r8
	mov 0x80(%rsp), %r9
        sub $0x20, %rsp
        // %rcx	: VMM_CONTEXT*
        // %rdx	: Guest resume IP
        // %r8	: Guest resume SP
        // %9	: Guest resume EFLAGS
        call HvCpuInit
        add $0x20, %rsp
.VmxGuestResume:
	RESTORE_CPU_STATE
	popfq
	ret
```

Simple, right? No. I have no idea why but KVM bricked itself depending on the number of APs available. If I attempted to execute QEMU with `-cores=3` instead of `-cores=2`, KVM would hang indefinitely. The APs were enabled then launched to an invalid address (the addess was noncanonical and outside the physical-address width) which caused the APs to hang. What's worse than 3 cores? 4! If we change `-cores=3` to `-cores=4`:

```
KVM internal error. Suberror: 1
emulation failure
RAX=0000000000000008 RBX=000000003f279248 RCX=000000003df75a17 RDX=0000000000000002
RSI=0000000000000000 RDI=0000000000000000 RBP=000000003f278008 RSP=000000003ff35000
R8 =000000003f2797a8 R9 =000000003fac644e R10=000000003df75a18 R11=0000000000000000
R12=0000000000000000 R13=0000000000000000 R14=0000000000000000 R15=000000003ff34f78
RIP=00000000000b0000 RFL=00010096 [--S-AP-] CPL=0 II=0 A20=1 SMM=0 HLT=0
ES =0030 0000000000000000 ffffffff 00c09300 DPL=0 DS   [-WA]
CS =0038 0000000000000000 ffffffff 00a09b00 DPL=0 CS64 [-RA]
SS =0030 0000000000000000 ffffffff 00c09300 DPL=0 DS   [-WA]
DS =0030 0000000000000000 ffffffff 00c09300 DPL=0 DS   [-WA]
FS =0030 0000000000000000 ffffffff 00c09300 DPL=0 DS   [-WA]
GS =0030 0000000000000000 ffffffff 00c09300 DPL=0 DS   [-WA]
LDT=0000 0000000000000000 0000ffff 00008200 DPL=0 LDT
TR =0000 0000000000000000 0000ffff 00008b00 DPL=0 TSS64-busy
GDT=     000000003f9ee698 00000047
IDT=     000000003f278248 00000fff
CR0=80010033 CR2=0000000000000000 CR3=000000003fc01000 CR4=00000660
DR0=0000000000000000 DR1=0000000000000000 DR2=0000000000000000 DR3=0000000000000000
DR6=00000000ffff0ff0 DR7=0000000000000400
EFER=0000000000000d00
Code=00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 <ff> ff ff ff ff ff ff ff ff ff ff ff ff ff ff ff ff ff ff ff ff ff ff ff ff ff ff ff ff ff
```

A definite, reportable, KVM bug. What caused it? No idea, I spent way too long trying to track it down but suddenly the bug disappeared (I think it was something to do with GCC mixing the [MS ABI](https://docs.microsoft.com/en-us/cpp/build/x64-calling-convention?) and [System V ABI](https://www.uclibc.org/docs/psABI-x86_64.pdf) which could be a bug in GCC itself).

# AMD SVM

These bugs were discovered and written up by [Máté Kukri](https://mkukri.xyz/) (check out his website). The AMD Secure Virtual Machine (SVM) extensions allows for the creation of encrypted VMs (encrypted virtualised guest). AMD SVMs have a different set of CPU structures, fields, but the only relevant change in terminology is that AMD SVM uses a VMCB (Virtual Machine Control Block) to set affect the execution of the guest as opposed to the VMCS used by Intel VMX.

## Nonspecific Behaviours

### Missing `#SX`

How could KVM possibly make SMP startup more "fun" to virtualize on SVM with no ability to catch startup IPIs at all? The answer is to not implement the half-broken mechanism SVM provides to catch INIT IPIs. According to the AMD manual INITs can be intercepted as follows:

> In the current implementation, the only use of the `#SX` is to redirect external INITs into an exception so that the VMM may — among other possibilities — destroy sensitive information before re-issuing the INIT, this time without redirection. The INIT redirection is controlled by the `VM_CR.R_INIT` bit.

So if the `VM_CR.R_INIT` bit is set and exception interception is enabled for `#SX` we should get a VMEXIT when an AP (Application Processor) recieves an INIT signal, right? The answer is yes on actual AMD hardware and VMWare, but not KVM. The latter of the 3 happily ignores the interception flag and just puts the AP into the "Waiting-For-SIPI" state.

So how do we fix this, how do we cause a `#VMEXIT` on an AP when it would have recieved an INIT? The answer is the MMIO emulator we painfully had to write anyways to catch startup IPIs. So let me rephrase the question, how do we make a piece of code (the MMIO emulator) running on the BSP (Bootstrap Processor) make a guest mode AP exit to *its own* VMM?

Well let me present the "finest" workaround I have ever designed:
1. Intercept the BSP's access to the LAPIC, catching all INIT signals.
2. Meanwhile make the APs wait in some hypervisor provided fake guest code
 that is polling a flag in a busy loop.
3. When the VMM on the BSP catches an INIT that the guest was trying to send instead of sending an actual INIT, the BSP sets the flag the APs are polling to the AP number.
4. The fake guest code on the target AP realizes the flag was set to its AP number, it does a VMMCALL to its own VMM which emulates the same behaviour as the `#SX` exception would have provided.

### No Nested Hugepages For You

AMD SVM provides a mechanism that helps hypervisors implement memory protection between host and guest called nested paging. It works by putting another set of page tables between the guest operating system's page tables and physical memory.

These "new" page tables are identical to the normal AMD64 page tables with the only difference being that user mode accessible flag always needs to be set.  When a virtual address from the guest operating system needs to be resolved the processor does a page walk with twice the depth, walking both the guest operating system's and nested page tables. So in theorey these nested page table should support all 3 page sizes supported by AMD64, right? Well, not so fast, on an actual AMD CPU they do indeed work, but KVM is not so friendly. The following code creates an identity map using 2MiB pages:

```c
cur_phys = 0;
pml4 = alloc_pages(1, NULL);
pdp = alloc_pages(1, NULL);

for (pdp_idx = 0; pdp_idx < 512; ++pdp_idx) {
	pd = alloc_pages(1, NULL);
	for (pd_idx = 0; pd_idx < 512; ++pd_idx) {
	    pd[pd_idx] = cur_phys | 0x87;
	    cur_phys += 0x200000;
	}
	pdp[pdp_idx] = (uint64_t) pd | 7;
}
pml4[0] = (uint64_t) pdp | 7;
```

KVM is happy with that, and the guest operating system boots fine with this as its nested page table. But then comes the 1GiB page version mapping the exact same region:

```c
cur_phys = 0;
pml4 = alloc_pages(1, NULL);
pdp = alloc_pages(1, NULL);

for (pdp_idx = 0; pdp_idx < 512; ++pdp_idx) {
pdp[pdp_idx] = (uint64_t) cur_phys | 0x87;
	cur_phys += 0x40000000;
}
pml4[0] = (uint64_t) pdp | 7;
```

and KVM just mysteriously locks up when the `vmrun` instruction is executed with no error message or any kind of crash. Sadly I haven't managed to find the root cause of this issue.

# Conclusion?

I could easily expand this to include rants about the working Linux Kernel tree (it has a _lot_ of duplicated APIs and inconsistent definitions) or GDB (it's on HP printer levels of consistently inconsistent). However, KVM caused the most difficulty for me because it affected my project at the lowest levels. Don't take this too seriously, KVM is fine but the nested guest support is untrustworthy.
