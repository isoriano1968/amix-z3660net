# Changelog

## 2026-07-26 — z3660eth defect packet (P0.3): three deferred STREAMS defects fixed

Found during the 2026-07-13 STREAMS "out of blocks" investigation and deliberately deferred then
to avoid perturbing a live RX-path hunt (that hunt closed: the wedge was the cdfs page-crossing
DMA wild write, since fixed). Host cross-build verified only — **bench/metal validation owed.**

- **[HIGH] RX trailer-arm removed** (`54729bb`). `toss_packet_up_stream()` carried stock `aen.c`'s
  4.2BSD trailer decode verbatim. An ethertype in 0x1000–0x1010 made `length = (sap-0x1000)*512`
  reach 8192 and read the trailer-count word at offset up to 8208 of the 1518-byte `rxbuff`
  (~6.7 KB out-of-bounds read); the derived `trailn` could go negative, making
  `allocb(length+trailn)` smaller than the following `bcopy(..., length)` — an mblk heap overflow.
  Dead for legitimate traffic (SVR4 IP never negotiates trailers; `IFF_NOTRAILERS`;
  `ifconfig -trailers`) but reachable by any crafted frame off the wire. Deleted rather than
  repaired. **Normal RX is byte-for-byte unchanged** — proven by exhaustive case split on the
  ethertype guard and confirmed by cross-disassembly (the range test, the ×512 shift and the
  indexed OOB read are gone; the former else-arm is now unconditional; nothing else in the
  function changes). Frames in 0x1000–0x1010 now demux on that sap, bind to no stream, and count
  as `rx_no_sap` — correct handling for an encapsulation this stack never negotiated.
- **[MED] ioctl NULL-deref fixed** (`cd97e53`). `Z3660ETH_NUMBER_OF_BOARDS` and
  `Z3660ETH_GET_STATUS` dereferenced `mp->b_cont->b_rptr` with no NULL check (the same defect is
  still live in stock `aen.c`); an `I_STR` ioctl with no data panicked the kernel from any process
  able to open `/dev/zen0`. New `z3660eth_ioc_arg()` / `z3660eth_ioc_nak()` helpers add the NULL
  and short-message checks and give the driver one NAK path.
- **[LOW] three items** (`364333c`): `DL_ERROR_ACK`/`DL_BADPRIM` now emitted for unknown DLPI
  primitives (previously silently freed, hanging the DLS user); `z3660eth_tx_drain()`
  `return` → `break` (one blocked stream no longer starves the other four for a poll tick);
  `Z3660ETH_CLEAR_STATUS` / `Z3660ETH_GET_CONFIG` given explicit EINVAL-NAK case labels.

Build evidence: driver cross-compile diagnostics byte-identical to baseline at every step and
`nm -u exp` symbol set unchanged (27); full kernel cross-link green (`unix` 1575484 B, `nm -u`
resolved, `unix.brel` 29250 relocs, `relsim.py` **0 complaints**).

Owed at the next bench run: RX soak confirming normal IP/ARP unchanged + `rx_no_sap` ticking for a
crafted 0x1000-ethertype frame; the status tool's TRANSPARENT ioctl still correct plus a malformed
`I_STR` returning EINVAL instead of panicking (needs a small test program that does not exist yet);
a bogus DLPI primitive drawing `DL_ERROR_ACK`; multi-stream TX exercising the `break` path.
