# Changelog

All notable changes to this project are documented here.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/), and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [Unreleased]

## [0.4.6] - 2026-09-03

### Added

- Generate the complete TEE, REE, PCPU, and AIDSP keyslot flush-status
  register family, including read-only busy, unlock-failure, and timeout-error
  fields.

## [0.4.5] - 2026-08-07

### Fixed

- Rename interrupt 47 from the misleading `SLE_INT` label to `GLE_INT`,
  matching the vendor SDK's BGLE baseband interrupt identity.

## [0.4.4] - 2026-08-04

### Added

- Generate the read-only WLMAC CCMP/TKIP replay, MIC, and key-search failure
  counters used by connectivity diagnostics.

## [0.4.3] - 2026-07-30

### Changed

- Rename the WLMAC diagnostic accessor from `rx_filter_command` to
  `rx_filter_control` because the register contains the packed state produced
  by the mask-ROM filter helper, not its input command.

### Fixed

- Correct the generated VAP0 station/BSSID field descriptions to network byte
  order, matching the register layout verified on WS63 silicon.

## [0.4.2] - 2026-07-30

### Added

- Generate read-only WLMAC receive-filter command and VAP0 station/BSSID
  diagnostic accessors for post-association data-path evidence.

## [0.4.1] - 2026-07-29

### Added

- Generate the read-only `WlmacStats` register block used to snapshot the six
  receive counters consumed by the WS63 mask-ROM statistics helper.

## [0.4.0] - 2026-07-17

### Added

- Model the SPACC hash-channel control, descriptor-ring, completion, error, and
  state-port registers required by the token-owned SHA/HMAC backend.
- Model SPACC symmetric DMA rings and KM/KLAD keyslot/clear-key registers needed
  by the token-owned AES backend.
- Add the `GLP_UART_RX_WAKE_INT` and `TIMING_GEN_INT` interrupt vectors and
  linker defaults.
- Complete the SSI v151 SPI0/SPI1 register window, including the 36-entry data
  register array and the missing DMA, interrupt-clear, ID, and extension
  registers.

### Changed

- **Breaking:** replace the incorrect single SPI data register at offset `0x60`
  with `spi_drnm(n)` covering the vendor-defined window at `0x2c..0xb8`.
- **Breaking:** regenerate UART registers as 32-bit register accesses while
  preserving each field's effective width.

### Fixed

- Correct SPACC clear-command, bus-error, and clear-finish access modes so
  consumers cannot read a write-only command or write a read-only status.
- Correct the AES key-length encodings and symmetric/KLAD W1C access modes.
- Correct audited read-only and write-only access semantics across UART, GPIO,
  I2C, PWM, DMA, SFC, and TSENSOR.
- Remove the duplicate PKE interrupt association from SPACC; IRQ 63 remains
  owned by PKE and IRQ 64 by SPACC.

## [0.3.0] - 2026-07-14

### Changed

- **Breaking:** regenerate timer EOI registers as write-only command fields.
  Consumers must write a complete command and can no longer call `read()` on
  these registers.

### Added

- Add the WS63 `SOFT_INT0`/`SOFT_INT1` interrupt vectors and generated
  `device.x` entries required by the single-hart scheduler port.

## [0.2.4] - 2026-07-14

Compatibility release from the `0.2.x` maintenance branch. It adds the software
interrupt vectors while retaining the old timer EOI API for already-published
HAL versions.

## [0.2.3] - 2026-07-14

Superseded by `0.2.4`: this release accidentally exposed the breaking timer EOI
access correction under a patch version. New consumers should use `0.3.0`;
existing `0.2.x` consumers resolve to the compatibility release `0.2.4`.

## [0.2.2] - 2026-07-12

### Added

- Generated shared-RAM bank/clock fields and the `BT_EM_CTL` peripheral used by
  WS63 Wi-Fi runtime memory configuration.
- Generated RF power-control registers used by the WS63 connectivity runtime.
- Generated CMU factory XO-trim fields used to preserve calibrated clock state.
- Generated `RiscvPatch` peripheral for the mask-ROM instruction patch
  controller and its 192 comparison registers.

## [0.2.1] - 2026-07-09

### Fixed

- Regenerated from the WS63 SVD register-access audit: adds the missing CMU
  block and CLDO_CRG/SYS_CTL1 clock-control fields used by the HAL clock, SPI,
  PWM, I2S, and DMA code paths.
- Tightened generated register access semantics for audited write-only /
  read-write fields so HAL code can avoid raw MMIO or stale read-modify-write
  patterns.
- Updates the nested `ws63-svd` pointer to the versioned mainline SVD release
  used for this PAC generation.

## [0.2.0] - 2026-06-15

Breaking: the SPI_WSR and TIMER fixes below change the generated public API
(removed/renamed accessors + changed `Mode` enum discriminants), so this is a
minor (0.x-breaking) bump, not a patch.

### Fixed

- **SPI_WSR (status register) bit layout** corrected to match the HiSilicon SSI
  v151 silicon (vendor `hal_spi_v151_regs_def.h` `spi_wsr_data`), which spreads the
  flags out instead of packing them like the textbook DesignWare SR:
  `rxfne` = bit 4, `rxff` = bit 5, `txfnf` = bit 11, `txfe` = bit 12,
  `busy` = bit 15, `dcol` = bit 0 (was the wrong `busy` = 0 / `txfnf` = 1 /
  `txfe` = 2 / `rxfne` = 3, plus bogus `rxfo`/`txfo`/`dcm`). With the old layout
  `txfnf` read a reserved bit that is always 0, so the HAL's SPI transfer spun until
  it returned `Timeout` on every call. Reset value updated to 0x1800 (TXFNF|TXFE,
  the idle state). Verified on WS63 silicon 2026-06-14: SPI0 MOSI→MISO HIL loopback
  now round-trips a 4-byte buffer (previously `Err(Timeout)`).
- **TIMER `%s_CONTROL`**: added the `cnt_req` (bit 5, rw) and `cnt_lock` (bit 6, ro)
  fields — the current-count latch handshake. They were missing; reading
  `CURRENT_VALUE` on real silicon needs them (it is a latch, not a live counter),
  so drivers previously had to poke raw bits. Verified against the vendor
  `hal_timer_v150` + on WS63 silicon.
- **TIMER `%s_CONTROL.mode`** enum corrected to the silicon/vendor
  `control_reg_mode_v150_t` values: `OneShot = 0`, `Periodic = 1`, `FreeRun = 3`
  (was the wrong `FreeRunning = 0 / OneShot = 1 / Periodic = 2`).

Regenerated from `ws63-svd/WS63.svd` via `regen.sh` (svd2rust 0.37.1); DMA and the
other blocks are unchanged (the DMA register layout was already silicon-correct —
the earlier DMA bring-up gap was a HAL/QEMU issue, not a PAC one).

## [0.1.3] - 2026-06-02

### Changed

- CI: first release cut by ws63-pac's own repo pipeline (no functional change since 0.1.2).

## [0.1.2] - 2026-06-02

### Added

- Vendored ws63-svd as a nested submodule (the PAC's generation source).

### Changed

- Regenerated from WS63.svd with eFuse/LSADC fixes and reproducible generation pipeline.

### Fixed

- Fixed ws63-pac to compile on non-RISC-V hosts by cfg-gating riscv-specific coupling.

## [0.1.1]

### Added

- KM keyslot registers: `KC_REECPU_LOCK_CMD`, `KC_PCPU_LOCK_CMD`, `KC_AIDSP_LOCK_CMD`,
  `KC_RD_SLOT_NUM` (additive, backwards-compatible API surface — hence the 0.1.1 bump).

### Note

- This 0.1.1 bump exists because public registers were added after the `v0.1.0` tag.
  Re-publishing `0.1.0` with a changed API would violate SemVer; downstream consumers
  (`ws63-hal`) require these registers, so the version is bumped accordingly.

## [0.1.0]

### Added

- SVD-based peripheral access crate for HiSilicon WS63 (RISC-V RV32IMFC_Zicsr)
- Register block definitions for all 35 on-chip peripherals via svd2rust
- Type-safe `Peripherals` singleton with `take()` and `steal()` access patterns
- External interrupt enum (`ExternalInterrupt`) covering all interrupt sources
- Interrupt vector table (`device.x`) with PROVIDE weak defaults
- Critical section support via `critical-section` feature
- Runtime support via `rt` feature (startup assembly + linker scripts)

### Changed

- Consumed by `ws63-hal` and `ws63-rt` as a registry (`version`) dependency; inside the
  `ws63-rs` monorepo it is redirected to the local path via `[patch.crates-io]`.
