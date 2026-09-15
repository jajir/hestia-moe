# Proposed Java API for Hestia block reads

Status: design sketch, not an implemented Hestia API.

Date: 2026-09-14. Source audit:
`f063b0525703756b97f7a36a5369f875b1c59885`.

This proposal makes the [storage requirements](hestia-storage-requirements.md)
concrete. The first adapter can use today's general index. The proposed API
can be prototyped in this playground before deciding whether a dedicated
binary-reader view belongs in HestiaStore. It does not imply a new storage
engine or a new file format.

## 1. What we can call today

An opened `SegmentIndex<Long, ByteArray>` already supplies point reads.
This complete helper uses existing public API and a reusable heap buffer:

```java
import org.hestiastore.index.IndexException;
import org.hestiastore.index.datatype.ByteArray;
import org.hestiastore.index.segmentindex.SegmentIndex;

final class ExistingBlockReads {

    /**
     * Copies one complete stored value into the caller's heap buffer.
     * The caller owns and closes the already-open index.
     */
    static int copyStoredValue(
            SegmentIndex<Long, ByteArray> index,
            long blockId,
            byte[] destination) {
        ByteArray value = index.get(blockId);
        if (value == null) {
            throw new IndexException("Missing block: " + blockId);
        }
        if (destination.length < value.length()) {
            throw new IllegalArgumentException("Destination is too small");
        }
        return value.copyTo(destination);
    }
}
```

Open one index and reuse it across reads; do not reopen it for every expert.
The stored value includes our block envelope. The playground adapter must
validate model/block identity, length and digest, then expose the exact payload
to the runtime. Hestia's CRC validates storage integrity but does not establish
that a value belongs to the requested model.

`copyTo()` avoids allocating the extra array returned by `getBytes()`, but the
engine still creates/copies values during decoding. Capacity checks after
`get()` cannot prevent those earlier allocations. This example is sufficient
for controlled fixtures; it makes no strict memory or performance promise.

## 2. The small reader contract we would like

This is a proposed generic Hestia-facing interface. Hestia sees block IDs and
bytes; the adapter understands layers, experts and quantization.

```java
import java.nio.ByteBuffer;

/**
 * Reads complete binary values from one opened index generation.
 * Initial contract: serialized calls and no close concurrent with a read.
 */
public interface BinaryBlockReader extends AutoCloseable {

    /**
     * Copies the complete stored value into destination at its position.
     * Returns the byte count and advances position by that count.
     * The destination limit is unchanged. No caller buffer is retained.
     */
    int read(long blockId, ByteBuffer destination);

    /** Releases reader-owned resources; repeated close is harmless. */
    @Override
    void close();
}
```

The reader returns the complete stored value, including any application
envelope. It does not strip a GGUF-specific header or check a model manifest.
The adapter does that before the native runtime consumes the payload. A native
slot may need a separate transfer after validation; include that copy in the
benchmark.

| Situation | Proposed behavior |
| --- | --- |
| Successful read | Complete value returned; result is its exact byte length; destination position advances, limit stays unchanged |
| Missing ID, corrupt value, unsupported size or closed reader | Throw `IndexException` or a specific subtype; never return fabricated data |
| Null/read-only/insufficient destination | Reject visibly before writing to the destination; use a documented argument error |
| Failure after a read has started | Restore destination position/limit, but its bytes may have been overwritten; caller must discard them |
| Heap or direct `ByteBuffer` | Both accepted; direct does not imply zero-copy or that the buffer is Metal-compatible |
| Read completion | All storage access to the caller buffer is finished on return; storage retains no reference |
| Concurrency | Serialized calls in the first adapter; concurrent reads are a later explicitly tested capability |
| Shutdown | Caller finishes reads before close; no in-flight asynchronous callbacks in this initial contract |

Caller ownership continues after `read()` returns. The bridge must not reuse
or free a destination while a transfer or GPU kernel still consumes it. The
reader cannot manage that later lifetime on the caller's behalf.
During `read()`, the caller grants exclusive access to the destination; other
threads must not change its contents, position or limit.

A manifest already supplies expected block lengths, so a separate `size()`
lookup is unnecessary for the first version. A bounded loop over `read()` is
enough for an expert request containing several blocks. If any block fails,
the adapter rejects the entire expert request and launches no dependent kernel.
Earlier successful reads in that request are not rolled back.

## 3. Proposed opening and resource configuration

The following is a usage sketch, not code supported by the current library.
`BinaryBlockReaders`, `BlockReadOptions` and their builder are proposed names.
They would be ordinary Java 17 classes. The values illustrate API usage; they
are not tuned settings for Qwen or a guaranteed sufficient memory allocation.

```java
final long MiB = 1024L * 1024L;

BlockReadOptions options = BlockReadOptions.builder()
        .maxValueBytes(4 * 1024 * 1024)
        .maxCacheBytes(128 * MiB)
        .maxWorkingBytes(32 * MiB)
        .build();

try (BinaryBlockReader reader =
        BinaryBlockReaders.openReadOnly(directory, options)) {
    ByteBuffer buffer = ByteBuffer.allocateDirect(4 * 1024 * 1024);

    int storedLength = reader.read(blockId, buffer);
    buffer.flip();

    // Adapter validates the envelope and expected model identity here.
    // Only a successful validation exposes the payload to native inference.
}
```

`directory` is a caller-owned Hestia `Directory`, and `blockId` comes from the
playground manifest. The reader owns its index handle but not the supplied
directory. The full value, including envelope overhead, must fit
`maxValueBytes`; the importer splits larger tensors at valid boundaries.

The proposed limits have distinct meanings:

- `maxValueBytes`: maximum stored value accepted. Check decoded lengths before
  allocating payload storage, with overflow checks and explicit rejection.
- `maxCacheBytes`: accounted retained payload/page-cache capacity owned by the
  reader, including simultaneous representations that remain cached. It does
  not describe the runtime's expert cache.
- `maxWorkingBytes`: accounted live storage scratch buffers for reads, decoding,
  copying and decompression. Acquire capacity before allocation and release it
  when the work ends. A request that can never fit must fail without waiting
  indefinitely.

These limits exclude caller buffers, routing/manifest metadata, JVM overhead,
native inference memory and the OS filesystem cache. Report those separately,
derive conservative limits for metadata, and include everything in the host
memory check. The builder does not claim a whole-process RSS limit. The
implementation must publish its accounting categories and peak measurements;
adding setters alone is not memory control. On Java heaps, released scratch
accounting also does not imply the garbage collector has immediately reclaimed
the underlying memory; reuse buffers and measure allocation pressure/RSS.
Zero cache bytes means caching is disabled. Maximum value size and working
capacity must be positive and sufficient for an admitted read; reject invalid
settings. No zero or negative value means an unlimited budget.

`openReadOnly` is a target lifecycle guarantee: open an already finalized index
without replaying writes, persisting configuration changes, creating missing
model data or running compaction. Reject a store requiring recovery. An
exclusive operational lock file may still require directory write access;
fully read-only filesystem access and multi-process readers are deferred.
Do not label ordinary `SegmentIndex.open()` as implementing this contract.

## 4. Distance from the target

The audited source already provides the basic storage operations. The table
distinguishes an adapter we can write now from deeper engine work; it is not
a completion percentage or a time estimate.

| Capability | Current state | Work remaining |
| --- | --- | --- |
| Long-key binary lookup | `SegmentIndex.get()` plus `TypeDescriptorByteArray` | Small adapter: required-ID errors and envelope validation |
| Reused heap destination | `ByteArray.length()` and `copyTo(byte[], offset)` exist | Small adapter; existing decoded-value allocations remain |
| Direct `ByteBuffer` destination | No direct index read-into method identified | A copying adapter is possible; fewer-copy engine path needs design and measurement |
| Group of expert blocks | Individual synchronous reads exist | Bounded loop and request validation; engine multi-get can wait |
| Model identity and publication | Generic store does not understand GGUF generations | Playground manifest/importer and correctness tests |
| Length limits before allocation | Current variable-length reader checks negative length, then allocates the declared array | Engine work for enforced maximum length; audit chunk/decompression allocation paths too |
| Byte-based cache/scratch budgets | Current relevant limits count keys/pages/segments; startup memory estimate is not a cap | Substantial accounting/admission work for strict bounds; controlled fixtures can start sooner |
| Opening without model mutation/recovery | Normal open acquires a directory lock and can persist configuration or perform recovery | Lifecycle investigation and likely engine changes for the proposed guarantee |
| Native runtime integration | Hestia returns Java values | Playground C++/Java bridge, validated transfers and expert-slot ownership |
| Competitive inference speed | No measurements yet | Compare same runtime/cache/model using direct files versus Hestia |

The most important hidden gap is allocation order. A wrapper that performs
`index.get(id)` and only then checks `value.length()` is too late to constrain
the array already allocated by the engine. That affects malicious or damaged
length fields as well as legitimate large blocks. Chunk/page decompression
also needs bounds; a per-value limit alone cannot bound the whole read path.

Large values make the existing sparse-index/local-chunk scan worth measuring:
loading neighboring entries can mean extra megabytes and allocations. The
current counts must be tuned for blocks rather than inherited from small-key
workloads.

## 5. Recommended implementation order

1. Prototype the synchronous reader contract in the playground using today's
   `SegmentIndex` and a separate ordinary-file implementation. Start with small
   controlled fixtures and report the engine memory guarantees still missing.
2. Prove envelope/tombstone handling, exact bytes, missing/corrupt values,
   position/limit behavior, reopen and caller buffer ownership.
3. Measure real block sizes, local-read amplification, copies and total memory.
   Decide whether direct-buffer reads or native multi-get justify engine work.
4. Implement pre-allocation limits and the required lifecycle/accounting gaps
   before advertising strict memory-bounded storage on a low-RAM machine.
5. Connect the native runtime after the storage contract is demonstrated.

We can begin the functional storage experiment with the existing Hestia API.
The work toward a strict, efficient reader is mainly lifecycle and memory
control inside the engine, plus the separate native bridge. Adding an
LLM-specific class to Hestia is not necessary.

## Evidence in the sibling HestiaStore checkout

- [SegmentIndex API](../../HestiaStore/engine/src/main/java/org/hestiastore/index/segmentindex/SegmentIndex.java)
- [ByteArray copy methods](../../HestiaStore/engine/src/main/java/org/hestiastore/index/datatype/ByteArray.java)
- [Reserved byte-array tombstone](../../HestiaStore/engine/src/main/java/org/hestiastore/index/datatype/TypeDescriptorByteArray.java)
- [Variable-length allocation](../../HestiaStore/engine/src/main/java/org/hestiastore/index/datatype/VarLengthReader.java)
- [Page-cache configuration](../../HestiaStore/engine/src/main/java/org/hestiastore/index/segmentindex/configuration/api/IndexChunkStoreCacheConfigurationBuilder.java)
- [Segment/cache count configuration](../../HestiaStore/engine/src/main/java/org/hestiastore/index/segmentindex/configuration/api/IndexSegmentConfigurationBuilder.java)
- [Memory-estimate limitations](../../HestiaStore/docs/operations/memory-estimate.md)

These links follow the local checkout; use the recorded revision to reproduce
the audit if the source changes. The installed Maven snapshot may differ from
that source and must be identified separately in benchmarks.
