# Apple IImax — Hardware Architecture

**Document:** 01 — hardware specification baseline  
**Repository:** `keroblabs/apple-ii-max`  
**Status:** Proposed architectural contract; RTL, board-level timing and integration not yet approved  
**Date:** 2026-10-19  
**Related:** [Project introduction](00-introduction.md) · [Software architecture](02-software-architecture.md)

## 1. Goals and constraints

Apple IImax uses a 65816 CPU and 1 MiB initial RAM allocation to extend an Apple IIe-compatible machine. It retains the original Apple IIe softswitch and peripheral address scheme, adds **bank `$EF` native registers**, provides **bank `$FF` firmware ROM**, and scans both legacy video and new **MHIRES** video through VGA. No Apple IIgs SHGR, `$E0/$E1` shadow banks, ADB or LCD output is required.

### Agreed or strongly preferred

- Hardware target: **QMTECH 5CEFA5 Cyclone V module + baseboard**, with 32-bit SDRAM data interface and VGA connector as described by the owner. Exact chip and pinout must be verified before writing constraints.
- CPU: 65816 RTL core, executing legacy code in emulation mode and new code in native mode.
- Clock limit: **2.8 MHz maximum initial target**, subject to board measurements and possible later revision; slower legacy timing may be necessary for particular programs.
- Physical memory: SDRAM with no full-frame video shadow copies.
- Standard Apple IIe I/O and slot decode preserved: `$C000–CFFF`, `DEVSEL`, `IOSEL`, `IOSTB`.
- Apple IImax-only registers: 64 KiB window **`$EF:0000–FFFF`**, `REGSEL`, independent of legacy decoding.
- Firmware: full 64 KiB read-only **bank `$FF`**; appropriate `$FF:D000–FFFF` content mirrored at `$00:D000–FFFF` for reset/legacy ROM whenever ROM is selected.
- Conventional Apple IIe video from MAIN/AUX banks `$00/$01`; new MHIRES framebuffer at a programmable SDRAM address.
- SDRAM reads: first prototype uses **8-byte (64-bit) cache lines** from two 32-bit SDRAM data beats, provided the chosen controller supports a suitable transfer.
- Cache: read-only; writes go through to SDRAM using byte masking and invalidate matching read buffers. No write-back cache.

### Explicitly deferred

Exact CPU core and timing interface, SDRAM controller implementation and measured latency, physical bank/page organisation, video palette wiring, storage-device electronics, exact `REGSEL` register offsets, full 640×480 MHIRES acceptance, 65816 legacy timing compatibility limits, and board pin assignments.

## 2. Logical system block diagram

```text
   65816 RTL CPU
      | 24-bit logical address, byte data, R/W, cycle qualifiers
      v
   CPU adapter / state sequencing
      |
      +--> Apple IIe address translator / ROM-I/O decode
      |       +-- $00:C000-CFFF original softswitch + slot decoder
      |       +-- $00:D000-FFFF language-card or mirrored ROM
      |       +-- $EF:xxxx REGSEL native hardware registers
      |       +-- $FF:xxxx direct firmware ROM
      |
      +--> RAM read buffer(s) / read bypass / byte-write path
                                      |
                            SDRAM request scheduler <--- Legacy video DMA
                                      ^                 MHIRES video DMA
                                      |                         |
                              32-bit SDRAM               video line buffers
                                                                |
                                                    legacy/MHIRES pixel pipeline
                                                                |
                                                       640x480 VGA output
```

ROM and device requests bypass SDRAM read caching. The arbiter owns real SDRAM transactions; the video pipeline owns line-rate output. CPU cache hits require no external RAM transaction.

## 3. The 65816 programming and bus model

- Address space: 24 bits (256 banks × 64 KiB); external data width is eight bits in the conventional 65816 model.
- Native mode provides selectable 8/16-bit accumulator and index widths; it is **not** a 16-bit-wide external memory bus.
- Program Bank Register plus 16-bit PC means sequential instruction execution does **not** automatically cross a 64 KiB bank boundary. Applications need long branches/jumps/calls or an ABI-aware veneer.
- Direct page and hardware stack are in bank `$00`; native software cannot treat bank `$00` as freely replaceable. Its I/O, ROM, vectors, and legacy video carve-outs must be respected.
- CPU emulation/native flag **E**, hardware memory-map selection, and CPU timing selection are separate state variables. Do not use E as the sole signal for selecting the physical address map.
- 65816 long accesses can address bank `$EF` even while the CPU is in emulation mode, assuming the chosen core faithfully implements those instructions.
- If a candidate core presents the next address earlier than a physical 65816, exploitation of that feature is conditional on a formal bus contract and verification. **Do not assume an address is available at the beginning of PHI2**.

Required CPU adapter fields: valid address, 24-bit address, read/write, write data, read data, ready/stall, and program-vs-data qualification if available. The precise synchronous handshake, clock domains and wait sequencing remain to be designed and tested.

## 4. Logical memory map

The following are **CPU-visible addresses**, not a claim that every RAM byte is linearly allocated without compatibility overlays.

| Bank/range | Mapping and function |
| --- | --- |
| `$00:0000–BFFF` | MAIN / logical Apple IIe low memory; subject to compatibility bank switching or direct native mapping |
| `$00:C000–C0FF` | Apple IIe internal softswitches and slot device I/O |
| `$00:C100–C7FF` | Legacy per-slot and internal ROM select (`IOSEL`) |
| `$00:C800–CFFF` | Legacy shared expansion/internal ROM select (`IOSTB`) |
| `$00:D000–FFFF` | Apple IIe language-card RAM when selected under compatibility rules, otherwise mirror of applicable `$FF:D000–FFFF` ROM |
| `$01:0000–FFFF` | Explicit auxiliary RAM backing for native 65816 use; details of upper-bank compatibility overlays to be resolved |
| `$02:0000–$0F:FFFF` | Extended RAM, nominally direct/linear in CPU address space |
| `$10–$EE` | Reserved; no RAM or I/O alias is implied |
| `$EF:0000–FFFF` | Apple IImax native register window, `REGSEL`, not ordinary RAM |
| `$F0–$FE` | Reserved; no alias assumed |
| `$FF:0000–FFFF` | 64 KiB ROM, directly executable/readable via long addresses |

The initial goal is **1 MiB RAM** distributed logically over banks `$00–$0F`. Some bank-zero addresses are hidden by I/O and ROM, so that is **not** a promise of 1 MiB simultaneously available to native applications. The original IIe language-card organisation contains alternate 4 KiB `$D000–DFFF` banks, requiring additional backing storage or a deliberate RAM carve-out. The physical-memory-allocation table must account for these bytes explicitly; any earlier rough “extra 8 KiB” estimate remains a verification item rather than a final bill of materials.

### Compatibility address translation

In IIe memory-map mode, CPU accesses to MAIN/AUX and language-card regions use the established Apple IIe rules: `RAMRD`, `RAMWRT`, `ALTZP`, `80STORE`, language-card bank selection, write-enable arming, `INTCXROM`, slot-3 ROM selection and relevant ROM/RAM overlays. Some accesses read and write different physical backing locations. This behaviour must be based on the Apple IIe references and tests, not approximated as “bank zero versus bank one.”

In native memory-map mode, the **proposed** policy is to use explicit bank addresses for RAM and not let the legacy `RAMRD/RAMWRT/ALTZP` switches remap ordinary CPU accesses. Video softswitches continue to affect legacy display generation. `$00:C000–CFFF` still decodes according to Apple IIe conventions, and `$00:D000–FFFF` ordinarily exposes the firmware mirror. The exact treatment of language-card controls in native mode is to be finalised with the CPU and OS interrupt/ROM design.

A change in mapping must invalidate or correctly retag the CPU's cached RAM lines. Mode changes must occur through a controlled firmware entry point—not in the middle of code whose fetch mapping is about to disappear.

### Slot compatibility and `REGSEL`

- Keep `DEVSEL`, `IOSEL` and `IOSTB` as distinct compatibility select functions; do not globally widen `$C0xx` to an Apple IImax-only 4 KiB register map.
- Preserve Apple IIe softswitch read/write side effects and status values. A shared physical peripheral may offer an Apple IIe interface and a richer `$EF` interface, but both must be explicitly specified.
- Decode bank `$EF` from the **full 24-bit qualified CPU address**, with a documented access-valid handshake. `REGSEL` must not spuriously assert during invalid/internal CPU cycles.
- Register space is a design capacity, **not** proof that all 64 KiB contain implemented registers.
- Hardware registers may have read side effects; never speculative-prefetch or RAM-cache them.

Example future subdivisions, **not register assignments**: configuration/MMU, interrupt controller, mouse/keyboard extensions, storage, MHIRES palette/framebuffer, timers and diagnostics.

## 5. Boot ROM and reset

A real 65816 resets in **emulation mode**, with PBR `$00`, and fetches the reset vector from **`$00:FFFC–FFFD`**. It does not reset into bank `$FF`.

On reset, ensure the `$00:D000–FFFF` ROM view contains the needed vector, sourced from bank `$FF`'s final 12 KiB. The boot-ROM path must be available before SDRAM has finished power-up/initialisation, or the CPU must remain reset/stalled until both boot and RAM dependencies are ready.

Proposed Apple IImax firmware sequence:

1. CPU starts in emulation state and fetches the conventional bank-zero reset vector.
2. The firmware runs a small compatibility-safe bootstrap.
3. For changes that affect bank-zero ROM mapping, transfer control with a long jump into **bank `$FF`** before changing the map.
4. Initialise/test SDRAM, configure the `$EF` MMU/device state and establish direct extended RAM access.
5. Install or validate the native interrupt dispatch strategy and any required bank-zero state.
6. Set the appropriate 65816 registers/flags, switch to native mode using the architecturally correct sequence, and launch the native kernel.

**Vector caveat:** 65816 native interrupt vectors occupy bank-zero upper-ROM locations (including `$00:FFEA–FFEF`). Fixed ROM vectors can target ROM trampolines that dispatch to RAM drivers. An **unmodified Apple IIe ROM image is not automatically a valid Apple IImax native-mode firmware**; a separate native-aware ROM image or explicitly designed vector overlay is needed.

The full 64 KiB of bank `$FF` is ROM. How enhanced IIe internal Cx ROM mappings are sourced from this image, and how to select between legacy and native firmware images, must be specified before firmware integration. Unmodified Apple IIe firmware boot is an acceptance target, not an established fact.

## 6. SDRAM and CPU buffering

### Physical interface versus transfer size

- External SDRAM data bus: **32 bits**.
- Proposed logical read transfer: **64 bits** in a two-beat burst of 32-bit words, aligned to eight bytes.
- A “64-bit cache line” **does not mean 64 physical SDRAM data pins**.
- SDRAM clock, read latency (including activate/precharge/CAS), turnaround, controller scheduling and refresh are presently **unmeasured for this board**.
- Actual sustained throughput depends on SDRAM type, command timing, burst capabilities, address locality and video arbitration.

### CPU read cache baseline

A simple 8-byte tagged read buffer contains: aligned physical SDRAM base address, data, valid bit and selection mux for one byte. Program/data accesses from RAM may use it. A hit avoids SDRAM; a miss enqueues a burst and may stall the CPU.

A **split program/data pair of 8-byte buffers** is a candidate to benchmark, not a required first implementation. Program/data qualification (VPA/VDA or equivalent) is needed before adopting that split. Cross-line sequential instruction fetches, branches, stack/data traffic and legacy memory mapping all affect hit rate.

The suggested “two hits per three reads” corresponds to a **66.7% hit rate for the measured reads** and is a performance hypothesis only. It cannot guarantee fixed completion of bus writes or VGA fetches, and no actual workload hit rate has been measured.

### CPU writes: no write-back

- Every CPU RAM write is a **byte-masked SDRAM write-through** transaction, subject to acknowledged completion and required ordering.
- If the write's translated physical address matches any valid read buffer line, **invalidate** that line. Do not implement dirty bits, write allocation or deferred write-back in the first version.
- Confirm byte-lane mask mapping for the exact 32-bit SDRAM organisation. The selected SDR SDRAM controller must honour mask/byte-write semantics at burst boundaries.
- Writes to I/O, ROM or unmapped space follow their distinct target's rules; never direct such writes through the RAM cache.
- CPU writes must be visible to later CPU reads and video reads; a video line already buffered may legitimately show earlier pixel data until the next fetch.
- Cache tags must use translated physical addresses or be invalidated on all mapping changes. Any future DMA engine other than VGA may require additional coherency rules.

### Timing and arbitration

A 2.8 MHz nominal cycle is about **357 ns**, with a roughly **179 ns half-cycle** for an ideal 50/50 clock. These are arithmetic bounds, not available SDRAM timing margins. The CPU address-valid time and data setup/hold remove usable time; **do not promise zero wait states**.

The SDRAM subsystem should run at a clock suitable for memory timing, decoupled from CPU-visible cycles through a correct handshake and, if clock domains differ, verified CDC.

Arbitration clients: CPU reads/cache fills, CPU byte writes, legacy video DMA, MHIRES DMA, and controller refresh. Use video FIFO/line-buffer low-water thresholds to grant urgency while ensuring CPU progress and refresh obligations. Test worst-case latency, not just average bandwidth.

## 7. Video architecture

**Output:** VGA on QMTECH baseboard. The LCD controller stays exclusively in Apple FPGA and is not an Apple IImax integration target.

### Legacy Apple IIe modes

Preserve text (40/80-column as supported), LORES, HIRES, DHIRES, mixed modes and the expected pages/softswitch interactions. Fetch directly from the physical MAIN/AUX backing in banks `$00/$01`, not cloned `$E0/$E1` or full-frame shadow RAM.

Retain the Apple IIe's non-linear address ordering and correct video-versus-CPU memory selection. A renderer outputs pixels into a VGA timing pipeline with appropriate centring/scaling. Existing `apple_fpga/verilog/apple_vga/` work is an **integration candidate**, not yet certified for this board.

### MHIRES native mode — initial acceptance target

| Attribute | Value |
| --- | --- |
| Logical dimensions | **640×200** |
| Pixel depth | **4 bpp, 16 indexed colours** |
| Packing | Two pixels per byte; nibble order to be specified |
| Stride | **320 bytes per source row** |
| Image bytes | **64,000 (62.5 KiB)** |
| Location | Programmable 24-bit SDRAM framebuffer base via `REGSEL` |
| Alignment | At least 8-byte alignment for 64-bit transfers |
| VGA timing | 640×480 display timing; target nominal 60 Hz |
| Scaling | Each source row duplicated twice → 640×400 centred vertically, 40 border rows top and bottom |
| Buffering | VGA line FIFO/buffers, separate from CPU read cache |
| Switching | Apply new display base/geometry at vertical blank to avoid mid-frame changes |

One complete 640×200 framebuffer fits into a 64 KiB processor bank **only if its placement does not cross the bank boundary**. With two framebuffers, two 64 KiB bank allocations would be a convenient *software* arrangement rather than fixed hardware banks.

### Later option: MHIRES 640×480

A 640×480×4 bpp framebuffer occupies **153,600 bytes (150 KiB)** and crosses bank boundaries; it needs no vertical duplication. The controller should be extensible to this geometry without promising it initially. Any additional source heights or configurable stride are **possible future features** rather than requirements for phase one.

### Bandwidth estimation (active source bytes)

| Mode | Bytes/frame | Source bytes/s at 60 Hz | 8-byte bursts/frame |
| --- | ---: | ---: | ---: |
| MHIRES 640×200 | 64,000 | 3.84 MB/s | 8,000 |
| MHIRES 640×480 | 153,600 | 9.216 MB/s | 19,200 |

These figures count each source row **once**. The repeated 200-line rows should come from the video line buffer rather than being fetched a second time. The estimates do **not** include refresh, command overhead, slot-mode rendering costs, SDRAM turnarounds or arbiter margins.

The dominant uncertainty may be **software drawing speed on the 2.8 MHz 65816**, not nominal SDRAM streaming bandwidth. Keep full-screen redraw optional and use partial updates on the software side.

### Palette and registers

Propose a 16-entry programmable MHIRES palette with an RGB representation matched to the **actual QMTECH VGA DAC wiring**. It is premature to claim a precise palette bit depth until board inspection. Bank-`$EF` video controls will include mode, framebuffer base, palette and vblank/status; exact offsets and atomic-commit behaviour are to be assigned in a register ABI review.

## 8. Device architecture

- The original Apple IIe keyboard, softswitch and slot-select semantics form the compatibility interface.
- Native mouse input should support motion, multiple buttons and wheel/scroll events through registers or a queue under `REGSEL`. Legacy mouse emulation is a separate compatibility feature, not automatically provided by those registers.
- Mass storage should have a native block interface; if desired, the same hardware can expose a slot-compatible legacy facade for ProDOS or firmware use.
- Sound and ADB are out of initial scope. Retaining side effects of Apple IIe speaker/paddle addresses is not the same as promising physical sound or analog input hardware.
- Driver interfaces belong to the software ABI; no assumption that the entire upper bank-zero area is writable.

## 9. Integration from Apple FPGA

The current source repository contains reusable candidates including `verilog/softswitch/softswitch.v` and `verilog/apple_vga/`, with separate `rtl_dev/` IIgs experiments. **Reuse is selective, source locations may change, and no RTL has been imported into Apple IImax by this document.**

Before importing: record source revision and provenance, identify dependencies, preserve Apple IIe register behaviour with regression tests, document adaptation for 65816 clocks/24-bit addresses/new SDRAM, and keep existing Apple FPGA work independent. The QMTECH migration and native IImax architecture can evolve independently.

## 10. Verification gates and unresolved decisions

### Functional tests

- Reset vector at `$00:FFFC` maps to correct ROM; bank `$FF` reads remain stable.
- Emulation/native CPU transitions and NMI/IRQ/BRK vector behaviour.
- MAIN/AUX read-versus-write remapping, `ALTZP`, `80STORE`, language-card banking and slot ROM decoding.
- No false `REGSEL`; I/O reads not cached; ROM writes appropriately handled.
- 64-bit cache line alignment, hit/miss, write invalidation, cache alias/mode changes, byte-mask lanes, SDRAM refresh and memory stalls.
- VGA text/LORES/HIRES/DHIRES and MHIRES colour, scanline duplication, FIFO underrun prevention, vblank base switching.
- Simultaneous worst-case CPU writes, cache misses and video requests.

### Hardware measurements

Measure the **actual** Cyclone V / SDRAM controller clocks, worst-case access latency, per-client arbitration delay, cache-hit rate under representative software, continuous video stability, and 65816 sustainable throughput. Initial 2.8 MHz limit may change **only after** this evidence.

### Still to decide

1. Exact 65816 RTL core, reset/interrupt signals and bus handshake.
2. Exact SDRAM chip, controller transfer geometry and byte mask behaviour.
3. Physical RAM backing and alternate language-card bank allocation.
4. Native-versus-compatibility mapping policy in upper MAIN/AUX regions and guardrails for switching.
5. Definitive firmware images/selection and Cx internal/slot ROM implementation.
6. `$EF` register addresses, read/write semantics and version identification.
7. VGA colour palette wiring, pixel nibble order, and 640×480 MHIRES viability.
8. Slot-compatible storage and legacy peripheral acceptance matrix.

## 11. Technical references

- [WDC W65C816S datasheet](https://www.westerndesigncenter.com/wdc/documentation/w65c816s.pdf) — registers, reset, bus qualification, interrupt vectors and timing.
- [ProDOS 8 technical reference](https://prodos8.com/docs/techref/) — legacy memory conventions and software-visible interfaces.
- [Apple FPGA source repository](https://github.com/keroblabs/apple_fpga) — candidate legacy softswitch, VGA and SDRAM-related implementation and verification work.

**Design status:** This document describes our current target architecture. It does not assert that the 65816 core, 5CEFA5 port, SDRAM timing, MHIRES or full legacy compatibility has been implemented or validated.
