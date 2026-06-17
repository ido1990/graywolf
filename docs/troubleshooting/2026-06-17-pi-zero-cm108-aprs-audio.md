# APRS TX silent and RX dead on Raspberry Pi Zero W (32-bit ARMv6) with C-Media CM108 USB audio

Field report from a debugging session (2026-06-17). Five independent bugs
stacked on top of each other; each had to be fixed before the next became
visible. Branch: `fix/aprs-tx-silent-f32-sample-rate`.

## Summary

On a Raspberry Pi Zero W with a CM108 USB sound dongle, graywolf keyed PTT
but transmitted silence most of the time, and RX never decoded. End state
after the fixes: TX transmits cleanly ~90%+ of the time, RX path runs, web
UI responsive. The remaining occasional glitch is residual marginal-USB
behavior on the Pi Zero (helped by a powered hub).

## Hardware / environment

- Board: Raspberry Pi Zero W -- 32-bit ARMv6 (BCM2835), single core, no NEON.
- OS: Raspberry Pi OS, kernel 6.12 v6, t64 userland (libasound2t64, 64-bit time_t).
- Audio/PTT: C-Media CM108 "USB PnP Sound Device" (idVendor=0d8c idProduct=013c),
  full-speed USB, used for both audio and CM108 HID PTT.
- Mode: APRS 1200 AFSK, digipeater + iGate.
- Build: graywolf-modem v0.14.0; also surfaced a mixed-build issue
  (Go graywolf v0.13.16 + modem v0.14.0).
- Capture device advertises native mono i16 @ 44100 and 48000 only.
  arecord/aplay stream it cleanly (incl. under full graywolf load); cpal did not.

## Root causes and culprit files

### 1. TX silent: F32 stream opened at 44.1 kHz
graywolf defaults to 44100 Hz. On the CM108 plughw: PCM only F32 is advertised
at 44.1 kHz (native I16 only at 48 kHz). Opening an F32 output stream makes
ALSA poll() return POLLERR every period -> crash/rebuild loop (up to 5 s
backoff). TX audio drains in ~1.4 s, so most transmits timed out mid-rebuild
-> silence. (TX twin of RX issue #227 / invariant 33.) The format picker chose
the format at the requested rate, but the rate decision was format-blind.
- graywolf-modem/src/audio/soundcard.rs (choose_stream_rate / pick_*_sample_format;
  spawn_output did no rate reconciliation)
- graywolf-modem/src/modem/mod.rs (synthesis + stream both used the raw rate)
- graywolf-modem/src/types.rs (DEFAULT_SAMPLES_PER_SEC = 44100)
Fix: choose_stream_rate_for_format prefers a native-I16 rate (44.1 -> 48 kHz);
the resolved rate drives both synthesis and the stream.

### 2. Crash-loop: 32-bit i32 overflow on ALSA hardware timestamp (t64)
After #1: "get_htstamp ... was earlier than get_trigger_htstamp 0.0". Two
layers: (a) t64 makes struct timespec 16 bytes on 32-bit; stock alsa-rs read
into 8 bytes -> stack smash (#231); the vendored fork fixed that but returned
the real non-zero htstamp. (b) cpal then takes its hardware-timestamp path
(only uses the software fallback when get_htstamp() == (0,0)) and computes
ts.tv_sec * 1_000_000_000 in i32, which overflows for uptime > ~2 s on 32-bit.
- cpal-0.17.3/src/host/alsa/mod.rs (timespec_to_nanos i32 multiply; (0,0) probe)
- vendored alsa-rs returning a real value defeated the (0,0) probe
Fix: vendored alsa-rs decode_htstamp returns (0,0) (graywolf discards callback
timestamps), forcing cpal's overflow-free software fallback.
- third_party/alsa-rs/src/pcm.rs, Cargo.toml ([patch.crates-io] alsa)

### 3. RX overrun: capture buffer too small for a loaded single core
arecord (default ~500 ms buffer) clean under load; graywolf overran. cpal
Default opens a small period; cpal double-buffers Fixed(x) as period=x,
buffer=2x. A periodic stall on the single ARMv6 core overran the small buffer.
- graywolf-modem/src/audio/soundcard.rs (spawn used BufferSize::Default)
Fix: BufferSize::Fixed(stream_rate/4) (~250 ms period -> ~500 ms buffer),
fallback to Default if rejected.

### 4. POLLERR handling: rebuilt on every error; cpal can't recover POLLERR in place
Decisive clue: arecord captured the same device under full graywolf load
cleanly for 15 s while graywolf thrashed. cpal does not die on POLLERR (it
reports and keeps polling) but it also does not recover it in place (only the
EPIPE XRUN path calls prepare()/try_recover). graywolf's wrapper rebuilt on
every error, so a transient POLLERR tore down a good stream and the rebuild
churn provoked POLLERR storms. (An interim over-correction left a single
POLLERR wedging the stream permanently.)
- graywolf-modem/src/audio/soundcard.rs (err_fn in spawn/spawn_output)
- cpal-0.17.3/src/host/alsa/mod.rs (poll path returns POLLERR as fatal)
Fix: recover BufferUnderrun in place (no rebuild); reopen the stream only on
BackendSpecific (POLLERR) / DeviceNotAvailable / StreamInvalidated.

### 5. RX runs but pegs the core: triple AFSK demod on ARMv6
Once RX was stable it pinned the single core at ~100% (top: modem 99.9%, Go 0%,
web UI unreachable). Default demod_ensemble = "triple" = 3 profiles x 9 slicers
= 27 slicers @ 48 kHz -- fine on x86, far too heavy for 1 GHz ARMv6 no-NEON.
- graywolf-modem/src/modem/mod.rs (create_demod defaults to RECOMMENDED_3DEMOD;
  per-channel demod_ensemble needs the v0.14 Go side)
Fix: GRAYWOLF_DEMOD_ENSEMBLE=single|dual|triple env override read in create_demod.

### 6. (Follow-on) Output stream POLLERR / drain-timeouts while idle
A persistent USB playback stream held open between transmissions clock-drifts
into an XRUN that arrives as POLLERR (cpal can't recover in place, see #4) ->
rebuild loop every few seconds; a rebuild colliding with a TX truncated it
("TransmitFrame: drain timeout").
- graywolf-modem/src/modem/tx_worker.rs (worker_loop kept the sink open forever)
Fix: idle-close the output sink after 2 s with no TX; reopen on the next TX
reusing the cached cpal Device (so reopen doesn't re-enumerate during capture).

## Reproduce
1. 32-bit ARMv6 t64 host + CM108 dongle; AFSK channel using CM108 audio + PTT, 44100 Hz.
2. Key a beacon: PTT keys, no RF audio; logs flood "alsa::poll() returned POLLERR"
   (+ on 32-bit "get_htstamp ... earlier than get_trigger_htstamp").
3. Contrast: arecord -D plughw:CARD=Device,DEV=0 -f S16_LE -r 48000 -c 1 -d 15 /tmp/t.wav
   is clean even under load -> bug is in graywolf/cpal, not the hardware.

## Affected files (summary)
- graywolf-modem/src/audio/soundcard.rs -- rate/format selection, capture buffer, cpal error recovery
- graywolf-modem/src/modem/mod.rs -- per-device resolved TX rate, demod ensemble + env override
- graywolf-modem/src/modem/tx_worker.rs -- output sink idle-close + device reuse
- graywolf-modem/src/types.rs -- default sample rate
- third_party/alsa-rs/src/pcm.rs + Cargo.toml -- vendored alsa-rs htstamp zeroing
- upstream: cpal-0.17.3 (i32 htstamp overflow on 32-bit; POLLERR not recovered in place)

## Notes
- arecord/aplay work because they recover from XRUN regardless of how it is
  signaled and use a large default buffer; cpal is stricter and (on 32-bit)
  miscomputes the timestamp.
- Mixed build (old Go graywolf + newer modem) hides the per-channel demod
  setting and is unsupported; keep both at the same version.
- Residual during-TX POLLERR on the Pi Zero is marginal USB power/signal; a
  powered hub helps. A complete software fix would require patching cpal to
  recover the output POLLERR in place (snd_pcm_recover) instead of rebuilding.

## See also
- Invariants 32, 33, 47, 48, 49, 50 in docs/wiki/invariants.md
- docs/plans/2026-06-11-armhf-t64-alsa-htstamp-fix.md (section 11 follow-up)
