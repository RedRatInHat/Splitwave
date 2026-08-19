[![Downloads](https://img.shields.io/badge/dynamic/json?url=https%3A%2F%2Fsplitwave.app%2Fapi%2Fdownloads&query=downloads&label=downloads&color=brightgreen)](https://github.com/Horuse/Splitwave/releases/latest)
[![Support](https://img.shields.io/badge/Support-donate-yellow)](https://github.com/Horuse/Splitwave#support)

# Splitwave

Splitwave is a node-based audio router for macOS, Linux, and Windows. Wire microphones, system audio, per-app capture, and WAV files into a visual graph, run them through a chain of effects — EQ, compression, reverb, limiting, and more, plus your own CLAP, VST3 and AU plugins — then send the result to speakers or record it in WAV, FLAC, AIFF, MP3, Opus, or AAC.

![Splitwave preview](./preview.webp)

## Installation

### macOS

Download the latest `.dmg` from [Releases](https://github.com/Horuse/Splitwave/releases/latest),
open it, and drag Splitwave to Applications.

**macOS will block the app on first launch** ("cannot verify developer") because the
binary is not notarized. To allow it, run once in Terminal:

```bash
xattr -cr /Applications/Splitwave.app
```

Then open Splitwave normally.

**After each update, Screen Recording permission resets** (macOS revokes it when the binary changes and the app is unsigned). To re-grant it: open System Settings → Privacy & Security → Screen Recording, click **−** to remove Splitwave, then click **+** and add it back.

### Linux

Requires a PipeWire-based audio session (default on most current distros).
Device-volume control additionally needs the PulseAudio compatibility layer
(`pipewire-pulse`, also default on most distros).
Download the build for your system from [Releases](https://github.com/Horuse/Splitwave/releases/latest):

- **AppImage** — `chmod +x Splitwave_*.AppImage && ./Splitwave_*.AppImage`
- **`.deb`** (Debian/Ubuntu) — `sudo apt install ./Splitwave_*.deb`
- **`.rpm`** (Fedora/RHEL/openSUSE) — `sudo rpm -i Splitwave-*.rpm`

### Windows

Requires Windows 10 version 2004 or newer (for per-app capture) and the
[WebView2 runtime](https://developer.microsoft.com/microsoft-edge/webview2/)
(preinstalled on current Windows 10/11). Download the `.exe` installer from
[Releases](https://github.com/Horuse/Splitwave/releases/latest) and run it.

Virtual audio devices are not available on Windows.

## Platform support

| Feature                                   |          macOS           |          Linux           |              Windows              |
| ----------------------------------------- | :----------------------: | :----------------------: | :-------------------------------: |
| Mic / speaker device I/O                  |            ✅            |            ✅            |                ✅                 |
| N-channel microphone arrays               | ⚠️ implemented, untested | ⚠️ implemented, untested |     ⚠️ implemented, untested      |
| System audio capture                      |   ✅ ScreenCaptureKit    |       ✅ PipeWire        |        ✅ WASAPI loopback         |
| Per-app audio capture                     |   ✅ ScreenCaptureKit    |       ✅ PipeWire        | ✅ Process Loopback (Win10 2004+) |
| App icons in the picker                   |            ✅            |            ✅            |                ✅                 |
| Device volume control                     |            ✅            |            ✅            |                ✅                 |
| Recording: WAV / FLAC / AIFF / MP3 / Opus |            ✅            |            ✅            |                ✅                 |
| Recording: AAC (M4A)                      |            ✅            |            ❌            |                ❌                 |
| Virtual audio devices                     |   ✅ AudioServerPlugin   |  ✅ PipeWire null-sinks  |  ❌ (no user-mode driver model)   |
| Effects, metering, file playback          |            ✅            |            ✅            |                ✅                 |
| CLAP plugins                              |            ✅            |            ✅            |                ✅                 |
| VST3 plugins                              |            ✅            |          ✅ X11          |                ✅                 |
| Audio Unit plugins                        |            ✅            |            ❌            |                ❌                 |

## Features

- **Inputs:** microphones, system audio, per-application audio, WAV files,
  virtual device loopback, and N-channel microphone arrays with shared-clock
  or experimental independent-device synchronization
- **Outputs:** physical speakers/interfaces, file recording in WAV (16/24-bit
  PCM + 32-float), FLAC, AIFF, Opus, MP3, AAC (M4A), virtual devices
- **Effects:** Gain, Mute, Channel Balance, Saturator, 10-band Graphic EQ,
  Brick-wall Limiter with look-ahead, Compressor (with sidechain), Noise Gate
  (with sidechain), Noise Suppressor, De-esser, Declick, Stereo Delay,
  Algorithmic Reverb (Freeverb), Level Meter, EBU R128 LUFS Meter, Waveform and
  Spectrum analyzers
- **Plugins:** host your own CLAP and VST3 plugins (all platforms) and Audio
  Unit plugins (macOS) as effect nodes, with the native editor embedded in the
  app, parameters editable in the node, and plugin state saved with the
  pipeline. On Linux, plugin editors need an X11 session (XWayland works);
  VST3 defines no Wayland embedding
- **Presets & templates:** shared effect presets with factory defaults, plus
  ready-made pipeline templates in the create flow, including
  `Spatial Voice — Multi-Mic`
- **Virtual devices:** create named virtual audio devices that appear system-wide.
  Use them to capture loopback audio from any app or to feed processed audio into
  apps that accept a microphone input (DAWs, Discord, etc.)

System and per-app capture use **ScreenCaptureKit** on macOS, **PipeWire** on
Linux, and **WASAPI loopback** / the **Process Loopback API** on Windows. Virtual
devices are AudioServerPlugin drivers on macOS and PipeWire null-sinks on Linux;
Windows has no user-mode virtual-device model, so they are unavailable there.

Microphone Array combines two or more selected physical input channels into one
calibrated mono `Spatial Voice` stream. It supports linear, circular,
rectangular, and custom geometry; fixed direction or point targets;
Delay-and-Sum, GSC, and MVDR processing; live diagnostics; and safe fallback.
One multichannel interface is recommended. Independent USB devices work in an
explicitly experimental clock-synchronization mode and are less stable. See the
[Microphone Array guide](docs/microphone-array.md),
[architecture](docs/microphone-array-architecture.md), and
[hardware test guide](docs/microphone-array-hardware-test.md).
The Microphone Array contribution is attributed to
[Red Rat in Hat](https://redratinhat.com/products/) in the node's lower-right
corner and in the feature guide.

<details>
<summary><strong>Full feature report: purpose, architecture, screenshots, validation, performance, and limitations</strong></summary>

## Universal N-channel Microphone Array

### What is this feature, why is it needed, and how does it work?

**Microphone Array is a native Splitwave input node that combines two or more physical microphone channels into one spatially focused mono stream named `Spatial Voice`.** It is intended for a fixed speaking position—a desk, podcast seat, recording position, lectern, or conferencing spot—where several microphones can observe the same acoustic field from known locations.

A single microphone only captures the waveform at one position. A conventional noise suppressor can reduce spectral noise after capture, but it cannot recover spatial information that was already mixed away. An array keeps every physical channel and its timing identity separate long enough to use a stronger fact: a wanted voice reaches known microphone positions with one predictable relative arrival-delay pattern, while a keyboard, fan, loudspeaker, or second talker at another position normally reaches them with a different pattern.

Splitwave implements that idea as one end-to-end path:

1. Capture each selected physical channel without mixing it first.
2. Preserve channels that share one hardware clock and synchronize each independent slave device with one ASRC per clock domain.
3. Calibrate fixed channel delay, gain, polarity, and quality.
4. Use measured microphone positions plus a fixed direction or point target to calculate target steering delays.
5. Align the target pattern and combine the channels with Delay-and-Sum, GSC, or MVDR processing.
6. Expose exactly one mono `Spatial Voice` output to the existing Splitwave graph.

The diagram below uses an intentionally simple circular example. Eight microphones are equally spaced around the target point, so the target has equal propagation distances and its calibrated copies line up. An off-axis source has unequal path lengths, so its copies remain staggered after target steering. The aligned target reinforces coherently; the off-target copies partly cancel in the weighted sum, and GSC/MVDR can deepen the attenuation.

![How microphones around a point create spatial focus](https://raw.githubusercontent.com/RedRatInHat/Splitwave/feat/microphone-array/docs/images/microphone-array-spatial-focus.png)

This is spatial filtering, not speaker recognition and not a promise that every non-target sound becomes zero. Same-delay ambiguity, reverberation, low frequencies, finite spacing, microphone mismatch, and calibration error all limit rejection. The exact, reviewable source for the polished illustration is [`microphone-array-spatial-focus.svg`](https://github.com/RedRatInHat/Splitwave/blob/feat/microphone-array/docs/images/microphone-array-spatial-focus.svg).

This experimental feature is published in the organization-owned [`RedRatInHat/Splitwave`](https://github.com/RedRatInHat/Splitwave) fork by [@RedRatInHat](https://github.com/RedRatInHat). Its feature commits use the verified organization identity `RedRatInHat <alexpacuk@redratinhat.com>` and an SSH signature.

### User-visible result

The user can:

- add one universal `Microphone Array` input node;
- select an arbitrary supported number of physical channels (`N >= 2`), rather than choosing a channel-count-specific node type;
- use channels from one multichannel interface, multiple wired USB microphones, or a mixed topology;
- see microphone count `N` separately from independent clock-domain count `K`;
- choose linear, circular, rectangular, or custom 3D geometry;
- steer to a fixed far-field direction or fixed near-field point;
- run a three-second calibration for delay, gain, polarity, and channel quality;
- choose Auto, Delay-and-Sum, GSC, or MVDR processing;
- A/B monitor best single, raw calibrated, Delay-and-Sum, and spatial output without adding graph outputs;
- inspect runtime synchronization, drift, ASRC, channel health, fallback, latency, load, and xrun diagnostics;
- connect the single mono result to any existing effect, recorder, speaker, or external virtual cable;
- create the complete chain from the `Spatial Voice — Multi-Mic` factory template.

The feature does not require a second application, a channel-count-specific set of nodes, or a new Windows virtual-audio driver.

### Product screenshots

#### Factory pipeline

The new `Spatial Voice — Multi-Mic` factory template creates one Microphone Array followed by a separate Noise Suppressor, Compressor, EQ, and Speaker. The fresh template deliberately leaves the physical inputs and output unassigned so the graph remains portable between machines.

![Spatial Voice Multi-Mic factory pipeline](https://raw.githubusercontent.com/RedRatInHat/Splitwave/feat/microphone-array/docs/images/microphone-array-pipeline.png)

#### Graph node

The compact node reports enabled microphones, clock domains, setup/calibration state, active algorithm, and strength. Configuration stays behind the node's existing task-oriented `Setup` action instead of adding another application page.

A subtle [`byRedRatInHat`](https://redratinhat.com/products/) attribution sits in the lower-right corner and opens the Red Rat in Hat products page through the system browser. It identifies this Microphone Array contribution only; it does not claim authorship of Splitwave as a whole.

![Microphone Array node](https://raw.githubusercontent.com/RedRatInHat/Splitwave/feat/microphone-array/docs/images/microphone-array-node.png)

#### Geometry, target, and calibration

The setup dialog keeps sources, geometry, target, calibration, processing, and diagnostics in one workflow. This screenshot shows a two-channel linear array, fixed-direction target, and a ready calibration.

![Microphone Array setup](https://raw.githubusercontent.com/RedRatInHat/Splitwave/feat/microphone-array/docs/images/microphone-array-setup.png)

#### A/B monitoring and diagnostics

Diagnostics do not fabricate stopped-state telemetry. When the graph runs, the same panel exposes measured ring fill, ASRC ratio, estimated ppm, correction, lock confidence, xruns, worker load, fallback state, latency, and per-member health.

![Microphone Array diagnostics](https://raw.githubusercontent.com/RedRatInHat/Splitwave/feat/microphone-array/docs/images/microphone-array-diagnostics.png)

### Signal-flow architecture

The core distinction is:

- `N`: enabled microphone channels;
- `K`: independent physical capture clock domains.

For example, one eight-channel interface is `N=8, K=1`. A four-channel interface plus two independent USB microphones is `N=6, K=3`.

![Microphone Array signal flow and clock domains](https://raw.githubusercontent.com/RedRatInHat/Splitwave/feat/microphone-array/docs/images/microphone-array-signal-flow.svg)

#### Capture ownership

- One physical device opens one native capture stream and owns one bounded SPSC ring.
- Channels from a multichannel device remain interleaved in the same stream and therefore retain their shared hardware timing.
- Capture callbacks bulk-copy complete interleaved frames and update atomics only.
- FFT, calibration, spatial DSP, allocation, file I/O, logging, IPC, and blocking locks do not run in the capture callback.
- The array capture object owns streams, runtime controls, metrics, stop state, and one worker. Producers stop before the worker joins.

#### Clock-domain synchronization

- The selected master domain sets transport cadence.
- The master channels pass through separately; they are not mixed and do not receive ASRC.
- Each slave domain has one timing model, one sample-rate-offset estimate, and one dynamic `rubato::SincFixedIn` ASRC shared by every channel in that domain.
- Ring occupancy steers the ratio so intra-device relative phase is not destroyed by per-channel resampling.
- Correction is bounded to 1,000 ppm and slewed by at most 5 ppm per update.
- The synchronization target is 3,072 frames (64 ms at 48 kHz), with 20 stable updates required before lock.
- A discontinuity, underrun, or device error resets only the affected domain and remains visible in diagnostics.

#### Calibration

- Captures three seconds through the same physical-domain plan used at runtime.
- Measures pairwise relative delay with GCC-PHAT.
- Rejects weak/outlier pair observations.
- Solves a weighted connected delay graph rather than trusting one reference pair.
- Estimates gain, polarity, per-channel quality, aggregate quality, and residual delay.
- Fingerprints source/channel topology, stream format, geometry, and target.
- Preserves stale calibration as `Needs review`; it is never silently deleted or accepted as current.
- Auto requires matching `Ready` calibration with quality at least 60 before adaptive MVDR is eligible.

#### Steering and beamforming

Geometry supports:

- fixed far-field direction from azimuth/elevation;
- fixed near-field point from `x/y/z` distances;
- linear, circular, rectangular, and custom member placement.

Steering uses 343 m/s and converts target propagation times into non-negative relative delays. A third-order fractional-delay interpolator applies fixed calibration plus target steering without allocating during processing.

- **Delay-and-Sum** aligns healthy channels and produces a normalized weighted sum. It is also the deterministic safe fallback.
- **GSC** forms blocking references from aligned channels and runs a bounded soft adaptive cancellation path. With two channels the blocking structure reduces to the familiar sum/difference form. Adaptation pauses while synchronization is unstable.
- **MVDR** uses a 512-frame Hann STFT with a 256-frame hop, updates complex spatial covariance, applies diagonal loading, solves the loaded complex system without an explicit inverse, and falls back per bin to Delay-and-Sum when the solve or normalization is unsafe.
- **Postfilter** applies a bounded soft residual gain, not a destructive hard mask.
- Algorithm changes, audition changes, and fallback transitions crossfade over 20 ms.

Frequency-domain processing reports 256 frames of algorithmic latency (about 5.3 ms at 48 kHz) to Splitwave's graph compensation. The processor remains mono; duplication happens only at the existing stereo graph boundary for compatibility.

### Runtime state, observability, and fallback

The worker publishes explicit `Starting`, `Syncing`, `Ready`, `Fallback`, `Bypassed`, and `Error` states. Reasons distinguish deliberate bypass, unlocked domain, missing healthy channel, source error, processor error, and CPU overload.

Runtime controls use atomics and can change algorithm, strength, bypass, postfilter, limiter, and audition mode without reopening devices. During a recoverable failure the output crossfades to the best healthy delayed channel or stable Delay-and-Sum rather than intentionally leaving permanent silence. MVDR additionally falls back per frequency bin.

Telemetry includes:

- requested and active algorithm;
- array state and fallback reason;
- algorithmic and synchronization latency;
- worker CPU/load estimate and deadline misses;
- calibration state, score, and residual;
- per-member level, enable/exclusion state, and health;
- per-domain ratio, estimated ppm, correction, ring fill, underflows, overflows, discontinuities, and lock progress;
- MVDR fallback-bin count.

### Graph, data model, and persistence

- `MicrophoneArrayData` contains dynamic `sources` and `members`; there are no hard-coded left/right or 2/4/8-channel node types.
- Each member references a source ID and physical channel index and carries position, enabled/quality state, weight, gain, polarity, and fixed calibration delay.
- Validation requires at least two usable members, rejects duplicate physical channel selection, validates geometry/controls, and preserves temporarily missing sources in serialization.
- The node is an **Input**, not an Effect or mixer bus, because source/channel/clock/timestamp identity must survive until spatial DSP.
- It has one public mono output. Diagnostics and A/B selection use control/telemetry APIs rather than extra graph edges.
- Rust types generate matching TypeScript bindings.
- The factory template stores stable ID `spatial_voice_multimic`, version `1`, and source-template provenance on instantiated graphs.
- Existing pipelines remain valid; no migration reinterprets an existing node.

### UI and workflow decisions

- One setup dialog, not a new application section.
- One universal node with a dynamic member list.
- `N` and `K` are shown separately so a six-channel, three-clock topology is not presented as six resamplers.
- Missing hardware remains visible and recoverable instead of being deleted from the saved graph.
- The fixed-target ambiguity warning is part of geometry setup and uses readable contrast.
- Calibration cannot run while the graph is active.
- Diagnostics state explicitly when live measurements are unavailable because the graph is stopped.
- Noise Suppressor remains a separate downstream node, so users can disable it, replace it, reorder effects, or use spatial processing alone.
- The lower-right `byRedRatInHat` attribution is an 8 px, non-dragging opener control scoped only to the contributed Microphone Array node.

### Platform boundary

| Platform | Backend implementation                                                             | Validation performed                                         | Remaining physical validation                          |
| -------- | ---------------------------------------------------------------------------------- | ------------------------------------------------------------ | ------------------------------------------------------ |
| Windows  | CPAL/WASAPI physical input, grouped multichannel streams                           | Full compile/test/build, browser UI, native executable smoke | Real shared-interface and multi-USB array measurements |
| Linux    | One configured PipeWire capture stream per physical source; monitor nodes rejected | WSL compile and pure DSP/template test coverage              | Real PipeWire session and physical array               |
| macOS    | CPAL/CoreAudio physical input; ScreenCaptureKit remains separate                   | Common code compiled through platform-gated review only      | Native compile/run and physical array                  |

No platform silently substitutes a device, rate, layout, or loopback source when the requested array cannot be built.

### Validation report

Development and primary validation were performed on Windows x64. After the attribution-only frontend commit, the final tree was rerun through formatting, frontend checks, tests, and the production frontend build. Rust audio sources were unchanged; the earlier final `cargo check` remains the Rust validation for the same DSP implementation.

#### Automated checks

| Check                                       | Result | Notes                                                                                                                                      |
| ------------------------------------------- | ------ | ------------------------------------------------------------------------------------------------------------------------------------------ |
| `bun run format`                            | pass   | Prettier + rustfmt; no unrelated semantic diff retained                                                                                    |
| `bun run check`                             | pass   | 0 errors; 13 pre-existing Svelte warnings in unrelated/shared components                                                                   |
| `bun run test`                              | pass   | 5/5 factory/template tests, 50 expectations                                                                                                |
| `bun run build`                             | pass   | SvelteKit/Vite production frontend build                                                                                                   |
| `cargo fmt --check`                         | pass   | Rust formatting clean                                                                                                                      |
| `cargo check`                               | pass   | Rust 1.97 MSVC, VS Developer environment, Ninja; 8 non-fatal dead-code warnings                                                            |
| `cargo clippy --all-targets --all-features` | pass   | Exit 0; warnings reported, no denied lint                                                                                                  |
| Windows release tests                       | pass   | 131 passed, 0 failed, 1 ignored benchmark                                                                                                  |
| Playwright feature smoke                    | pass   | Existing 1200x800 feature flow plus attribution bounds/click check; control stayed inside the node and did not overlap the strength slider |
| `bun run tauri build --no-sign`             | pass   | `.exe`, MSI, and NSIS artifacts produced                                                                                                   |
| Native `.exe` smoke                         | pass   | Window responded and title was `Splitwave`                                                                                                 |
| Type generation                             | pass   | Microphone Array bindings generated; rerun was idempotent                                                                                  |

The default system Rust on this workstation is 1.85 and is below the lockfile's current dependency MSRV. Successful Rust validation used the already installed `1.97.0-x86_64-pc-windows-msvc` toolchain. `audiopus_sys` was built with CMake/Ninja inside the Visual Studio Developer environment.

#### Linux/WSL evidence

- Linux `cargo check`: pass.
- Pure/common library tests: 129 passed, 1 ignored.
- Four full-suite failures were environment-bound PipeWire/config cases because WSL had no configured PipeWire session; filtered DSP, synchronization, calibration, graph, serialization, and template tests passed.
- This is not presented as physical Linux hardware validation.

#### Dependency and license checks

- `cargo deny` licenses: pass.
- `cargo deny` bans: pass.
- `cargo deny` sources: pass.
- Full advisory check still reports pre-existing unmaintained transitive dependencies; this feature introduces no new package for those advisories.
- The feature adds a direct `rustfft` declaration but no new runtime package beyond packages already present in the dependency graph.

#### Release artifact hashes

The unsigned Windows build produced:

| Artifact       | SHA-256                                                            |
| -------------- | ------------------------------------------------------------------ |
| executable     | `2BEEF9A1F6A27789DC12D3606ED64F1202A88B9B82E6594FF8ABDA41AFB90D77` |
| MSI            | `54EBCE91C3719F399684F9573C68D05ED12E3955F6B0C2C9FDB2DDCE18FAE20C` |
| NSIS installer | `4FFBE1AAF421AC414F0ABAE0CAE5A7EAC270BB69BC1E3CF7A621504FF16B3A06` |

### Deterministic performance evidence

Benchmark host: Intel Core i9-13900F, Windows x64, release build, 48 kHz, 256-frame worker block, 512-frame MVDR FFT, 256-frame hop.

| Enabled channels | Approx. one-core load | Block p50 | Block p95 |
| ---------------: | --------------------: | --------: | --------: |
|                2 |                 0.21% |   17.8 µs |   18.1 µs |
|                4 |                 0.39% |   33.0 µs |   33.4 µs |
|                8 |                 0.67% |   19.7 µs |   84.1 µs |
|               16 |                 1.55% |   59.1 µs |  317.8 µs |

The frequency-domain algorithmic latency is 256 frames / 5.333 ms at 48 kHz. Independent clock domains additionally target a 3,072-frame / 64 ms synchronization buffer.

One deterministic four-channel target/interferer fixture measured:

- target level change: `-0.01 dB`;
- interference reduction: `69.07 dB`;
- SNR improvement: `69.06 dB`;
- calibration residual: `0.000 samples`;
- maximum recovered-delay error: `0.247 samples`;
- calibration quality: `100`.

These are synthetic deterministic fixtures and CPU measurements, not claims about every room or physical microphone set.

### Failure and negative-path coverage

Tests cover:

- arbitrary-N steering and Delay-and-Sum for 2, 4, 8, and 16 channels;
- two-channel GSC blocking behavior;
- finite GSC/MVDR output and singular MVDR per-bin fallback;
- explicit frequency-domain latency;
- target/interferer response;
- calibration outliers and disconnected delay graphs;
- calibration fingerprints and stale-topology review state;
- serialization and missing devices;
- multiple clock-domain counts and acoustic TDOA preservation;
- hot runtime control changes;
- synchronizer discontinuity, underflow, overflow, and recovery;
- safe fallback and A/B audition selection;
- stable factory IDs/version/provenance and 4-channel plus mixed 4+2 scenarios.

### Explicit limitations and non-goals

- **No moving-target tracking.** Direction and point targets are fixed.
- **No speaker identification.** Spatial delay patterns are selected, not people.
- **No guaranteed total cancellation.** Same-TDOA sources, reflections, low frequencies, compact spacing, mismatch, and calibration error reduce rejection.
- **No Bluetooth array input.** Codec buffering and variable transport latency are incompatible with stable relative timing.
- **Independent USB devices remain experimental.** A shared-clock multichannel interface is the recommended configuration.
- **No bundled denoiser.** Noise Suppressor remains a separate graph node.
- **No new Windows virtual-audio driver.** Existing outputs or an external virtual cable remain the boundary.
- **No physical hardware claim yet.** The manual acceptance guide is included, but real multichannel/USB microphone-array measurements have not been completed for this branch.
- **macOS has not been run.** This is named explicitly rather than implied by common-code coverage.

The same-TDOA boundary is illustrated separately:

![Same-TDOA ambiguity](https://raw.githubusercontent.com/RedRatInHat/Splitwave/feat/microphone-array/docs/images/microphone-array-ambiguity.svg)

### Research and implementation provenance

The implementation was written independently in Rust for Splitwave. No code, tests, configuration, or assets were copied or translated from the reviewed projects.

| Source                                                                          | License/status                                | Used for                                              | Incorporated code             |
| ------------------------------------------------------------------------------- | --------------------------------------------- | ----------------------------------------------------- | ----------------------------- |
| [GCC-PHAT, Knapp and Carter (1976)](https://doi.org/10.1109/TASSP.1976.1162830) | paper                                         | relative-delay terminology and expectations           | none                          |
| [GSC, Griffiths and Jim (1982)](https://doi.org/10.1109/TAP.1982.1142739)       | paper                                         | generalized sidelobe-canceller structure              | none                          |
| [MVDR/Capon (1969)](https://doi.org/10.1109/PROC.1969.7278)                     | paper                                         | minimum-variance constrained beamforming              | none                          |
| [pyroomacoustics](https://github.com/LCAV/pyroomacoustics)                      | MIT                                           | geometry and reference scenarios                      | none                          |
| [ODAS](https://github.com/introlab/odas)                                        | MIT                                           | N-channel mapping and system boundaries               | none                          |
| [paderwasn](https://github.com/fgnt/paderwasn)                                  | MIT                                           | independent-clock SRO/STO concepts                    | none                          |
| [openMHA](https://github.com/HoerTech-gGmbH/openMHA)                            | AGPL-3.0                                      | public real-time beamforming/calibration descriptions | none; excluded as code source |
| [BeamformIt](https://github.com/xanguera/BeamformIt)                            | no repository license identified during audit | public variable-channel feature description           | none; excluded as code source |
| [rubato](https://github.com/HEnquist/rubato)                                    | MIT OR Apache-2.0                             | existing resampler API                                | existing dependency only      |

`NOTICE` and the architecture guide record this boundary explicitly.

### Documentation included

- [`docs/microphone-array.md`](https://github.com/RedRatInHat/Splitwave/blob/feat/microphone-array/docs/microphone-array.md): user guide, feature purpose, illustrated physics, setup, screenshots, algorithms, limitations, and troubleshooting.
- [`docs/microphone-array-architecture.md`](https://github.com/RedRatInHat/Splitwave/blob/feat/microphone-array/docs/microphone-array-architecture.md): implementation architecture, RT boundary, validation, and provenance.
- [`docs/microphone-array-hardware-test.md`](https://github.com/RedRatInHat/Splitwave/blob/feat/microphone-array/docs/microphone-array-hardware-test.md): repeatable shared-clock, independent-USB, mixed-topology, ambiguity, and recovery acceptance procedure.
- README, DEVELOPMENT, CHANGELOG, and NOTICE updates.

The primary concept image was first authored as an exact SVG, rendered and visually checked, then restyled with the built-in image generator using the SVG render as a topology-locked reference. The final raster was reviewed for exactly eight microphones, correct nearest/farthest paths, eight aligned target rows, eight staggered off-axis rows, and the non-guaranteed-rejection caveat. The exact SVG remains in the repository for scientific review.

### Implementation map

| Area                                                | Primary files                                                                             |
| --------------------------------------------------- | ----------------------------------------------------------------------------------------- |
| Data model and validation                           | `src-tauri/src/audio/graph.rs`                                                            |
| Shared runtime, synchronizer, worker, steering, GSC | `src-tauri/src/audio/microphone_array.rs`                                                 |
| Offline calibration                                 | `src-tauri/src/audio/microphone_array/calibration.rs`                                     |
| Frequency-domain MVDR                               | `src-tauri/src/audio/microphone_array/mvdr.rs`                                            |
| Platform input construction                         | `src-tauri/src/audio/pipeline/input/{windows,macos,linux}.rs`                             |
| CPAL grouped capture                                | `src-tauri/src/audio/streams/cpal_stream.rs`                                              |
| PipeWire physical-source support                    | `src-tauri/src/audio/{capture/linux,device/linux,pw_enum}.rs`                             |
| Commands and runtime update/metrics                 | `src-tauri/src/commands.rs`, `src-tauri/src/audio/pipeline/mod.rs`                        |
| Node and setup UI                                   | `src/lib/modules/flow/ui/input/microphone_array.svelte`, `_microphone_array_setup.svelte` |
| Factory pipeline                                    | `src/lib/modules/template/catalog.ts`                                                     |
| Generated bindings/defaults                         | `src/lib/modules/pipeline/generated/`, `defaults.ts`, `microphone_array.ts`               |
| Tests                                               | Rust module tests plus `src/lib/modules/template/catalog.test.ts`                         |
| User and implementation documentation               | `docs/microphone-array*.md`, `docs/images/`                                               |

### Implementation checklist

- [x] One universal N-channel input node; no per-count node variants.
- [x] Shared-clock and independent-clock topologies represented explicitly.
- [x] One ASRC per slave clock domain, not per device channel.
- [x] Calibration, steering, Delay-and-Sum, GSC, and MVDR implemented.
- [x] One mono graph output; diagnostics kept out of graph routing.
- [x] Runtime metrics, recovery, and deterministic fallbacks exposed.
- [x] Factory pipeline, persistence, generated bindings, and template tests included.
- [x] Formatting, frontend checks/tests/build, Rust check/test/clippy, package build, and native smoke completed on Windows.
- [x] Linux common/pure coverage performed and environment failures named.
- [x] Platform/hardware gaps named explicitly.
- [x] User guide, architecture/provenance guide, hardware test guide, exact SVGs, imagegen rendering, and actual UI screenshots included.

</details>

## Stack

- **Frontend:** Svelte, Tauri, @xyflow/svelte
- **Engine:** Rust -- `rtrb` (SPSC ring buffers), `rubato` (resampling),
  `hound` (WAV), `flac-codec`, `opus`, `mp3lame-encoder`, `ebur128`
- **macOS:** `cpal` device I/O; custom Swift static library for ScreenCaptureKit,
  compiled by `build.rs` via `swiftc`; CoreAudio HAL FFI for device enumeration;
  libASPL-based AudioServerPlugin for the virtual device driver
- **Linux:** `pipewire` for device I/O, system/app capture, and virtual
  null-sinks; `libpulse-binding` for device-volume control (talks to
  `pipewire-pulse`); `freedesktop-desktop-entry` / `freedesktop-icons` for app
  icons
- **Windows:** `cpal` (WASAPI) device I/O; the `windows` crate for WASAPI
  loopback + Process Loopback capture, `IAudioEndpointVolume`, audio-session
  enumeration, and exe icon extraction (`png` for encoding)

## Development

See [DEVELOPMENT.md](DEVELOPMENT.md) for prerequisites, setup, useful commands,
and project layout.

## License

Splitwave is licensed under [MIT](LICENSE).  
Third-party component notices (LGPL, MPL-2.0, etc.) are in [NOTICE](NOTICE).

## Support

If you find this app useful, consider supporting it:

- Tether USDT (TRC20): `TLhTvnn8CtVuQZruLXmRurGhR9GWd7DrWZ`
- TON: (TON) `UQCpokpaZfwmVTjKDj0LrAbEPO-65c81-MiuBQOa7lTXbMGR`
- Bitcoin (BTC): `bc1q6tusr5rht7dgmw8gqzkx7rwdg4q8932lwn2rsy`
