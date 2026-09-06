# SDR Signal Monitoring Platform — Project Plan

## Purpose

A spine project for systematically deepening .NET/C# skills across CLR internals, modern C#,
WPF/MVVM, multithreading, network programming, low-allocation patterns, SIMD, SQL Server, and
software architecture — using a single well-scoped system rather than isolated topic study.

The domain is software-defined radio (SDR) signal monitoring, built around **synthetic I/Q
sample data** — no physical SDR hardware required. The architecture is designed so real SDR
hardware could later replace the synthetic emulator without touching downstream components.

## Architecture

```
[I/Q Emulator] --TCP--> [Receiver/Ingestion] --> [DSP Pipeline] --> [Storage]
                                                        |               |
                                                        +--> [WPF Dashboard] <--+
```

Each phase is a standalone deliverable connected to the next over a plain socket or in-process
queue interface. This is deliberate: Phase 1 could be swapped for a real SDR later without
touching phases 2–5.

## Phases

### Phase 1 — I/Q Emulator
Console app generating synthetic I/Q signals (tones, chirps, noise floor) and streaming them
over TCP in a defined binary frame format (interleaved I/Q, little-endian, `cs16` or similar).
**Deliverable:** a client can connect and read a well-formed, continuous byte stream.
**Skills touched:** binary protocol design, TCP fundamentals.

### Phase 2 — Receiver / Ingestion Service
Connects to the emulator, reads the stream via `System.IO.Pipelines`, handles frame boundaries
across `ReadOnlySequence<byte>` segments, and implements reconnect logic (backoff, resuming
cleanly after a dropped connection).
**Deliverable:** reliably reports "received frame N, M samples" even when the emulator is
killed and restarted mid-stream.
**Skills touched:** `Pipelines`, `Span<T>`/`Memory<T>`, backpressure, reconnect logic,
low-allocation parsing, SIMD-friendly segment alignment (`minimumSegmentSize` tuned to frame
size; `buffer.IsSingleSegment` fast path).

### Phase 3 — DSP Pipeline
A TPL Dataflow graph: windowing (Hann/Hamming) → FFT → magnitude/power computation, with
bounded block capacities for end-to-end backpressure. FFT/windowing implemented once naively
and once with `System.Numerics.Vector<T>` or hardware intrinsics, benchmarked with
BenchmarkDotNet.
**Deliverable:** spectrum bins printed per block; a benchmark report showing the naive-vs-SIMD
delta.
**Skills touched:** TPL Dataflow block composition, synchronization primitives, SIMD,
performance benchmarking.

### Phase 4 — Storage
Persist detected signal events (peak frequency, power, timestamp) to SQL Server — implemented
twice, once with EF Core and once with Dapper, to compare.
**Deliverable:** a queryable events table plus execution-plan analysis for a representative
insert/read pattern.
**Skills touched:** T-SQL, execution plans, EF Core vs Dapper tradeoffs.

### Phase 5 — WPF Dashboard (MVVM)
Live-consumes the pipeline and renders a scrolling waterfall/spectrogram via `WriteableBitmap`,
a live spectrum line chart, and a virtualized signal-event list.
**Deliverable:** the visual payoff — a working real-time dashboard.
**Skills touched:** `VirtualizingPanel`/binding cost, `Freezable`, `Dispatcher` patterns for
high-frequency UI updates, custom controls/`ControlTemplate`, high-DPI handling.

## Key design principles

- **Spine project approach**: one well-chosen project spanning multiple competency areas beats
  isolated topic study.
- **Hardware independence**: the emulator sits behind a clean interface boundary so real SDR
  hardware can replace it later without cascading changes.
- **I/Q fundamentals**: interleaved `I, Q, I, Q, ...`; common sample types `cu8`/`cs8`/`cs16`/
  `cf32`; little-endian layout; tone generation via `Math.Cos`/`Math.Sin`.
- **Pipelines + SIMD risk**: non-contiguous segment splits can break vectorized conversion
  paths — mitigate via `minimumSegmentSize` alignment and the single-segment fast path.
- **TPL Dataflow vs. alternatives**: best for multi-stage pipelines with branching, joining, or
  batching and varying per-stage parallelism; `Channels`/plain `async`-`await` suit simpler
  linear flows.

## Status

Plan confirmed. Next step: concrete spec for the Phase 1 I/Q frame format and emulator design.
