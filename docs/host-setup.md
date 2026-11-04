# Host setup

The tools in this repo assume a host that already passes a GPU through to a guest. This page
covers getting to that point. It is deliberately short — the hard parts are IOMMU grouping and
driver binding, and both are hardware-specific.

## 1. Enable IOMMU

Add to the kernel command line and reboot:

```
amd_iommu=on iommu=pt        # AMD
intel_iommu=on iommu=pt      # Intel
```

Verify groups exist, and that your GPU is isolated in one:

```bash
for g in /sys/kernel/iommu_groups/*/devices/*; do
  echo "group ${g%%/devices*} -> $(basename "$g") $(lspci -nns "$(basename "$g")" | cut -d' ' -f2-)"
done | sort -V | grep -i vga
```

If the GPU shares a group with unrelated devices, you must pass all of them through or use ACS
override patches. That is outside this repo's scope.

## 2. Bind the GPU to `vfio-pci`

Find the vendor:device IDs of **both** functions — the VGA device and its HDMI audio function:

```bash
lspci -nn | grep -iE 'vga|audio.*nvidia|audio.*amd'
# 01:00.0 VGA compatible controller [0300]: ... [10de:2f18]
# 01:00.1 Audio device [0403]: ...              [10de:2f80]
```

Bind them on the **kernel command line**, which is the reliable mechanism because it takes effect
before any real driver can claim the card:

```
vfio-pci.ids=10de:2f18,10de:2f80
```

A `modprobe.d` snippet is a useful belt-and-braces addition, but should not be the only measure:

```
# /etc/modprobe.d/vfio.conf
options vfio-pci ids=10de:2f18,10de:2f80
softdep nouveau pre: vfio-pci
softdep nvidia  pre: vfio-pci
softdep nvidia_drm pre: vfio-pci
softdep nvidia_modeset pre: vfio-pci
softdep nvidia_uvm pre: vfio-pci
```

> **The softdep lines have a side effect worth knowing.** Because `nvidia` softdeps `vfio-pci`,
> running `modprobe -r nvidia` unloads **vfio-pci too**, leaving the GPU with no driver at all.
> If you ever unload the NVIDIA stack by hand, use `rmmod` on each module instead — see
> [sharing-a-gpu.md](sharing-a-gpu.md).

Verify after reboot:

```bash
lspci -nnk -d 10de: | grep -A3 VGA     # expect: Kernel driver in use: vfio-pci
```

## 3. Reserve hugepages at boot

1 GiB pages must be reserved on the command line. Runtime reservation is unreliable on a host
that has been up for any length of time — see [performance.md](performance.md#hugepages).

```
default_hugepagesz=1G hugepagesz=1G hugepages=12      # 12 GiB for a 12 GiB guest
```

```bash
grep -i huge /proc/meminfo      # HugePages_Total: 12
```

Then in the domain XML:

```xml
<memoryBacking><hugepages><page size='1048576' unit='KiB'/></hugepages></memoryBacking>
```

Guest memory must be an exact multiple of the page size, and the pool must cover it fully. A pool
that is one page short will stop the domain from starting.

## 4. Optional: host CUDA on the same GPU

If you also want to run CUDA on the host while the guest is off, see
[sharing-a-gpu.md](sharing-a-gpu.md), and blacklist the NVIDIA display modules so they never
auto-load. Without it, the desktop compositor opens the card's DRM
render node and blocks handing it back to `vfio-pci` until you log out.

## 5. Optional: custom KVM modules

If you build your own `kvm.ko` / `kvm-amd.ko`, your distro's kernel package owns those paths and
will overwrite them. See [kvm-modules.md](kvm-modules.md).
