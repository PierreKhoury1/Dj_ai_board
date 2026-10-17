# DJ AI Board — Architecture Plan

Status: draft v0.3 (design discussion, nothing built yet)

## Decision log

| Date | Decision | Consequence |
|---|---|---|
| 2026-10-17 | This is a **product**, not a one-off. | Custom carrier board is mandatory. BOM cost, lifecycle, licensing, compliance and manufacturing enter the design from day one. |
| 2026-10-17 | Working assumption: **file-based library, no live line-in** in v1. | High-accuracy stem separation of a live stream is not practical. Revisit for v2 only with a lighter, causal model. |
| 2026-10-17 | **No Jetson.** Compute module is the **Raspberry Pi Compute Module 5**. | Module cost drops from $399 to $105-170 (1K-unit list prices, July 2026). Fanless, simpler 4-6 layer carrier, open KiCad reference design, pre-certified wireless on-module. |
| 2026-10-17 | **Heavy separation runs off the unit**, in a companion app on the user's laptop/phone (optionally a cloud service). The unit keeps a light on-device fallback. | CM5 CPU cannot run RoFormer/Demucs-class models in acceptable time. DJs already prep libraries on a laptop (rekordbox/Serato model), so this matches existing workflow. Accuracy is unchanged because separation was never real-time. |

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
- Consequence: the heavy models can run wherever there is a GPU or NPU (the
  user's laptop or phone, or a server), and the unit itself only needs
  enough compute for playback, mixing and UI. That is what makes a CM5
  viable.

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

Prices are 1K-unit list prices from the vendors as of July 2026 (NVIDIA raised
all Jetson prices that month; Raspberry Pi raised CM5 prices with DRAM costs).
Numbers marked "est." are estimates to be measured in Phase 1, not benchmarks.

| | Jetson Orin Nano 8GB module | **Raspberry Pi CM5 (chosen)** | Rockchip RK3588 module (e.g. Radxa CM5) |
|---|---|---|---|
| Module price | **$399** (4GB: $349). Single units $500-770 at resellers. | 4GB Lite wireless **$105**; 8GB Lite wireless **$170**; 8GB + 32GB eMMC wireless $205 (full table below) | launch pricing $75-89 for 8GB; 2026 pricing unverified |
| Storage needed on carrier | NVMe mandatory (no eMMC) | NVMe or eMMC variant; NVMe recommended for the library | eMMC on module |
| Cooling | fan + heatsink, 15-25 W | passive, 5-8 W | passive/small fan |
| Carrier board | 6-8 layer, 260-pin SO-DIMM, PCIe/DP/USB3 routing | 4-6 layer, 2x 100-pin Hirose, open KiCad IO-board reference from Raspberry Pi | 4-6 layer, CM4-style connectors |
| Wireless certification | separate M.2 module needed | pre-certified module on the CM5 itself (+$5) | varies |
| Audio bring-up | custom device-tree work | existing I2S DAC overlays (HiFiBerry-class) work out of the box | moderate |
| On-device heavy separation (4-min track) | ~1-3 min est. | 20-60 min est. CPU-only | somewhat faster CPU; NPU unproven for these models |
| Software ecosystem / longevity | JetPack; NVIDIA lifecycle commitment | Raspberry Pi OS; **in production until at least January 2036** | vendor BSPs, weaker long-term support |
| Approx. compute-related BOM delta vs CM5 | **+$330-400** (module, NVMe, fan, bigger PSU, harder board) | baseline | roughly CM5-class |

CM5 price list (Raspberry Pi product brief, July 2026, list prices):

| RAM | Lite (no eMMC) | 16GB eMMC | 32GB eMMC | 64GB eMMC |
|---|---|---|---|---|
| 2GB | $67.50 | $92.50 | $102.50 | $102.50 |
| 4GB | $100 | $125 | $135 | $135 |
| 8GB | $165 | $190 | $200 | $200 |
| 16GB | $295 | $320 | $330 | $330 |

Wireless adds $5 to each (e.g. 4GB Lite wireless $105, 8GB Lite wireless $170).

Recommendation: **CM5 8GB Lite wireless ($170)** for development and the first
build; decide after Phase 1 whether the 4GB part ($105) is enough for the
playback/mix workload plus UI, which it probably is if the on-device fallback
separator is a light model.

Important framing: the "PCB" you design is a **carrier board around a compute
module**, plus audio and controls. Start from Raspberry Pi's open CM5 IO Board
KiCad files.

What dropping the Jetson costs you: heavy separation cannot happen on the box.
That moves the "super accurate" separation into a companion app (and/or a cloud
service) and adds a second codebase to build and maintain. It does not reduce
separation accuracy, because the heavy models were always going to run at
import time, not during the mix.

## Product implications

### Positioning and rough BOM (CM5 unit, low hundreds of units)

| Item | Rough cost |
|---|---|
| CM5 8GB Lite wireless (or 4GB, $105) | $170 |
| 512 GB NVMe (boot + stem library) | ~$40 |
| Carrier PCB (4-6 layer) + assembly | $50-100 |
| DAC + analog output stage + headphone amp | ~$25 |
| 7" touchscreen (DSI) | $50-80 |
| Faders, encoders, buttons, LEDs, RP2040 | $40-80 |
| Passive heatsink | ~$5 |
| Enclosure (low volume) | $60-150 |
| External certified PSU | ~$15 |
| **Total** | **~$450-650** |

For comparison the Jetson version of the same unit came out at roughly
$750-1,000 after NVIDIA's July 2026 price increase. At a 2.5-3.5x hardware
multiplier the CM5 unit lands in the $1,000-1,800 retail band, with room to
target $999-1,499. The companion app is the extra engineering cost that buys
that saving.

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
  PCIe Gen2, USB 3, DSI and GbE still needs a disciplined stackup, ground
  plane strategy and shielded connectors to pass, but it is a tamer problem
  than a Jetson carrier. Budget a pre-compliance scan at
  DVT and $10-30k for the full pass.
- Radio: the CM5 wireless variant carries a **pre-certified Wi-Fi/BT module**
  (modular approval), so the radio portion of FCC/RED is inherited as long
  as the carrier uses an approved antenna configuration.
- Safety: keep mains out of the box. External certified 12-20 V adapter or
  USB-PD 20 V input.
- RoHS, REACH, WEEE; battery-free design keeps shipping simple.

### Thermal and acoustics

- CM5 at 5-8 W runs fanless with a modest heatsink coupled to the enclosure.
  No booth-noise problem. The on-device fallback separator should be
  throttled (nice level, core limit) so it never competes with playback.
- Validate the thermal design in the real enclosure at EVT, including a
  worst case of fallback separation running during a performance in a hot
  booth.

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
- CM5 stock has been tight at retailers in 2026; set up an approved
  Raspberry Pi reseller / design-partner relationship before EVT and place
  the pilot-run order early.

## Hardware block diagram (CM5 version)

```
 DC 12-20 V in / USB-PD (external certified PSU)
        |
   5 V / 3 A buck  ----> Raspberry Pi CM5 8GB Lite wireless (2x 100-pin Hirose)
        |                      |  |  |  |  |
   3.3 V / 1.8 V rails         |  |  |  |  +-- M.2 M-key NVMe over PCIe Gen2 x1 (boot + stem library)
                               |  |  |  +----- Wi-Fi/BT: on-module, pre-certified (antenna on carrier or external)
                               |  |  +-------- GbE (on-module PHY; carrier adds magnetics + RJ45)
                               |  +----------- DSI -> 7" touchscreen (HDMI available as alternative)
                               +-------------- USB 3 host x2 (USB sticks, MIDI controllers)
        I2S + MCLK (44.1 kHz family, 22.5792 MHz local oscillator)
                               |
                         DAC (PCM5122 budget / ES9038Q2M hi-fi)
                               |
            +------------------+------------------+
      headphone amp      balanced master out     booth out
      (TPA6120 class)    (THAT1646 / DRV134, TRS/XLR)

 Control surface: RP2040 scans faders/pots/encoders/buttons/LEDs,
 presents as USB-HID or UART to the CM5. Keeps timing off Linux.

 Thermal: passive heatsink to the enclosure, no fan.
```

Notes:
- Lite variant has no eMMC; boot from NVMe. The NVMe also holds the stem
  library (~80-100 MB/track as FLAC; 512 GB holds ~5,000 tracks).
- I2S DACs with existing Raspberry Pi device-tree overlays (PCM5122 and
  similar) bring up in minutes. Keep the DAC in clock-master mode from a
  local audio-grade oscillator rather than deriving MCLK from the SoC.
- Open reference design: Raspberry Pi publishes the CM5 IO Board schematics
  and KiCad files. Start from those.
- Audio spec targets: 24-bit / 44.1 or 48 kHz, DAC dynamic range > 110 dB,
  balanced master at +4 dBu, headphone out ~2 x 250 mW into 32 ohm.

## Software stack

Two deliverables: the **unit firmware** (CM5) and the **companion app**
(macOS / Windows / iOS / Android) that does the heavy analysis.

| Stage | Choice | Runs where |
|---|---|---|
| Heavy separation | Licensed on-device SDK (AudioShake Local Inference SDK ships for macOS, Windows, iOS, Android, Linux with GPU backends; Music.AI Embedded SDK) or legally cleared Demucs-class model, using the user's GPU / Apple Neural Engine. Optional cloud service for users without capable hardware. | **Companion app** (or cloud) |
| Light on-device fallback | A small separator (Open-Unmix class, or a quantized light model) so a track dropped straight onto the unit via USB gets usable stems in a few minutes; replaced by HQ stems when the companion app syncs them. Quality to be measured; may be "drums/vocals only". | Unit, background, throttled |
| Beats / downbeats / structure / key | BeatNet (CC BY 4.0) or in-house on the drum stem; allin1-style structure model after license check; in-house key detector | Companion app, with a CPU-feasible subset on the unit |
| Cache / sync | per-track folder: 4 stems as FLAC + `analysis.json`. Synced over Wi-Fi (unit advertises an mDNS service) or copied via USB stick in the same layout. | Both |
| Mix planner | rule engine over `analysis.json` pairs: phrase-aligned entry/exit points, Camelot key check, energy curve matching, per-stem swap schedule, EQ/filter automation. Optional LLM step for style selection (cloud). | Unit, at load, <1 s |
| Real-time engine | Rust (cpal) or C++ (commercial JUCE): 8 stream readers, per-stem time-stretch (signalsmith-stretch MIT, or licensed Rubber Band R3), per-stem EQ/filter/gain automation lanes, sum, limiter, 44.1 kHz / 24-bit out over I2S | Unit, always |
| OS / updates | Raspberry Pi OS Lite (64-bit) or Yocto; A/B rootfs with signed OTA; encrypted partition for licensed model files | Unit |
| UI | touchscreen app (web view or LGPL Qt) + RP2040 control surface | Unit |

## Development plan

Do not spin a PCB until the software pipeline is proven on dev hardware.

0. **Phase 0 — product definition (parallel with Phase 1).**
   Target price and first-run volume; separation-model strategy (licensed SDK
   vs. cleared Demucs vs. in-house); request SDK quotes from AudioShake and
   Music.AI; distributor contact for module pricing and EOL date; counsel
   review of the licensing table above.
1. **Phase 1 — pipeline on dev hardware (2-4 weeks).**
   Raspberry Pi 5 8GB (same silicon as CM5) + a USB audio interface + NVMe
   HAT, plus a laptop for the heavy side.
   Deliverables: the stem cache format; the companion-side analysis
   pipeline (separation on the laptop GPU, beat grid, structure, key);
   measured time and RAM for the light on-device fallback separator on the
   Pi 5; a Python prototype that renders a stem-level transition between
   two analyzed tracks to a WAV. Community checkpoints are fine here for
   benchmarking only.
   Exit criterion: a blind listening test says the transitions are "clean"
   on 10 varied track pairs, and the Pi 5 plays 8 stretched stems with
   headroom to spare.
2. **Phase 2 — real-time engine.** Port the renderer to Rust/C++ with
   per-stem stretch and automation lanes; play live through the USB
   interface; add the mix planner as a service.
3. **Phase 3 — audio + control daughterboard (EVT-0).** A simple 2-4 layer
   HAT on the Pi 5 40-pin header: I2S DAC, headphone amp, balanced outs,
   RP2040 control surface. Validates the analog design at low risk.
4. **Phase 4 — custom CM5 carrier (EVT).** 4-6 layer board derived from the
   CM5 IO Board reference: 5 V buck, M.2 NVMe, DSI display, USB 3, GbE,
   antenna, plus the Phase 3 audio section, in the production enclosure
   with passive cooling. Companion app reaches feature parity with the
   laptop prototype on macOS and Windows; mobile follows.
5. **Phase 5 — DVT.** Pre-compliance EMC scan, thermal/acoustic validation,
   drop and connector-cycle tests, firmware feature-complete, OTA tested.
6. **Phase 6 — PVT and pilot run.** Test fixtures, flashing station, first
   50-100 units, full compliance pass, then MP.

## Open questions (answers change the design)

1. Separation-model strategy for the companion app: licensed SDK, legally
   cleared Demucs, or in-house training? This is the biggest cost and
   quality lever. (AudioShake's SDK covering macOS/Windows/iOS/Android fits
   the companion-app design well; get a quote.)
2. Target retail price and first-run volume?
3. Companion app scope at launch: desktop only (macOS + Windows), or mobile
   too? Is a cloud separation service wanted for users without a capable
   laptop?
4. Confirm the working assumption of a file-based library only in v1 (no
   live line-in).

Answered: this is a product; no Jetson, CM5 it is (2026-10-17).

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
- NVIDIA Jetson price increase, July 2026 (Orin Nano 8GB module $299 -> $399 at 1KU): https://www.cnx-software.com/2026/07/22/nvidia-increases-the-price-of-jetson-modules-and-devkits-by-up-to-101/
- Raspberry Pi CM5 product brief with price list and 2036 availability (July 2026): https://pip-assets.raspberrypi.com/categories/944-raspberry-pi-compute-module-5/documents/RP-008181-DS-10-cm5-product-brief.pdf
- Radxa CM5 (RK3588S) launch pricing: https://radxa.com/blog/CM5-Official-Announcement
