# Caprock

A capability-based microkernel written in Rust — and being moved onto Gabbro, the language built alongside it — designed to be verified rather than trusted.

Caprock has no users, no root, no ambient authority and no global namespace. A process can do exactly what it holds a capability for — nothing else is nameable, so nothing else is reachable. Drivers, the network stack, filesystems and platform services are ordinary processes with narrow capability sets, not privileged kernel code.

The goal is a system where isolation is a property you can check, not a promise you have to accept.

---

## Why

A conventional server runs tens of millions of lines of kernel code in a single privilege domain, most of it device drivers. Every isolation guarantee above it — containers, namespaces, disk encryption, network policy — rests on the assumption that none of those lines contains an exploitable bug. Historically, that assumption has not held, and drivers are where it fails most often.

Caprock takes the opposite approach:

- **The trusted core is small.** On the order of ten thousand lines, with its security-relevant properties machine-checked in Isabelle/HOL.
- **Drivers are not privileged.** A driver holds an MMIO capability for its own registers, an IRQ capability for its own interrupt, an IOMMU domain, and buffers. A faulty NIC driver cannot read foreign memory, cannot touch another device, and cannot take the machine down with it.
- **DMA is typed.** A bus-mastering device bypasses CPU memory protection unless the IOMMU is configured correctly — the usual shortcut is an identity mapping that silently disables it. In Caprock, an identity mapping is not constructible without a witness type. The mistake isn't made unlikely; it isn't expressible.
- **Resource accounting is exact.** Scheduling contexts are first-class and are donated across calls, so CPU time, interrupts and DMA transfers are attributable to the request that caused them — even when the work happens in another process or in a driver.

## What follows from the design

These are not features bolted on top. They fall out of the fact that a process is fully described by its address space, its thread states and its capability space — there is no fourth place where state hides.

### Live driver reload

A driver is a process holding four kinds of capability: MMIO for its own registers, its own IRQ line, an IOMMU domain, and buffers. Replacing it means quiescing the queue, draining in-flight requests, fencing outstanding DMA through the IOMMU, handing the serialised device-independent state to a new process, and giving that process **the same capabilities** — not new ones. The device is not reinitialised when the handover permits.

A driver security fix becomes a process restart instead of a reboot window. The hard part is the quiesce protocol: a NIC cannot be stopped instantly, so per device class you either wait for completion or invalidate the mappings and handle the resulting faults. That is a design decision per driver, not a generic mechanism.

*Status: the structure exists; the reload path is not built yet.*

### Freezing threads and processes

`freeze` halts a thread at a nameable boundary — not on a core, no open IPC relation — and refuses with a reason otherwise. Refusal is the useful part: a freeze that succeeds when it shouldn't produces a checkpoint that isn't one.

What state a thread carries is **enumerated, not serialised ad hoc**: a closed set of parts, so a new TCB field fails a `match` at compile time rather than showing up as wrong registers after the first thaw. Refusal reasons are explicit, including domains a checkpoint cannot honestly cross — a device window is not reconstructible elsewhere, and a core affinity is created fresh elsewhere, so they are different reasons rather than one vague one.

Two properties worth naming:

- **A full image is confidential, always.** That is a machine-readable rule, not a convention, so there is no case in which the storage and transport rules quietly lapse.
- **The clock does not pause.** Userspace reads the hardware counter directly (`rdtsc` / `CNTVCT_EL0`), so a paused kernel clock would be a promise the hardware does not back. Instead the thaw reports the elapsed ticks to the holder of the freeze authority. The frozen process itself does not learn the duration — a deadline it computed from the counter before the freeze cannot be corrected afterwards. This is stated because it is a real limit, not because it is comfortable.

*Status: freeze, thaw and the state enumeration are built and measured on x86_64 and AArch64.*

### Migration — resuming elsewhere

Because a process is the transitive closure over its capability space, "what belongs to this process" is an answerable question. On Linux it is not, which is why checkpoint/restore there is a collection of special cases that grows with every kernel release.

What can move is every capability that is either location-independent (memory, endpoints, notifications, scheduling contexts, address spaces) or has an equivalent at the destination. What cannot move is named and refused: device-bound capabilities, anything that has handed out a physical address, and anything derived from a machine-local number.

**Migration is bounded by class, not by wish.** A process cannot move between x86_64 and AArch64 — instruction set, register state and memory model differ, so "resume identically" is physically excluded. The same applies within one architecture across differing CPU feature sets. A migration class is therefore (ISA, feature mask, page size, memory model), and movement is only permitted inside one.

*Status: not built. The state enumeration above is the precondition for it — the checkpoint image deliberately carries no thread state yet.*

## What this is not

We try to be precise about claims, because vague ones are worthless in this domain.

- **Not "a formally verified OS."** The kernel core is verified. Drivers and platform code are written in Gabbro, which makes a defined set of error classes syntactically inexpressible — that is correctness by construction, not functional correctness. Guest programs are not verified at all.
- **Not free of unverified code.** Caprock runs unmodified Linux binaries through a syscall compatibility layer. Those programs are unverified; what is verified is their containment. There is no Linux kernel here. There are Linux programs.
- **Not a drop-in Linux replacement.** No shell, no ambient permissions, no `ptrace`, no namespaces, no container runtime. Some of that is missing on purpose and will stay missing.
- **Not production-ready.** See status below.

## Verification levels

Every component is assigned exactly one level, and the level is stated rather than implied.

| Level | Meaning |
|---|---|
| **V0** | Unverified. Contained, not trusted. Guest binaries, third-party code. |
| **V1** | Correct by construction. Written in Gabbro; a defined set of error classes cannot be expressed. Not functional correctness. |
| **V2** | Functional correctness against a specification, machine-checked in Isabelle/HOL. |
| **V3** | Refinement down to the binary artefact. Target, not yet reached — the C compiler currently sits in the trusted base. |

The full assumption list — hardware behaves to spec, firmware is correct, no microarchitectural side channels, no physical attacks — is published alongside the proofs rather than buried. A proof without its assumptions is marketing.

---

## Repositories

Caprock is developed as several repositories under this organisation. Some exist, some are planned; the table says which.

### Core

| Repository | What it is | Status |
|---|---|---|
| **caprock** | The microkernel: capabilities, IPC, address spaces, scheduling contexts, IOMMU/DMA subsystem. x86_64 and AArch64. | In development |
| **gabbro** | Systems language targeting Caprock. Linear and ghost types, flow narrowing, atomic pairing, runtime-bounded values. Compiles to freestanding C11. The kernel itself is being migrated onto it. | In development |
| **caprock-proofs** | Isabelle/HOL specifications and proofs for the kernel core, plus the published assumption list. | In development |

### Drivers

Every driver is a normal process. Written in Gabbro, one repository per family, so a driver can be reviewed, replaced and eventually hot-reloaded on its own.

| Repository | What it is | Status |
|---|---|---|
| **caprock-drivers** | Platform drivers: PCIe enumeration, interrupt controllers, timers, IOMMU (VT-d, SMMUv3), TPM. | In development |
| **caprock-net** | NIC drivers and the network stack. | Planned |
| **caprock-storage** | NVMe and the block path, including durability semantics and block-layer encryption. | Planned |

### System

| Repository | What it is | Status |
|---|---|---|
| **caprock-linux** | Linux syscall compatibility layer. Runs unmodified binaries under a capability set. | In development |
| **caprock-tools** | Build, image and provisioning tooling; measured-boot image assembly. | Planned |
| **caprock-debug** | Debug authority as a capability over exactly one protection domain — delegable, revocable, auditable. A domain for which the authority was never minted cannot be debugged, and that is checkable rather than asserted. Includes a GDB remote-serial-protocol server. | Early |

### Desktop

A long-term direction, not a near-term deliverable. The same properties that make Caprock interesting for servers — per-process capability sets, no ambient authority, exact resource accounting — matter at least as much on a desktop, where the software you run is far less trustworthy.

| Repository | What it is | Status |
|---|---|---|
| **caprock-desktop** | Display server, input, compositing, and a session model where an application receives capabilities for what the user actually granted it. | Exploratory |

Nothing here is committed to a timeline. Server and platform work comes first, because it is what forces the driver and compatibility work that a desktop would need anyway.

---

## Method

A few working rules that show up throughout the code and the documents, stated here because they explain why things look the way they do.

- **Measure before building.** Predictions are written down before the measurement, not adjusted afterwards.
- **A test that cannot fail proves nothing.** Every claimed property has a counter-proof: a specific mutation that must turn exactly that line red. One mutation, one failing conjunct — a mutation that breaks two things proves nothing about either.
- **Silence is a failure.** A check that reports success without having exercised anything is a bug, not a pass.
- **Name the limits.** Where a guarantee rests on policy rather than proof, the document says so.

## Status

Early. This is research-grade systems work: the kernel boots on real hardware on two architectures, parts of the driver set and the compatibility layer exist, and the proof effort is ongoing. It is not ready to run anything you care about, and the interfaces will change.

If that is interesting to you anyway, the individual repositories carry build instructions and current state.

## Licence

Per repository; see each one.

---

*Built in Germany.*
