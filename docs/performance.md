# Performance and tuning

Measured findings from one AMD Ryzen + NVIDIA laptop host running a 32-vCPU Windows guest. The
numbers are specific to that machine; the mechanisms are not.

## CPU pinning

Pin vCPUs 1:1 onto host threads and keep the guest's topology honest — if you expose 16 cores ×
2 threads, pin sibling vCPUs onto real sibling threads:

```xml
<vcpu placement='static'>32</vcpu>
<cputune>
  <vcpupin vcpu='0' cpuset='0'/>
  <vcpupin vcpu='1' cpuset='16'/>   <!-- 0 and 16 are SMT siblings on this host -->
  ...
  <emulatorpin cpuset='15,31'/>
</cputune>
```

Check which threads are siblings before writing the map:

```bash
cat /sys/devices/system/cpu/cpu0/topology/thread_siblings_list
```

**Leave the emulator its own threads.** This is the mistake worth avoiding: pinning all 32 vCPUs
across all 32 host threads and *also* giving `emulatorpin` two of those same threads means the
QEMU emulator, the host desktop, and memory compaction all fight two vCPU threads. On this host
that collapsed the all-core clock roughly 8× whenever the guest ran. Either give the emulator
threads nothing else, or leave headroom:

```xml
<!-- vCPUs on 0-14,16-30; emulator alone on 15,31 -->
```

**Isolate the rest.** `nohz_full` and `rcu_nocbs` keep housekeeping work off the pinned threads.
Leave at least one core pair out of the list to absorb it:

```
nohz_full=1-15,17-31 rcu_nocbs=1-15,17-31     # CPUs 0 and 16 stay as housekeeping
```

## Hugepages

Use 1 GiB pages, reserved at boot (`hugepages=` on the command line). Two things are worth
knowing.

**Runtime reservation does not reliably work.** On a host up only ~2 hours, with 19 GiB free and
almost no fragmentation, asking for 12 pages returned 11. Repeated `drop_caches` and
`compact_memory` never recovered the twelfth: a single unmovable allocation anywhere inside a
1 GiB-aligned region is enough to block that page. At longer uptimes it is hopeless. Reserve at
boot or not at all.

**`hugetlb_cma` looks like the answer and is incompatible with VFIO.** CMA solves the
fragmentation problem completely — allocation succeeded 12/12 instantly, every time, and the
host could use the region for movable allocations while the guest was off, which recovered
around 12 GiB of usable host RAM. But the guest then will not start:

```
vfio 0000:01:00.0: failed to setup container for group 15:
memory listener initialization failed: Region pc.ram:
vfio_container_dma_map(...) = -12 (Cannot allocate memory)
```

VFIO pins all guest memory for DMA, and pinned pages are by definition unmovable, which is
exactly what a CMA region cannot hold. Tried and reverted — do not spend time on it.

The cost of boot-time reservation is that the memory is gone while the guest is off.
Writing `0` to `nr_hugepages` releases the pool; re-reserving afterwards may need a reboot, for
the fragmentation reason above.

## KVM module parameters (AMD)

```
options kvm_amd nested=1 avic=0
options kvm ignore_msrs=1 report_ignored_msrs=0
```

**`avic=0` is load-bearing on this host.** With AVIC enabled the guest hung at AP startup — 20 of
32 vCPUs parked at the firmware AP stub and disk I/O froze. The cause is the known AMD AVIC
limitation around INIT/SIPI with the all-but-self shorthand, producing an
`kvm_avic_incomplete_ipi` storm. Windows 10 uses xAPIC rather than x2APIC, so x2AVIC does not
help. Re-test a cold boot before changing this.

`nested=1` is only needed if your domain exposes `svm` to the guest. `ignore_msrs=1` stops
Windows' WHEA driver tripping on AMD SMCA MSRs; `report_ignored_msrs=0` silences a printk storm
from RAPL MSR polling (~10k/s) that otherwise floods the journal.

## Guest-side

**Turn off VBS/HVCI.** Windows' virtualization-based security nests a hypervisor inside your
guest and costs a lot. It can be disabled offline in a shut-off guest. Note it has
*two* switches and both must be cleared — the registry `DeviceGuard` keys and the BCD
`hypervisorlaunchtype` element. Clearing only the registry side leaves the hypervisor launching
every boot.

**Power plan.** Set the guest to High Performance; the default Balanced plan parks cores.

## Host latency

Holding a PM-QoS constraint on `/dev/cpu_dma_latency` for as long as a process runs stops the cpuidle governor entering deep C-states with long exit latencies. The constraint
is bound to the open file descriptor, so the kernel releases it automatically when the process
exits for any reason — that auto-release is why this is preferable to writing to per-core
cpuidle sysfs, which has no such safety.

```python
# hold a 0 µs CPU-DMA-latency target until this process exits
import os, struct
fd = os.open('/dev/cpu_dma_latency', os.O_WRONLY)
os.write(fd, struct.pack('i', 0))
# ...keep the fd open; closing it releases the constraint
```
