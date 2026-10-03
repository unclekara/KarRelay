# Changelog

User-facing changes to KarRelay. The source is not published, so this
describes behaviour rather than code.

## [0.5.0] — 2026-10-04

First public release.

### Switching between sources is now seamless, audio included

This is the whole story of the release. Switching a live output from one
source to another was audible and visible: the two sources seemed to mix
for a second or two, audio drifted out of sync with the picture, and
there was a freeze on the cut. It turned out to be several separate
faults sitting on top of each other, which is why it took measuring
individual packets rather than reasoning about the design.

- **The cut was aligned to the wrong clock.** The relay worked out how
  far to move the incoming stream from the difference between the two
  sources' PCR — the transport clock — and then applied that same number
  to the presentation timestamps, which answer a different question. The
  two sources' relationship between those clocks is not the same, so the
  programme was moved by an amount that had nothing to do with the
  picture. There are now two offsets derived from two clocks: the
  transport clock continues from the last value sent, and presentation
  continues one frame after the last frame sent.

- **Audio now moves with its own source.** It is shifted by the offset
  that source's video got, so an encoder's own lip sync survives the
  switch intact.

- **The first switch after a start no longer freezes.** It used to step
  the whole programme more than a second backwards — one visible freeze,
  and one discontinuity error in the receiver's log — while every later
  switch was clean. The source already on air never had its video
  identified, so the first switch had nothing to align to and fell back
  to the transport clock. It now has a reference from the first packet
  sent.

- **No moment of audio is ever sent twice.** The outgoing source's last
  frame is now cut on where it *ends* rather than where it starts, and
  any leading frame of the incoming source covering a moment already
  sent is dropped — including across the boundary between processing
  ticks, which is where every overlap actually turned out to be.

- Measured over ten switches between two SRT outputs of one vMix
  instance: no audio gaps, no backward steps, no duplicated audio, and
  video and audio offsets agreeing exactly. Before this release, every
  switch carried a gap of between 2.2 and 10.2 seconds. Confirmed by
  ear and eye on an Android receiver as well as in the traces.

- Transport conformance at the cut was fixed alongside: continuity
  counters per stream, frame boundaries honoured per stream, no
  duplicated frames, and the transport clock no longer corrected twice.

### Receivers stop logging a false error several times a second

Any libsrt-based receiver — KarPlayer, ffmpeg, vMix — logged
`EPE: incoming UMSG: 6 INVALID SIZE: 0` continuously while connected to
KarRelay. Nothing was lost and the connection was fine, but it filled
the log, and a log full of errors that are not errors is a log you
cannot find a real error in.

The cause was in the SRT library: an acknowledgement packet was sent
with an empty body, which the RFC allows and libsrt rejects. Measured at
405 of those lines in 8 seconds; now zero. Fixed and
[proposed upstream](https://github.com/datarhei/gosrt/pull/161).

### Added: operator credentials on the web interface and API

The relay listens on every network interface and previously asked
nothing of whoever connected, so anyone who could reach the port could
switch sources, disconnect viewers and move the service port.

- A fresh install accepts `admin` / `admin`, so the interface is
  reachable immediately with no setup step. Change both on the **About**
  tab. Until you do, the relay says so in its log on every start and the
  interface shows a banner.
- Changing the credentials requires the current password, even when you
  are already signed in, and ends every open session.
- Passwords are stored hashed, never in the clear.
- Browsers get a session cookie; scripts can use HTTP Basic against the
  same credentials.
- Clearing the activation now requires credentials. Previously it could
  be done by anyone who could reach the port, and it was the one call
  that could drop a running relay to the free tier mid-production.
- Two read-only endpoints stay open, because they are how a player keeps
  its clock and finds out why it was turned away, and a viewer should not
  need the operator password to watch.

**The credentials are not confidential in transit** — the interface
speaks plain HTTP. This stops unauthenticated changes and anything
routed in from outside; it does not hide the password from someone
capturing traffic on the same network. Keep the interface on a network
you trust. Encrypted transport is planned and will never be a paid
feature.

---

## Earlier versions

0.1 to 0.4 were internal. In summary, and in order: SRT fan-out with
caller and listener input and per-subscriber buffering; the web
interface and live statistics; stream-id whitelisting; the system-tray
application; SEI timecode injection for frame-accurate sync with
KarPlayer; H.265 support; connection-rejection reasons surfaced to both
the operator and the rejected subscriber; a configurable service port;
switching between sources with an IDR-aligned cut; and the delayed audio
handover that 0.5.0 then went on to get right.
