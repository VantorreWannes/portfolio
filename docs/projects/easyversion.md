---
icon: material/server
---

# EasyVersion

**An artists and digital creatives first version control system.**

[:simple-github: Source Repository](https://github.com/VantorreWannes/easyversion){: .md-button .md-button--primary }

---

!!! abstract "Project Overview"
    Standard version control systems like Git are optimized for line-by-line text diffing, making them notoriously inefficient and complex for 3D artists, game developers, and musicians handling massive binary assets. EasyVersion (`ev`) replaces branching and committing with a simplified "Save and Split" mental model, utilizing global deduplication to handle large files seamlessly.

## Core Mechanics & Impact

For the end-user, EasyVersion entirely eliminates the infamous `Project_Final_v2` folder clutter. 

Unlike traditional VCS tools, `ev` does not pollute the working directory with hidden tracking folders (like `.git`). Instead, it securely routes all snapshots to a centralized, OS-level data directory. This ensures the user's workspace remains perfectly clean while background deduplication drastically minimizes disk usage across all their projects.

---

## Technical Architecture

=== "Storage Engine (CAS)"

    At its core, EasyVersion is built on a **Content-Addressable Storage (CAS)** model written in Rust. 
    
    Files are not stored by their path or name, but by a deterministic `GxHash` signature of their contents. A thread-safe `Mutex<HashMap<FileId, usize>>` tracks global reference counts. If a user modifies a 10GB project but only changes a single 2MB asset, `ev` simply increments the reference pointers for the unchanged data, storing only the new 2MB file.

=== "Streaming Pipeline"

    To prevent memory exhaustion when snapshotting massive binaries (e.g., raw audio or uncompressed textures), the system processes data in streams.
    
    A custom `HashingReader` wraps the file stream, computing the `GxHash` on-the-fly as the data is piped directly into an `lz4_flex` frame encoder. This guarantees a bounded, minimal memory footprint regardless of the file size.

=== "Advanced Optimization"

    To achieve maximum I/O throughput and minimal CPU overhead, the release binaries are compiled using advanced compiler toolchains. The build pipeline utilizes **Profile-Guided Optimization (PGO)** combined with **LLVM BOLT** layout optimization to restructure the compiled binary based on actual workload profiling, resulting in peak runtime execution speeds.

---

## Implementation Highlight

Below is the core function demonstrating the zero-allocation streaming pipeline. By wrapping the reader, the system calculates the file's cryptographic signature at the exact same time it compresses the data, requiring only a single pass over the disk.

```rust title="src/file/storage.rs" hl_lines="6 9 12"
fn compute_and_write<R: Read>(
    reader: &mut R,
    directory: &Path,
) -> anyhow::Result<(FileId, NamedTempFile)> {
    let temp_file = NamedTempFile::new_in(directory)?;

    // Computes the GxHash on the fly while data is read
    let mut hashing_reader = HashingReader::new(reader, GxHasher::default());
    
    // Streams data directly into the LZ4 compressor
    let mut writer = FrameEncoder::new(temp_file);
    io::copy(&mut hashing_reader, &mut writer)?;
    
    let temp_file = writer.finish()?;
    let hash = hashing_reader.finalize(); // Final hash retrieved after stream finishes
    
    Ok((FileId::new(hash), temp_file))
}
```

## Available Commands

!!! info "CLI Interface"
    * `ev save -c "Message"` — Takes a deduplicated snapshot of the current directory.
    * `ev list` — Displays the linear history of snapshots.
    * `ev split -p <Path>` — Reconstructs a previous snapshot into a completely separate, clean directory for safe experimentation.
    * `ev clean` — Safely decrements global reference counts and purges orphaned data chunks to free up system disk space.
