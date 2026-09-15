# Hestia MoE Playground

A separate Maven project for evaluating HestiaStore as storage for quantized
mixture-of-experts (MoE) language-model weights on a computer with limited RAM.

## Start here

Read [Hestia storage requirements](docs/hestia-storage-requirements.md).
It defines the first workload, required behavior, ownership of each capability,
current API evidence, and acceptance criteria. Proposed requirements are not
claims that Hestia already implements them or that it outperforms model files.

See the [proposed Java API](docs/java-api-proposal.md) for an existing-API
example, a small proposed binary reader, and the remaining adapter/engine work.
The Java snippets are design documentation, not a shipped Hestia extension.

The project currently contains the specification and a Java 17 Maven scaffold
with the Hestia engine dependency. The source directories are empty. There is
no importer, model runner, native bridge, benchmark, or automated test yet.

## Build

Requirements: JDK 17 or newer and Maven. The initial Hestia dependency follows
the sibling checkout's `1.1.1-SNAPSHOT` version. If that snapshot is not already
installed locally, build it from the sibling repository:

```sh
mvn -f ../HestiaStore/pom.xml -pl engine -am install
```

Then validate this project:

```sh
mvn clean verify
```

At this stage, a successful build validates Maven configuration and dependency
resolution only; there is no Java implementation to test. Maven may report an
empty JAR. No model download or inference is part of the build.

Plugin versions are pinned to versions available in the existing local Maven
environment. The engine snapshot is mutable: future benchmark reports must
record the actual engine JAR checksum and source revision, not just its Maven
version.

## First implementation milestone

Create a small block-store contract and fixtures, then compare Hestia reads
with ordinary file reads under the same byte budget. Use packed tensor blocks,
preserve every byte, and prove reopen/corruption/failure behavior before adding
a model runtime. The specification describes the later path to inference.

Keep downloaded models in `models/`, imported stores in `data/`, and local
measurements in `runs/`; these directories are ignored by Git. Retain concise,
reproducible benchmark reports under `docs/` when results are available.
