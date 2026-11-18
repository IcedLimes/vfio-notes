# How virtual machines get detected

Any guest can be identified as a guest. This page is about *why* that is structurally true,
what the detection surface actually consists of, and what it costs to reduce it — because the
answer shapes how much effort is worth spending.

The practical audience for this is sandbox work: malware routinely checks whether it is being
analysed and changes behaviour if it thinks it is, so people who run analysis environments care
a great deal about how visible the hypervisor is. The same mechanics explain why a passthrough
guest behaves differently from bare metal even when you are not trying to hide anything.

> These notes come from a real build. The passthrough guest they describe was tuned until it
> came back clean against a comprehensive public VM-detection suite — so the trade-offs and dead
> ends below are what actually happened, not what ought to in theory. The configuration itself is
> deliberately not reproduced here; see the [scope note](../README.md#scope).

## Why a hypervisor cannot be transparent

Everything follows from one fact: **virtualisation is implemented by trapping.**

When a CPU runs a guest, the hypervisor configures which instructions and events cause a
**VM exit** — a forced transfer of control from guest to host. An intercepted instruction costs
on the order of 1,000–15,000 cycles instead of the 20–200 it would take natively. That gap is
the root of **timing** detection, and it is not an implementation flaw; it is the mechanism.

Separately, whatever the hypervisor intercepts it must then *emulate*, and emulation is rarely
bit-identical to silicon — different values, different exception types, different side effects
in corner cases. That is the root of **artifact** detection.

Neither can be eliminated. Both can be reduced.

## The four families of check

**1. Ask politely.** `CPUID.1:ECX[31]` is the architectural "hypervisor present" bit.
`CPUID.0x40000000` returns a vendor string — `KVMKVMKVM`, `Microsoft Hv`, `VMwareVMware`. A
hypervisor that does not deliberately suppress these is identified in one instruction. These are
the easiest checks to defeat and no serious detector relies on them alone, but failing them is
immediate, so they are table stakes rather than an achievement.

**2. Look for artifacts.** By far the largest and most mechanical surface, and what public test
suites mostly exercise:

| Layer | What leaks |
|---|---|
| SMBIOS / DMI | vendor and product strings (`QEMU`, `innotek GmbH`), default or absent serials |
| ACPI | OEM IDs and table signatures (`BOCHS`, `BXPC`), emulator-specific tables |
| `fw_cfg` | QEMU's firmware-config interface answers with the literal string `QEMU` |
| PCI | IDs that exist only in emulators — `1AF4` (virtio), `1234:1111` (Bochs VGA), `80EE` (VirtualBox) |
| Storage | model strings, serial formats, and `rotation_rate` (the SSD-vs-HDD tell) |
| Network | MAC OUI ranges assigned to hypervisor vendors |
| TPM | the emulated TPM's manufacturer ID |
| Display | EDID vendor/product/serial, especially identical values across every unit of a dummy plug |

**3. Measure time.** The classic is `rdtsc; cpuid; rdtsc` — if `CPUID` took 8,000 cycles,
something trapped it. More sophisticated versions compare the TSC against `APERF`/`MPERF`, which
count real core cycles, or look at the *variance* of an instruction's cost rather than its mean.

**4. Probe for behavioural divergence.** Execute something that should fault and check *which*
fault arrives. Read an MSR that does not exist on real hardware and see whether it returns a
value instead of faulting. Put the CPU in an impossible state and see whether the emulator
handles it the way silicon does. This family is the hardest to defend against, because defending
requires the emulator to be *exactly as wrong as the hardware is*, in every corner case — and
corner cases are where emulators differ by definition.

## What reducing the surface actually costs

Three things are worth understanding before deciding how far to go.

**Consistency is the hard problem, not any individual value.** A machine that reports a
plausible laptop model but whose ACPI tables, PCI topology, CPU enumeration, display EDID and
storage identifiers describe three different machines is *more* identifiable than an obvious
VM, because no real product looks like that. Changing one value in isolation usually creates a
contradiction somewhere else. The work is in making a single coherent machine, which means the
effort scales with the number of layers, not the number of checks.

**Every change to emulation trades correctness.** The emulator behaves the way it does largely
because something depends on that behaviour. Altering how faults are delivered, or which
instructions are intercepted, changes semantics for legitimate guest code too — and the
failures show up far from the change, as instability rather than as an error message.

**Two approaches are known dead ends**, both learned expensively here:

- *TSC compensation* — subtracting VM-exit cycles from the guest's TSC offset to hide intercept
  latency. Easy to get subtly wrong, and getting it wrong makes the guest clock run **backwards**,
  which hangs the boot. It also creates cross-clock drift between the TSC and every other time
  source, which is itself a stronger signal than the latency it hid. (This is distinct from
  synchronising cross-core TSCs, which is a legitimate and separate concern.)
- *Disabling CPUID interception wholesale* — during SMP bring-up this makes the host's real
  topology disagree with the guest's configured topology, and the result is all vCPUs spinning
  at 100% with no boot at all.

## Usermode and ring 0 are different problems

This distinction matters more than any individual technique, and it is the one most often
glossed over.

Public detection suites are **usermode programs**. A kernel-mode detector can read MSRs, inspect
page tables, issue privileged instructions, and time operations in ways usermode simply cannot.
A clean usermode result therefore tells you very little about what a ring-0 observer sees — the
two can disagree completely and for long periods.

The corollary is that a usermode score is not a measure of transparency. It is a measure of
usermode transparency, which is a much smaller claim than it appears.

## Measuring without fooling yourself

Most of the time lost in this kind of work goes to false readings rather than to hard problems.

- **Measure in the right context.** A detector run in an interactive desktop session and the
  same detector run from a service or over SSH see different machines — some graphics APIs
  cannot reach the GPU from a non-interactive session and produce phantom results.
- **Let the guest settle.** Boot-time behaviour is not steady-state behaviour; give it several
  minutes before comparing anything.
- **Check content, not exit status.** A tool can succeed, write a well-formed output file, and
  tell you nothing — a screenshot of a guest that has not initialised its display is a valid
  image of a placeholder. Discriminate on what the output *contains*.
- **Prefer host-side observation.** Tracing from the host (ftrace on the hypervisor's own entry
  points) requires no guest changes and cannot be observed by the guest. Instrumenting from
  inside changes the thing you are measuring.
- **Verify by hash, never by timestamp.** Build scripts re-copy artifacts and bump mtimes
  without changing bytes.

## The honest bottom line

`APERF` and `MPERF` count real core cycles and cannot be offset from inside a guest. A
privileged detector that cross-references them against the TSC will always see the cycles
consumed by VM exits. There is no configuration or patch that closes this; it is a direct
consequence of trapping being how virtualisation works.

So the realistic goal is not transparency. It is reducing the mechanical artifact surface — the
strings, IDs, tables and topology inconsistencies in the table above — which is worth doing
because it is the bulk of what is actually checked, and achievable because it is all just
configuration. Beyond that point the returns drop sharply and the correctness costs rise.

It is also worth being clear that software which does not want to run in a VM is usually making
a policy decision rather than a technical one, and policy is enforced in places you cannot see
from inside the guest. Making a detector quiet is not the same as being undetected.
