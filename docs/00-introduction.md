# Apple IImax — Project Introduction

**Project:** Apple IImax  
**Repository:** `keroblabs/apple-ii-max`  
**Document:** 00 — project purpose and end goal  
**Status:** Initial architectural baseline; implementation and benchmarks pending  
**Date:** 2026-10-19
**Development strategy:** emulator-first; see [03-implementation-roadmap.md](03-implementation-roadmap.md)

## 1. Vision

Apple IImax is a new FPGA-based computer inspired by the Apple IIe, built around a 65816 processor and an extended memory and graphics architecture. Its distinguishing ambition is to be **a useful Apple IIe-compatible computer and a new, independent 16-bit Apple-like platform** on which we develop a native graphical operating environment.

It is **not** an Apple IIgs clone, and full IIgs compatibility is not a goal. We deliberately retain the parts of the Apple IIe that are useful for software compatibility while replacing or augmenting the subsystems that constrain native software: memory capacity, CPU capabilities, storage interfaces, input facilities, and graphics.

The machine is intended to boot and run Apple IIe software in a compatibility configuration, and then support a dedicated Apple IImax native environment with windows, icons, menus, dialogs, a file manager, launchable applications, and a programmable graphics system. It should remain approachable to develop for, understandable in RTL, and testable before the FPGA integration is complete.

**Guiding principle:** preserve the Apple IIe contract where compatibility requires it; add Apple IImax capabilities without reproducing the complexity of the Apple IIgs.

## 2. What makes the machine different

| Area | Apple IImax direction |
| --- | --- |
| CPU | 65816, emulation mode for legacy software and native mode for extended software |
| CPU target | Nominal maximum 2.8 MHz initially; provisional until measured |
| Memory | Initially 1 MiB of 24-bit-addressable RAM, with Apple IIe MAIN/AUX compatibility |
| FPGA platform | QMTECH Cyclone V 5CEFA5 module and baseboard |
| External memory | 32-bit SDRAM interface; actual timing and part-specific details to be validated |
| Native hardware registers | Separate bank `$EF` selected by `REGSEL` |
| Firmware | 64 KiB ROM in bank `$FF`; appropriate mirror into bank `$00` for reset and legacy firmware |
| Legacy I/O | Retain Apple IIe softswitches and `DEVSEL` / `IOSEL` / `IOSTB` decoding |
| Legacy display | Apple IIe text, LORES, HIRES, and double-high-resolution (DHIRES), including applicable mixed/80-column variations |
| Native display | **MHIRES**, initially 640×200 at 4 bits per pixel, presented via VGA with vertical replication |
| Display output | VGA only; the separate Apple FPGA project's LCD is explicitly out of scope |
| Native software | **GEOS-Max** (provisional name), a GEOS-inspired graphical operating environment with a documented application and driver ABI |
| Legacy software strategy | Native access to ProDOS-formatted storage; Apple IIe compatibility boot; investigate launching selected ProDOS 8 applications from the native desktop |

## 3. Intended user experience

The end goal is a computer that can:

1. Start in a documented, reproducible configuration and provide a dependable Apple IIe-compatible experience.
2. Boot an Apple IImax firmware and enter a native graphical desktop.
3. Display windows, menus, icons, text, buttons, lists, dialogs, and application screens through a reusable GUI kernel.
4. Browse storage, copy and organise files, and launch native applications.
5. Use modernised native input and storage devices through bank-`$EF` interfaces while retaining appropriate legacy-facing peripheral functionality.
6. Develop, transfer, load and debug applications using a host-side toolchain and an executable simulator/emulator.
7. Read ProDOS file systems and, progressively, run a useful subset of existing Apple II programs without pretending that every legacy program is automatically compatible with the graphical desktop.

The desktop should feel like a small, complete operating environment, **not merely a file launcher**. Its interaction model is inspired by Apple Desktop/GEOS rather than an attempt to port an entire modern X11 stack. The project may reuse operating-system abstraction and event-queue ideas learned from the separate Embedded X11 project, but it should not inherit X11 protocol or runtime complexity.

## 4. Graphics philosophy

All original Apple IIe display modes remain part of the compatibility design. Apple IImax adds **MHIRES (Max High Resolution)** as its own native graphics mode.

### Initial MHIRES target

- Logical framebuffer: **640×200 pixels, 4 bpp, 16 palette indices**.
- Framebuffer size: **64,000 bytes (62.5 KiB)**; one image fits in a single 64 KiB 65816 bank if appropriately placed.
- Display: normal **640×480 VGA output**; each source row initially occupies two VGA scanlines, giving a centred 640×400 image with 40 blank lines above and below.
- Address: programmable 24-bit SDRAM framebuffer base via bank-`$EF` registers.
- Framebuffer organisation: packed, linear, 320 bytes per source row.
- Updates: the displayed base can be changed at vertical blank to support page flipping; double buffering is a software policy, not a requirement for the first working screen.

### Open possibility: 640×480×4 bpp

A **640×480 MHIRES variant** remains a serious candidate, not a promised feature of the first implementation. It requires 153,600 bytes (150 KiB), crosses 65816 bank boundaries, increases display bandwidth, and increases the CPU's drawing workload. Raw SDRAM throughput alone is unlikely to be the decisive problem; CPU rendering cost, actual arbitration latency, and GUI ergonomics also matter.

The video architecture should avoid assuming a hard-coded 200-line source if this adds little complexity, while **acceptance of 640×480 is contingent on benchmarks**. We should not expand native graphics at the expense of a responsive GUI or a reliable Apple IIe fallback.

No LCD path, Apple IIgs SHGR compatibility, ADB compatibility, or IIgs-specific audio system is required for the first Apple IImax.

## 5. Native software end goal

The planned operating environment, provisionally named **GEOS-Max**, includes:

- A small native kernel/runtime with memory allocation, drivers, file operations, timers, input events and application loading.
- A graphics and widget library with a stable API, resource handling, clipping and redraw/dirty-region semantics.
- A resident desktop and file manager with launch, open/save dialogs, disks/volumes, and application discovery.
- One foreground application initially, with cooperative tasks or desk accessories only after the basic execution model proves reliable.
- A development workflow in which GUI layouts may be authored in JSON on the host, compiled into compact binary resources, and consumed by the 65816 runtime. Runtime JSON parsing is optional, not a design dependency.
- A documented native executable format and cross-bank calling conventions suitable for the 65816's 24-bit data addresses and 16-bit program counter.

The OS should be designed **for** the 65816, not as a collection of 6502 applications with ad hoc bank switches.

## 6. Legacy software and ProDOS

Legacy compatibility is an important part of the machine's identity, but has several levels:

1. **Apple IIe hardware/software compatibility:** run compatible programs in a legacy machine state using the authentic Apple IIe memory and I/O conventions.
2. **ProDOS storage interoperability:** mount/read/write ProDOS volumes from the native environment using a native filesystem implementation, independent of the original ProDOS kernel.
3. **Native-desktop launch of selected ProDOS 8 programs:** investigate a compatibility runtime akin in purpose to Wine, using either controlled execution of 6502/65C02 code, a ProDOS MLI compatibility layer, or a transition to an exclusive legacy session.
4. **Selective application adaptation:** understand, document and—where lawful and maintainable—adapt individual legacy programs when direct I/O access, hard-coded memory assumptions or other behaviour defeats a generic layer.

**Not a baseline promise:** seamless windowed execution of arbitrary ProDOS applications alongside native apps. ProDOS 8 programs can manipulate video RAM, zero page, stack, language-card memory, slot hardware and operating-system globals directly; this makes binary compatibility substantially harder than translating a file API.

**Research-only restriction:** Reverse engineering or adaptation of third-party legacy applications is exclusively private, noncommercial research. Application-specific disassemblies, traces, patches and derived binaries are **not to be published or distributed**. Redistributable project-created compatibility tools are separate. The public repository must not include proprietary ROM or application images without established rights. See [Document 02](02-software-architecture.md) and [Document 03](03-implementation-roadmap.md).

The detailed alternatives, trade-offs and proof-of-concept plan belong in [02-software-architecture.md](02-software-architecture.md).

## 7. Relationship to other projects

- **[Apple FPGA](https://github.com/keroblabs/apple_fpga):** separate Apple IIe development. Its Apple IIe softswitch, existing VGA logic, memory knowledge and verification lessons are candidates for deliberate reuse. Apple IImax must not silently modify that project or assume any particular historical directory layout is fixed.
- **[Embedded X11](https://github.com/keroblabs/embedded_x11):** separate, higher-end portable graphical/windowing project. Share architectural lessons where useful; do not make the Apple IImax OS depend on TinyX, JWM or X11.

Apple IImax is a **separate repository and product**, not a permanent development branch of Apple FPGA. We may import or reimplement validated components with appropriate provenance and tests. Each project retains its own scope, compatibility goals and roadmap.

## 8. Engineering principles

1. Keep legacy and native interfaces distinct and explicit.
2. Build the smallest correct memory and video hardware, then optimise based on measurements.
3. Provide wait-state capability from the outset; a cache-hit estimate is not a timing guarantee.
4. Avoid permanent video shadow memory: legacy scanning uses MAIN/AUX backing; MHIRES scans its selected SDRAM framebuffer.
5. Use host simulation, automated test benches, reference traces and repeatable acceptance criteria before relying on board observations.
6. Keep both the hardware register ABI and the software API documented and versioned.
7. Mark unverified claims as provisional; do not call an RTL component hardware-validated solely because simulations pass.
8. Preserve an Apple IIe-compatible fallback throughout native feature development.

## 9. Initial delivery milestones

**Sequence decision (2026-10-19):** begin with the owner's existing functioning IIgs-oriented emulator, adapt it to the entire Apple IImax machine contract, and validate original Apple IIe firmware/peripherals in that emulator. Then use the validated emulator to develop GEOS-Max and native applications **before migrating the Apple IImax hardware implementation into RTL in this repository**. The table below describes broad deliverables rather than a strict chronological gate sequence; [Document 03](03-implementation-roadmap.md) is authoritative for the implementation order and acceptance gates.


| Phase | Milestone | Proof |
| --- | --- | --- |
| A | Board and SDRAM bring-up | Deterministic memory tests, measured timing, working VGA output |
| B | Apple IIe compatibility foundation | ROM reset, legacy memory/softswitch tests, text and original graphics modes |
| C | 65816 native memory | Validated 24-bit addressing, ROM bank, `REGSEL`, RAM banks and CPU stalls |
| D | Cached SDRAM + display arbitration | Burst/cache tests, byte writes, continuous legacy/MHIRES scan without underruns |
| E | MHIRES and input/storage | Native framebuffer, palette, mouse, storage and framebuffer switching |
| F | Native GUI and application platform | Desktop, dialogs, file manager and first native application |
| G | ProDOS interoperability | Filesystem interchange followed by independently evaluated executable-compatibility pilots |

The phases are an ordering guide, not an assertion that previous Apple FPGA hardware work has already passed acceptance on the 5CEFA5 board.

## 10. Documentation map and decision discipline

- [00-introduction.md](00-introduction.md) — motivation, intended experience and non-goals.
- [01-hardware-architecture.md](01-hardware-architecture.md) — working hardware contracts, timing assumptions and open decisions.
- [02-software-architecture.md](02-software-architecture.md) — GEOS-Max native OS, applications, development workflow and ProDOS compatibility research.
- [03-implementation-roadmap.md](03-implementation-roadmap.md) — emulator-first migration, Apple IIe validation, debugging infrastructure, GEOS-Max development and RTL parity.

These initial documents intentionally distinguish **agreed direction**, **provisional engineering targets**, and **open investigations**. Future design-review sessions may refine individual documents without rewriting the project's overall purpose.
