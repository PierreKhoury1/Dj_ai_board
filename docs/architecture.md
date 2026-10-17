# DJ AI Board — Architecture Plan

Status: draft v0.2 (design discussion, nothing built yet)

## Decision log

| Date | Decision | Consequence |
|---|---|---|
| 2026-10-17 | This is a **product**, not a one-off. | Custom carrier board is mandatory. BOM cost, lifecycle, licensing, compliance and manufacturing enter the design from day one. |
| 2026-10-17 | Working assumption: **file-based library, no live line-in** in v1. | High-accuracy stem separation of a live stream is not practical. Revisit for v2 only with a lighter, causal model. |
| 2026-10-17 | Working assumption: **separation runs on the unit** (offline-capable), cloud is an optional accelerator later. | Compute module must run heavy separation models in a wait-a-minute-or-two timeframe. DJs work in venues without reliable internet; this is the "Pro" positioning. |

## Goal

A standalone hardware unit that takes two tracks, separates each into stems
(vocals / drums / bass / other), and performs an automatic, beat- and
phrase-accurate transition between them at the stem level, output as analog
audio. Differentiator vs. existing DJ gear with "stems" features: separation
quality (heavy, non-real-time models run at import), and stem-aware
transition planning.

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

## Accuracy targets

| Layer | Target | How |
|---|---|---|
| Stem separation | ~10-11 dB vocal SDR ("best") or ~9 dB ("fast") | BS-RoFormer / Mel-Band RoFormer class for best; HT-Demucs class for fast. **For a product, see the licensing section: the shipping model will likely be a licensed commercial SDK or an in-house trained model, not a community checkpoint.** |
| Beat grid | downbeat-accurate, few-ms jitter | BeatNet (CC BY 4.0, commercial OK) or in-house tracker; run on the **drum stem**, not the mix (cleaner onsets). madmom's trained models are non-commercial (see licensing). |
| Phrase / structure | 8/16/32-bar boundaries, section labels | `allin1`-style structure model (MIT code; check its weights and madmom dependency), or in-house. |
| Key | Camelot-wheel compatibility, optional per-stem pitch correction | In-house HPCP + key-profile detector (small, avoids GPL/AGPL libraries). |
| Transition alignment | sub-beat | Cross-correlate onset envelopes of deck A drums vs deck B drums inside the overlap window, then nudge. Stems make this far more robust than mix-vs-mix. |
| Per-stem stretch | artifact-free at +/-8% | Different stretch settings per stem: transient-preserving for drums, formant-preserving for vocals. This is a real audible win of stem-based mixing. |

Reference points for separation quality (MUSDB18-HQ): BS-RoFormer reports
9.80 dB average SDR across 4 stems and won SDX'23; MVSEP's best ensembles
report ~11.9 dB on vocals; HT-Demucs fine-tunes sit around 9.0-9.2 dB vocals.
Ensembles buy accuracy at multiplied compute.

## Compute options

Numbers marked "est." are estimates to be measured in Phase 1, not benchmarks.

| Option | On-device separation (4-min track) | Real-time 8-stem mix | Power | Carrier PCB difficulty | Module cost (rough) | Verdict |
|---|---|---|---|---|---|---|
| **NVIDIA Jetson Orin Nano 8GB module (Super mode, 67 INT8 TOPS)** | htdemucs FP16: ~1-3 min est.; RoFormer: several min est. | easy | 7-25 W, needs fan | high (260-pin SO-DIMM, PCIe/DP/USB3 impedance-controlled routing) but reference designs exist | single-unit resellers list $500-770; volume pricing only via NVIDIA distributors, expect well under that | **Recommended for the on-device "Pro" product** |
| Raspberry Pi CM5 | tens of minutes est. CPU-only | easy | 5-10 W, passive OK | medium | ~$45-95 | For a cheaper tier where separation runs in a companion app or cloud |
| RK3588 NPU / Hailo-8 / Hailo-10H / Qualcomm QCS8550 | unproven for STFT + long-sequence transformers; Qualcomm is viable at scale but is a large porting effort | easy | low | medium | varies | Not for v1. Candidate for a cost-down v2 once the model is frozen. |
| MCU / audio DSP (STM32H7, SHARC) | impossible | fine | very low | low | low | Rule out for separation; right part for the **control surface** |

Important framing: the "PCB" you design is a **carrier board around a compute
module**, plus audio and controls. Nobody should route an SoC + LPDDR from
scratch for this, at any volume a new product will see.

Lifecycle: NVIDIA publishes multi-year availability commitments for Jetson
Orin modules and Raspberry Pi does the same for CM5. Get the current end-of-
life date in writing from the distributor before freezing the design.

## Product implications

### Positioning and rough BOM (Pro unit, low hundreds of units)

| Item | Rough cost |
|---|---|
| Jetson Orin Nano 8GB module | volume quote needed; budget $250-350 |
| 512 GB NVMe | ~$40 |
| Carrier PCB (6-8 layer) + assembly | $80-150 |
| DAC + analog output stage + headphone amp | ~$25 |
| 7" touchscreen | $50-80 |
| Faders, encoders, buttons, LEDs, RP2040 | $40-80 |
| Pre-certified Wi-Fi/BT module | ~$15 |
| Heatsink + fan | ~$15 |
| Enclosure (low volume) | $60-150 |
| External certified PSU | ~$15 |
| **Total** | **~$600-900** |

At the usual 2.5-3.5x hardware multiplier that puts retail in the
$1,500-2,500 range, which is normal for the DJ market (pro media players and
standalone controllers sit at $1,500-2,700). The accuracy story must justify
the price against Engine OS / rekordbox gear that already ships lighter
on-device stems.

### Licensing audit for commercial use (verified 2026-10-17)

| Component | Code license | Models / data | Product verdict |
|---|---|---|---|
| Demucs (facebookresearch) | MIT | README states no license for pretrained weights; htdemucs trained on MUSDB HQ + an unnamed 800-song set | Usable only after legal review of the weights. Many products do ship it; still get counsel. |
| BS-RoFormer / Mel-Band RoFormer community checkpoints (ZFTurbo training repo) | MIT (code) | Checkpoint licenses and training data not stated | **Do not ship.** Use for Phase 1 quality benchmarking only. |
| Commercial on-device SDKs: AudioShake Local Inference SDK (encrypted model file, Linux + GPU backends), Music.AI Embedded SDK (embedded Linux) | commercial | vendor-licensed | **Most realistic path to best-in-class separation in a product.** Get quotes early; per-unit royalty shapes the BOM. |
| In-house trained separator | - | needs **licensed multitrack training data**; research datasets such as MUSDB18 are not cleared for commercial training | Long-term option. Only worth it if separation quality is the company's moat. |
| madmom | BSD-style | **Trained models are CC BY-NC-SA 4.0** (non-commercial); the project says to contact JKU for commercial products | Do not ship the trained RNN models. The DBN tracker is an algorithm, not a trained model, but get counsel before relying on that distinction. |
| allin1 (structure analysis) | MIT | weights license unstated; depends on madmom and NATTEN | Verify which madmom components it calls and the weights license before shipping. |
| BeatNet | CC BY 4.0 | same | Commercial OK with attribution. |
| essentia | AGPL-3.0 (commercial license from UPF) | - | Avoid or license. |
| libkeyfinder | GPL-3.0 | - | Avoid in proprietary firmware; key detection is small enough to write in-house. |
| Rubber Band (time-stretch) | GPL or commercial | - | License it, or use signalsmith-stretch. |
| signalsmith-stretch | MIT | - | OK. |
| JUCE | AGPL or commercial | - | Commercial license, or use Rust (cpal / Apache-MIT). |
| Qt | LGPL or commercial | - | LGPL with dynamic linking is workable; a web-view UI avoids it. |
| FFmpeg | LGPL build | - | Build LGPL-only. **AAC decoding needs a patent license (Via LA) if you support .m4a**; MP3 patents have expired. |
| Streaming services (Beatport, Tidal, SoundCloud) | partner agreements | - | Caching stems of DRM'd streams needs explicit contractual permission. Not a v1 feature. |

### Compliance

- EMC: FCC Part 15 Class B, CE (EN 55032 / 55035), UKCA. A board carrying
  PCIe, DisplayPort, USB 3 and GbE needs a disciplined stackup, ground plane
  strategy and shielded connectors to pass. Budget a pre-compliance scan at
  DVT and $10-30k for the full pass.
- Radio: use a **pre-certified Wi-Fi/BT module** (modular approval) so the
  radio portion of FCC/RED is inherited.
- Safety: keep mains out of the box. External certified 12-20 V adapter or
  USB-PD 20 V input.
- RoHS, REACH, WEEE; battery-free design keeps shipping simple.

### Thermal and acoustics

- The module needs 15-25 W during separation. Fan noise at a DJ booth is not
  acceptable, so: oversized heatsink, slow large fan, and schedule separation
  into a "quiet" power mode when the unit is idle, dropping to the 7-15 W
  mode during performance where only the cheap real-time path runs.
- Validate the thermal design in the real enclosure at EVT, not on the bench.

### Software as a product

- Secure boot and encrypted storage for licensed model files (the AudioShake
  SDK already uses encrypted model blobs).
- A/B root filesystem with OTA updates; signed update images.
- Analysis queue with progress UI; library management; USB export/import of
  analysis caches so a user can pre-analyze on another unit.
- Crash reporting and a hardware self-test mode for RMA triage.

### Manufacturing

- EVT (does it work) -> DVT (does it pass compliance and thermals in the
  enclosure) -> PVT (does the line build it) -> MP.
- Test fixture: bed-of-nails for the carrier, audio loopback test (THD+N,
  channel balance), control-surface test, NVMe image flashing station,
  serialization.
- Module and NVMe procurement lead times drive the schedule; set up the
  NVIDIA distributor relationship before EVT.

## Hardware block diagram (Pro / Jetson version)

```
 DC 12-20 V in / USB-PD 20 V (external certified PSU)
        |
   5 V / 6 A buck  ----> Jetson Orin Nano 8GB module (260-pin SO-DIMM)
        |                      |  |  |  |  |
   3.3 V / 1.8 V rails         |  |  |  |  +-- M.2 M-key NVMe (boot + stem cache; ~80-100 MB/track as FLAC)
                               |  |  |  +----- M.2 E-key pre-certified Wi-Fi/BT module
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
 and fan noise; separation is scheduled in a quieter lower-power mode.
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
- Audio spec targets: 24-bit / 44.1 or 48 kHz, DAC dynamic range > 110 dB,
  balanced master at +4 dBu, headphone out ~2 x 250 mW into 32 ohm.

## Software stack

| Stage | Choice | Runs |
|---|---|---|
| OS | JetPack 6 (Ubuntu 22.04, CUDA, TensorRT), A/B rootfs, secure boot | - |
| Import / analysis | Separation: licensed SDK (AudioShake / Music.AI) or legally cleared Demucs, FP16 on CUDA. Beats/downbeats: BeatNet or in-house on the drum stem. Structure: allin1-style model (after license check) or in-house. Key: in-house. Loudness: EBU R128. | background worker, at import, quiet power mode |
| Cache | per-track folder: 4 stems as FLAC + `analysis.json` (beat grid, downbeats, sections, key, loudness, waveform overview) | NVMe |
| Mix planner | rule engine over `analysis.json` pairs: phrase-aligned entry/exit points, Camelot key check, energy curve matching, per-stem swap schedule (e.g. B-drums under A-vocal at phrase boundary, bass swap at downbeat, A-vocal out before B-vocal in), EQ/filter automation. Optional: an LLM picks between candidate plans / "styles" given the structured features. | at load, <1 s |
| Real-time engine | Rust (cpal) or C++ (commercial JUCE): 8 stream readers, per-stem time-stretch (signalsmith-stretch MIT, or licensed Rubber Band R3), per-stem EQ/filter/gain automation lanes, sum, limiter, output at 44.1 kHz / 24-bit | always, performance power mode |
| UI | touchscreen app (web view or LGPL Qt) + RP2040 control surface | - |

## Development plan

Do not spin a PCB until the software pipeline is proven on a dev kit.

0. **Phase 0 — product definition (parallel with Phase 1).**
   Target price and first-run volume; separation-model strategy (licensed SDK
   vs. cleared Demucs vs. in-house); request SDK quotes from AudioShake and
   Music.AI; distributor contact for module pricing and EOL date; counsel
   review of the licensing table above.
1. **Phase 1 — pipeline on a dev kit (2-4 weeks).**
   Jetson Orin Nano Super dev kit + a USB audio interface + NVMe.
   Deliverables: measured separation time per model per 4-min track
   (community checkpoints are fine here for benchmarking only); stem cache
   format; beat grid + structure analysis; a Python prototype that renders a
   stem-level transition between two analyzed tracks to a WAV.
   Exit criterion: a blind listening test says the transitions are "clean"
   on 10 varied track pairs, and separation time per track is acceptable.
2. **Phase 2 — real-time engine.** Port the renderer to Rust/C++ with
   per-stem stretch and automation lanes; play live through the USB
   interface; add the mix planner as a service.
3. **Phase 3 — audio + control daughterboard (EVT-0).** A simple 2-4 layer
   PCB on the dev-kit 40-pin header: I2S DAC, headphone amp, balanced outs,
   RP2040 control surface. Validates the analog design and device-tree work
   at low risk.
4. **Phase 4 — custom carrier (EVT).** 6-8 layer board derived from a
   reference carrier: 5 V high-current buck, M.2 x2, DP, USB3, GbE, plus the
   Phase 3 audio section, in the production enclosure with the production
   thermal solution.
5. **Phase 5 — DVT.** Pre-compliance EMC scan, thermal/acoustic validation,
   drop and connector-cycle tests, firmware feature-complete, OTA tested.
6. **Phase 6 — PVT and pilot run.** Test fixtures, flashing station, first
   50-100 units, full compliance pass, then MP.

## Open questions (answers change the design)

1. Separation-model strategy: licensed SDK, legally cleared Demucs, or
   in-house training? This is the biggest cost and quality lever.
2. Target retail price and first-run volume? (Decides whether the Pro/Jetson
   path alone is enough or a cheaper CM5 tier is needed at launch.)
3. Confirm the two working assumptions in the decision log: file-based
   library only in v1, and separation on the unit.

Answered: this is a product (2026-10-17).

## Proposed repo layout

```
docs/          architecture, measurements, decisions, licensing
analysis/      Python: separation, beat grid, structure, key, cache format
planner/       transition planning (rule engine, optional LLM step)
engine/        Rust/C++ real-time stem mixer
firmware/      RP2040 control surface
hardware/      KiCad: daughterboard (Phase 3), carrier (Phase 4)
```

## Sources consulted

- BS-RoFormer paper: https://arxiv.org/pdf/2309.02612
- MVSEP algorithm/quality list: https://mvsep.com/es/algorithms
- BSMamba2 vocal separation paper (2025): https://arxiv.org/pdf/2508.14556
- Jetson Orin Nano Super announcement summary: https://www.infoq.com/news/2024/12/nvidia-jetson-ai-supercomputer
- Demucs CPU timing guidance: https://aireiter.com/blog/demucs-stem-separation-models-flags-cost
- madmom license (models CC BY-NC-SA): https://github.com/CPJKU/madmom/blob/main/LICENSE
- BeatNet (CC BY 4.0): https://github.com/mjhydri/BeatNet
- allin1 (MIT): https://github.com/mir-aidj/all-in-one
- Demucs README (code MIT, weights unstated): https://github.com/facebookresearch/demucs
- RoFormer training repo (MIT code, checkpoint licenses unstated): https://github.com/ZFTurbo/Music-Source-Separation-Training
- AudioShake Local Inference SDK: https://developer.audioshake.ai/sdk/overview
- Music.AI Embedded SDK: https://music.ai/embedded-ai/
- Orin Nano 8GB module single-unit pricing examples: https://thinkrobotics.com/ar/blogs/product-reviews-buying-guides/jetson-orin-nano-price-complete-buying-guide-for-ai-robotics-development
