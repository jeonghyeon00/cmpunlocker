# 170tune HBM validation record

This document records the live HBM qualification performed on 2026-09-03. It is a result for one
card, not a guaranteed setting for every CMP 170HX.

## Tested system

- GPU: NVIDIA CMP 170HX 8GB (`0x20C2`), serial `1322421000100`
- Unlocked memory geometry: 64GB (`65536 MiB`)
- NVIDIA kernel and userspace version: `610.43.02`
- Stock HBM clock: NDIV 64 (`1728 MHz`)
- Stock refresh field: `REFRESH 6`
- cmpunlocker branch: `codex/hbm-memory-overclock`
- PLM fix commit: `5927762`
- Driver installation: no `--mclk-ndiv`; the card always starts from its stock NDIV

Boot-time preflight confirmed that both live-tuning privilege windows were open:

```text
PLL PLM  0x00903C7C = 0xFFFFFFDF (host writes enabled)
MEM PLM  0x009A0168 = 0xFFFFFFFF (host writes enabled)
```

## Qualification results

Each row was qualified with `170tune hbm-gate`, four hot sweeps, 95% resident VRAM coverage
(`61376 MiB`), and the compute checker. Every recorded sweep returned `mem_errors=0` and
`compute_ok=1`.

| NDIV | HBM clock | Timings | Sweeps | Peak HBM | Result |
|---:|---:|---|---:|---:|---|
| 64 | 1728 MHz | stock (`REFRESH 6`) | 4/4 | 60 C | qualified |
| 65 | 1755 MHz | stock (`REFRESH 6`) | 4/4 | 65 C | qualified |
| 66 | 1782 MHz | stock (`REFRESH 6`) | 4/4 | 65 C | qualified |
| 67 | 1809 MHz | stock (`REFRESH 6`) | 4/4 | 66 C | qualified |
| 68 | 1836 MHz | stock (`REFRESH 6`) | 4/4 | 65 C | qualified |
| 69 | 1863 MHz | stock (`REFRESH 6`) | 4/4 | 64 C | qualified |
| 70 | 1890 MHz | `REFRESH 24` | 4/4 | 64 C | qualified |

No new Xid or uncorrectable-memory error was observed during this session. The gates ran without
the production vLLM container, which was not running after the reboot. Requalify under the actual
long-running workload before promoting a more aggressive profile.

## Conservative active profile

The selected initial profile is NDIV 66 (`1782 MHz`, approximately 3.1% above stock) with all stock
timings, including `REFRESH 6`. This is intentionally below the highest qualified profile.

```bash
sudo 170tune persist save --ndiv 66
sudo 170tune persist enable
```

The exact per-card gate receipt is:

```text
/var/lib/170tune/gated-hbm/1322421000100/2776e45749e638f5.json
```

Verify the live state with the direct PLL tool because `nvidia-smi` continues to display the VBIOS
memory clock for a userspace PLL change:

```bash
sudo hbm_mclk get
sudo fbpa_regs get REFRESH
sudo 170tune persist status
```

Expected values are NDIV 66 (`1782 MHz`), PLL lock 1, and `REFRESH 6`.

## Recovery

For a live fault or failed gate:

```bash
sudo 170tune recover
```

To prevent automatic reapplication on the next boot:

```bash
sudo systemctl mask 170tune-persist.service
```

If the GPU no longer responds, perform a cold power cycle. A reboot starts from stock because the
driver was installed without `--mclk-ndiv`; persistence is applied later by a userspace oneshot
service. Never persist a profile without an exact gate receipt.
