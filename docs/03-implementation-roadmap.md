# Apple II Pro — Implementation Roadmap

**Document:** 03 — execution strategy, project structure and acceptance gates  
**Repository:** `keroblabs/apple-ii-pro`  
**Status:** Initial roadmap for independent review; schedules, tool choices and emulator design intentionally open  
**Date:** 2026-10-19  
**Related:** [Introduction](00-introduction.md) · [Hardware architecture](01-hardware-architecture.md) · [GEOS-Pro software architecture](02-software-architecture.md)

## 1. Executive decision: emulator first, FPGA later

Apple II Pro must be built **as a computer platform in software before committing its complete architecture to FPGA RTL**. The owner already has a functioning Apple computer emulator based on an architecture similar to the proposed Apple II Pro, originally intended for an Apple IIgs. That existing work is the starting asset for this project, subject to a source-level inventory and review; this roadmap does **not** select or redesign the emulator yet.

Our order of work is:

1. Bring the existing emulator development into the **Apple II Pro repository**, preserving provenance and a known-working baseline.
2. Adapt its machine model to the Apple II Pro hardware contract in Document 01.
3. Validate faithful Apple IIe firmware, memory, video, I/O and peripheral behaviour inside the emulator.
4. Use that validated machine as the primary development and research platform for **GEOS-Pro** (provisional OS name), plus native applications.
5. After the emulated hardware architecture and representative software workloads are stable, bring appropriate Apple FPGA RTL into this repository and implement the Apple II Pro FPGA machine.
6. Continuously compare FPGA behaviour against the emulator's accepted functional reference, retaining emulator-driven software development even after real hardware runs.

**This is not an instruction to wait for Apple FPGA to move to Cyclone V first.** Apple II Pro is now a separate project with its own machine definition and repository; its eventual RTL comes from deliberate migration/adaptation of reusable Apple FPGA components and new RTL developed here. Do not mutate or abandon the separate Apple FPGA project as a side effect.

The progression is **architecture emulation → IIe verification → GEOS-Pro → RTL hardware implementation**, with tests and tools developed throughout. These stages may overlap once the upstream acceptance gates are met, but they must not be reversed merely to get pixels on the FPGA sooner.

## 2. Core deliverables in one repository

The repository should contain five first-class development workstreams:

| Workstream | Purpose | Long-term deliverable |
| --- | --- | --- |
| Emulator development | Functional and diagnostic Apple II Pro machine model, originating from the existing emulator | Executable reference computer, automated tests and interactive research/debug environment |
| Apple IIe validation artefacts | Firmware/peripheral behaviour tests, sample disk images, test programs, manifests and expected results | Reproducible compatibility suite with explicit provenance and licensing |
| RTL development | 65816 integration, address/MMU, SDRAM, cache, VGA, I/O, ROM and board bring-up | Synthesizable Apple II Pro on QMTECH Cyclone V |
| GEOS-Pro development | Native firmware, OS, HAL, drivers, GUI, filesystem, application loader, ProDOS interoperability | Usable graphical operating environment on emulator and FPGA |
| Applications | Native GUI applications and test tools | File manager, utilities and demonstrator programs, plus application SDK examples |

Cross-cutting tooling includes cross-builds, documentation, image preparation, automated regression, diagnostics, trace analysis and continuous integration.

**Important scope boundary:** legacy Apple IIe compatibility is part of the Apple II Pro platform; **Apple IIgs hardware and software compatibility is not a project target**. The existing emulator's IIgs origins do not change the new contract.

## 3. Proposed repository organisation

This is a **target layout**, not a command to move or rename code before inventory.

```text
apple-ii-pro/
├── docs/
│   ├── 00-introduction.md
│   ├── 01-hardware-architecture.md
│   ├── 02-software-architecture.md
│   ├── 03-implementation-roadmap.md
│   └── ... future interface, test and compatibility notes
├── emulator/
│   ├── core/                 # CPU, memory map, device/bus model
│   ├── devices/              # IIe and Apple II Pro peripherals
│   ├── debug/                # machine inspection and research instrumentation
│   ├── frontends/            # CLI/GUI/automation as chosen later
│   ├── tests/                # emulator correctness and timing tests
│   └── ...                   # exact implementation deferred
├── validation/
│   ├── apple_iie/
│   │   ├── manifests/        # names, checksums, expected hardware, rights
│   │   ├── tests/            # independently authored/redistributable fixtures
│   │   ├── expected/         # permitted golden outputs/traces
│   │   └── tools/            # local fixture preparation and verification
│   ├── apple_ii_pro/          # native hardware-specific scenarios
│   └── prodos/              # redistributable format/API tests
├── rtl/
│   ├── cpu/
│   ├── memory/
│   ├── video/
│   ├── io/
│   └── system/
├── fpga/
│   └── qmtech_5cefa5/       # constraints, PLL, pinout, programming, reports
├── firmware/                 # startup, ROM images built from project sources
├── geos-pro/
│   ├── kernel/
│   ├── hal/
│   ├── drivers/
│   ├── fs/
│   ├── gui/
│   ├── desktop/
│   ├── compat/              # generic, independently written compatibility code
│   └── resources/
├── applications/
├── tools/                    # assemblers/build helpers/resource/image tools
├── tests/                    # end-to-end hardware/software tests
└── .github/                  # CI once suitable
```

Keep `emulator/`, `rtl/`, `geos-pro/`, `applications/` and `validation/` independently buildable/testable where possible. Shared hardware address constants and register semantics must have **one authoritative specification** (Document 01 plus a future versioned register ABI); any generated headers should derive from that, not become conflicting independent truths.

### Public repository and legacy ROM/disk-image handling

This repository is **public**. The goal of having validation artefacts “in the repo” must not lead us to publish software, firmware, ROM dumps or commercial disk images without redistribution permission.

- **Commit**: original test programs authored for this project; redistributable/open-licensed firmware or images with their licence notices; test harnesses, hashes, manifests, scripts, documentation of public interfaces, and test expectations that we are authorised to distribute.
- **Keep local/private**: proprietary Apple IIe ROM dumps, copyrighted application media and research-only modified binaries or disassemblies unless distribution rights are expressly established.
- The validation harness should resolve private local fixtures through configurable paths or an ignored artefact directory, verify their checksums and report **skipped/missing-private-fixture** distinctly from test failure.
- A public manifest may describe required *types* of media and version constraints without including confidential research notes, copyrighted bytes or a proprietary derivative.
- Reverse engineering of third-party applications is **strictly private, noncommercial research only**. **Do not publish or distribute application-specific reverse-engineering results or derived patches.** Research-only intent does not itself remove copyright/licensing restrictions. The normal, independent Apple II Pro emulator/RTL/GEOS-Pro development work is a distinct workstream.

Do not make public CI depend on unavailable proprietary fixtures. Provide redistributable smoke tests that run everywhere and an optional private compatibility suite for the developer's environment.

## 4. Phase 0 — Inventory and establish the existing emulator baseline

**Objective:** establish exactly what is already working before modifying the IIgs-oriented emulator for Apple II Pro.

Work:

- Locate the existing emulator code, its build chain, dependencies, test fixtures and entrypoint. Record its original repository/revision when imported.
- Build and execute an existing known-good machine scenario; capture repeatable startup and representative outputs.
- Inventory CPU core capabilities: 6502/65C02 emulation, 65816 native mode, interrupt vectors, long addresses, cycle progression and CPU debug hooks.
- Inventory memory mapping, original IIgs assumptions, peripheral interfaces, storage, screen output and existing debugging capabilities.
- Identify reusable Apple IIe hardware behaviour versus IIgs-only features that must be removed/disabled/not modelled.
- Preserve a comparative baseline test so that a regression introduced by the migration can be located.

**Do not specify a new emulator framework or architecture during this phase.** The first outcome is an inventory and gap analysis of the emulator that already exists, not a premature rewrite.

**Gate E0 — baseline accepted:** emulator builds repeatably and runs at least one original reference scenario; inventory lists supported and absent hardware behaviour; imported code has traceable provenance. No conclusion about Apple II Pro compatibility is implied yet.

## 5. Phase 1 — Implement the full Apple II Pro machine contract in the emulator

**Objective:** make the emulator's guest-visible behaviour conform to Document 01, independently of physical SDRAM performance.

Necessary capabilities include:

- 65816 reset in emulation mode; correct bank-zero reset vector, program bank and interrupt-vector behaviour.
- MAIN/AUX Apple IIe memory map, bank-zero I/O, `RAMRD/RAMWRT/ALTZP/80STORE`, language-card and internal/slot ROM selection.
- Direct extended RAM banks `$02–$0F`, with agreed native-versus-compatibility mapping policy.
- `$EF:0000–FFFF` native `REGSEL` register window; `$FF` ROM and its conditional `$00:D000–FFFF` mirror.
- `DEVSEL`, `IOSEL`, `IOSTB` and the relevant slot/expansion devices expected by validation programs.
- Existing Apple IIe TEXT/LORES/HIRES/DHIRES and mixed/80-column display behaviours.
- PHIRES 640×200×4 bpp and a programmable framebuffer base, palette and scan/video mode controls.
- Native storage and input facilities sufficient for later GEOS-Pro work; reserve contracts for multi-button/wheel mouse and block-device semantics.
- A well-defined way to inspect/configure guest memory and ROM state for research and tests without changing guest-visible hardware behaviour.

The emulator must distinguish the **functional memory model** from the **real SDRAM scheduling model**. A functional RAM read may initially be immediate, while a configurable performance/timing model is added for cache misses, CPU stalls, 8-byte bursts and VGA-DMA arbitration. Neither model is permitted to contradict the address-map or register-visible state.

Treat cache behaviour and SDRAM delay modelling as independent verification concerns. CPU caching in hardware must not be relied on for correct application behaviour. Correct software should behave identically with a cache disabled, cache hits changed or wait states inserted.

**Gate E1 — Apple II Pro architecture accepted:** automated tests cover logical decoding, ROM/reset transitions, native/compatibility execution, display modes, I/O read/write side effects, extended-bank access and basic device integration. The emulator must produce actionable failures rather than merely opening a VGA window.

## 6. Phase 2 — Apple IIe firmware and peripheral compatibility

**Objective:** establish a credible, regression-tested Apple IIe-compatible computer **before** using the emulator as the primary GEOS-Pro platform.

Build a compatibility validation matrix with these categories:

| Domain | What to validate |
| --- | --- |
| Reset and firmware | Apple IIe ROM boots correctly; monitor/BASIC and relevant firmware calls behave as expected |
| CPU | Emulation-mode instructions, decimal arithmetic, branches, stack, interrupts and timing-sensitive cases |
| Memory | MAIN/AUX, 80STORE, ALTZP, RAMRD/RAMWRT, language-card bank switching and ROM overlays |
| Text/graphics | 40/80-column text, characters, LORES, HIRES, DHIRES, page switching, mixed modes and relevant timing |
| Input | Keyboard strobes, modifier handling, polling and required paddles/mouse facades where implemented |
| Storage | Bootable compatible storage, slot ROM/firmware interactions and representative file operations |
| Slots/peripherals | DEVSEL/IOSEL/IOSTB selection and side effects, open-bus/floating-bus behaviour where required |
| ProDOS | Boot and run appropriately licensed ProDOS media, test directory/file operations and selected standard programs |
| Stability | Reset/reboot, repeatability, state snapshots and failure reports |

The validation suite must distinguish:

1. **Required baseline IIe features** for an accepted compatibility configuration.
2. **Optional peripherals** supported with explicit claims and tests.
3. **Known incomplete or unsupported behaviour**, especially cycle-exact dependencies and unusual cards.

Do not state “perfect Apple IIe compatibility” on the strength of a ROM boot screen alone. Test original firmware and peripherals with both synthetic fixtures and lawfully held real media where rights permit private testing.

**Gate E2 — Apple IIe compatibility baseline accepted:** agreed required tests pass, known limitations are documented, and failures have reproducible traces. Only after this gate does the emulator become the trusted primary platform for native OS work. Compatibility testing continues thereafter.

## 7. Emulator as a first-class engineering and research tool

The user's requirement is **substantially more capable than a conventional GDB-style source-level debugger**. We should build a comprehensive, extensible machine-observation and intervention infrastructure, while deferring decisions about the emulator's internals, UI toolkit and command syntax until its source has been reviewed.

### 7.1 Execution control

- Run, stop, reset, pause, resume; instruction step, step-over/step-out and controlled run-until.
- CPU cycle/phase stepping where supported; stop on bus transactions, interrupts, DMA and selected video timing points.
- Breakpoints by logical `bank:offset`, translated physical address, opcode, symbol, interrupt/vector or device event.
- Conditional breakpoints using CPU registers, softswitch state, memory values, counters or structured predicates.
- Watchpoints on reads, writes, execution, defined bit transitions and address ranges; capture **source, target and side effects** of each access.

### 7.2 Memory and state exploration

- Read/write guest memory by **CPU-visible logical address**, by **translated physical RAM address**, and (separately) by raw snapshot inspection. These are deliberately different operations.
- Inspection of Apple IIe MAIN/AUX and language-card backing, bank `$EF` registers, bank `$FF` firmware, slot state, native 65816 registers and stack.
- Hex editor, memory comparison, memory search, watch regions, disassembler and bank-aware code/data views.
- Controlled ROM/firmware image loading and disk/image mounting through explicitly selected modes.
- Memory writes via a debugger must clearly distinguish a **real emulated bus write**, which triggers softswitch/device side effects, from a **side-effect-free raw memory patch** used for analysis.
- Inspect current mapping and the chain from CPU logical address to ROM/I/O/RAM physical destination.

### 7.3 Tracing and deep diagnostics

- Configurable instruction trace including `PBR:PC`, opcode, disassembly, register/flag state and effective operands.
- Bus trace including access type, logical/translated address, data, timing/cycle, device responder and side effects.
- Softswitch/register-change trace, interrupt/vector trace, device commands, disk block operations and video fetches.
- Trace filters, trigger windows, history buffers, export, comparisons and correlation of CPU ↔ MMU ↔ device events.
- ProDOS-specific instrumentation for MLI calls at `$BF00` with inline parameters, return state and relevant globals; optional firmware/slot-call annotations.
- Debug views of framebuffers, text/video pages, palette, line buffers, scan position and video-mode switching.
- CPU and memory performance statistics: cycles, cache misses/hits (in timing model), bus stalls and per-device transaction counts.
- Clear reproducibility metadata: emulator build, ROM/disk identity, test seed, machine config and capture version.

### 7.4 Repeatability and reverse execution

Target deterministic execution from a saved machine state plus recorded external inputs. Desired tools:

- Save/restore full machine snapshots including CPU, RAM, softswitches, ROM selection and device state.
- Recording/replaying keyboard, mouse, storage responses, timers and other nondeterministic inputs.
- Checkpoints, state diffs, last-write provenance and reverse-step/reverse-continue **if feasible**.
- Scriptable scenario control, assertions, golden traces and automated failure capture.

Reverse-step and very long instruction histories can have substantial memory/disk costs. Prefer an extensible checkpoint/replay architecture and scoped tracing over assuming unlimited in-memory history. **“Without limitation” here means avoiding arbitrary constraints on inspection and extensibility—not promising physically unbounded storage or perfect reverse execution from day one.**

### 7.5 Debugger surfaces and interfaces

Aim for a reusable debug/introspection API that can support a CLI, scripting client, optional graphical debugger, automated tests, and later remote hardware traces. GDB compatibility may be useful as a bridge but must not limit Apple-specific features.

Possible future tools: breakpoint scripting, trace queries, annotated assembly listings, symbol/source integration, video inspection windows, storage/block viewers and protocol/event visualisation.

All research instrumentation must be disableable for normal-speed execution and must avoid mutating guest state except through explicit commands.

### Acceptance of the debugger foundation

Before calling it ready for GEOS-Pro work, demonstrate repeatable instruction stepping, bank-aware logical/physical memory inspection, register/softswitch tracing, conditional break/watchpoints, loaded media tests and snapshot/replay for a nontrivial Apple IIe boot-and-I/O scenario. The advanced components can be delivered iteratively; their interfaces should leave room for extensions.

## 8. Phase 3 — Develop GEOS-Pro on the validated emulator

**Objective:** implement and debug native software against the agreed machine interface before FPGA hardware integration.

### GEOS-Pro work packages

1. **Native bootstrap and ABI:** initialisation from firmware, 65816 native-mode entry, native vectors/interrupts, near/far calling rules and register preservation.
2. **Kernel and HAL:** extended-bank allocator, system-service dispatch, input/timer queues, native storage and errors.
3. **PHIRES API:** 640×200×4 framebuffer, palette, clipping, fonts, text, rectangles/blits, dirty-region tracking and cursor.
4. **Windowing and widgets:** menus, modal dialogs, lists, buttons, text fields, event dispatch and resource loading.
5. **Desktop/file manager:** mount volumes, browse/copy/open/rename files, launch native applications, display failures.
6. **Application SDK:** compact native binary format, loader, linker conventions, host resource compiler and sample templates.
7. **Applications:** text/editor or drawing utility, file manager, diagnostics, graphics and storage demos.
8. **ProDOS interoperability:** native filesystem support first, a controlled exclusive ProDOS session next, then a bounded MLI compatibility experiment if worthwhile.

Native GUI layouts may be authored in JSON and compiled on the **host** into compact resources; native runtime JSON parsing is optional. One foreground application and cooperative event handlers are the first scheduling model.

All GEOS-Pro features must have emulator-driven regression scenarios and remain aligned to the bank-`$EF` hardware register contract rather than depending on emulator-private APIs.

**Gate S1 — usable native machine in emulation:** boots GEOS-Pro, opens a window, handles mouse/keyboard input, renders PHIRES, mounts storage and launches a native sample application repeatably.

**Gate S2 — useful desktop baseline:** file manager, dialogs, loader and at least one nontrivial native application operate with stable ABI and actionable test failures.

ProDOS reverse engineering/application adaptation is **optional private research**, not a condition for first GEOS-Pro delivery.

## 9. Phase 4 — Translate the validated architecture into RTL

**Trigger:** emulator Apple IIe compatibility is accepted (E2), Apple II Pro machine architecture is stable (E1), and GEOS-Pro has begun exercising native interfaces. This protects RTL work from premature architecture churn.

**Destination:** `rtl/` and `fpga/` of **`keroblabs/apple-ii-pro`**, which becomes the authoritative home of Apple II Pro hardware. The independent **`keroblabs/apple_fpga`** repository remains available for its own Apple IIe work.

### Migration method

- Inventory source modules and dependencies from Apple FPGA and the previous IIgs work; import only what serves the Apple II Pro contract, with recorded source commit and licensing/provenance.
- Preserve the verified Apple IIe softswitch behaviour and slot decoding, rather than reimplementing them unnecessarily.
- Bring up the QMTECH 5CEFA5 board clocking, reset, VGA timing and 32-bit SDRAM interface.
- Integrate a suitable 65816 RTL core, 24-bit bus adapter, legacy MMU, bank-`$EF` register decoder and bank-`$FF` firmware access.
- Implement SDRAM byte masking, 64-bit read-buffer prototype, explicit CPU stalls/waits and video-DMA scheduling/line buffers.
- Preserve original Apple IIe display modes; implement PHIRES-200 with programmable framebuffer base. Keep PHIRES-480 conditional on measured performance.
- Test the same ROM, devices and representative programs used in the emulator, with automated differential checks on defined functional outputs.

### Emulator-to-RTL correspondence

Maintain a parity matrix with each feature's status in: **architecture spec → emulator → simulation testbench → hardware test**.

Examples: reset vector, language-card side effects, slot select, program/data read, write-invalidation, `REGSEL`, PHIRES scanline mapping, byte-lane mask, interrupt timing and storage transfer.

The emulator defines **guest-visible functional expectations**. Hardware cycle timing is separately validated against the SDRAM datasheet and actual board; do not assume emulator performance proves physical wait-state-free operation.

**Gate H1 — board/memory foundation:** boot ROM, memory diagnostics, VGA output and CPU-to-SDRAM transactions work on hardware with measured latency and no unexplained underruns.

**Gate H2 — compatibility parity:** Apple IIe ROM and core compatibility test suite behave consistently on emulator and FPGA, with documented permissible timing differences.

**Gate H3 — native parity:** GEOS-Pro boots on the FPGA, accesses bank-`$EF` native devices, displays PHIRES and runs the same application tests as in the emulator.

Do not declare H2/H3 solely because a screenshot looks correct.

## 10. Phase 5 — Application and compatibility expansion

After native functionality is repeatable on emulator and FPGA:

- Expand application APIs, documentation, templates and reusable GUI resources.
- Measure real redraw/input latency, CPU stalls, storage throughput, video DMA margins and code/data memory consumption.
- Decide on PHIRES 640×480 support using **actual workloads**, not SDRAM peak bandwidth alone.
- Grow ProDOS filesystem compatibility and broaden the exclusive guest-session test matrix.
- Privately investigate the selected MLI shim/application adaptation research path; keep research results and third-party derivatives out of public commits.
- Optionally add small hardware graphics assists only after measurable need is demonstrated.
- Keep emulator and RTL regression suites running as the OS evolves; the emulator remains the primary debugging environment.

The success criterion is a useful, maintainable Apple II Pro computer and software ecosystem—not matching every Apple IIgs feature or transparently executing every Apple II application.

## 11. Milestone dependency summary

```text
Existing IIgs-oriented emulator
           |
           v
 E0  Known-good baseline and inventory
           |
           v
 E1  Apple II Pro hardware model complete
           |
           v
 E2  Apple IIe firmware/peripheral compatibility proven
           |
           +----------------------+
           |                      |
           v                      |
 GEOS-Pro native development      |
           |                      |
           v                      |
 S1 / S2 usable software          |
           |                      |
           +----------+-----------+
                      |
                      v
         RTL / QMTECH hardware port
                      |
                      v
            H1 → H2 → H3 parity
                      |
                      v
          Apps and optional expansion
```

RTL bring-up may begin once the software effort has begun and the architecture is stable; **S2 need not be fully completed before H1 work begins**. The essential sequence is emulator hardware → Apple IIe validation → native software as the development driver → FPGA port and parity.

## 12. Work tracking and review policy

- Treat this roadmap as a **sequencing and acceptance document**, not a micro-level emulator design. Its emulator requirements define what the tooling must enable, not how a specific codebase implements them.
- Review each technical domain in separate discussions and specifications: emulator architecture, Apple IIe compatibility matrix, register ABI, GEOS-Pro ABI, RTL memory subsystem and ProDOS research strategy.
- Keep a running architectural decision log recording accepted choices, rejected alternatives, measurements and affected docs.
- Land changes with specific test evidence and commit messages that identify scope and status; do not confuse an experimental implementation with a completed milestone.
- Update Documents 00–02 when roadmap decisions alter the product end goal, memory/hardware contract or software architecture.
- Keep private reverse-engineering material private throughout emulator and compatibility development.

## 13. Immediate next actions

1. Inspect the existing functioning emulator and produce an **inventory/gap report** against Document 01—without committing to a rewrite or debugger framework.
2. Draft the Apple IIe firmware/peripheral **compatibility matrix** and identify which fixtures can legally be redistributed versus supplied locally.
3. Decide the emulator's minimum machine-state/debug API contract and establish automated build/run tests.
4. Port the smallest Apple II Pro memory-map, ROM and CPU-mode semantics into the emulator; validate them before expanding devices.
5. Establish the first full Apple IIe boot-and-peripheral compatibility gate.
6. Begin GEOS-Pro bootstrap, native GUI and application development **on that validated emulator**.
7. Only then integrate the Apple II Pro FPGA/RTL work and measure the real SDRAM/cache/VGA constraints.

**Current status:** Repository documentation exists; the owner's existing emulator is reported functional but has **not yet been inventoried, imported or benchmarked here**. No FPGA or software gate in this roadmap is presently claimed as passed.
