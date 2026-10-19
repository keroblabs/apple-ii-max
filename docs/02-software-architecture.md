# Apple IImax — GEOS-Max Software Architecture

**Document:** 02 — GEOS-Max native operating environment and compatibility strategy  
**Repository:** `keroblabs/apple-ii-max`  
**Status:** Software design direction and research programme; interfaces and tooling not yet frozen  
**Date:** 2026-10-19  
**Related:** [Introduction](00-introduction.md) · [Hardware architecture](01-hardware-architecture.md) · [Implementation roadmap](03-implementation-roadmap.md)

## 1. Mission

**GEOS-Max** (provisional name) is the native Apple IImax operating environment: a **GEOS-inspired desktop and application platform**, purpose-built for a 65816 running on a machine with approximately 1 MiB RAM, a preliminary maximum CPU clock of 2.8 MHz, an MHIRES framebuffer, and a separate bank-`$EF` native-device interface.

The goal is **more than a graphical launcher**. It is a coherent software stack providing startup, drivers, filesystem access, a window/widget API, a desktop/file manager, an application loader and a viable development workflow. Historical Apple II software matters too: the native environment should exchange files with ProDOS, allow a dependable way to enter legacy Apple IIe execution, and investigate whether selected ProDOS 8 programs can run from the graphical desktop.

GEOS-Max is **our own operating system**, not a port of Apple IIgs GS/OS or a copy of GEOS internals, and it does not depend on the X11 protocol. We may take inspiration from GEOS, the Apple II Desktop, Smalltalk-like resource composition and our separate Embedded X11 work, but Apple IImax has its own ABI and constraints.

## 2. Constraints that shape the OS

| Constraint | Software implication |
| --- | --- |
| 65816 executes with a 16-bit PC plus program bank | Code is split into bank-aware modules; ordinary sequential code cannot run beyond a 64 KiB bank without explicit transfer |
| Native accumulator/index widths can be 8/16 bits | Establish a consistent ABI for widths and call boundaries |
| Native direct page and hardware stack live in bank `$00` | Kernel, applications and compatibility programs must coordinate these resources |
| Bank `$00` includes IIe softswitches and mirrored ROM | It is not general-purpose continuous RAM; never assume arbitrary writable addresses |
| Bank `$01` backs auxiliary legacy memory | Allocator and guest-execution plan must reserve display and compatibility regions appropriately |
| `$02–$0F` intended as ordinary extended memory | Prefer native code/data and large resources here |
| Bank `$EF` maps hardware registers | Access via drivers/HAL, not arbitrary application writes |
| MHIRES-200 is 64,000 bytes | A single bank-aligned framebuffer is convenient; double buffering uses two such allocations |
| 2.8 MHz and byte-oriented external CPU transfers | Minimise copying and redraw; use compact data structures and event-driven execution |
| No memory protection | Applications are trusted initially; robust contracts and debugging matter more than isolation claims |

A different CPU emulation/native flag is **not** itself a protected process boundary or a complete operating-system context switch.

## 3. Layered architecture

```text
  Desktop / File Manager / Utilities / Native Applications
                         |
           GUI widgets, dialogs, layout, fonts
                         |
       Window manager + graphics + event dispatch
                         |
     Application loader + native ABI + system services
                         |
     File systems / block devices / input / timers / HAL
                         |
       65816 + SDRAM + legacy video + MHIRES + REGSEL
```

### 3.1 Bootstrap and runtime kernel

Apple IImax-aware firmware starts from the bank-zero reset mirror, completes hardware initialisation, establishes native 65816 state and transfers to the resident kernel. The kernel exposes a **versioned system-service interface** rather than requiring applications to call undocumented ROM entry points.

Kernel responsibilities initially include:

- RAM-bank/region allocation with reservations for bank-zero stack/direct page, Apple IIe video areas, device-backed I/O and ROM overlays.
- Resident interrupt and timer handling, using the bank-zero vector policy specified in the hardware document.
- Block device and filesystem interfaces.
- Input event collection and dispatch.
- Loading, relocation (if needed), application startup/exit and resource cleanup.
- Logging/diagnostics and system-level failure reporting.
- Transitions to controlled compatibility execution modes.

**First implementation model:** one foreground application with a resident desktop and modal dialogs. Cooperative background jobs or desk accessories are later additions. No pre-emptive scheduler or virtual memory is required for the first deliverable.

### 3.2 Device drivers and hardware abstraction

Native drivers are allowed to communicate with bank-`$EF` `REGSEL` registers and selected legacy hardware services. Applications generally see device-neutral APIs, not fixed register addresses.

Proposed service classes:

| Class | API responsibility |
| --- | --- |
| Video | Set mode, framebuffer base, palette, vblank; present dirty regions/page flip |
| Input | Keyboard, mouse motion, multiple buttons, wheel/scroll, focus events |
| Block storage | Enumerate devices, query geometry, read/write blocks, flush and report errors |
| Filesystem | Open/read/write/close, enumerate paths, attributes and directory operations |
| Timing | Ticks, monotonic timers and event wakeups |
| System | Memory allocation, application/process state, error reporting |

The Apple IIe slot device API remains available in compatibility execution, but native apps should not assume a specific slot. Native and legacy interfaces may drive the same underlying hardware using different register formats.

The proposed resident driver registry uses bank-aware function pointers and entry thunks. Driver **implementations can reside in extended RAM**, with compact dispatch structures in bank `$00` where needed; do not require all driver code to fit under `$00:D000` (which remains ROM-mapped in the native design).

### 3.3 Window manager and graphics kernel

The graphics kernel handles logical surfaces, clipping, dirty rectangles, bitmap/text rendering, palette management, z-order (when applicable), menus and controls.

**Initial policy:** the desktop owns the MHIRES framebuffer. Applications draw via a supported API into it or into small off-screen bitmaps, not unconditionally into video memory. The kernel coalesces dirty regions and minimises writes. Avoid compulsory full-screen redraws at 2.8 MHz.

Proposed minimal drawing API semantics:

- Set clip region, foreground/background colour and font.
- Draw pixels, lines, filled rectangles, blit icons and text.
- Measure strings and widget bounds.
- Invalidate a rectangle and process paint events.
- Present accumulated updates, optionally flipping the configured framebuffer on vblank.

The MHIRES palette has 16 entries; design icons/widgets to be legible at 640×200. The VGA presentation of MHIRES-200 initially doubles rows and centres the 640×400 active area. A future 640×480 framebuffer is possible, **not promised**.

Legacy text/LORES/HIRES/DHIRES modes are separate display contracts; a generic native window is not required to composite over arbitrary legacy pixels in the first version.

### 3.4 Input and event model

Use a compact queue of typed events: key down/up, text character, pointer motion, button down/up, wheel scroll, timer expiry, window expose/paint, application command and device notifications.

The desktop routes input according to focus and capture. Dispatch should be deterministic, with bounded per-event cost and explicit queue-overflow behaviour. The first version uses **cooperative event processing**: handlers return promptly; long file operations report progress or yield at controlled points.

Our separate Embedded X11 work provides useful conceptual examples for queues and OS abstraction, but this is **not** Xlib, JWM, TinyX or an X11 wire protocol.

## 4. Native executable format, ABI and resource model

A native application is not merely a 65C02 binary stored in a larger bank. It needs a loader-aware container and bank-aware calling convention.

### 4.1 Loader and executable container — proposal

A compact native container could include:

- Magic/version and minimum OS/API version.
- Entrypoint bank/address and required CPU mode.
- Code/data/BSS segments, preferred or relocatable destinations, alignments.
- Relocations for 16-bit near and 24-bit far addresses.
- Requested resident-memory and graphics-resource budget.
- Embedded icon, name, menus/dialog resources and optional file-type associations.
- Declared driver/service dependencies.

The exact file layout is **to be designed and frozen after a prototype loader**. We should initially support static known-address modules, then add relocation where genuinely useful.

### 4.2 ABI rules to specify

- Which register widths callers and callees expect (`REP`/`SEP` discipline).
- Which A/X/Y, DBR, D, PBR, flags and stack state a system call preserves.
- Near vs far calling, return-address sizes, parameters and error conventions.
- Who owns zero/direct page and how a service acquires temporary DP workspace.
- How interrupts preserve a native application context.
- How kernel calls remain valid regardless of caller code bank.
- How application exit restores desktop state and releases allocations.

The 65816 has direct 24-bit addressing but **not** a flat instruction space across banks. Cross-bank call thunks are a fundamental ABI concern.

### 4.3 Declarative GUI resources

The development host may describe menus/dialogs/widgets using JSON or a similarly readable language, for example:

```json
{
  "window": {
    "id": "files",
    "title": "Files",
    "width": 480,
    "height": 160,
    "widgets": [
      {"type": "list", "id": "entries", "x": 8, "y": 20, "w": 464, "h": 104},
      {"type": "button", "id": "open", "text": "Open", "x": 392, "y": 132}
    ]
  }
}
```

A **host-side resource compiler** turns readable definitions into compact binary tables; native code loads them and registers event callbacks. The resource format should use offsets/IDs rather than requiring a heap of dynamically allocated JSON trees.

Runtime JSON parsing (for configuration or user-created resources) is an optional experiment, not a dependency. This keeps authoring pleasant without taxing the 65816 at startup.

## 5. Desktop and file manager

The resident desktop owns the initial workspace, icon layout and application launch process. The file manager should provide:

- Volume discovery and directory navigation.
- File icon/name/type, selection and contextual commands.
- Open/launch, copy/move/rename/delete and confirmation dialogs.
- Program-file associations and human-readable metadata.
- Open/save dialogs shared by native applications.
- Handling of storage errors and removable media changes.

A simple desktop and file manager are **part of** the operating environment, not the whole end goal. Later native example applications could include a text editor, simple drawing tool, terminal/monitor and disk/image utility.

A native filesystem implementation should initially be straightforward and testable. **ProDOS volume compatibility is a priority**, because it also supports the legacy application strategy.

## 6. ProDOS: distinguish three very different compatibility objectives

**Research boundary:** Reverse engineering and adaptation of third-party legacy applications are strictly **private, noncommercial research activities**. They are **not for publication, distribution or commercial reuse**. This restriction applies to disassemblies, annotated proprietary binaries, patches derived from third-party code, application-specific traces and private research notes. The public Apple IImax project may describe generic compatibility interfaces and tests that it independently owns, but it must not publish the private reverse-engineering outputs. No copyrighted ROM/application/disk image may be committed to this public repository unless redistribution rights have been established. Research-only intent does not itself grant copyright or licence permission; obtain and use software lawfully.


“Run ProDOS applications like Wine runs Windows software” is an interesting and valuable research direction, but it has technical limits. Wine translates operating-system APIs in an environment with process/address-space conventions; an Apple II ProDOS 8 application may depend on direct 6502 execution **and** arbitrary Apple IIe hardware state. We therefore must not equate implementing ProDOS file calls with full binary compatibility.

### Level 1 — ProDOS filesystem interoperability (recommended first)

Implement a native ProDOS filesystem driver that can mount supported ProDOS volumes/images through the Apple IImax block-device API and manipulate files without starting the historical ProDOS kernel.

This includes block layout, pathnames, file types/auxiliary types, file metadata, directory traversal and data/resource conventions as applicable to the supported ProDOS version. Establish read-only correctness first, then safe writes and image-level regression tests. The native desktop can browse ProDOS volumes even if no legacy program execution is implemented.

**Result:** useful interoperability immediately, relatively low risk to the native OS.

### Level 2 — Exclusive Apple IIe/ProDOS execution session (recommended second)

Launch a ProDOS 8 image/program by entering a controlled **Apple IIe compatibility execution state**, preferably with genuine ProDOS and expected ROM/slot services available.

The 65816 can execute 6502/65C02-compatible code in emulation mode. The real machine still needs appropriate zero page, stack, language-card state, softswitches, video mode and peripheral behaviour. Use authentic firmware/OS media that the developer is entitled to use.

For the first reliable version, accept that this is **exclusive**, not a window inside MHIRES. The native GUI can hand off to the legacy session, and a reset/reboot return path is acceptable. Preservation and restoration of the entire native desktop without reboot is an investigation, not an initial promise.

A later fast-switch design might save all required CPU state, MAIN/AUX RAM, video/softswitch state and device context into reserved extended RAM, then restore it. This is a **software session snapshot**, not IIgs-style hardware video shadow RAM. It requires careful accounting for zero page, stack, hardware side effects, interrupts and any direct memory writes.

**Result:** highest near-term confidence for running a broad class of real legacy applications, though not integrated into a native window.

### Level 3 — MLI-compatible execution inside an Apple IImax runtime (research)

Investigate launching **selected ProDOS 8 binaries** with a compatibility layer that implements their expected ProDOS Machine Language Interface (MLI), related globals and common firmware conventions.

A central documented entrypoint is **`JSR $BF00`**. In ProDOS 8, the call is followed inline by a command byte and a 16-bit parameter-list pointer. The MLI processes the call, returns beyond those inline arguments and reports success/failure through the specified registers/flags. The **`$BF00–BFFF` global page** contains additional conventions. Merely intercepting a jump to `$BF00` does not reproduce the full environment.

Possible implementation approaches:

| Approach | Attraction | Major problem |
| --- | --- | --- |
| Install an MLI-compatible dispatcher/trampoline in a prepared legacy RAM environment | Native storage services can answer familiar ProDOS calls | Need exact MLI semantics, global page, CPU/stack state, memory ownership and firmware behaviour |
| Run guest 6502 code directly in 65816 emulation mode | No CPU instruction translation needed for compatible code | Guest writes directly to bank-zero memory, video softswitches and device registers; no hardware process isolation |
| Execute a software 6502/65C02 emulator inside the native OS | Allows true guest address-space and I/O virtualisation, potentially a window | Large CPU performance cost at 2.8 MHz; requires hardware/video modelling |
| Binary patching or targeted program adaptation | Can support useful, well-characterised individual programs | Per-program work, compatibility drift, potentially fragile assumptions |

**Best first research candidate:** a very small, documented ProDOS 8 machine-language program that uses standard MLI file operations, not BASIC.SYSTEM or a game that owns the machine. Develop a traceable minimal dispatcher in an **exclusive, controlled test environment** before claiming that such code can coexist with the desktop.

#### Constraints a ProDOS runtime must handle

- ProDOS MLI entry convention, inline command/pointer bytes, carry/error results and other documented register effects.
- ProDOS globals and memory bitmap under `$BF00–BFFF`, plus code/data below `$C000`.
- Bank-zero stack/direct page and CPU 8-bit-width assumptions; emulation/native transitions and interrupts.
- Direct access to Apple IIe text/graphics pages, keyboard, slot ROMs, language card and legacy peripherals.
- Programs that patch vectors, call monitor/firmware routines, bypass MLI and access raw disks.
- Programs dependent on timing, undocumented behaviour, specific cards or precise display effects.
- Failure recovery when a guest corrupts RAM, disables interrupts, never returns or assumes it controls the entire machine.

For a native MHIRES desktop and a directly executed legacy guest to coexist **without an isolated address space**, these conflicts are fundamental. A full windowed solution likely requires **software emulation, additional memory-context translation hardware, program-specific adaptation**, or an explicitly restricted subset of guest programs.

The OS must never claim universal “Wine-like” compatibility merely because the MLI subset works.

### Level 4 — Reverse engineering / program adaptation (selective and later)

For important applications that are not clean MLI clients, analyse their behaviour using documentation, disassembly of legitimately obtained binaries and emulator traces. Identify the APIs, memory assumptions, firmware entrypoints and direct hardware interactions they actually use.

Potential outcomes include:

- A compatibility profile that prepares the appropriate environment.
- A narrow binary patch or loader transformation for one known version.
- A replacement device/firmware thunk for a well-documented interface.
- A **native reimplementation** of the application's useful function instead of attempting fragile binary translation.

Prioritise documented interfaces and software we may lawfully inspect. **Keep reverse-engineering artefacts, proprietary binaries, application-specific traces, disassemblies, annotations, transformed executables and patches in private research storage only**; never publish them or use them commercially. A public generic compatibility test suite may include independently written sample programs and materials confirmed redistributable. Track private application versions, hashes, observations and limitations in non-public research notes rather than publishing them in this repository.

The phrase “reverse engineer ProDOS applications” is therefore a **private research route**, not a requirement to disassemble or rewrite every existing application and not a distribution goal.

## 7. Recommended compatibility experiment sequence

| Stage | Experiment | Pass criterion |
| --- | --- | --- |
| P0 | Read a ProDOS volume/image in host tests | Directory/file data matches independent reference |
| P1 | Mount the same data from native Apple IImax OS | File manager can open files reliably |
| P2 | Boot genuine ProDOS in exclusive IIe mode | Repeatable boot, file access and keyboard/video behaviour |
| P3 | Trace a minimal MLI-only program | Every `$BF00` call and its parameters/results are accounted for |
| P4 | Substitute selected MLI file services in an isolated test session | Same observable behaviour as reference ProDOS for that test |
| P5 | Test controlled return from legacy execution | Desktop recovers correctly or explicitly reboots cleanly |
| P6 | Profile one lawfully obtained real application in private research | Retain non-public compatibility notes, failure causes and adaptation-cost findings |
| P7 | Decide whether GUI-windowed compatibility is worth it | Evidence-based decision; no automatic commitment |

Keep **binary execution tests separate from filesystem parsing tests**. Success in P0/P1 says nothing about P3–P7.

## 8. Development workflow and tools

GEOS-Max development starts **on the Apple IImax emulator**, after its hardware model has passed the Apple IIe compatibility gate. The emulator is the primary execution, diagnostic, automated-test and investigation environment, not a stopgap used only until FPGA hardware exists. Full emulator capability, including bank-aware tracing, stepping, breakpoints, watchpoints, device inspection, deterministic replay and state snapshots, is a project-level requirement specified by [Document 03](03-implementation-roadmap.md); the particular implementation is deliberately not selected here.

The software project should be developed and verified **before depending on a physical VGA display or native storage controller**.

### Host-first loop

1. Author native kernel, GUI and applications in a version-controlled tree.
2. Build with a cross-assembler/compiler/linker targeting 65816; choose a toolchain only after checking 65816 native-code quality, banked relocations and debugging support.
3. Run host-side unit tests for filesystems, event queues, GUI resources and geometry.
4. Build and run the Apple IImax machine model/emulator using the same firmware images, address map and hardware-register contract as FPGA.
5. Test native programs, ProDOS guest sessions and display updates using automated traces/screenshots and deterministic disk images.
6. Run RTL/SDRAM/video integration tests with cycle-level diagnostics.
7. Deploy to the QMTECH hardware only once the relevant integration gate passes; compare traces and performance against the emulator.

An SDL- or equivalent-based host display can emulate VGA output for testing; that is a **development backend**, not an LCD feature and not part of the embedded software ABI.

### Suggested source organisation

```text
emulator/       machine model, diagnostics, stepping and trace tools
validation/     Apple IIe and IImax test images, manifests and fixtures
rtl/            hardware RTL (later migration)
fpga/           QMTECH 5CEFA5 constraints/implementation
firmware/       boot code, vectors, fixed ROM services
geos-max/
  kernel/       memory, interrupts, event loop, loader, diagnostics
  hal/          hardware-neutral driver contracts
  drivers/      video, input, block devices, timers
  fs/           ProDOS (first), filesystem abstractions
  gui/          fonts, drawing, windows, widgets, resource loader
  desktop/      file manager and shell
  compat/       legacy session manager, MLI investigations, test tools
applications/   native sample applications
tools/          host resource compiler, image conversion, executable packer
tests/          unit tests, emulator scenarios, compatibility fixtures
docs/           specifications and future ABI/compatibility reports
```

This is a suggested eventual organisation, not a claim that those files currently exist. [Document 03](03-implementation-roadmap.md) is authoritative for workstream boundaries, emulator-first sequencing, and the handling of validation fixtures in the public repository. Proprietary media and private reverse-engineering material must not be committed.

### Regression test priorities

- API/ABI unit tests for far calls, argument passing, register preservation and stack use.
- A test program whose code and data reside in different banks.
- Buffer/bitmap clipping at 64 KiB boundaries.
- Native RAM allocator that never returns ROM/I/O or reserved legacy video ranges.
- Desktop rendering at 640×200, including dirty-region accounting, text and mouse events.
- Filesystem tests for malformed images, read-only mounting and write consistency.
- Controlled guest start, hardware accesses, interrupt behaviour and recovery.
- Performance counters for cache misses, memory stalls, redraw cost and input latency.

## 9. Practical milestones

**SW-0 — boot/diagnostics:** firmware enters native execution; prints or records diagnostics and uses bank-`$EF` hardware safely.

**SW-1 — platform services:** memory allocator, bank-aware calls, input queue, video driver, block-device API.

**SW-2 — GUI baseline:** MHIRES screen, palette, fonts, pointer, clipping, widgets, dialogs and retained desktop.

**SW-3 — executable platform:** native application format, loader, stable ABI and resource compiler.

**SW-4 — useful desktop:** files and volumes, file manager, save/open dialogs, application launching and recovery.

**SW-5 — ProDOS interoperability:** mount and use ProDOS storage natively, demonstrate faithful exclusive legacy execution.

**SW-6 — selected ProDOS binary compatibility research:** bounded MLI shim proof-of-concept and compatibility profiles; only expand scope if objective tests justify it.

These are **GEOS-Max software milestones**, subordinate to the emulator-first acceptance gates in [03-implementation-roadmap.md](03-implementation-roadmap.md). Hardware RTL migration starts after the emulator has validated the Apple IImax hardware contract and software development is underway; **none of these milestones is assumed complete today**.

## 10. Explicit decisions still open

1. Which cross-toolchain and native executable/relocation format are practical for the chosen 65816 RTL implementation?
2. What should the native calling convention and driver/service table look like?
3. How are bank-zero RAM, guest zero page/stack, original graphics buffers and firmware trampolines reserved?
4. What is the minimum native block-storage device/driver API, and how does it coexist with the Apple IIe slot facade?
5. How much GUI state can be buffered at 640×200 without harming memory availability or interactivity?
6. Should foreground-only operation remain the initial process model, or are cooperative desk accessories sufficiently cheap?
7. Which first ProDOS 8 target(s) will supply a representative, legally usable MLI trace and hardware-compatibility test?
8. Can a direct-execution legacy session safely return to the desktop, or should the first implementation intentionally reboot?
9. Is there enough measured performance for a software-emulated, windowed guest, or is exclusive guest execution the better end state?
10. Under what measurements is 640×480 MHIRES worth enabling?

## 11. Research references

- [ProDOS 8 Technical Reference](https://prodos8.com/docs/techref/) — official-era MLI, memory map, program and file conventions.
- [ProDOS MLI calls](https://prodos8.com/docs/techref/calls-to-the-mli/) — `JSR $BF00`, inline arguments, status and services.
- [ProDOS memory use](https://prodos8.com/docs/techref/memory-use/) — global page, system bitmap and memory constraints.
- [WDC W65C816S documentation](https://www.westerndesigncenter.com/wdc/documentation/w65c816s.pdf) — native/emulation states and bank-aware execution.
- [Apple FPGA source](https://github.com/keroblabs/apple_fpga) and [Embedded X11](https://github.com/keroblabs/embedded_x11) — separate projects with reusable lessons, not implementation dependencies.

**Status note:** This document intentionally describes a native desktop **and** a compatibility research agenda. Running arbitrary ProDOS programs as safe, concurrently windowed native applications is not an established capability.
