# LANforge cross-tool campaign

The LANforge L4 HTTPS campaign cited in Section 8.2 of the paper, corroborating
the parity claim under a different traffic-generation model. The values below
are computed from the per-endpoint CSV exports in this directory; each condition
is a single 120 s run.

## Setup

| Element | Configuration |
| --- | --- |
| DUT | EAP101, Dome per condition |
| Foreground | 100 LANforge L4 endpoints, HTTPS GET of a fixed 10 KB object, over a Raspberry Pi 5 station (Intel BE200, 5 GHz HE 80 MHz, channel 100, RSSI -33 dBm, 1200 Mbps PHY) |
| Wired background | LANforge L3 TCP pair between eth2 and eth3 across the DUT bridge, 1 Gbps offered (500 threads, 2 Mbps each, 1460 B payload, 1500 B MTU) |
| Background ports | outside Dome's L7 inspection set (DNS/HTTP/TLS on 53/80/443), so the bridge `dpi` chain does not mark them for the Lua hook |
| Run length | 120 s per condition, first 15 s discarded as warmup |
| Repetitions | 1 per condition |

Conditions: **baseline** (Linux bridge, no Dome), **Dome** (verdict cache
active), **Dome without cache** (cache rules removed, Lua hook on every packet).

## Reduction

Steady-state slice `[t0 + 15 s, t0 + 120 s]`, where `t0` is the first LANforge
sample timestamp; all metrics are over this 105 s slice.

- **L4 URL rate**: sum of `Δurls_processed` across the 100 `wlan` endpoints over
  the slice, divided by its duration. Per endpoint, the slice is bracketed by
  the last sample with `ts <= t_lo` and the last with `ts <= t_hi`.
- **L4 throughput (Mbps)**: sum of `Δrx_bytes` across the same endpoints over
  the slice.
- **Background TCP (Mbps)**: median of `bps_rx_rate_3s` on the `eth2-eth3-A`
  endpoint over the slice (single direction; the ACK direction, about 27 Mbps,
  is excluded).
- **Within-run CV**: coefficient of variation of the URL rate across the ten
  consecutive 10 s sub-windows that span the slice. At 3 s sub-windows the 1 Hz
  LANforge sampling cadence dominates the variance, so 10 s windows are used to
  reflect workload variance rather than sampling jitter.

## Results

| Condition | URLs/s | L4 Mbps | BG Mbps | CV (10 s windows) |
| --- | ---: | ---: | ---: | ---: |
| baseline | 6 180 | 506 | 940 | 2.9 % |
| Dome | 5 994 | 491 | 940 | 3.6 % |
| Dome without cache | 924 | 76 | 739 | 0.6 % |

Per-window URL rate (ten 10 s windows per condition):

- **baseline**: 6158, 6182, 6428, 5903, 6277, 6324, 5893, 6368, 6264, 6223
- **Dome**: 6168, 5993, 6205, 6376, 5802, 6478, 5920, 5876, 6188, 6053
- **Dome without cache**: 913, 922, 929, 934, 922, 926, 918, 918, 925, 922

Dome tracks baseline: 5 994 versus 6 180 URLs/s (within 3.0 %) on the foreground
and 491 versus 506 Mbps on L4 throughput, with the wired background at 940 Mbps
in both. Without the cache the foreground drops to 924 URLs/s (76 Mbps), 6.5x
below baseline, and the background falls to 739 Mbps as the four Cortex-A53
cores saturate under per-packet Lua.

## Caveats

1. **Single run.** One 120 s run per condition, not the ten-repetition medians
   with bootstrap 95 % CIs of the `wrk2` production cells. Within-run CV is
   below 4 % on the baseline and Dome cells and below 1 % without the cache;
   between-run variance is unmeasured.
2. **No latency.** The L4 endpoint's `urllat_*` columns are empty
   (`urllat_width = 0`), so p50 and p99 are not recoverable from these exports;
   the campaign reports throughput-side metrics only.
3. **DUT CPU not here.** LANforge samples CPU on its own host, not the DUT; the
   DUT sys% characterisation is in the `wrk2` matrix (Table 2).

## Provenance

Captured 2026-05-24. The per-endpoint CSVs in this directory are the primary
source: per condition, 100 `wlan<N>_<ts>.csv` (with `urls_processed` and
`rx_bytes`) and one `eth2-eth3-A*.csv` (with `bps_rx_rate_3s`). The values in
this file are computed from those CSVs over the steady-state slice above.

