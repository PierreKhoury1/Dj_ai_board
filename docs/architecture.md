# DJ AI Board — Architecture Plan

Status: draft v0.1 (design discussion, nothing built yet)

## Goal

A standalone hardware unit that takes two tracks, separates each into stems
(vocals / drums / bass / other), and performs an automatic, beat- and
phrase-accurate transition between them at the stem level, output as analog
audio.

## The one decision that shapes everything: separate at import, mix in real time

Stem separation is about three orders of magnitude more expensive than
everything else in the pipeline, and the accurate models are non-causal (they
look at several seconds of context on both sides). So:

- Separation runs **once, when a track is added to the library**, and the stems
  are cached to disk. This is the same UX as Rekordbox/Serato "analysis".
- The **real-time path is cheap**: 8 stems (4 per deck) x (time-stretch + EQ +
  gain automation) + sum + limiter. Any modern ARM SoC does this easily.
- Consequence: the board's compute is sized by "how long will the user wait
  after loading a new track", not by real-time constraints.
- Consequence: live line-in from external decks is **out of scope for v1**.
  High-accuracy separation of a live stream is not practical (multi-second
  latency at best, large quality loss).

## Accuracy targets

| Layer | Target | How |
|---|---|---|
| Stem separation | ~10-11 dB vocal SDR ("best") or ~9 dB ("fast") | BS-RoFormer / Mel-Band RoFormer checkpoints for best; HT-Demucs (htdemucs_ft) for fast. Both available via `python-audio-separator` or `demucs`. |
| Beat grid | downbeat-accurate, few-ms jitter | madmom DBN or BeatNet; run on the **drum stem**, not the mix (cleaner onsets). |
| Phrase / structure | 8/16/32-bar boundaries, section labels | `allin1` (All-In-One music structure analyzer; it already consumes Demucs stems). |
| Key | Camelot-wheel compatibility, optional per-stem pitch correction | essentia / libkeyfinder on the "other"+"bass" stems. |
| Transition alignment | sub-beat | Cross-correlate onset envelopes of deck A drums vs deck B drums inside the overlap window, then nudge. Stems make this far more robust than mix-vs-mix. |
| Per-stem stretch | artifact-free at +/-8% | Different stretch settings per stem: transient-preserving for drums, formant-preserving for vocals. This is a real audible win of stem-based mixing. |

Reference points for separation quality (MUSDB18-HQ):
BS-RoFormer reports 9.80 dB average SDR across 4 stems and won SDX'23;
MVSEP's best ensembles report ~11.9 dB on vocals; HT-Demucs fine-tunes sit
around 9.0-9.2 dB vocals. Ensembles buy accuracy at multiplied compute.

## Compute options

Numbers marked "est." are estimates to be measured in Phase 1, not benchmarks.

| Option | On-device separation (4-min track) | Real-time 8-stem mix | Power | Carrier PCB difficulty | Verdict |
|---|---|---|---|---|---|
| **NVIDIA Jetson Orin Nano 8GB module (Super mode, 67 INT8 TOPS)** | htdemucs FP16: ~1-3 min est.; RoFormer: several min est. | easy | 7-25 W, needs fan | high (260-pin SO-DIMM, PCIe/DP/USB3 impedance-controlled routing) but reference designs exist | **Recommended if separation must run on the box** |
| Raspberry Pi CM5 / RK3588 CM (CPU only) | tens of minutes est. (Demucs docs quote ~1.5x realtime on a desktop CPU; these are far slower) | easy | 5-10 W, passive OK | medium | Only if separation is offloaded to a phone/PC/cloud companion |
| RK3588 NPU / Hailo-8 / Hailo-10H | unproven; STFT + long-sequence transformer ops map poorly to these compilers | easy | low | medium | Do not bet the project on it without a working port first |
| MCU / audio DSP (STM32H7, SHARC) | impossible | fine | very low | low | Rule out for separation; fine as the **control-surface MCU** |

Recommendation: **Jetson Orin Nano 8GB on a custom carrier** for a standalone
"super accurate" unit. If you are willing to run separation in a companion app,
a Pi CM5 carrier is cheaper, fanless, and simpler, and the mixing accuracy is
identical because separation is not time-constrained either way.

Important framing: the "PCB" you design is a **carrier board around a compute
module**, plus audio and controls. Nobody should route an SoC + LPDDR from
scratch for this.

## Hardware block diagram (Jetson version)

```
 DC 12-20 V in / USB-PD 20 V
        |
   5 V / 6 A buck  ----> Jetson Orin Nano 8GB module (260-pin SO-DIMM)
        |                      |  |  |  |  |
   3.3 V / 1.8 V rails         |  |  |  |  +-- M.2 M-key NVMe (stem cache; ~80-100 MB/track as FLAC)
                               |  |  |  +----- M.2 E-key Wi-Fi/BT
                               |  |  +-------- GbE (magnetics + RJ45)
                               |  +----------- DisplayPort -> touchscreen (DP->HDMI bridge if HDMI needed)
                               +-------------- USB 3 host x2 (USB sticks, MIDI controllers)
        I2S + MCLK (44.1 kHz family, 22.5792 MHz local oscillator)
                               |
                         DAC (PCM5122 budget / ES9038Q2M hi-fi)
                               |
            +------------------+------------------+
      headphone amp      balanced master out     booth out
      (TPA6120 class)    (THAT1646 / DRV134, TRS/XLR)

 Control surface: RP2040 scans faders/pots/encoders/buttons/LEDs,
 presents as USB-HID or UART to the Jetson. Keeps timing off Linux.

 Thermal: active heatsink required at 15-25 W. Plan the enclosure for airflow
 and fan noise; separation can be scheduled in a quieter lower-power mode.
```

Notes:
- The module boots from NVMe (no eMMC on Orin Nano), so the M.2 slot is mandatory.
- Jetson 40-pin header exposes I2S on the same pins as a Pi; `jetson-io` and a
  device-tree overlay bring up a PCM5122-style codec. Budget time for ASoC
  bring-up; use a USB audio interface during Phase 1 to decouple it.
- Keep the DAC in clock-master mode from a local audio-grade oscillator rather
  than deriving MCLK from the SoC.
- Open reference designs: NVIDIA publishes the Orin Nano dev-kit carrier design
  files (Jetson Download Center), and Antmicro publishes an open-source KiCad
  Jetson Orin baseboard. Start from one of these, do not start from a blank
  schematic.

## Software stack

| Stage | Choice | Runs |
|---|---|---|
| OS | JetPack 6 (Ubuntu 22.04, CUDA, TensorRT) | - |
| Import / analysis | `python-audio-separator` (Demucs + RoFormer models) or `demucs` directly, FP16 on CUDA; `allin1` for beats/downbeats/sections; key detection; loudness (EBU R128) | background worker, at import |
| Cache | per-track folder: 4 stems as FLAC + `analysis.json` (beat grid, downbeats, sections, key, loudness, waveform overview) | NVMe |
| Mix planner | rule engine over `analysis.json` pairs: phrase-aligned entry/exit points, Camelot key check, energy curve matching, per-stem swap schedule (e.g. B-drums under A-vocal at phrase boundary, bass swap at downbeat, A-vocal out before B-vocal in), EQ/filter automation. Optional: an LLM picks between candidate plans / "styles" given the structured features. | at load, <1 s |
| Real-time engine | C++ (JUCE or plain ALSA) or Rust (cpal): 8 stream readers, per-stem time-stretch (signalsmith-stretch, MIT, or Rubber Band R3), per-stem EQ/filter/gain automation lanes, sum, limiter, output at 44.1 kHz / 24-bit | always, low priority CPU |
| UI | touchscreen app (Qt or web view) + RP2040 control surface | - |

## Development plan

Do not spin a PCB until the software pipeline is proven on a dev kit.

1. **Phase 1 — pipeline on a dev kit (2-4 weeks).**
   Jetson Orin Nano Super dev kit + a USB audio interface + NVMe.
   Deliverables: measured separation time per model per 4-min track; stem
   cache format; beat grid + structure analysis; a Python prototype that
   renders a stem-level transition between two analyzed tracks to a WAV.
   Exit criterion: a blind listening test says the transitions are
   "clean" on 10 varied track pairs, and separation time per track is
   acceptable.
2. **Phase 2 — real-time engine.** Port the renderer to C++/Rust with
   per-stem stretch and automation lanes; play live through the USB
   interface; add the mix planner as a service.
3. **Phase 3 — audio + control daughterboard.** A simple 2-4 layer PCB that
   plugs onto the dev-kit 40-pin header: I2S DAC, headphone amp, balanced
   outs, RP2040 control surface. Low risk, validates the analog design and
   the device-tree work.
4. **Phase 4 — custom carrier (only if a compact product is the goal).**
   6-8 layer board derived from a reference carrier: 5 V high-current buck,
   M.2 x2, DP, USB3, GbE, plus the Phase 3 audio section. Alternatively
   ship the dev-kit carrier inside the enclosure and skip this entirely.

## Open questions (answers change the design)

1. Must separation run on the box, or may a phone/PC app do it and sync stems
   over? (Jetson carrier vs Pi CM5 carrier.)
2. File-based library only, or do you want live line-in from external decks?
   (Live-in with accurate stems is not feasible in v1.)
3. One-off / small batch, or a product? (Decides whether Phase 4 happens.)

## Proposed repo layout

```
docs/          architecture, measurements, decisions
analysis/      Python: separation, beat grid, structure, key, cache format
planner/       transition planning (rule engine, optional LLM step)
engine/        C++/Rust real-time stem mixer
firmware/      RP2040 control surface
hardware/      KiCad: daughterboard (Phase 3), carrier (Phase 4)
```

## Sources consulted

- BS-RoFormer paper: https://arxiv.org/pdf/2309.02612
- MVSEP algorithm/quality list: https://mvsep.com/es/algorithms
- BSMamba2 vocal separation paper (2025): https://arxiv.org/pdf/2508.14556
- Jetson Orin Nano Super announcement summary: https://www.infoq.com/news/2024/12/nvidia-jetson-ai-supercomputer
- Demucs CPU timing guidance: https://aireiter.com/blog/demucs-stem-separation-models-flags-cost
