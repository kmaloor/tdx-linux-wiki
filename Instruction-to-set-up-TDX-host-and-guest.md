# Introduction
This wiki is your comprehensive guide to building and configuring a TDX host Linux kernel, and then booting a TD VM. Whether you're a developer, an enthusiast, or just curious about how to work with the source code, this resource is designed to provide you with clear and detailed instructions to help you navigate the process effectively.

# Bill of materials
For the version information of various components like Linux kernel, QEMU, OVMF, etc. please refer to each release's notes.

# Supported hardware
4th Generation Intel® Xeon® Scalable Processors or newer with Intel® TDX.

# Setup TDX host
## Distro information
Most information are distro-agnostic and when some distro specific command is used like apt, you will need to change it accordingly. These build instructions are tested on Ubuntu 23.04 and Fedora 39.

## Build a TDX host
Please apply the tarball from this repo on top of the related kernel version (see the NEWS file for kernel version).

As for kernel configs: make sure your base config is usable on your test machine. And set the following configs to Y
```
CONFIG_X86_INTEL_TSX_MODE_ON=y
CONFIG_X86_X2APIC=y
CONFIG_IRQ_REMAP=y
CONFIG_EXPERT=y
CONFIG_KVM_SW_PROTECTED_VM=y
CONFIG_TDX_HOST=y
CONFIG_TDX_GUEST=y
CONFIG_VIRTIO_BLK=y
CONFIG_VIRTIO_NET=y
CONFIG_ZRAM=y
```

Set the following configs to N
```
CONFIG_EISA=n
CONFIG_KSM=n
CONFIG_HYPERV=n
```

## Host kernel cmdline
Make sure the following two options are in host kernel cmdline: "kvm_intel.tdx=on nohibernate".

## Setup TDX host in BIOS
Note: The following is a sample setting, different vendors may have different settings.
Go to Socket Configuration > Processor Configuration > TME, TME-MT, TDX.
```
Memory Encryption (TME): Enabled
Total Memory Encryption (TME) Bypass: AUTO
Total Memory Encryption Multi-Tenant (TME-MT): Enabled
memory integrity: Disabled
Trust Domain Extension (TDX): Enabled
TDX Secure Arbitration Mode Loader (SEAM Loader): Enabled
Disable excluding Mem below 1MB in CMR: AUTO
TME-MT/TDX key split: 1
SW Guard Extensions (SGX): Enabled
```

## Check TDX host kernel runs correctly
```
$ dmesg |grep tdx
... ...
[   52.900653] virt/tdx: module initialized
```
If you see "virt/tdx: module initialized", that means the host side enable is going well.

# Setup a TDX guest
A special version of TDVF will be needed. The TDVF is used as the firmware of the TD guest.
The TDX-enabled Qemu is needed to boot a TD guest VM. 

The guest image can be created as usual or use the existing one, with TDX-enabled kernel.

## Build TDVF image
* Below are example steps to build TDVF image on Ubuntu based system. Commands will be slightly different if different distro OS is used. The branch information is just an example, please refer to the NEWS file to use the correct branch/tag.
* The resulting ovmf binary is at: Build/IntelTdx/DEBUG_GCC5/FV/OVMF.fd
```
$ sudo apt build-dep ovmf
$ git clone --branch edk2-stable202302 https://github.com/tianocore/edk2.git
$ cd edk2
$ git submodule update --init
$ make -C BaseTools
$ source ./edksetup.sh
$ build -p OvmfPkg/IntelTdx/IntelTdxX64.dsc -a X64 -t GCC5 -D SECURE_BOOT_ENABLE=TRUE
```

## Build TDX enabled Guest kernel
The same host kernel can be used as the guest kernel.

# Boot TD
## Build TDX enabled QEMU
The branch information is just an example, please refer to the NEWS file to use the correct branch/tag.
```
$ git clone --branch tdx-qemu-next-2023.9.21-v8.1.0 https://github.com/intel/qemu-tdx
$ sudo apt build-dep qemu
$ mkdir build; cd build
$ ../configure --target-list="x86_64-softmmu" --enable-kvm --prefix=XXX
$ make & make install
```

## Qemu cmdline
Below is a reference Qemu cmdline, you may want to change settings like number of CPUs, memory size, etc.
Also, XXX needs be replaced accordingly.
```
qemu-system-x86_64                  \
    -enable-kvm                     \
    -drive file=XXX.qcow2,if=virtio \
    -smp cpus=4,cores=2,threads=2   \
    -m 4G                           \
    -cpu host                       \
    -netdev user,id=n1,ipv6=off,hostfwd=tcp::2022-:22      \
    -device virtio-net-pci,netdev=n1,mac=XX:XX:XX:XX:XX:XX \
    -nographic                      \
    -kernel /boot/vmlinuz-`uname -r`\
    -append "root=XXX console=ttyS0"\
    -object tdx-guest,id=tdx        \
    -object memory-backend-ram,id=mem0,size=4G,private=on  \
    -machine q35,kernel-irqchip=split,confidential-guest-support=tdx,memory-backend=mem0 \
    -bios OVMF.fd                   \
    -vga none                       \
    -nodefaults                     \
    -serial stdio
```

## Verify TD guest is running well
```
$ dmesg | grep tdx
... ...
[    0.000000] tdx: Guest detected
```
If you see above messages, that means TD is running correctly.

# Test
Some sanity tests are done. Refer to https://github.com/intel/tdx/wiki/Tests for details of the sanity tests. 