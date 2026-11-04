# Why ejecting a Thunderbolt eGPU hangs, and the ordering that avoids it

Bringing a Thunderbolt eGPU up is easy. Taking it down without hanging the kernel — or
disturbing an unrelated passthrough GPU on the same machine — is the hard half.

## The two mechanisms that matter

**`amdgpu` cannot survive the link disappearing while it is bound.** Its teardown path waits on
GPU work that can never complete once the device is gone, and the wait does not time out. The
task parks in `D` state and the only exit is a reboot. So the cable may only be pulled *after*
the drivers are already unbound — not during, not "it'll notice".

**Unbind first, then remove.** `pciehp`, sysfs `remove`, and every PCI rescan take one
kernel-wide lock: `pci_rescan_remove_lock`. Every other hot-plug-capable slot on the machine
contends for that same lock — including the slot your passthrough GPU sits behind. Unbinding a
driver does *not* take the lock, and removing an already-driverless device completes in
milliseconds. Doing it in that order keeps the lock held for a negligible window instead of for
the length of a driver teardown.

So the safe sequence is:

```bash
# 1. unbind every function, no lock taken
echo 0000:38:00.0 | sudo tee /sys/bus/pci/devices/0000:38:00.0/driver/unbind   # audio
echo 0000:38:00.1 | sudo tee /sys/bus/pci/devices/0000:38:00.1/driver/unbind   # GPU

# 2. remove the now-driverless devices, lock held briefly
echo 1 | sudo tee /sys/bus/pci/devices/0000:38:00.1/remove

# 3. de-authorize the enclosure
echo 0 | sudo tee /sys/bus/thunderbolt/devices/<dev>/authorized
```

**Never use the global `/sys/bus/pci/rescan`.** It walks every bus on the machine, takes that
same lock for the whole walk, and can disturb devices that have nothing to do with the eGPU.
Scope every operation to exact addresses.

**Set the enclosure's bolt policy to `manual`.** Plugging it in then only brings up the
Thunderbolt link; the PCIe tunnel and the GPU appear only when you explicitly authorize it. A
flapping cable can no longer re-attach a GPU behind your back — which matters a great deal if
the host is in the middle of something.

## What to identify on your own machine

Topology is the least portable part of this. Match the enclosure by Thunderbolt vendor/device
ID, and the ports by **sysfs path** — `pciehp` slot names are assigned in probe order and are
not stable across boots.

```bash
ls /sys/bus/thunderbolt/devices/          # enclosures; cat device_name, vendor_name
lspci -t                                  # the tree: find your USB4/TB PCIe switch
udevadm info -q path -n /dev/dri/card1    # sysfs path of the eGPU once attached
```

You need: your enclosure's IDs, the sysfs path of the USB4/Thunderbolt PCIe switch, and the
downstream ports the eGPU can appear behind.

## What any automation here must refuse to touch

If you script this, the exclusion list matters more than the happy path. Closing a process that
holds the eGPU is reasonable; closing these is not:

- the VM and everything around it — `qemu*`, `libvirtd`, `virt*`, `swtpm*`, `looking-glass*`,
  or anything in `machine.slice`
- the desktop session's plumbing — compositor, PipeWire, portals, keyring, systemd
- the passthrough GPU's address, which should be excluded by explicit match rather than by
  assuming your patterns will never happen to match it

And if a kernel step never returns, record that fact somewhere and refuse to make things worse
until a reboot clears it. A hung `pci_rescan_remove_lock` holder turns every subsequent attempt
into another `D`-state task.

## Diagnosing a hang

Tasks stuck in these frames mean an eGPU operation is wedged, not slow:

```
pciehp_*            drm_dev_unplug        pci_lock_rescan_remove
device_release_driver   amdgpu_device_fini_*   ttm_device_fini
```

```bash
cat /proc/<pid>/stack          # where it is parked
dmesg -w | grep -iE 'pciehp|thunderbolt|amdgpu'
```

Frames in `amdgpu_device_fini*` or `ttm_device_fini` specifically mean the driver's final
teardown is stuck — which is the failure this ordering exists to prevent.
