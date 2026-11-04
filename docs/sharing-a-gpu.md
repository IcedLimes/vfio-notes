# Sharing one dGPU between the guest and host CUDA

A GPU bound to `vfio-pci` is unavailable to the host. If you also want to run CUDA on the host
while the guest is shut off, the card has to move between `vfio-pci` and the NVIDIA driver. That
is doable without rebooting, but two things reliably break the hand-back.

## 1. The display modules must never load

`nvidia_drm` creates a DRM render node. Desktop compositors — KWin certainly, others likely —
open every render node they find and keep it open. Once that has happened, the card cannot go
back to `vfio-pci` until you log out of your session entirely.

CUDA needs only `nvidia` and `nvidia_uvm`, so nothing is lost by refusing to load the rest:

```
# /etc/modprobe.d/nvidia-compute-only.conf
blacklist nvidia_drm
blacklist nvidia_modeset
```

This must be `blacklist`, not `install … /bin/false`. kmod ignores `install` commands for
modules that have a softdep, and the usual `vfio.conf` softdeps both of these.

## 2. Unload with `rmmod`, not `modprobe -r`

If `vfio.conf` contains `softdep nvidia pre: vfio-pci` — which the standard passthrough advice
tells you to add — then `modprobe -r nvidia` removes **`vfio-pci` along with it**, leaving the
GPU bound to no driver at all. The card is still there; nothing owns it.

`rmmod` touches only the module you name. It also *refuses*, rather than hanging, while any file
descriptor on `/dev/nvidia*` is still open, which is the behaviour you want: it tells you a CUDA
process is still alive instead of wedging.

## The procedure

**To the host (guest must be shut off, cleanly — `virsh shutdown`, not `destroy`):**

```bash
echo 0000:01:00.0 | sudo tee /sys/bus/pci/drivers/vfio-pci/unbind
echo nvidia       | sudo tee /sys/bus/pci/devices/0000:01:00.0/driver_override
echo 0000:01:00.0 | sudo tee /sys/bus/pci/drivers_probe
```

**Back to the guest:**

```bash
sudo rmmod nvidia_uvm        # in dependency order; each may refuse if busy
sudo rmmod nvidia
echo vfio-pci     | sudo tee /sys/bus/pci/devices/0000:01:00.0/driver_override
echo 0000:01:00.0 | sudo tee /sys/bus/pci/drivers_probe
```

Check what holds the device before and after:

```bash
lspci -nnk -d 10de:                                   # which driver is bound
sudo fuser -v /dev/nvidia* 2>/dev/null                # CUDA processes
ls /dev/vfio/                                         # VFIO groups in use
```

**Leave the audio function on `vfio-pci` throughout.** It is never needed by CUDA, and keeping it
bound avoids a second unbind/rebind cycle that can fail independently of the VGA function.

## Hugepages

A boot-reserved 1 GiB pool is unavailable to the host even with the guest off, which hurts when
you want that memory for a model:

```bash
echo 0 | sudo tee /sys/kernel/mm/hugepages/hugepages-1048576kB/nr_hugepages
```

> Re-reserving at runtime is **not reliable** — see
> [performance.md](performance.md#hugepages). Expect to reboot to get the pool back, and do not
> release it if you plan to start the guest again soon.

## Before starting the guest

Check the binding. If the GPU shows `nvidia`, hand it back first. The domain will not start with
the card on the NVIDIA driver, and with `managed='yes'` hostdevs libvirt's own rebind attempt can
race against a compositor that is still holding a render node.
