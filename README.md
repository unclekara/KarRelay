# KarRelay

Low-latency SRT fan-out relay with frame-accurate timecode injection.
One binary, no dependencies to install, a web interface for everything.

[**Download 0.7.0**](https://github.com/unclekara/KarRelay/releases/latest)
· Windows and Linux, 64-bit

> This repository carries the description and the releases. **KarRelay
> is not open source and its source is not published** — see
> [LICENSE](LICENSE). The binaries are free to download and run.

[SRT](https://github.com/Haivision/srt) (Secure Reliable Transport) is
an open-source video transport protocol originally created and
open-sourced by [Haivision](https://www.haivision.com/). SRT is a
trademark of Haivision Systems Inc. KarRelay is not affiliated with,
endorsed by, or sponsored by Haivision or the SRT Alliance.

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
                        │     ──SRT──▶ KarRelay at another site
                        └── web UI on :8484
```

## Features

**Transport**

- SRT **caller** and **listener** input, per source — the relay dials
  out to your encoder, or waits for it to dial in.
- **Fan-out** with per-subscriber buffering. A slow viewer is dropped
  frames, never backpressure on the source.
- **Stream-id whitelist** per source, in and out.
- **Push replication** to a second KarRelay: one copy across the
  network instead of one per viewer, fanned out again at the far end,
  and a standby path if this relay becomes unreachable.
- **SRT link encryption** (passphrase, AES-128/192/256), per direction
  on a source and for a channel's subscribers. A peer with the wrong
  passphrase, or none, is rejected rather than quietly served in the
  clear.
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

**A second site**

Viewers in more than one place do not each need a link back to the
encoder. KarRelay pushes a single copy to a second KarRelay, which
fans it out locally:

```
              ┌──SRT──▶ viewer
  encoder ──▶ │──SRT──▶ viewer        site A
   KarRelay   │
              └──SRT──▶ KarRelay ──┬──SRT──▶ viewer
                         site B    └──SRT──▶ viewer
```

Targets hang off a source or a switchable output — a switchable one
sends the programme as cut, not one of its candidates. They are added
and removed while everything is running, reconnect by themselves if
the far relay restarts, and can carry their own passphrase. A target
pointing back at this relay's own port is refused, because that feeds
the stream into itself and nothing about the symptoms would say so.

Free on every licence.

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
- **Listen address** — restrict the interface to one network, or to the
  relay machine itself, and the port disappears from everywhere else.
- **TLS** for the interface, one click, with a certificate generated for
  you. HTTPS and plain HTTP share the port, so KarPlayer keeps syncing
  its clock without needing the certificate.
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
| SRT link encryption, stream-id whitelist, statistics, H.265 | ✓ | ✓ |
| Encrypted web interface (TLS) | ✓ | ✓ |
| Push replication to another KarRelay | ✓ | ✓ |

Activation is checked once and cached locally, signed, so the relay
keeps running with no network. A seven-day trial is available per
machine from the licence panel in the UI.

## Not built yet

Nothing in the table above is outstanding — push replication was the
last of it, and arrived in 0.7.

Custom metadata injection, webhooks on events and interface branding
have been mentioned in earlier plans. They are not built and are not
planned; if they ever appear they will not change what is free and what
is paid.

## Security

Three things, none of them behind a paid tier, and worth the minute
each takes:

1. **Change the password.** A fresh install accepts `admin` / `admin`
   so you can get in; until you change it on the About tab, so can
   anyone who can reach the port. The relay says so in its log on every
   start.
2. **Turn on TLS**, also on the About tab. Without it the password
   crosses the network in the clear. The certificate is generated for
   you and your browser will warn once because nobody signed it —
   compare the fingerprint it shows with the one in the UI, then accept
   it.
3. **Narrow the listen address** if the interface is only ever opened
   on the relay machine. That removes the port from every other network
   rather than defending it.

Separately, set a passphrase on your sources and channels if the path
to your viewers is not one you control.

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
