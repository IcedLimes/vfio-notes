# VFIO passthrough notes

Notes from building and running a **GPU-passthrough Windows guest on Linux** with libvirt/QEMU:
host setup, performance tuning, unattended OS installation, Thunderbolt eGPU hot-plug, sharing
one discrete GPU between the guest and host CUDA, and why a guest is always identifiable as one.

**This repository contains no code.** It is a writeup of mechanisms and findings — what the
failure looked like, what was actually going on, and what to do about it. Commands are given
where they are the clearest way to say something, not as a tool you install.

Most of this was learned the expensive way. Where something cost a day, that is usually noted.

## Scope

These notes describe mechanisms. They are deliberately not a configuration you can apply: there
is no patch set here, no domain XML, and no scorecard against any particular detector.

That is a judgement about what stays useful, not modesty. A working configuration is specific to
one machine and one moment and is stale within months, while the reasons it worked keep
transferring. The [detection notes](docs/vm-detection.md) in particular are an explanation of
why a guest is always identifiable as one — including the limit that no amount of configuration
gets past — rather than instructions for hiding one.

---

## Contents

Start here if you do not yet have a passthrough host:

- **[host-setup.md](docs/host-setup.md)** — IOMMU, binding both GPU functions to `vfio-pci`,
  reserving hugepages, and a `modprobe` side effect that can leave your GPU with no driver at all

Then, per topic:

- **[performance.md](docs/performance.md)** — CPU pinning and why the emulator needs its own
  threads, hugepages (including why `hugetlb_cma` is incompatible with VFIO), AVIC on AMD, and
  the guest-side settings that matter
- **[unattended-install.md](docs/unattended-install.md)** — installing Windows into a guest with
  no keyboard input: answer-file layout, the installer-media partition type, and boot ordering
- **[sharing-a-gpu.md](docs/sharing-a-gpu.md)** — moving one dGPU between `vfio-pci` and the
  NVIDIA driver without rebooting, and the two things that reliably break the hand-back
- **[kvm-modules.md](docs/kvm-modules.md)** — keeping locally built `kvm.ko`/`kvm-amd.ko` alive
  across package upgrades, and the one case nothing can save you from
- **[egpu.md](docs/egpu.md)** — why ejecting a Thunderbolt eGPU hangs the kernel, and the
  ordering that avoids it
- **[vm-detection.md](docs/vm-detection.md)** — why a hypervisor cannot be transparent, what the
  detection surface consists of, and why the general problem is not winnable
- **[troubleshooting.md](docs/troubleshooting.md)** — symptoms, and what they actually turned
  out to be

---

## The setup these notes come from

| | |
|---|---|
| Host | CachyOS (Arch-based), kernel 6.19, AMD Ryzen, 32 threads |
| Guest | Windows 10, 32 vCPUs, 12 GiB on 1 GiB hugepages |
| Passthrough | NVIDIA dGPU (VGA + HDMI audio functions), laptop/muxless |
| Stack | libvirt 12.6, QEMU 11.0, OVMF, swtpm |

Numbers are specific to that machine. The mechanisms are not — the reasoning should transfer to
any IOMMU-capable host.

---

## A caveat on everything here

Passthrough work means editing kernel command lines, module parameters, and libvirt domains, and
occasionally writing to disks. Nothing in these notes is risk-free, and a few items —
reserving hugepages, disabling AVIC, wiping a disk during an unattended install — will stop a
machine booting or destroy data if applied carelessly.

Read the reasoning before copying a command. Back up a domain's XML (`virsh dumpxml --inactive
<domain>`) before changing it; that one habit has undone more mistakes here than anything else.

## License

MIT — see [LICENSE](LICENSE). The notes are offered as-is, with no warranty and no claim that
they are correct for your hardware.
