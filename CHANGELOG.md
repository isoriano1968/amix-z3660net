# Changelog

## 2026-07-30 — cdevsw major renumbered 48 → 51; Q1 (AMIX level-6 servicing) resolved

Renumbered `z3660eth`'s char major **48 → 51** repo-wide (`c949380`, date note corrected
`b9991e3`). 48 was chosen in 2026-06 only because it was an empty `nostr` row on the
stock/a4091 kernel; the 2026-07 community sweep shows it is contested — `zz9000-amix`'s
`integration/kernel.c.txt` documents an unpublished dev tree using "47 for Hydra, 48 for
random, 49 for ZZ9000, 50 for sad", 49 (zz9000) and 68 (va2000) are published claims, and 47
is a live double claim by hydra and the stock 1992 `contrib/mx` driver. 51 is the lowest slot
of the ecosystem-verified-free set 51–67/69 (CDEVSIZE = 70; 7/38/39/44/45 are traps).
Convention and evidence: grimoire-amix's Major-Number Registry.
Sites: `driver.conf`, `tools/patch-kernel.sh`, `userland/{S99zen,bringup.sh,network-config.zen}`,
`BUILD.md` (carries the authoritative why-51 note), `docs/ETHERNET-SCOPING.md`,
`docs/ETHERNET-SCOPING-evidence.md` (supersession banner + prescriptive sites).
**Docs-and-config only — built kernels and shipped `/dev/zen0` nodes still carry 48 until the
next build (P5.4). A 51-built kernel with a stale `c 48 0` node opens an empty cdevsw row, and
`S99zen` only mknods when the node is absent — existing roots need the stale node removed or
re-mknod'd at P5.4.**

Q1 RESOLVED (`f12b2d5`) — `docs/ETHERNET-SCOPING.md` §2.5 now reads resolved instead of OPEN.
AMIX does wire the level-6 autovector (`vec.s:36` → `p6int`), but `p6int` (`ttrap.s:118-134`) is
hardwired to CIA-B: it gates on Paula `INTREQR` bit 13, dismisses only Paula's latch, and calls
the single fixed handler `aciabintr`. There is no `int6_tbl` and no level-6 registration API.
So the Z3660's `/INT6` passes the gate but nothing ever acks the board, and because `/INT6`→EXTER
is level-sensitive it re-latches on every `rte` — the unbounded IPL6 storm behind the 2026-06-21
hard-lock. Confirms the shipped `ZZ_CONFIG_DISABLE` + bounded polled drain (`b06cf45`) as correct
architecture, not a workaround.

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
