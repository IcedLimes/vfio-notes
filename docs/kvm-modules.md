# Keeping locally built KVM modules alive across package upgrades

If you build your own `kvm.ko` / `kvm-amd.ko` — for tracing, debugging, or any local patch —
you have a problem that nothing warns you about: those paths under
`/usr/lib/modules/<KVER>/kernel/arch/x86/kvm/` are **owned by your distro's kernel package**.
Any reinstall or upgrade silently replaces your build with the stock one.

This happened here. A kernel package upgrade reverted the live `kvm-amd.ko` to stock and it went
unnoticed for roughly three weeks, during which every conclusion drawn about guest behaviour was
being drawn against the wrong module.

## The shape of a fix

Three parts, none complicated:

1. **Keep a copy** of your built modules somewhere the package manager does not own, with a
   manifest recording the kernel version they were built for and their sha256 sums.
2. **Restore after any transaction** that touches those paths or the kernel package.
3. **Detect the case you cannot fix** (below) and say so loudly rather than silently restoring
   something that will not load.

On Arch, step 2 is a pacman hook. Number it to run *before* `90-mkinitcpio-install`, in case the
initramfs ever picks these modules up:

```ini
# /etc/pacman.d/hooks/05-custom-kvm.hook
[Trigger]
Operation = Install
Operation = Upgrade
Type = Path
Target = usr/lib/modules/*/kernel/arch/x86/kvm/*

[Action]
Description = Restoring locally built KVM modules...
When = PostTransaction
Exec = /path/to/your/restore-script
```

The equivalents elsewhere are dpkg triggers, a dnf `%posttrans` scriptlet, or a systemd unit
ordered after your upgrade job.

## The case nothing can save you from

**A kernel *version bump* defeats this entirely.** Your stored copy belongs to a specific
`KVER`. A new kernel version creates a whole new module tree, your manifest no longer matches
anything, and there is nowhere to restore *into* — a module built for the old version will not
load on the new one anyway (vermagic mismatch). Your patches are simply gone and you are running
stock, probably without noticing.

The only real protection is to hold the kernel package:

```
# /etc/pacman.conf
IgnorePkg = <your-kernel-package> <your-kernel-headers-package>
```

Re-seed the stored copy whenever you deliberately rebuild, and re-pin after any intentional
kernel upgrade. Accept that pinning a kernel means you are choosing to miss its security
updates — that is a real cost, not a formality.

## Verifying a build actually landed

Two traps, both of which produced false "success" readings here:

**Never verify by mtime.** Build scripts that re-copy artifacts bump the timestamp even when the
bytes are identical. Compare hashes.

**A changed hash alone does not prove your change landed** if anything in your build randomises
output — the bytes then differ every run regardless of whether the patch applied. Grep the built
module for the actual value or symbol you expect to find.

```bash
modinfo -F srcversion kvm_amd                    # what is loaded right now
sha256sum /usr/lib/modules/$(uname -r)/kernel/arch/x86/kvm/kvm-amd.ko.zst
```

Comparing `srcversion` between the loaded module and your stored copy is the quickest honest
check that the running kernel has the build you think it has.
