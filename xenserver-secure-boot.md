XenServer Secure Boot
=====================

Overview
--------

XenServer is a virtualization platform based around Xen, a type-1 hypervisor.
The system is controlled by Xen in conjunction with a privileged domain, dom0.
For the purposes of Secure Boot, both Xen and the dom0 Linux kernel need to be
verified and prevent execution of unauthenticated code.

The boot process is a little more complicated than a regular Linux distro:

    UEFI firmware -> Shim -> GRUB -> Xen -> Linux (dom0)

Rather than booting Linux, GRUB boots Xen instead, passing it the dom0 kernel
and an initrd. Xen creates domain 0 and starts running the kernel inside it.
Xen and dom0 share control of the hardware.

The diagram below shows a high-level overview of the system at runtime. The
components highlighted in red are privileged and cannot run unauthenticated code
when Secure Boot is enabled.

![System Overview](./overview.png)

Shim
----

Shim does not require any modifications to be used with XenServer aside from a
bugfix. See [README.md](./README.md) for details of the patches applied to
Shim.

GRUB
----

The `xen_boot` loader is used to boot Xen. Xen is an EFI binary and the
`xen_boot` loader loads it by extending the Linux EFI loader in GRUB, reusing
much of the code, including verification. The main difference is that instead
of GRUB advertising an initrd using the LoadFile2 protocol on a well-known
GUID, GRUB advertises a kernel, an initrd, and a config file using the Simple
Filesystem Protocol.

Although an ARM64 version of the `xen_boot` loader exists upstream, the x86_64
implementation used here is currently a downstream patch, though we do intend
to upstream it.

While the `multiboot2` loader is included in the GRUB build, it cannot be used
to boot Xen (or anything else) with Secure Boot enabled. Therefore it is not a
security vulnerability.

Xen
---

A lockdown mode has been added to Xen for the purpose of protecting Secure Boot.
Lockdown mode is always enabled when Secure Boot is enabled. Lockdown mode
requires that Xen verifies the dom0 kernel using Shim before executing it.

Certain Xen command-line arguments may be unsafe for Secure Boot.
Lockdown mode implements an allowlist mechanism to restrict non-essential and
potentially unsafe command-line arguments when Secure Boot is enabled. The
arguments we do allow have been audited and are deemed safe.

Other ways that unauthenticated code may be executed have been restricted when
lockdown mode is enabled:

* Xen livepatches are signed and the signature is verified by Xen using a
  built-in key before they can be loaded.

* Kexec image signatures are checked before they are loaded to prevent
  execution of unauthenticated code. This is similar to how Linux verifies
  kexec images when Secure Boot is enabled.

* PCI passthrough is restricted in various ways to ensure that a malicious VM
  with a passthrough device cannot use the device to attack Xen or the dom0
  kernel.

Linux
-----

For the purposes of Secure Boot, the dom0 kernel is secured much like a
standard distro kernel:

* The dom0 kernel uses lockdown mode (separate to Xen lockdown mode) in the
  normal way to prevent executing unauthenticated code, lockdown /dev/mem, access
  to I/O ports, etc.

* Linux kernel modules are required to be signed and the signatures are verified.

* Similarly, dom0 kernel livepatches (which are also kernel modules) are
  signed and the signature is verified.

One aspect where it does differ is that dom0 userspace can make Xen hypercalls
in addition to normal system calls. Previously, a XenServer system's dom0
userspace was fully trusted and could invoke privileged Xen hypercalls
to compromise Secure Boot. This has been updated so that when lockdown
mode is enabled, dom0 userspace can only use a restricted subset of hypercalls.
An extensive audit of the entire Xen hypercall interface was performed to
determine what was safe to allow. This restriction is implemented in
[filter-hypercalls.patch](./kernel-patches/filter-hypercalls.patch).

To avoid any issues where Xen loads a dom0 kernel with a hypercall filter built
for different version of Xen (i.e. there is an ABI mismatch), the kernel
exports an ELF note (XS_ELFNOTE_PRIVCMD_FILTERING) containing the ABI version
it is built against. When loading the kernel, Xen refuses to load the dom0
kernel if the ABI version does not match Xen's current ABI version.
