# Experimental AMD KVM RDTSC handler for Linux 7.3.3

This is an AMD-only port of [WCharacter/RDTSC-KVM-Handler](https://github.com/WCharacter/RDTSC-KVM-Handler) to upstream Linux **7.3.3**. Intel VMX is not supported by this port. The old files under `Linux kernel/` are retained for historical reference and must not be copied into Linux 7.3.3; use the patch instead.

**Experimental:** the patch compiles, but has not been booted or tested in a guest. The handler scales elapsed time by 1/20, as in the original AMD implementation. Per-vCPU clocks can diverge and disagree with KVM's paravirtual clock, TSC MSR reads, timers, and other time sources. Guest clock jumps, hangs, and data corruption are possible. Use disposable test guests and keep a known-working kernel available. Virtualization concealment has not been verified.

The experiment is disabled by default. Enabling it affects non-SEV-ES AMD KVM guests on the host, not just one selected guest. SEV-ES/SNP, nested guests, live migration, save/restore, and guest TSC adjustments are not supported or verified. Avoid using these features with the experiment enabled. RDTSC is intercepted; RDTSCP is not, so the interface is inconsistent when RDTSCP is exposed.

## Apply and build

1. Download and extract [Linux 7.3.3](https://cdn.kernel.org/pub/linux/kernel/v7.x/linux-7.3.3.tar.xz). Its SHA-256 is `79c66dde66cd96877fcde8f7d36aec946b1f423d2b97b5a7b1751b75c764f3d1`. Distribution kernels may have additional changes and need a separate port.
2. From the kernel source directory, check and apply the patch, replacing `REPO` with this repository's absolute path:

   ```sh
   patch -p1 --dry-run < REPO/patches/linux-7.3.3-amd-experimental.patch
   patch -p1 < REPO/patches/linux-7.3.3-amd-experimental.patch
   ```

3. Configure and build using your distribution's kernel instructions. For a compile-only check with the default configuration:

   ```sh
   make defconfig
   scripts/config -e VIRTUALIZATION -e KVM -m KVM_AMD
   make olddefconfig
   make -j"$(nproc)" arch/x86/kvm/kvm-amd.o
   ```

   This checks the affected object; it does not create an installable kernel or module. A full build and installation require the normal kernel build dependencies and distribution-specific bootloader steps. Keep a recovery kernel available.

4. After building, installing, and booting the matching patched kernel, opt in with the kernel command-line parameter `kvm_amd.rdtsc_experiment=1`. For a modular `kvm_amd`, a modprobe configuration can instead contain `options kvm_amd rdtsc_experiment=1`; update the initramfs where necessary. Verify the value at `/sys/module/kvm_amd/parameters/rdtsc_experiment`. Set it to `0` at the next load or boot to disable the experiment; the parameter is read-only after loading.
5. For a test guest, do not expose RDTSCP (for example, use QEMU's `-cpu host,-rdtscp` where appropriate), and disable nested virtualization. Neither setting makes the resulting clock internally consistent.

## Implementation and verification

The SVM intercept is enabled only when the module parameter is set and the guest is not SEV-ES. On each intercepted instruction, the handler reads KVM's L1 guest TSC, advances a per-vCPU value by one twentieth of the elapsed TSC ticks, and returns it in EDX:EAX. State is reset on vCPU reset. CR4.TSD privilege restrictions are checked before emulation.

The patch applies cleanly to the upstream 7.3.3 files. The `arch/x86/kvm/kvm-amd.o` target built successfully using the default x86-64 configuration with KVM AMD enabled as a module. No full kernel, boot, guest, multi-vCPU, timing, migration, nested virtualization, or SEV test has been performed.

## Building a package on Arch-based systems

This additional route has not been tested for this patch. From the extracted, patched Linux 7.3.3 directory, a machine with `/proc/config.gz` can start with its running configuration:

```sh
zcat /proc/config.gz > .config
scripts/config --set-str LOCALVERSION "-rdtsc-amd" --disable LOCALVERSION_AUTO
make olddefconfig
PACMAN_EXTRAPACKAGES=headers make -j"$(nproc)" pacman-pkg
```

Install the build dependencies listed in `scripts/package/PKGBUILD` and `base-devel` first. Inspect the resulting kernel and headers packages before installing them with `pacman -U`. The running kernel configuration may contain distribution-specific options or external modules that upstream Linux does not provide; review the configuration and build output. The unique local version prevents replacing the existing kernel's modules. Kernel packaging, bootloader integration, initramfs generation, third-party modules, and Secure Boot signing depend on the machine and have not been verified here. Keep the existing kernel and confirm a recovery boot entry before trying the new one. See the [Arch traditional kernel compilation guide](https://wiki.archlinux.org/title/Kernel/Traditional_compilation).
