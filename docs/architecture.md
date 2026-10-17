# DJ AI Board — Architecture Plan

Status: draft v0.5 (design discussion, nothing built yet)

## Decision log

| Date | Decision | Consequence |
|---|---|---|
| 2026-10-17 | This is a **product**, not a one-off. | Custom carrier board is mandatory. BOM cost, lifecycle, licensing, compliance and manufacturing enter the design from day one. |
| 2026-10-17 | Working assumption: **file-based library, no live line-in** in v1. | High-accuracy stem separation of a live stream is not practical. Revisit for v2 only with a lighter, causal model. |
| 2026-10-17 | **No Jetson.** First alternative was the Raspberry Pi CM5 ($105-170); superseded by the next row. | Fanless, simpler carrier, open KiCad reference designs, pre-certified wireless on-module. |
| 2026-10-17 | **Much cheaper compute wanted.** Primary target is the **Raspberry Pi Zero 2 W ($15)**; step-up is the **CM4 1GB Lite wireless ($46.25)**. CM5 stays documented as the "if we outgrow it" tier. | Requires the **pre-stretch architecture** below (time-stretch in the background, only gain/EQ live), a minimal on-board UI with the phone as the rich screen, microSD storage, and dropping the on-device fallback separator. Target BOM ~$100-120, retail ~$249-349. |
| 2026-10-17 | **Heavy separation runs off the unit**, in a companion app on the user's laptop/phone (optionally a cloud service). No on-device separation on the cheap tier. | Small ARM CPUs cannot run RoFormer/Demucs-class models in acceptable time. DJs already prep libraries on a laptop (rekordbox/Serato model), so this matches existing workflow. Accuracy is unchanged because separation was never real-time. |

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
  enough compute for playback, mixing and UI. That is what makes a $15 board
  viable.

## The second decision that makes cheap compute possible: pre-stretch, live-mix

The mix is planned by the AI, not performed by a human on a jog wheel. That
means the only expensive part of the live path, **time-stretching 8 stems**,
does not have to happen in real time either:

- A background **renderer** follows the planner's tempo curve and writes
  pre-stretched versions of the 8 stems a few seconds ahead of the playhead
  (incrementally, streaming, not whole-transition renders).
- The **live engine** only reads those 8 pre-stretched streams and applies the
  automation lanes: per-stem gain, EQ, filter, crossfade, sum, limiter. That is
  a handful of biquads and multiplies per sample, trivial on any ARM core.
- As long as the renderer runs faster than 1x real time in aggregate, plan
  changes (user picks a different next track) take effect within a buffer or
  two. If it runs slower than 1x on the chosen SoC, the UI simply commits a
  transition a few seconds ahead. Phase 1 measures this.
- Physical controls still work live: stem level, EQ, filter and crossfader
  act on the live engine. Tempo nudging is the one control that goes through
  the renderer and therefore has a small lag; that is acceptable for an
  auto-mix product.

Estimate for a signalsmith-stretch class phase-vocoder: on the order of 1
GFLOP/s for 16 channels at 44.1 kHz, which four Cortex-A53 cores at 1 GHz
should sustain at roughly 1-3x real time. To be measured; this number decides
between the Zero 2 W and the CM4.

## What the AI actually is

There is no single "AI". The product is a pipeline of four different kinds of
model, and only two of them are neural networks.

| Stage | Kind of AI | What it produces | Where |
|---|---|---|---|
| 1. Stem separation | Deep neural network (band-split transformer or hybrid U-Net on spectrograms), supervised on multitrack data | 4 stems per track | companion app / cloud |
| 2. Music analysis | Small neural nets + probabilistic decoding for beats/downbeats (CRNN/TCN + dynamic Bayesian network) and section structure (transformer); plain signal processing for key, loudness, per-bar energy, vocal activity, bass presence | `analysis.json`: beat grid, downbeats, phrases (8/16/32-bar), section labels (intro/build/drop/break/outro), key, LUFS, per-stem per-bar energy, vocal-activity per bar | companion app |
| 3. Mix planning (the "DJ brain") | **Constraint search + scoring function**, i.e. classical AI. Rules encode DJ craft; a cost function ranks candidate transitions; priors come from analyses of real DJ mixes. Later: weights learned from user feedback. Not an LLM; an LLM cannot hear. | a transition plan: exit point in A, entry point in B, tempo curve, template, per-stem automation lanes | unit, at load |
| 4. Rendering | DSP, not AI: phase-vocoder time-stretch, biquad EQ/filters, gain lanes, limiter | audio | unit |

The planner is where "sounds good" is decided, and it is deliberately not a
black box: every rule below is inspectable and tunable, and a failed
transition can be traced to a rule or a bad analysis value.

### The planner's rules (DJ craft, written down)

1. **Phrase alignment.** Transitions start and end on phrase boundaries
   (8/16/32 bars), downbeat to downbeat. Analyses of 20k real transitions
   show cue points cluster at phrase boundaries; this single rule is most of
   what separates "DJ" from "crossfade".
2. **Tempo.** Stretch no stem beyond about +/-8% (quality cliff); prefer
   <= 4%. Match B to A's tempo during the overlap, then drift to B's native
   tempo after A is out. Allow half/double-time relationships.
3. **Harmony.** Camelot neighbours (same, +/-1, relative). When keys clash,
   overlap **drums only**; stems make a harmonically safe transition always
   available. Allow +/-1 semitone shift of one track as a last resort.
4. **One bass at a time.** Bass stems never overlap; the swap happens on a
   downbeat at a phrase boundary. Two kick/bass stacks is the classic
   amateur mud.
5. **Vocals never collide.** Using per-bar vocal activity, B's vocal enters
   only when A's is silent, or after A's vocal stem is out.
6. **Energy continuity.** Match section types: A's outro or breakdown into
   B's intro or build; land B's drop on the phrase boundary where A's energy
   would have dropped. Never drop into an intro.
7. **Choreography templates.** Transitions are instances of parameterised
   templates, e.g. *drum swap* (B drums under A for 16 bars, bass swap at
   the boundary, B tonal stems in as A's fade), *breakdown bridge* (B drums
   enter during A's breakdown and carry into B's drop), *acapella bridge*
   (A vocal over B instrumental for 8-16 bars, key-matched), *cut* (on a
   downbeat, with echo-out) for incompatible pairs.
8. **Gain staging.** Per-stem gains from analysis keep the summed output at
   a constant short-term loudness; a transparent limiter catches peaks. A
   high-pass on B until the bass swap; a gentle low-pass on A as it leaves.

The search: for each candidate (exit phrase in A, entry phrase in B,
template, length), compute a cost from phrase misalignment, stretch %, key
clash on overlapping tonal stems, vocal overlap seconds, energy
discontinuity, section compatibility and analysis confidence. Pick the
minimum. Weights are hand-tuned first, then fitted to listening-test and
user-skip data. This is the AutoMashUpper idea (compatibility measured after
allowed tempo/key adjustment) plus DJ-specific constraints.

### Safe fallbacks beat clever failures

When the analysis is uncertain (low beat-tracker confidence, no phrase
structure found, ambiguous key, tempo gap too large), the planner degrades to
a safe template: drums-only overlap, a short 8-bar blend, or a clean cut on a
downbeat. A product is judged on its worst transition, not its best.

## Making it sound good: the quality plan

Most "AI mixes" sound bad for three reasons, in this order: wrong beat grid
(flams and drift), audible time-stretch and stem artefacts, and musically
wrong transition points. The plan attacks each.

1. **Play the original, not the stems, outside transitions.** Separation
   artefacts are only audible when stems are soloed. The engine crossfades
   from the original mixdown to the stem sum a bar before a transition and
   back a bar after. Stems are exposed for maybe 10% of the set.
2. **Beat grids on the drum stem, verified.** Track beats on the isolated
   drums, snap the grid to transients, and align A to B during the overlap
   by cross-correlating drum onset envelopes. Target < 5 ms alignment error;
   flams become audible around 10-20 ms.
3. **Per-stem stretch settings and a hard stretch budget.** Drums with
   transient preservation, vocals with formant preservation, tonal stems
   with the default. Never exceed the per-genre stretch budget; prefer a
   different entry point over a bigger stretch.
4. **Genre scoping.** Launch on 4/4 electronic (house, techno, DnB) where
   structure and tempo are regular. Hip-hop and pop have irregular structure
   and tempo and come later, with their own templates.
5. **A test corpus with ground truth.** 200+ licensed tracks per launch genre
   with human-verified beat grids, keys and section labels; 100+ track
   pairs. Every analysis model is scored against it before shipping, and
   every planner change is regression-tested on it.
6. **Objective metrics per rendered transition**, logged in CI: alignment
   error (ms), stretch %, overlapping-tonal-stem key distance, vocal overlap
   seconds, loudness jump (LU), limiter gain reduction (dB), separation SDR
   of the shipping model on a held-out set.
7. **Blind listening tests with DJs.** MUSHRA-style: our transition vs. a
   human DJ's vs. baselines (crossfade, Mixxx AutoDJ, djay Automix). Rate
   smoothness, musicality, artefacts, "would you play this". Launch
   criterion: >= 80% of in-genre transitions rated playable, and no
   transition rated "broken". A 2018 automatic Drum-and-Bass DJ built on
   these same MIR ingredients scored highly with genre fans in exactly this
   kind of test, so the bar is reachable.
8. **Feedback loop in the product.** Thumbs-down / skip on a transition is
   logged with the plan and the analysis values; this is the training
   signal for planner weights and the queue for analysis-bug triage.

Prior art worth reading before writing the planner: Vande Veire & De Bie,
*From raw audio to a seamless mix* (EURASIP 2018, open source, AGPL so read
it, do not copy it); Kim et al., *A Computational Analysis of Real-World DJ
Mixes* (ISMIR 2020, cue-point statistics from 1,557 mixes); Davies et al.,
*AutoMashUpper* (IEEE/ACM TASLP 2014, the mashability score).

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

Prices are vendor list prices as of mid-2026 (Raspberry Pi product briefs and
the 14 Aug 2026 pricing guide; NVIDIA's July 2026 price list). Numbers marked
"est." are estimates to be measured in Phase 1.

### Cheap tier (what we are designing for)

| | **Raspberry Pi Zero 2 W (primary)** | **Pi CM4 1GB Lite wireless (step-up)** | Pi CM4 2GB Lite wireless | Radxa Zero 3W (RK3566) |
|---|---|---|---|---|
| Price | **$15** (unchanged since launch) | **$46.25** (non-wireless $41.25) | $62.50 | retail listings $44-62, out of stock; list price unverified |
| CPU / RAM | 4x Cortex-A53 @1 GHz, 512 MB | 4x Cortex-A72 @1.5 GHz, 1 GB | same, 2 GB | 4x Cortex-A55 @1.6 GHz, 1-4 GB |
| Wireless | 2.4 GHz Wi-Fi 4 + BT 4.2, on board, certified | dual-band Wi-Fi 5 + BT 5.0 on module, certified | same | dual-band Wi-Fi 5, on board |
| Display | mini-HDMI on board, SPI TFT, or DPI over GPIO (pin-plan against I2S) | DSI, HDMI x2, DPI | same | mini-HDMI, DSI |
| Storage | microSD | microSD/USB (Lite) or eMMC variant; PCIe x1 available | same | microSD/eMMC |
| Carrier | 2-layer, board sits on a 40-pin header + USB pads | 4-layer, 2x 100-pin Hirose, open CM4 IO Board KiCad design | same | 2-layer |
| Form factor | an SBC, not a module; fine for low/mid volume, Raspberry Pi supports industrial use | true compute module | same | SBC |
| Pre-stretch render speed, 8 stems | ~1-3x real time est. | ~3-6x real time est. | same | ~2-4x est. |
| Supply commitment | in production until at least **January 2030** | until at least **January 2034** | same | until Sept 2033 |

Recommendation: **design for the Zero 2 W**, and keep the carrier's audio and
control sections identical to a CM4 variant so the step-up is a connector
change rather than a redesign if Phase 1 shows the Zero 2 W is too slow, or if
you want DSI, 5 GHz Wi-Fi or more RAM. The 1 GB CM4 is the cheapest true
compute module Raspberry Pi sells.

What the cheap tier gives up:
- No on-device separation at all (not even a light fallback). Stems come from
  the companion app (laptop/phone GPU) or a cloud service, synced over Wi-Fi
  or USB stick.
- 2.4 GHz-only Wi-Fi on the Zero 2 W: syncing 100 tracks (~8-10 GB of FLAC
  stems) takes tens of minutes; a USB stick is the fast path.
- A small on-board display (or none): the phone/laptop app is the rich UI.
- Time-stretch via the background renderer (see pre-stretch section).

### Reference: the tiers we moved away from

| | Jetson Orin Nano 8GB module | Raspberry Pi CM5 |
|---|---|---|
| Module price (1K units) | $399 (4GB: $349) | 2GB Lite wireless $72.50; 4GB Lite wireless $105; 8GB Lite wireless $170 |
| Why not | +$330-400 compute BOM, fan, 6-8 layer carrier | still $70-170 of compute for a job that needs far less once separation is off-device and stretching is pre-rendered |
| Where it would still make sense | on-device heavy separation | on-device fallback separation, rich touchscreen UI, 5 GHz Wi-Fi, NVMe library |

CM5 list prices (July 2026) for reference: 2GB Lite $67.50 / 4GB Lite $100 /
8GB Lite $165 / 16GB Lite $295; eMMC +$25-35; wireless +$5. In production
until at least January 2036.

Important framing: the "PCB" you design is a **carrier board around a compute
module or SBC**, plus audio and controls. Nobody should route an SoC + DRAM
from scratch for this.

## Product implications

### Positioning and rough BOM (cheap tier, low hundreds of units)

| Item | Zero 2 W unit | CM4 1GB unit |
|---|---|---|
| Compute | $15 | $46.25 |
| 128 GB microSD (stem library, ~1,300 tracks) | $10 | $10 |
| Carrier PCB + assembly | $15-25 (2-layer) | $30-50 (4-layer) |
| DAC (PCM5102A), headphone amp, RCA line out | $6-8 | $6-8 |
| Balanced TRS outs (optional) | +$10 | +$10 |
| Display: 3.5-4" SPI touch TFT, or OLED + encoders with the phone as rich UI | $5-20 | $5-20 (or $40 DSI 5") |
| Controls (encoders, pots, buttons, LEDs) + I2C ADC/expander or RP2040 | $10-20 | $10-20 |
| Enclosure (low volume) | $20-40 | $25-50 |
| 5 V USB-C PSU | $6 | $6 |
| **Total** | **~$95-150** | **~$140-220** |

At a 2.5-3x multiplier the Zero 2 W unit retails around **$249-349** and the
CM4 unit around **$399-549**. Previous tiers for comparison: CM5 unit
~$450-650 BOM, Jetson unit ~$750-1,000 BOM.

Positioning note: at this price point the buyer will ask "why not just the
app?" The hardware's answer has to be: dedicated audio outputs, physical
controls, and a standalone box that plays the set without a laptop at the
gig. Keep those three things good; everything else can be in the app.

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

- EMC: FCC Part 15 Class B, CE (EN 55032 / 55035), UKCA. A 2-layer carrier
  with I2S, SPI and USB 2.0 is about as easy as EMC gets; keep the DAC
  clock and switching regulator away from the analog section. Budget a pre-compliance scan at
  DVT and $10-30k for the full pass.
- Radio: the Zero 2 W and the CM4 wireless variant carry their own radio
  certifications, so the radio portion of FCC/RED is inherited as long as
  the board is used unmodified with its on-board or approved antenna.
  Product-level EMC testing is still required.
- Safety: keep mains out of the box. External certified 12-20 V adapter or
  USB-PD 20 V input.
- RoHS, REACH, WEEE; battery-free design keeps shipping simple.

### Thermal and acoustics

- Zero 2 W / CM4 run at 2-5 W. Passive, no heatsink beyond the enclosure
  contact in most cases; the CM4 wants a small heatsink if the renderer runs
  flat out. No fan anywhere in the product.

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
  channel balance), control-surface test, microSD image flashing station,
  serialization.
- Raspberry Pi stock has been tight at retailers in 2026; set up an
  approved reseller / design-partner relationship before EVT and place the
  pilot-run order early.

## Hardware block diagram (cheap tier)

```
 5 V USB-C in (external certified PSU)
        |
   5 V rail -----> Raspberry Pi Zero 2 W on 40-pin header (+ USB pads)
   3.3 V LDO            |   |   |   |
                        |   |   |   +-- microSD on the Zero (OS + stem library)
                        |   |   +------ Wi-Fi 2.4 GHz / BT on board (certified)
                        |   +---------- USB 2.0 OTG -> USB-A host port on carrier (USB sticks, MIDI)
                        +-------------- display: SPI touch TFT on GPIO, or mini-HDMI panel,
                                        or none (OLED + encoders; phone is the rich UI)
        I2S (GPIO 18/19/21) + MCLK from a local 22.5792 MHz oscillator
                        |
                  DAC PCM5102A (budget) / PCM5122 (I2C control) / ES9023
                        |
            +-----------+-----------+
      headphone amp   RCA line out   optional balanced TRS (THAT1646)
      (TPA6132 class)

 Controls: encoders + buttons on GPIO, pots via I2C ADC (ADS1115 class) or an
 RP2040 scanning everything and talking UART/USB-HID. LED ring / meters via
 I2C or SPI driver.

 Step-up variant: same carrier sections, with the Zero 2 W header replaced by
 2x 100-pin Hirose for a CM4 (adds DSI display, 5 GHz Wi-Fi, more RAM).
```

Notes:
- PCM5102A/PCM5122-class DACs have stock Raspberry Pi device-tree overlays;
  bring-up is minutes.
- Keep the DAC in clock-master mode from a local audio-grade oscillator rather
  than deriving MCLK from the SoC.
- Audio spec targets: 24-bit / 44.1 kHz, DAC dynamic range > 100 dB,
  line out 2 Vrms, headphone out ~2 x 100 mW into 32 ohm. Balanced +4 dBu
  outs as a cost option.
- Open reference designs: Raspberry Pi publishes the CM4 IO Board KiCad files
  for the step-up variant; the Zero 2 W carrier is simple enough to draw from
  the Zero 2 W schematics and pinout.

## Software stack

Two deliverables: the **unit firmware** (Zero 2 W / CM4) and the **companion app**
(macOS / Windows / iOS / Android) that does the heavy analysis.

| Stage | Choice | Runs where |
|---|---|---|
| Heavy separation | Licensed on-device SDK (AudioShake Local Inference SDK ships for macOS, Windows, iOS, Android, Linux with GPU backends; Music.AI Embedded SDK) or legally cleared Demucs-class model, using the user's GPU / Apple Neural Engine. Optional cloud service for users without capable hardware. | **Companion app** (or cloud) |
| On-device separation | None on the cheap tier. A track dropped onto the unit without stems plays as a plain file with mix-level transitions until the app supplies stems. | - |
| Beats / downbeats / structure / key | BeatNet (CC BY 4.0) or in-house on the drum stem; allin1-style structure model after license check; in-house key detector | Companion app only |
| Cache / sync | per-track folder: 4 stems as FLAC + `analysis.json` (~80-100 MB/track). Synced over Wi-Fi (unit advertises an mDNS service) or copied via USB stick in the same layout; USB is the fast path on 2.4 GHz-only hardware. | Both |
| Mix planner | rule engine over `analysis.json` pairs: phrase-aligned entry/exit points, Camelot key check, energy curve matching, per-stem swap schedule, EQ/filter automation. Optional LLM step for style selection (cloud). | Unit, at load, <1 s |
| Background renderer | Rust or C++: follows the planner's tempo curve, time-stretches the 8 stems (signalsmith-stretch, MIT) a few seconds ahead of the playhead into shared ring buffers | Unit, background thread(s) |
| Live engine | Rust or C++ on ALSA: reads 8 pre-stretched streams, per-stem EQ/filter/gain automation lanes + physical controls, sum, limiter, 44.1 kHz / 24-bit out over I2S. No stretching in this path. | Unit, real-time thread |
| OS / updates | Raspberry Pi OS Lite (64-bit) or Buildroot/Yocto for a small, fast-booting image; A/B rootfs with signed OTA | Unit |
| UI | minimal on-board UI (LVGL or Slint on a small TFT/OLED; no browser on 512 MB) + physical controls; the companion app is the rich UI over Wi-Fi/BLE | Unit + app |

## Development plan

Do not spin a PCB until the software pipeline is proven on dev hardware.

0. **Phase 0 — product definition (parallel with Phase 1).**
   Target price and first-run volume; separation-model strategy (licensed SDK
   vs. cleared Demucs vs. in-house); request SDK quotes from AudioShake and
   Music.AI; distributor contact for module pricing and EOL date; counsel
   review of the licensing table above.
1. **Phase 1 — pipeline on dev hardware (2-4 weeks).**
   A Raspberry Pi Zero 2 W and a CM4 IO Board + CM4 1GB Lite, each with a
   $10 I2S DAC HAT, plus a laptop for the heavy side.
   Deliverables: the stem cache format; the companion-side analysis
   pipeline (separation on the laptop GPU, beat grid, structure, key); a
   Python prototype that renders a stem-level transition between two
   analyzed tracks to a WAV; and the **key measurement**: aggregate
   time-stretch throughput for 8 stems on the Zero 2 W and the CM4
   (x real time) and RAM headroom with the live engine and a small UI
   running. Community checkpoints are fine here for benchmarking only.
   Exit criterion: a blind listening test says the transitions are "clean"
   on 10 varied track pairs, and the chosen board renders faster than 1x
   real time with the UI running.
2. **Phase 2 — renderer + live engine.** Port to Rust/C++: background
   pre-stretch renderer feeding ring buffers, live engine with automation
   lanes and physical-control input, playing through the I2S DAC HAT; mix
   planner as a service; first companion-app sync over Wi-Fi.
3. **Phase 3 — carrier board rev A (EVT).** The 2-layer Zero 2 W carrier:
   5 V input, I2S DAC + headphone amp + line outs, display connector,
   controls, USB-A host. Because the compute is a $15 board on a header,
   this *is* the product PCB; there is no separate daughterboard phase.
   Lay out the CM4 variant of the same carrier in parallel if Phase 1 was
   marginal on the Zero 2 W.
4. **Phase 4 — rev B + enclosure (EVT-2).** Fixes from rev A, production
   enclosure, companion app at feature parity on macOS and Windows; mobile
   follows.
5. **Phase 5 — DVT.** Pre-compliance EMC scan, thermal/acoustic validation,
   drop and connector-cycle tests, firmware feature-complete, OTA tested.
6. **Phase 6 — PVT and pilot run.** Test fixtures, flashing station, first
   50-100 units, full compliance pass, then MP.

## Open questions (answers change the design)

1. Separation-model strategy for the companion app: licensed SDK, legally
   cleared Demucs, or in-house training? This is the biggest cost and
   quality lever. (AudioShake's SDK covering macOS/Windows/iOS/Android fits
   the companion-app design well; get a quote.)
2. On-board UI: a small touchscreen, or just OLED + encoders with the phone
   as the screen? (Roughly $15 of BOM and a lot of firmware either way.)
3. Physical controls wanted on the box: just play/next/volume, or stem
   faders + EQ + crossfader? (Decides the control-scan hardware and the
   front panel.)
4. Companion app scope at launch: desktop only (macOS + Windows), or mobile
   too? Is a cloud separation service wanted for users without a capable
   laptop?
5. Confirm the working assumption of a file-based library only in v1 (no
   live line-in).

Answered: this is a product; no Jetson; compute must be much cheaper, so
Zero 2 W primary / CM4 1GB step-up (2026-10-17).

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
- Raspberry Pi 2026 pricing guide (Zero 2 W $15, Pi 5 1GB $45 etc., updated 14 Aug 2026): https://magazinmehatronika.com/en/the-2026-raspberry-pi-pricing-and-buying-guide/
- Raspberry Pi Zero 2 W product page ($15, in production until at least January 2030): https://www.raspberrypi.com/products/raspberry-pi-zero-2-w/
- Raspberry Pi CM4 product brief with price list (1GB Lite $41.25, wireless $46.25; until January 2034): https://pip-assets.raspberrypi.com/categories/634-raspberry-pi-compute-module-4/documents/RP-008169-DS-8-cm4-product-brief.pdf
- Radxa Zero 3W product brief (availability until Sept 2033): https://dl2.radxa.com/zero3/docs/hw/3w/radxa_zero_3w_product_brief.pdf
- Vande Veire & De Bie, automated DJ system for Drum and Bass (EURASIP 2018): https://link.springer.com/article/10.1186/s13636-018-0134-8
- Kim et al., computational analysis of real-world DJ mixes (ISMIR 2020): https://arxiv.org/abs/2008.10267
- Davies et al., AutoMashUpper (IEEE/ACM TASLP 2014): https://staff.aist.go.jp/m.goto/PAPER/IEEETASLP201412davies.pdf
