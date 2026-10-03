# KarRelay

Low-latency SRT fan-out relay with frame-accurate timecode injection.
One binary, no dependencies to install, a web interface for everything.

[**Download 0.5.0**](https://github.com/unclekara/KarRelay/releases/latest)
· Windows and Linux, 64-bit

> This repository carries the description and the releases. **KarRelay
> is not open source and its source is not published** — see
> [LICENSE](LICENSE). The binaries are free to download and run.

[SRT](https://github.com/Haivision/srt) (Secure Reliable Transport) is
an open-source video transport protocol originally created and
open-sourced by [Haivision](https://www.haivision.com/). KarRelay is not
affiliated with, endorsed by, or sponsored by Haivision or the SRT
Alliance.

---

## What it does

You have one SRT stream coming out of an encoder — vMix, OBS, ffmpeg, a
hardware box — and you need several people to watch it. Most encoders
will not send to more than one or two destinations, and asking them to
is the fastest way to ruin the feed.

KarRelay sits in between. It takes the stream once and re-publishes it
to as many subscribers as you like, each with its own buffering, so a
viewer on a bad connection cannot slow down anyone else or stall the
encoder.

```
  encoder ──SRT──▶  KarRelay  ──SRT──▶ viewer
                        │     ──SRT──▶ viewer
                        │     ──SRT──▶ recorder
                        └── web UI on :8484
```

## Features

**Transport**

- SRT **caller** and **listener** input, per source — the relay dials
  out to your encoder, or waits for it to dial in.
- **Fan-out** with per-subscriber buffering. A slow viewer is dropped
  frames, never backpressure on the source.
- **Stream-id whitelist** per source, in and out.
- **SRT link encryption** (passphrase).
- H.264 and H.265, passthrough — nothing is re-encoded, so there is no
  added latency and no quality loss.
- Live stats over a WebSocket: RTT, bitrate, loss, drops, viewer count,
  uptime, and the codec read out of the transport stream itself.

**Switching between sources**

One output can sit in front of several candidate sources and switch
between them while people are watching. The cut happens at an IDR
boundary, and the relay rewrites continuity counters and re-anchors the
PCR, PTS and DTS so a receiver sees one continuous timeline — even when
the two encoders run on clocks that were never synchronised.

Audio is spliced separately from video, because audio has no equivalent
of an IDR to cut at. The outgoing source keeps providing audio for up to
100 ms while the new source's video is already on air, and the audio
switch commits on the first clean frame boundary. No instant of audio is
ever sent twice, and none is skipped.

Measured over ten switches on two outputs of one vMix instance: no
gaps, no backward steps in the timeline, no duplicated audio, nothing
audible. See the [0.5.0 notes](CHANGELOG.md) for how that was arrived
at — it took a while.

> All candidate sources in one switchable output must share a codec:
> H.264 with H.264, or H.265 with H.265. Hardware decoders on the
> receiving side are set up for one codec at the first frame and cannot
> be changed mid-stream without dropping the session.

**Timecode for frame-accurate sync**

KarRelay can inject a small SEI message carrying UTC microseconds into
every I-frame. [KarPlayer](https://github.com/unclekara/KarPlayer), the
companion Android receiver, uses it to hold a constant, known lag —
so several screens showing the same feed show the same frame.

**Operation**

- **System tray** app on Windows and desktop Linux: status icon, open
  the UI, quit. No console window.
- **Headless** build for servers and systemd.
- **Web interface** for sources, viewers, settings and licence state.
- **Operator credentials** on the interface and the API. A fresh install
  accepts `admin` / `admin`; change both on the About tab.
- **Rejection reasons** surfaced both to the operator and to the
  subscriber that was turned away, so "it won't connect" has an answer.

## Quick start

1. [Download](https://github.com/unclekara/KarRelay/releases/latest) and
   unpack. There is nothing to install and no DLLs to place.
2. Run `karrelay-tray.exe` (Windows desktop) or `./karrelay` (Linux).
3. Open <http://localhost:8484/> and sign in with `admin` / `admin`.
4. **Change the password** on the About tab.
5. Sources → Add. Pick listener mode, give it an input port and an
   output port.

Point your encoder at the input port, point your viewers at the output
port:

```
srt://<relay-host>:<output-port>?mode=caller
```

### Try it with ffmpeg

```bash
ffmpeg -re -f lavfi -i testsrc=size=1280x720:rate=30 \
       -f lavfi -i sine=frequency=440 \
       -c:v libx264 -preset ultrafast -tune zerolatency -c:a aac \
       -f mpegts "srt://<relay-host>:9000?mode=caller&latency=120"
```

With a source on input port 9000 and output port 9001, any SRT receiver
can then watch `srt://<relay-host>:9001?mode=caller`.

## Free and paid

Everything above works without a licence key, within these limits:

| | Free | Paid |
|---|---|---|
| Sources at once | 1 | unlimited |
| Viewers per source | 3 | unlimited |
| Switching between sources | — | ✓ |
| Timecode injection | — | ✓ |
| API access from outside the local network | — | ✓ |
| SRT encryption, stream-id whitelist, stats, H.265 | ✓ | ✓ |
| Encrypted web interface | ✓ | ✓ |

Activation is checked once and cached locally, signed, so the relay
keeps running with no network. A seven-day trial is available per
machine from the licence panel in the UI.

Transport encryption for the web interface will never be behind a paid
tier.

## Security

The web interface speaks plain HTTP today. The operator password stops
unauthenticated changes and anything routed in from outside, but it is
not hidden from someone capturing traffic on the same network. **Keep
the interface on a network you trust.** Encrypted transport is the next
thing on the list.

If you find something security-relevant, please open an issue saying
only that you have, and I will follow up for the detail.

## Platforms

| | Status |
|---|---|
| Windows 10/11, 64-bit | released, tray and headless |
| Linux, x86-64 | released, headless |
| Linux ARM64, macOS | build on request |

## Issues and requests

Bug reports and feature requests are welcome in
[Issues](https://github.com/unclekara/KarRelay/issues), source or no
source. Please say which version, which platform, what the encoder is,
and what the log said.

## Licence

KarRelay is proprietary — see [LICENSE](LICENSE).

The binary includes open-source components under their own licences,
reproduced in full in
[THIRD-PARTY-NOTICES.md](THIRD-PARTY-NOTICES.md).

One of them is worth a word: the SRT implementation is
[datarhei/gosrt](https://github.com/datarhei/gosrt), pure Go, which is
why KarRelay is a single binary with no `libsrt` to ship alongside.
KarRelay currently builds against a fork carrying one fix, which has
been [proposed upstream](https://github.com/datarhei/gosrt/pull/161).

## Author

Alexander Karabatov · <hellokarabatov@gmail.com>
