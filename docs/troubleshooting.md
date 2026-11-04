# Troubleshooting

Symptoms that cost real time here, and what they actually were.

## The guest hangs early in boot, some vCPUs never start

Most vCPUs parked at a firmware AP stub, disk I/O frozen. On AMD this is AVIC. Set
`avic=0` in `/etc/modprobe.d/kvm.conf` and cold-boot. See
[performance.md](performance.md#kvm-module-parameters-amd).

## The domain refuses to start

**Hugepage pool too small.** The pool must cover guest memory *completely*; one page short is
a hard failure.

```bash
grep -i huge /proc/meminfo       # HugePages_Total vs the guest's memory
```

**You enabled `hugetlb_cma`.** It is incompatible with VFIO — the error names
`vfio_container_dma_map(...) = -12`. Revert to plain `hugepages=`. See
[performance.md](performance.md#hugepages).

**The GPU is bound to the wrong driver.** `lspci -nnk -d 10de:` should show `vfio-pci`. If it
shows `nvidia`, hand it back to `vfio-pci` first — see [sharing-a-gpu.md](sharing-a-gpu.md).

## The GPU ends up with no driver at all

You ran `modprobe -r nvidia` and `vfio-pci` came off with it, because `vfio.conf` softdeps the
two. Rebind by hand:

```bash
echo vfio-pci | sudo tee /sys/bus/pci/devices/0000:01:00.0/driver_override
echo 0000:01:00.0 | sudo tee /sys/bus/pci/drivers_probe
```

Use `rmmod` rather than `modprobe -r` in future. See [sharing-a-gpu.md](sharing-a-gpu.md).

## The GPU will not go back to `vfio-pci`

Something on the host holds it. Usually the desktop compositor, via the DRM render node that
`nvidia_drm` creates. `sudo fuser -v /dev/nvidia*` and `lspci -nnk` show what holds it. The fix
is to stop `nvidia_drm` ever loading; once a compositor has it, logging out is the reliable way
to get it back.

## The guest hangs at power-on with the GPU attached

OVMF stalling on the GPU option ROM. Recovery is to remove and re-add the device so the ROM is
re-read: unbind it, `echo 1 > .../remove`, then rescan that bridge only.

## The VM appears hung and nothing responds — but it is running

With `<video model='none'/>` there is **no display at all**, so anything Windows puts on screen
is invisible. Repeated forced power-offs put Windows into Automatic Repair, after which *every*
boot stops at a recovery prompt you cannot see. The VM looks dead; it is sitting at a menu.

Temporarily adding a `<graphics type='vnc'>` device plus a VGA model lets you look (the domain
must be shut off — graphics cannot be hot-added). This masqueraded as
"the kernel module won't boot" for four different modules here.

## Offline tooling fails on the guest's disk

If the guest was not shut down cleanly, the NTFS volume is dirty and offline registry/image
tools will refuse it or mount read-only. Always shut the guest down from inside Windows before
reaching for offline tools. If it is already dirty, boot it once and shut it down properly.

Mounts via `ntfs3` are also **case-sensitive**, which matters for hive paths: a fresh install
has `Windows/System32/config/SOFTWARE` in uppercase.

## Windows Setup says it cannot find installation media

The FAT32 partition on your installer image is an EFI System Partition. Windows deliberately
hides ESPs and assigns them no drive letter, so Setup cannot read `install.wim` from its own
boot media. Use type GUID `EBD0A0A2-B9E5-4433-87C0-68B6B72699C7` (Microsoft basic data).
Set it when you build the media.

## The install restarts and installs again, forever

Boot order. The setup media keeps winning against the freshly installed disk. Give the target
disk `boot order='1'` and the setup media `order='2'` — an empty target falls through to the
installer on the first pass and the installed system wins on every pass after.

Do **not** try to detach the media partway through on a timer. An auto-detach heuristic
here fired mid-install and produced
`Windows could not apply the unattend answer file's <DiskConfiguration> setting`.

## A note on measuring

Three habits, each of which came from a false reading:

- **Check the content, not the exit status.** `virsh screenshot` returns 0 and writes a
  perfectly valid PNG when the guest has not initialised the display — the image is QEMU's
  placeholder. Discriminate on geometry or content, not on success.
- **Let the guest settle** before comparing exit rates or timings; give it ~10 minutes.
- **Verify builds by hash, never by mtime.** See
  [kvm-modules.md](kvm-modules.md#verifying-a-build-actually-landed).
