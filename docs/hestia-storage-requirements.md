# Hestia storage requirements for MoE inference

Status: proposed specification; implementation has not started.

Date: 2026-09-14.

The [Java API proposal](java-api-proposal.md) shows the current call surface,
a possible reader contract and the distance between them. Its proposed names
and configuration methods are not implemented Hestia APIs.

## 1. Objective and decision

Evaluate whether HestiaStore can supply quantized model-weight blocks to an
existing mixture-of-experts (MoE) inference runtime while using a limited
amount of RAM. The first result must be a measured comparison with ordinary
file-backed reads, followed by a working model if the storage experiment is
promising.

The experiment must establish both correctness and practical value. A model
producing text is not sufficient evidence that Hestia is useful. Hestia may
prove slower than direct reads from an immutable GGUF file; that is a valid
experimental outcome.

This document specifies required behavior at the boundary between storage and
inference. It does not mandate that every capability become a public Hestia
API. Implement adapter behavior in this playground first where possible;
propose a focused library change only when a measured or correctness-related
gap requires it. Potential engine changes remain proposals until a fixture or
measurement establishes their necessity.

## 2. First workload and resource envelope

The initial target is Qwen3-30B-A3B in the official Q4_K_M GGUF representation.
It has 30.5 billion total parameters and approximately 3.3 billion active per
token. Each MoE layer selects 8 of 128 experts. An expert is a learned neural
subnetwork; storage follows the model's router rather than choosing experts
by a human-assigned topic. See [the model card][model].

| Item | Initial choice or planning value |
| --- | --- |
| Hardware | Mac mini M4, 16 GB unified CPU/GPU memory |
| Model file | `Qwen3-30B-A3B-Q4_K_M.gguf` |
| Download size | 18,556,685,824 bytes, approximately 18.56 GB / 17.28 GiB |
| Initial inference workload | One sequence, short prompts, context limit 2,048 tokens, 32–128 generated tokens |
| Storage | Internal SSD first; external USB storage only as a separately measured case |
| Working disk reserve | Approximately 80 GB for source, imported representation, temporary files, tools and a small comparison model |
| Memory objective | Fit the combined experiment on the 16 GB host; sweep bounded cache budgets and report total memory |
| First data | Small deterministic synthetic blocks; real model blocks before performance conclusions |

The 80 GB allowance is a planning estimate, not a guaranteed Hestia footprint
or import peak. Do not assume additional compression of quantized weights.
Download only the selected GGUF, not all quantizations in its repository.
Import must stream existing packed bytes and must not materialize an entire
unquantized model. See [the exact GGUF file][gguf].

An 8 GB expert-cache setting is not an 8 GB process limit. The JVM, Hestia
caches, native buffers, Metal allocations, shared model tensors, attention
cache and macOS all require memory. CPU/GPU memory on this machine is shared;
moving work to the CPU does not add capacity. Cache budgets must be selected
after measuring fixed overhead. Initial defaults must leave room for the OS
and other applications. Run timing experiments separately from the active
Senku workload; do not stop or alter that job as part of setup.

## 3. Ownership and architecture

| Component | Responsibilities |
| --- | --- |
| Hestia engine | Persist and retrieve exact binary values; provide indexing, lifecycle, integrity and storage configuration |
| Playground importer | Read GGUF metadata, identify expert matrix slices, assign block IDs, copy packed bytes, publish a verified model manifest |
| Playground storage adapter | Apply byte budgets, map logical blocks to Hestia, enforce manifest identity, expose failures and measurements |
| Inference runtime | Tokenization, routing, numerical kernels, attention cache, model scheduling and output generation |
| Expert cache / bridge | Transfer missing blocks to native slots, track ownership, retain reusable experts and coordinate completion |
| Benchmark harness | Drive identical requests through ordinary files and Hestia; measure correctness, resource use and timing |

The runtime requests a selected expert's tensor blocks. The cache serves hits;
the adapter loads misses from Hestia and transfers them to native buffers. The
runtime waits for all required blocks before computing. Missing experts must
never be replaced with zero values or silently skipped.

The first implementation should hide ordinary files and Hestia behind the
same small logical read contract. No interface signatures or transport are
fixed yet. A later C++/Java bridge can use a persistent local binary connection
or JNI. Its time, copies and memory count toward Hestia's result. A chat HTTP
endpoint cannot replace the runtime's internal weight-loading path.

An experimental [Metal expert-loader fork][fork] is a candidate integration
point. It is research code, not a promised upstream capability. Reproduce it
and pin a reviewed revision before integration. Standard [llama.cpp][llama]
supports file mapping and CPU/GPU placement, but those alone are not an
application-enforced byte budget for SSD expert residency.

## 4. Data model

Use a model-scoped mapping from `long blockId` to packed binary bytes. Do not
create one database record per scalar weight. Start with expert-aligned tensor
slices, allowing a large slice to span several bounded blocks. The manifest
must record every split and its reconstruction order. Block boundaries must
respect the source quantization layout; the prototype must not requantize.

The model manifest must contain:

- Manifest version, model repository/revision and source-file size/checksum.
- Model architecture/configuration and the tokenizer metadata needed by the
  selected runtime, either retained in the GGUF or referenced explicitly.
- Every tensor's identity, dimensions, numeric/quantization type and relevant
  byte order/alignment information.
- A collision-free assignment of IDs to `(tensor, layer, expert, slice)` where
  applicable, including slices for shared tensors if those are imported.
- Block lengths, logical offsets and integrity digests, plus a digest covering
  the manifest's mapping. Equal bytes at different coordinates still retain
  their distinct logical identity.
- The Hestia storage format/configuration needed to reopen the index, adapter
  version and completion state.

The existing byte-array descriptor reserves a particular byte sequence as a
deletion tombstone. Therefore a raw payload must be wrapped in a versioned
envelope whose fixed prefix cannot equal that marker. Include block identity,
payload length and digest in the envelope, and verify them before unwrapping.
An input payload equal to the descriptor's tombstone must round-trip as ordinary
model data. A custom descriptor is an alternative only if it provides equally
unambiguous deletion semantics. Envelope overhead and copies count toward the
resource budget.

All file sizes and logical offsets must support values above 2 GiB. Individual
block sizes must be bounded before allocating Java arrays or native buffers;
unsupported blocks must be split on valid boundaries or rejected explicitly.
The minimum cache/buffer requirement for a requested expert set must be
checked before scheduling it, so an oversized request cannot wait forever.

## 5. Required behavior and acceptance checks

MUST describes a correctness or resource requirement for the complete
prototype. It is not an assertion that the library currently provides the
behavior. SHOULD identifies an optimization to investigate after measuring a
correct baseline.

| ID | Requirement | Initial owner | Acceptance check |
| --- | --- | --- | --- |
| R01 | MUST preserve packed tensor bytes and coordinates exactly | Importer + adapter | Every imported block's digest and length matches the source; reconstruct test tensors byte-for-byte after reopen, including a payload equal to the byte descriptor's tombstone |
| R02 | MUST read a requested block without a full model scan or full model load | Engine + adapter | Retrieve first, middle, last and pseudorandom blocks; instrument bytes decoded/read and distinguish bounded local scans from whole-model scans |
| R03 | MUST reject unknown IDs, wrong model identities, corrupt or truncated blocks and unsupported manifest versions before their bytes reach a kernel | Adapter + engine integrity path | Inject each failure into fixtures and verify a visible error, no fabricated data, and no successful compute completion |
| R04 | MUST bound allocations owned by import, reads, queues, caches and transfers in bytes | Adapter + engine configuration | Use variable-size fixtures and repeated reads; show each configured budget, peak usage, oversize rejection and stable memory over repeated cycles |
| R05 | MUST define buffer ownership and completion | Adapter + native bridge | A buffer is not reused/evicted while a read, transfer or kernel can access it; exercise delayed completion, cancellation and errors |
| R06 | MUST expose only a completed, verified model generation to readers | Importer + engine lifecycle | Interrupt import before/after finalization and publication; only the previous or new complete generation opens, or none if no generation was previously published |
| R07 | MUST preserve logical model contents during inference | Adapter + engine lifecycle | Reopen the same generation repeatedly; verify model/manifest identity and absence of model-data writes or active ingestion/compaction |
| R08 | MUST support batched expert requests with deterministic ID-to-result association | Adapter; engine multi-get optional | Mix out-of-order, repeated and missing IDs; verify association and define whole-request failure; chunk a large request under the byte budget |
| R09 | MUST have a documented concurrency and shutdown contract | Adapter + engine lifecycle | Concurrent reads if advertised; close/cancel with pending work must neither reuse live buffers nor deadlock; serial reads are valid for the first baseline |
| R10 | MUST stream model payloads with a bounded working set; account for manifest and index metadata separately | Importer + engine configuration | Import a fixture larger than the data-buffer budget without accumulating all payloads, then real model blocks; record metadata growth, peak RAM and temporary disk usage |
| R11 | MUST expose measurements needed for fair attribution | Adapter + harness | Report logical bytes, returned bytes, cache hits/misses, queue time, read/decode/copy times, and peak memory/disk; label unavailable device metrics |
| R12 | MUST release resources and report partial failures | Adapter + importer + bridge | Inject failed reads, exhausted output space and bridge disconnects; model generation remains unready or intact, and buffers/handles are released |
| P01 | SHOULD reduce avoidable allocations and copies | Adapter + possible engine extension | Count copies and allocation rate against the existing byte-value API; test caller-owned/native-buffer reads only if the measured cost warrants them |
| P02 | SHOULD overlap bounded reads with useful computation | Scheduler + bridge | Compare synchronous and prefetched runs with the same memory budget; unused predictions cannot replace required reads or cause unbounded work |
| P03 | SHOULD arrange blocks to reduce read amplification | Importer + engine configuration | Sweep block/chunk sizes and layout; report requested bytes versus bytes decoded and physical disk bytes where observable |
| P04 | SHOULD offer compression only when beneficial | Engine configuration + harness | Compare lossless compression settings and uncompressed storage with identical source bytes; measure size, CPU and end-to-end time |

R06 is an application publication contract. A concrete first design is an
isolated staging directory, finalized/closed index, verified manifest and
atomic publication on the same filesystem. Visibility after process failure
and durability after power loss are different guarantees. Document the
engine's flush/sync guarantees and parent-directory persistence requirements;
do not claim power-loss durability from a rename alone. Resumable import is
optional; interrupted staging data must never masquerade as a ready model.

For R07, distinguish immutable model data from operational lock files. If
opening an index needs a writable directory for a lock, record that limitation.
Opening an artifact on a filesystem that is entirely read-only is desirable,
but is not required for the first private, single-process experiment. Multiple
independent readers/processes and live model updates are also deferred.

Verification scans must explicitly use `FULL_ISOLATION`, close their streams,
and check every expected ID, count, length and digest against the manifest.
Default `FAIL_FAST` scans can finish normally with an incomplete range after
invalidation; absence of an exception is not proof of complete verification.
`FULL_ISOLATION` does not provide a transactional whole-index snapshot, so
validate an isolated, finalized generation with no writer or maintenance work.

For R04, count simultaneous copies, in-flight reads and decompression workspace,
not only retained cache entries. Budget model/index metadata too, and reject a
configuration that cannot fit rather than ignoring that overhead. An
entry-count limit can be a conservative bound only if maximum entry size and
concurrency are also bounded. A JVM heap
limit alone is not a limit on native/Metal memory or OS page caching.
Maximum decoded lengths must be checked before allocation, including chunk
and decompression paths. Checking a value's size only after `get()` returns
cannot enforce this requirement.

## 6. Current Hestia fit and gaps

This audit refers to the clean HestiaStore source checkout at
`f063b0525703756b97f7a36a5369f875b1c59885`, inspected on 2026-09-14. The Maven
coordinate is `org.hestiastore:engine:1.1.1-SNAPSHOT`; its installed JAR may
differ from that source revision. Record the actual artifact checksum in any
future benchmark. Source references below point to the sibling checkout;
consult the recorded commit when that checkout advances. This does not assume
the local commit has been published to the remote repository.

| Capability | Current evidence | Implication |
| --- | --- | --- |
| Long keys and binary values | `TypeDescriptorLong` and `TypeDescriptorByteArray` exist; the byte descriptor reserves a tombstone | A block store is possible through the general index with an unambiguous payload envelope |
| Point lookup and scans | `SegmentIndex.get(key)` and stream/iterator APIs exist; the documented read path may scan a local chunk | Suitable functional starting point; measure amplification for large values |
| Scan completion | Default `FAIL_FAST` scans can end early without an error | Use explicit isolation and manifest-completeness checks for verification |
| Current Senku path | Senku is a write/finalize/stream index with no point-read API | Do not use the billion-key set benchmark as evidence for selective expert reads |
| Missing values | General point lookup returns a missing value result (`null`) | Adapter must turn an absent required block into a clear error |
| Binary value ownership | Existing byte-array wrappers/readers allocate and copy payloads | Functional baseline; no zero-copy or native-buffer performance claim |
| Integrity | Chunk read pipeline supports magic/CRC validation | Explicitly select and verify configuration; also bind block contents to the model manifest |
| Memory limits | Existing entry/page-oriented settings do not establish a complete byte cap across engine, adapter and runtime | Bound block size/concurrency and inspect hidden caches before calling memory bounded |
| Opening completed data | No dedicated read-only opening mode identified in the reviewed public API; normal open takes an exclusive directory lock and can write configuration/recovery state | Share one index handle, verify open-time writes and enforce immutable generations in the adapter |
| Batched selected reads | No public multi-get identified in the reviewed `SegmentIndex` API | Start with a bounded adapter loop; native multi-get is an optimization candidate |
| GPU/native tensor consumption | Public index returns Java values | Native transfer, slot lifetimes and computation scheduling belong to the adapter/runtime |

Evidence: [general index API][index], [byte value descriptor][bytes],
[byte value wrapper][bytearray], [variable-length reader][varreader],
[read-path documentation][readpath], and [Senku design][senku]. A capability
not identified here is an audit scope statement, not proof that no related
internal mechanism exists. Keep this matrix current as implementation begins.

## 7. Benchmark contract and decision gates

### Gate A: storage correctness with small fixtures

Implement a minimal ordinary-file reader and Hestia reader under one logical
contract. Cover R01–R12 as applicable without downloading a large model. Use
known bytes, heterogeneous block sizes, duplicates, reopen cycles, corruption,
failure injection and requests larger than a cache window. Add a sparse-file
fixture with an offset above 2 GiB to catch narrowing errors without allocating
a multi-gigabyte array. Any failure blocks inference integration.

### Gate B: representative block retrieval

Use real quantized expert blocks and, when available, a trace of IDs selected
by the target runtime. Synthetic locality patterns are useful diagnostics but
must not be described as observed model behavior. Compare ordinary-file reads
and Hestia with identical logical data, request order, budgets and concurrency.

Report per-request median and tail latency, effective payload throughput,
bytes read/decoded, copying/allocation costs, warm/cold behavior, import time,
and steady/peak disk footprint. Measure the storage medium separately when
possible. Filesystem-cache hits are not SSD reads. A model larger than RAM by
itself does not prove the requested experts are being served from disk.

Keep raw measurements separate from derived metrics. Record hardware, OS/JDK,
engine JAR hash and source revision, model file/revision/hash, adapter/runtime
revision, storage configuration, cache sizes, context, token counts, commands
and cache preparation method. Do not clear system-wide caches or disrupt the
active Senku run for a benchmark.

### Gate C: native integration and output correctness

First reproduce expert streaming from ordinary files, then replace only its
reader with Hestia. Use the same runtime build, model quantization, expert
selection, buffers and workload for the comparison. Preserve all eight
selected experts; expert dropping, changed quantization and reduced context
are separate experiments, not storage improvements.

Verify exact block bytes and compare fixed-input logits using a stated numeric
tolerance appropriate for the backend. Generated text alone is an unreliable
equivalence check because sampling and numerical differences can change it.
Measure time to first token separately from steady decoding and whole-request
time. Count both JVM and native memory if separate processes are used, and
avoid double-counting shared pages when reporting combined physical usage.

### Gate D: usefulness

Select a small repeatable set of useful document assessment or classification
tasks with short outputs and a stated quality rubric. Compare the disk-backed
MoE model with a smaller resident model as a separate product-level comparison.

Before performance tuning, record the acceptable response-time and memory
targets for that workload. No fixed tokens-per-second promise or arbitrary
speedup threshold is specified before a baseline exists. Continue Hestia
integration only if the measurements establish useful storage behavior or an
explicit experimental benefit. If direct files are faster and Hestia adds no
useful capability, document the result rather than forcing a new engine API.

## 8. Deferred capabilities

The first prototype does not require training, fine-tuning, new numerical
kernels, automatic expert pruning, distributed storage, ANN/vector search,
RAG, model hot-swapping, multi-user serving, or conversion of model knowledge
into a database of facts. These are separate projects.

Strict zero-copy, native engine multi-get, fully read-only filesystem access,
parallel read speedups, prefetch prediction and additional compression are
optional until the baseline demonstrates their value. A first byte-preserving
adapter may copy data and perform synchronous reads.

## 9. First implementation backlog

- [ ] Define the smallest block-read contract, including identity, lengths,
  budgets, failures and resource ownership.
- [ ] Implement tiny fixtures and the ordinary-file reader as an independent
  correctness reference.
- [ ] Implement a general-index Hestia reader and staging/publication import.
- [ ] Verify the selected engine's lifecycle, integrity configuration and cache
  behavior against R01–R12; update the capability matrix with results.
- [ ] Measure real-block reads before proposing a direct-buffer or batch API.
- [ ] Pin and reproduce the chosen native expert loader with ordinary files.
- [ ] Choose the Java/native bridge after measuring its transfer costs.
- [ ] Run the same-runtime comparison and record the Gate D decision.

Decisions still open: exact block size/layout, checksum representation,
cache ownership between engine and adapter, socket versus JNI, whether true
read-only engine opening is worth adding, and workload-specific latency goals.
Resolve them with small fixtures and measurements; avoid building all options
at once.

[model]: https://huggingface.co/Qwen/Qwen3-30B-A3B
[gguf]: https://huggingface.co/Qwen/Qwen3-30B-A3B-GGUF/blob/main/Qwen3-30B-A3B-Q4_K_M.gguf
[llama]: https://github.com/ggml-org/llama.cpp/blob/master/tools/cli/README.md
[fork]: https://github.com/kisasexypantera94/llama.cpp/pull/2
[index]: ../../HestiaStore/engine/src/main/java/org/hestiastore/index/segmentindex/SegmentIndex.java
[bytes]: ../../HestiaStore/engine/src/main/java/org/hestiastore/index/datatype/TypeDescriptorByteArray.java
[bytearray]: ../../HestiaStore/engine/src/main/java/org/hestiastore/index/datatype/ByteArray.java
[varreader]: ../../HestiaStore/engine/src/main/java/org/hestiastore/index/datatype/VarLengthReader.java
[readpath]: ../../HestiaStore/docs/architecture/segmentindex/read-path.md
[senku]: ../../HestiaStore/docs/development/senku-index.md
