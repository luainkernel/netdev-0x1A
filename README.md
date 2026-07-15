# *Scripting Netfilter with Lua*: measurement data

Measurement data for the evaluation (Section 8) of **"Scripting Netfilter with
Lua: A Cooperative Kernel-Userspace Pipeline"** (Netdev 0x1A). Each top-level
directory corresponds to one experiment; Section 8 of the paper describes how
the reported medians and bootstrap confidence intervals are computed from these
logs.

The paper, slides, and abstract are on the [Netdev 0x1A session page](https://netdevconf.info/0x1A/sessions/talk/scripting-netfilter-with-lua-a-cooperative-kernel-userspace-pipeline.html).

The device under test (DUT) throughout is an Edgecore EAP101 (Qualcomm IPQ6018,
four ARM Cortex-A53 cores at 1.8 GHz) running OpenWiFi. Three conditions recur:

- **`baseline`**: plain Linux bridge, Dome not loaded
- **`dome`**: Dome with the nftables verdict cache active
- **`dome-no-cache`**: Dome with the cache rules removed, so the Lua hook runs on every packet

## `wrk2/`: Table 2 and the aggregate-throughput figure

A combined Ethernet and Wi-Fi HTTPS workload driven by `wrk2`, one generator per
path. Directories are `<condition>/<size>/`, over response sizes 10, 100, 300,
and 1024 KB; each cell holds 10 repetitions of 30 s.

| File | Contents |
| --- | --- |
| `wrk-eth<n>-rep<NN>.log` | `wrk2` report (HdrHistogram latency and request rate) for the Ethernet generator, repetition `NN` |
| `wrk-wlan<n>-rep<NN>.log` | `wrk2` report for the Wi-Fi generator |
| `dut-sample-rep<NN>.log` | DUT `/proc/stat`, one line per second; the source for the sys% column |
| `dut-netstat-rep<NN>-{pre,post}.txt` | DUT interface byte and packet counters, captured before and after the repetition (wire-level throughput at the DUT) |
| `params.txt` | cell configuration, including the calibrated per-generator target request rates |

## `lanforge/`: Table 3, the cross-tool campaign

A LANforge L4 HTTPS campaign: 100 endpoints each fetching a fixed 10 KB object
over Wi-Fi, concurrent with a 1 Gbps wired TCP background traversing the DUT
bridge; one repetition per condition.

- `<condition>/`: the per-endpoint CSV time series exported by LANforge, 100
  `wlan<N>_<ts>.csv` files (per-sample `urls_processed`, `tx_bytes`, `rx_bytes`,
  among other columns) and one `eth2-eth3-A*.csv` for the wired background.
- `measurements.md`: the per-condition measurements and how they are computed
  from those CSVs.

---

IP addresses in the logs are RFC1918 lab-internal addresses.

## Netdev 0x1A BoF: *Your Fu is Better Than Mine, 3.0!*

The same Netdev 0x1A hosted the BoF *Your Fu is Better Than Mine, 3.0!*, where we
demonstrated a vibe-coded [lunatik](https://github.com/luainkernel/lunatik) tool
that answers "why is my packet dying?": a 57-line Lua kprobe on
`kfree_skb_reason()` that names every drop in the system from the kernel's own
headers, counts them in a table queried live from the in-kernel REPL, and traps
one chosen reason to dump the registers and call site of the exact drop.

The code lives on lunatik's
[`netdev0x1A-fu`](https://github.com/luainkernel/lunatik/tree/netdev0x1A-fu)
branch, and [`LUA-FU.md`](LUA-FU.md) is the companion artifact: the (translated)
prompts that produced that script, and the steps to reproduce the demo.

