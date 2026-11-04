# Installing Windows into a guest with no keyboard input

A fully unattended Windows install needs three things to line up: bootable media OVMF will
actually boot, an answer file that answers *every* question Setup can ask, and a boot order that
does not loop. Each of the items below cost a failed install before it was understood.

## The installer media

**The FAT32 partition must be `Microsoft basic data`, not an EFI System Partition.** Windows
deliberately hides ESPs and assigns them no drive letter — so Setup boots from your media and
then cannot read `install.wim` off that same media. The symptom is Setup starting normally and
then claiming it cannot find installation media, which sends you looking at the wrong thing
entirely.

Use type GUID `EBD0A0A2-B9E5-4433-87C0-68B6B72699C7`:

```
label: gpt
start=2048, type=EBD0A0A2-B9E5-4433-87C0-68B6B72699C7, name="WINSETUP"
```

**Export one edition rather than splitting the WIM.** FAT32 caps a file at 4 GiB and a full
`install.wim` is usually larger. Splitting into `.swm` works but complicates the answer file.
Exporting a single edition instead typically lands around 3.7 GiB and fits whole:

```bash
wimlib-imagex info   source/install.wim                      # what editions exist
wimlib-imagex export source/install.wim "Windows 10 Pro" dest/install.wim --compress=LZX
```

## The answer file

**`<DynamicUpdate><Enable>false</Enable>`** stops Setup pulling updates mid-install, which
otherwise changes the resulting build from run to run and makes results irreproducible.

**`Microsoft-Windows-International-Core` must appear in *both* the `windowsPE` and `oobeSystem`
passes.** Present in only one, OOBE still stops to ask about region and keyboard — and the
install is no longer unattended.

**Keep the NIC link down until OOBE finishes.** With no network, Setup skips the
Microsoft-account prompts and the network questionnaire entirely, which removes the last
interactive stop. Raise the link afterwards for provisioning.

```bash
virsh domif-setlink <domain> <mac> down
```

> Note: on some libvirt builds `domif-setlink` has no `--live` flag. If yours lacks it, the
> command silently does nothing to a running domain — check, rather than assuming it worked.

Windows Setup reads `autounattend.xml` from the **root of removable media**, so the file can
live on the installer image itself, on a separate small ISO, or both.

## Boot order, not media detachment

The obvious approach — pull the install media once Setup finishes — is a race against a process
whose timing you do not control. Guessing the moment wrong interrupts the install and produces
*"Windows could not apply the unattend answer file's `<DiskConfiguration>` setting"*.

Use boot order instead. Give the target disk `boot order='1'` and the setup media
`boot order='2'`: an empty target falls through to the installer on the first pass, and an
installed target wins on every pass afterwards. No detachment, no race, and no reinstall loop.

## Safety

An unattended answer file that partitions a disk will contain something like:

```xml
<DiskID>0</DiskID>
<WillWipeDisk>true</WillWipeDisk>
```

**That wipes DiskID 0 of whatever domain boots it, without asking.** Two habits make this safe:

- Point it only at a throwaway harness domain that has exactly one disk, and verify with
  `virsh domblklist <domain>` before you start anything.
- **Undefine the harness the moment you are done with it.** A leftover harness domain still has
  bootable install media attached — and if you later adopt its disk image into a production
  domain, that media is now pointed at data you care about.

```bash
virsh undefine <harness-name> --nvram --tpm     # note: no storage flags
```

That exact situation occurred here: an image was adopted into the production domain while the
harness stayed defined against the same file, with the wipe-on-boot media still attached. Boot
order meant a stray start would have booted Windows rather than wiping it, but it was one
boot-menu selection away from being destructive.
