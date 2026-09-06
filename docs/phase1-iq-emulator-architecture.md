# Phase 1 — I/Q Emulator Architecture

**Status:** Confirmed design direction. Frame format and signal generator interfaces still to be spec'd in detail.

## Purpose

Console app that generates synthetic I/Q signal data and streams it over TCP, standing in for real SDR hardware. Sits behind a clean interface boundary so real hardware can replace it later without touching Phase 2+ (receiver, DSP pipeline, storage, dashboard).

## Pipeline stages

```
Signal generators → Mixer/compositor → Sample formatter → Frame encoder → Real-time pacer → TCP server
```

1. **Signal generators** — tone generator (frequency offset, amplitude, phase via `Math.Cos`/`Math.Sin`), noise generator (Gaussian, configurable SNR), optional modulation (AM/FM, or BPSK/QPSK for richer downstream DSP testing). Optional impairments: DC offset, I/Q gain/phase imbalance, frequency drift.

2. **Mixer/compositor** — superimposes N independent signal sources plus a noise floor into a single I/Q stream, simulating a spectrum with multiple emitters rather than one clean tone.

3. **Sample formatter** — converts synthesized double-precision samples to the wire format. Start with `cs16` (interleaved 16-bit signed I/Q, little-endian) since that's the target for later SIMD conversion work. Keep this conversion isolated so `cu8`/`cf32` can be added later. Scale correctly to each format's dynamic range for accurate SNR downstream.

4. **Frame encoder** — packages samples into frames for transport. Proposed header fields:
   - Sync/magic bytes (resync after dropped connection)
   - Sequence number (uint64) — gap detection
   - Timestamp (int64)
   - Sample format enum
   - Sample count in this frame
   - Sample rate / center frequency — either per-frame or as a separate low-frequency control-frame type (open decision — affects Phase 2's framing/state machine design)
   - Payload: raw interleaved I/Q bytes

5. **Real-time pacer** — paces output to the configured sample rate using a high-resolution timer rather than blasting data as fast as the CPU allows, so the receiver has to handle realistic flow control and backpressure. Configurable multiplier (1× real-time, N× for throughput stress testing).

6. **TCP server** — accepts one or more client connections, handles disconnects cleanly (so Phase 2's reconnect logic has something correct to reconnect to), avoids unbounded buffering if a receiver is slow.

## Configuration / scenario definition

- Config file (JSON) describing: base sample rate, center frequency, output format, list of synthetic signals (offset, power, modulation type).
- Stretch goal: scripted scenarios — signals appearing/disappearing at specific times — to give Phase 4/5 event-detection and persistence something meaningful to log.

## Hardware-swap boundary

Signal generation, mixing, and formatting (stages 1–3) sit behind an `ISampleSource`-style interface producing frames of samples. Transport (stages 4–6) is a separate concern. Real SDR hardware would implement the same interface later without cascading changes downstream.

## Open decisions / next steps

- [ ] Finalize the binary frame header spec (field order, sizes, endianness)
- [ ] Decide per-frame vs. separate control-frame metadata
- [ ] Design `ISignalSource` interface and concrete tone/noise/modulation implementations
