---
icon: material/zip-box
---

# skim

**The fastest block compression algorithm implementation in the world.**

[:simple-github: Source Repository](https://github.com/VantorreWannes/skim){: .md-button .md-button--primary }

---

!!! abstract "Project Overview"
    `skim` is a zero-allocation block compression algorithm built in Zig, heavily inspired by the Density (Chameleon) algorithm. It is engineered with a singular focus: absolute maximum data throughput. Benchmarks demonstrate that it is currently the fastest implementation of this compression paradigm globally.

## Core Mechanics & Impact

While standard algorithms like Zstandard or gzip offer excellent compression ratios, they require significant CPU cycles, creating bottlenecks in high-frequency data pipelines. 

`skim` sacrifices a small amount of compression density in exchange for blazing, near-memory-limit speeds. It is designed for use cases where data must be compressed synchronously on the hot path without blocking the main CPU thread—such as high-frequency trading logs, real-time telemetry, or live memory dumping.

---

## Technical Architecture

=== "Algorithmic Foundation"

    The algorithm operates on a high-speed tokenization loop. It reads multi-byte words, computes a mathematically optimized hash, and checks a pre-allocated Lookup Table. If the word was seen recently, it emits a bit-packed header and the short hash signature. If not, it stores the word and emits the raw bytes.

=== "Zero-Cost Hash Engine"

    To prevent the hashing step from slowing down the compression loop, `skim` utilizes a custom `NumberHasher`. By leveraging Zig's compile-time generics (`comptime`), the engine uses wrapping multiplication (`*%`) against prime constants and bitwise shifts (`>>`) to compute hashes in essentially a single CPU cycle.

=== "Memory & Safety Bypasses"

    Absolute speed requires absolute control. Memory is allocated exactly once during `init()`. During the `compressBlockToBuffer` cycle, there are exactly zero allocations. Furthermore, in the absolute hottest path of the loop, Zig's standard runtime bounds checking is explicitly disabled to maximize raw CPU cache throughput.

---

## Implementation Highlight

Below is the core of the compression loop. Notice the deliberate use of `@setRuntimeSafety(false)`—a decision made carefully after rigorous mathematical validation to prevent the compiler from inserting CPU-taxing bounds checks on every memory read.

```zig title="src/root.zig" hl_lines="6 14 15"
pub fn compressBlockToBuffer(self: *Self, input: []const u8, output: []u8) usize {
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

## Build System Integration

!!! info "Advanced Toolchain"
    The project relies on a modern `build.zig` pipeline, enabling seamless cross-compilation and the ability to strictly toggle the LLVM backend (`-Dllvm`) for further micro-optimizations depending on the deployment target architecture.
