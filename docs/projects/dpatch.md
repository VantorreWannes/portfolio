---
icon: material/source-commit
---

# dpatch

**A highly efficient binary delta diffing tool.**

[:simple-github: Source Repository](https://github.com/VantorreWannes/dpatch){: .md-button .md-button--primary }

---

!!! abstract "Project Overview"
    `dpatch` is a stateful delta patching library built in Zig. By analyzing the structural differences between a source and target byte sequence, it generates a highly compressed set of instructions—a "patch"—that can precisely reconstruct the target data from the original source without transmitting the entire file.

## Core Mechanics & Impact

In distributed systems, game development, and IoT environments, transmitting entire files for minor updates incurs heavy bandwidth costs and latency. 

`dpatch` solves this by calculating the exact minimal difference. Instead of sending a 100MB updated binary over the network, systems can transmit a fractionally sized patch. The client then applies this patch to their local 100MB file, seamlessly upgrading it while drastically reducing payload sizes and network congestion.

---

## Technical Architecture

=== "Algorithmic Foundation"

    The core differencing engine relies on a custom Longest Common Subsequence (LCS) implementation. By identifying the longest continuous blocks of shared data between the source and the target, the encoder guarantees an optimal sequence of `COPY` operations, strictly minimizing the need for raw data insertions.

=== "Smart Insertion Buffering"

    To further compress the output, `dpatch` employs an internal state buffer during encoding. If a new block of data needs to be inserted, the engine checks if this exact byte sequence was already inserted earlier in the patch stream. If a match is found, it emits a recursive `INSERT_COPY` instruction rather than duplicating the raw bytes in the payload.

=== "LEB128 Bytecode"

    Patch instructions are serialized into a highly compact, tag-based binary format. Integer values (such as read lengths and buffer offsets) are encoded using Unsigned Little Endian Base 128 (`leb128`). This variable-length encoding ensures that small offset integers consume only a single byte, maximizing spatial efficiency.

---

## Implementation Highlight

Below is the core serialization loop. It consumes the high-level `DPatchInstruction` union and writes the resulting bytecode to an allocating stream. Notice the strict instruction tagging system and the reliance on `leb128` to maintain a minimal binary footprint.

```zig title="src/root.zig" hl_lines="10 13 20"
while (try dpatch_encoder.next()) |delta| {
    switch (delta) {
        .insert => |instruction| {
            switch (instruction) {
                .data => |data| {
                    try writer.writeByte(INSERT_DATA_TAG);
                    try std.leb.writeUleb128(writer, data.len); // (1)!
                    try writer.writeAll(data);
                },
                .copy => |details| {
                    try writer.writeByte(INSERT_COPY_TAG);
                    try std.leb.writeUleb128(writer, details.start);
                    try std.leb.writeUleb128(writer, details.len);
                },
            }
        },
        .copy => |details| {
            try writer.writeByte(COPY_TAG);
            try std.leb.writeUleb128(writer, details.start);
            try std.leb.writeUleb128(writer, details.len);
        },
    }
}
```

1. Unsigned LEB128 encoding is used to compress `usize` types dynamically based on their magnitude.

---

## Robustness & Validation

!!! info "Memory Safety & Fuzz Testing"
    The library is built with strict memory safety in mind, passing custom allocators down the stack to ensure zero hidden allocations. The codebase is validated using **Fuzz Testing** (`std.testing.fuzz`) to programmatically ensure absolute encode/decode parity across thousands of randomized, mutated byte streams.
