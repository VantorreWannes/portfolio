---
icon: material/zip-box
---

# skim

**The fastest block compression algorithm implementation in the world.**

[:simple-github: Source Repository](https://github.com/VantorreWannes/skim){: .md-button .md-button--primary }

---

!!! abstract "Project Overview"
    `skim` is a zero-allocation `u64` word dictionary compression algorithm built in Zig, heavily inspired by the Density (Chameleon) algorithm. It is engineered with a singular focus: absolute maximum data throughput. Benchmarks demonstrate that it is currently the fastest implementation of this compression paradigm globally.

## The Profile & Impact

`skim` makes a single, extreme trade-off: **It sacrifices compression ratio to achieve absolute maximum throughput.** Under the hood, it avoids complex math and relies entirely on a hash-based lookup table, allowing it to operate near hardware memory-bandwidth limits.

* **Speed:** ~12.6 GB/s encoding | ~15.5 GB/s decoding
* **Efficiency:** Compresses 100MB in 34ms (using ~80% less memory than `xz`).
* **Ratio:** Low.

### Targeted Use Cases
Because `skim` requires almost zero CPU overhead, it is designed specifically for situations where traditional compression (like Zstandard or gzip) is too slow to be viable on the hot path:

1. **In-Memory Caching:** Compressing database pages or RAM caches where read/write speed is the primary bottleneck.
2. **High-Volume IPC:** Shrinking Inter-Process Communication payloads without stalling the pipeline.
3. **Real-Time Telemetry:** Buffering massive streams of unoptimized log/sensor data on the fly.

---

## Technical Architecture

=== "Algorithmic Foundation"

    The algorithm is fundamentally a `u64` word dictionary coder. It operates on a high-speed tokenization loop: reading multi-byte words, computing a mathematically optimized hash, and checking a pre-allocated Lookup Table. If the word was seen recently, it emits a bit-packed header and the short hash signature. If not, it stores the word and emits the raw bytes.

=== "Zero-Cost Hash Engine"

    To prevent the hashing step from slowing down the compression loop, `skim` utilizes a custom `NumberHasher`. By leveraging Zig's compile-time generics (`comptime`), the engine uses wrapping multiplication (`*%`) against prime constants and bitwise shifts (`>>`) to compute hashes in essentially a single CPU cycle.

=== "Memory & Safety Bypasses"

    Absolute speed requires absolute control. Memory is allocated exactly once during `init()`. During the `compressBlockToBuffer` cycle, there are exactly zero allocations. Furthermore, in the absolute hottest path of the loop, Zig's standard runtime bounds checking is explicitly disabled to maximize raw L1 cache throughput.

---

## Implementation Highlight

Below is the core of the compression loop. Notice the deliberate use of `@setRuntimeSafety(false)`, a decision made carefully after rigorous mathematical validation to prevent the compiler from inserting CPU-taxing bounds checks on every memory read.

```zig title="src/root.zig" hl_lines="6 14 15"
pub fn compressBlockToBuffer(self: *Self, input:[]const u8, output:[]u8) usize {
    std.debug.assert(output.len >= outputBufferBound(input.len));

    // In the hottest path of the application, runtime safety checks are 
    // explicitly bypassed to unlock maximum L1 cache throughput.
    @setRuntimeSafety(false);

    while (input_index < loop_limit) {
        const header_pos = output_index;
        output_index += HEADER_BYTES;
        var header: Header = 0;

        inline for (0..HEADER_BITS) |token_index| { // (1)!
            const word: Word = std.mem.readInt(Word, input[input_index..][0..WORD_BYTES], .little);
            const hash = Hasher.hash(word);
            
            // Evaluates lookup tables and packs bits into the header...
        }
    }
}
```

1. The `inline for` construct forces the Zig compiler to fully unroll the inner loop, eliminating branch prediction penalties entirely.

---

## Usage Interfaces

`skim` provides both a standalone executable and an embeddable library.

=== "CLI Tool"
    
    The executable is incredibly simple and memory-maps files for zero-copy reads, ensuring disk I/O is the only bottleneck.

    ```bash
    # Compress a file
    skim -c <input_file> <output_file>

    # Decompress a file
    skim -d <input_file> <output_file>
    ```

=== "Zig Library"
    
    `skim` can be seamlessly embedded into other Zig projects via `build.zig.zon`. 
    
    To see the canonical implementation of how to initialize the memory arena, encode data to a buffer, and decode it back, **inspect the existing source code in `src/main.zig` and `src/bench.zig`**. These files serve as the single source of truth for programmatic integration.

---

## Build System Integration

!!! info "Advanced Toolchain"
    The project relies on a modern `build.zig` pipeline, enabling seamless cross-compilation and the ability to strictly toggle the LLVM backend (`-Dllvm`) for further micro-optimizations depending on the deployment target architecture.
