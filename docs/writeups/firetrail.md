---
icon: material/fire
---

# Firetrail

**A zero-allocation, hash-based word-dictionary compressor written in Zig - engineered for one thing: absolute maximum throughput.**

[:simple-github: Source Repository](https://github.com/VantorreWannes/firetrail){: .md-button .md-button--primary }
[:material-speedometer: Skip to benchmarks](#benchmarks){: .md-button }

---

!!! abstract "Project overview"
`firetrail` is a zero-allocation, generic word-dictionary lossless compressor built in Zig, inspired by the Density (Chameleon) family. It makes a single, extreme trade-off - **ratio for throughput** - and executes it with one hash-based lookup stage: no match finder, no entropy coder, nothing but a 512 KiB table between input and output. The result is encode throughput 1.6×–2.5× ahead of its speed-tier peers (`lz4 --fast=20`, `zstd --fast=5`), while still beating lz4's ratio on natural text.

## How it works

All three modes share the same core: hash 8-byte words into a 512 KiB lookup table; on a match, emit a 2-byte hash - otherwise emit the raw word.

```text
8-byte word ──hash──▶ 512 KiB LUT
                       ├─ match → emit 2-byte hash
                       ╰─ miss  → emit raw word
```

| Mode     | Profile       | Lookup table                                    |
| -------- | ------------- | ----------------------------------------------- |
| `red`    | Ratio-focused | Trains the LUT and **exports** it (512 KiB)     |
| `orange` | Balanced      | Self-contained - no external LUT                |
| `white`  | Speed-focused | **Imports** the LUT `red` exported (`--import`) |

## Benchmarks

Everything is single-threaded. Baselines are `zstd v1.5.7 --fast=5` and `lz4 v1.10.0 --fast=20` (both pinned to `-T1`), chosen as speed-tier peers - no claims are made about zstd at default levels.

### At a glance - geomean across all corpora

| Codec                     | Encode (MB/s) | Decode (MB/s) | Size (% of original) |
| ------------------------- | ------------: | ------------: | -------------------: |
| `firetrail red`           |         1,167 |         1,331 |                 63.4 |
| `firetrail orange`        |         1,674 |         1,895 |                 63.6 |
| `firetrail white`[^white] |     **1,760** |         1,983 |                 69.1 |
| `zstd --fast=5`           |           717 |         1,703 |             **40.2** |
| `lz4 --fast=20`           |         1,097 |     **2,629** |                 56.2 |

[^white]: `white`'s compressed sizes exclude the 512 KiB LUT, which in this test was trained on the same file being compressed. Amortized at these sizes that's +0.03–0.52 pp, so the picture doesn't change here - but it would be unfair on small files, and real deployments would train on a sample set.

### Full results

=== "Encode throughput (MB/s)"

    | Codec              |    silesia |     HDFS |   enwik9 |   enwik8 |  geomean |
    | ------------------ | ---------: | -------: | -------: | -------: | -------: |
    | `firetrail red`    |      1,081 |    1,755 |    1,018 |      962 |    1,167 |
    | `firetrail orange` |      1,536 |    2,417 |    1,513 | **1,399**|    1,674 |
    | `firetrail white`[^white] | **1,669** | **2,716** | **1,548** | 1,368 | **1,760** |
    | `zstd --fast=5`    |        688 |    1,214 |      602 |      526 |      717 |
    | `lz4 --fast=20`    |        981 |    1,252 |    1,109 |    1,064 |    1,097 |

=== "Decode throughput (MB/s)"

    | Codec              |    silesia |     HDFS |   enwik9 |   enwik8 |  geomean |
    | ------------------ | ---------: | -------: | -------: | -------: | -------: |
    | `firetrail red`    |      1,225 |    1,856 |    1,215 |    1,138 |    1,331 |
    | `firetrail orange` |      1,781 |    2,446 |    1,815 |    1,631 |    1,895 |
    | `firetrail white`[^white] | 1,892 |    2,566 |    1,869 |    1,704 |    1,983 |
    | `zstd -d`          |      1,656 |    2,036 |    1,616 |    1,543 |    1,703 |
    | `lz4 -d`           | **2,526** | **2,961** | **2,793** | **2,288** | **2,629** |

=== "Compressed size (% of original)"

    | Codec              |    silesia |     HDFS |   enwik9 |   enwik8 |  geomean |
    | ------------------ | ---------: | -------: | -------: | -------: | -------: |
    | `firetrail red`    |       65.6 |     41.2 |     74.2 |     80.7 |     63.4 |
    | `firetrail orange` |       64.8 |     39.5 |     76.5 |     83.6 |     63.6 |
    | `firetrail white`[^white] | 76.6 |     45.7 |     78.7 |     83.0 |     69.1 |
    | `zstd --fast=5`    |   **48.6** | **15.6** | **54.9** | **62.5** | **40.2** |
    | `lz4 --fast=20`    |       64.1 |     22.1 |     79.5 |     88.9 |     56.2 |

=== "Raw output size (bytes)"

    | Codec   | silesia.tar |   HDFS.log |     enwik9 |    enwik8 |
    | ------- | ----------: | ---------: | ---------: | --------: |
    | `red`   | 138,960,934 | 650,174,322| 742,118,280| 80,693,334|
    | `orange`| 137,280,094 | 623,821,368| 764,848,386| 83,586,414|
    | `white`[^white] | 162,270,340 | 720,875,910| 787,229,844| 83,038,608|
    | `zstd`  | 103,066,261 | 246,427,519| 548,501,676| 62,529,666|
    | `lz4`   | 135,964,759 | 348,070,094| 794,588,177| 88,919,770|

### Key takeaways

- **Encode (geomean):** `white` and `orange` are 2.45× / 2.33× vs. `zstd --fast=5`, and 1.60× / 1.53× vs. `lz4 --fast=20`. `red`: 1.63× / 1.06×.
- **Natural text:** `red`/`orange` beat lz4's ratio while encoding ~40% faster (enwik9: `red` 742.1 MB vs. lz4 794.6 MB).
- **Structured logs:** lz4 wins ratio comfortably on HDFS (22.1% vs. `orange` 39.5%) - the comparison is content-dependent.
- **Decode order is identical on every corpus:** lz4 → white → orange → zstd → red. `red` is the only mode that loses decode to zstd.
- **zstd remains the ratio king** by a mile (6.4:1 on HDFS). `firetrail` targets the lz4-and-faster tier, not zstd's Pareto point.

??? info "Test setup & methodology"

    **Machine**

    - AMD Ryzen 7 7800X3D (Zen 4, 8C/16T, 96 MB L3, max 4.2 GHz, no frequency pinning)
    - CachyOS x86-64, kernel 7.1.5-1-cachyos
    - `firetrail` built with `zig build --release=fast`

    **Method**

    - Wall-time mean over N runs (4–115 depending on input size), warm page cache
    - Round-trip verified with `cmp`
    - Single-threaded throughout; baselines pinned to `-T1`

    **Corpora**

    | File          |         Bytes | Notes                                 |
    | ------------- | ------------: | ------------------------------------- |
    | `silesia.tar` |   211,948,544 | standard tarball                      |
    | `HDFS.log`    | 1,577,982,906 | loghub HDFS_v1, structured log text   |
    | `enwik9`      | 1,000,000,000 | Wikipedia XML dump                    |
    | `enwik8`      |   100,000,000 | Wikipedia XML dump (first 10⁸ bytes)  |

## Why it's fast

| Metric                               | `firetrail` | `lz4 --fast=20` | `zstd --fast=5` |
| ------------------------------------ | ----------: | --------------: | --------------: |
| Instructions per input byte (encode) | **1.2–2.0** |         4.2–4.9 |       10.6–20.5 |

- `white` decode runs at **under 1 instruction per output byte** on enwik/silesia.
- RSS is flat at **~15–20 MB** from 95 MiB to 1.5 GiB inputs - genuinely streaming. (For reference: `zstd -d` ~6 MB, `lz4 -d` ~36 MB.)
- Known weak spot: `red` is branch-mispredict heavy (~2.9% of instructions, ~0.8 IPC on text) - that's the current optimization target.
