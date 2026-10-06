# Changelog

User-facing changes to KarRelay. The source is not published, so this
describes behaviour rather than code.

> **Releases before 0.7.5 have been withdrawn.** Their entries stay
> below, because the history of what changed is worth keeping, but the
> binaries are no longer downloadable: 0.6.0 and earlier validate a
> cached licence without checking the machine it was issued for, 0.7.0
> reports a version that is not its own, 0.7.1 can run a stream
> unencrypted while telling you it is encrypted (see 0.7.2), 0.7.2
> cannot show or change a caller source's stream-id once it is set
> (see 0.7.3), and 0.7.3 can tell you a change was saved when it was
> not (see 0.7.4). 0.7.4 itself is sound — it is withdrawn only
> because 0.7.5 replaces it. If you are running any of them, take
> 0.7.5.

## [0.7.5] — 2026-10-06

A contact address on the About card, and nothing else. Everything in
0.7.4 below applies unchanged.

## [0.7.4] — 2026-10-06

### A change that could not be saved is no longer reported as saved

Most of the settings pages applied your change to the running relay
and then wrote it to the configuration file — and if that write
failed, said nothing. Rename, the subscriber whitelist and the SEI
toggle answered "done" with the relay running one thing and the file
holding another. Delete was worse: it stopped the source, failed to
write the file, answered "done", and the source came back the next
time the relay started.

These now store the change first and tell you if that fails. Nothing
is applied that was not saved.

The same went for the file itself. It was written in place, which
means it was emptied and then filled: a save interrupted part-way — a
full disk, a machine switched off at the wrong moment — left a
configuration the relay could not read at all. It is now written
alongside and swapped in, so the file on disk is always one complete
configuration, the old one or the new one.

On Windows a save can now fail with a sharing error if another program
is holding the file open — a backup, an editor, a scanner. It retries
for a fifth of a second first, and if it still cannot, it tells you
and changes nothing. That is the trade for never leaving the file
half-written.

### Two smaller ones

Editing a source or channel that is saved but not running — one with
autostart off, after a restart — reported a failure. The edit had been
saved; there was simply no live stream to apply it to.

The stream-id chip could be left showing a value the relay had
refused, or have a correction you were typing wiped out by the refusal
of the previous attempt.

### The About tab is now Settings

Which is what it holds: your credentials, the service port, the listen
address and TLS. The product information moves up beside the
credentials, and the card that was itself called Settings is now
**Web interface**.

Unpack over the old one as usual.

## [0.7.3] — 2026-10-06

### The caller stream-id, visible and changeable

0.7.2 added the field and left it write-once. The Add dialog was the
only place that sent it, nothing showed it afterwards, and there is no
edit dialog for a source — so once a caller source existed there was
no way to see what stream-id it presents, and a typo meant deleting
the source and building it again.

The card now carries it as a **sid** chip. Click it to change it:

  * the source re-dials with the new value immediately, and
    **nobody watching the output is disturbed** — the output listener
    and every subscriber on it stay up, only the input leg reconnects;

  * if the source is not connected at all, which is the usual reason
    you are changing this, the correction is tried at once rather than
    waiting out the retry interval. That interval grows to five
    seconds while a source is being refused, and the fix used to sit
    behind it;

  * clearing the field is a setting, not a cancel: an empty stream-id
    means the source dials without one;

  * a listener-mode source has no chip, because a listener receives a
    stream-id rather than presenting one.

Nothing else changed. Upgrading from 0.7.2 is worth it only if you use
caller sources with a stream-id; unpack over the old one as usual.

## [0.7.2] — 2026-10-06

What the relay runs is now what you saved. Every path that writes a
source or a channel was checked against that, and four of them were
not keeping it.

### A replaced source could lose its encryption without saying so

This is the one worth upgrading for.

The interface never shows you a stream passphrase — it cannot, it is
never sent to the browser — so saving a source you did not re-type one
for means "keep the one you had". That worked for the stored
configuration and not for the running stream: if the source was saved
with autostart off and the relay had been restarted since, saving it
again started it **without** the passphrase, while the file and the
interface both went on reporting encryption.

A subscriber with no passphrase was then let in, and one with the
right passphrase was refused. Nothing in the log connected the two.

If you have ever saved an encrypted source without re-entering its
passphrase, check it: on 0.7.2, a client with no passphrase is refused
as it should be.

### A rejected change no longer half-happens

Saving a source or channel the relay then refused to store used to
leave it running anyway — bound to its port, pushing replicas, and
absent from the configuration file. The interface said 400 and the
relay carried on serving it until the next restart.

Three more of the same shape, all now fixed:

  * a rejected save could **delete the source it was replacing**,
    from the file as well as from memory — you asked for a change,
    got an error, and lost what was there;

  * an unusable replication target started the source before it was
    rejected, leaving it retrying against a target that was never
    saved;

  * two changes made at the same time — a rename while a replication
    target was being edited, say — could each overwrite the other's,
    with one of them answering success and quietly storing a stale
    copy of the whole entry.

Switching a channel is unaffected by any of this and never waits for a
save: the cut is still the one thing that cannot be made to queue.

### Caller sources can present a stream-id

A source in caller mode now carries a stream-id of its own, which is
what a sender filtering on one expects to see. Until now that field
only existed for the subscriber side, so a caller could not reach such
a sender at all. Up to 512 bytes, as SRT allows; over that it is
refused when you save it rather than failing to connect for reasons
nothing explains.

### Upgrading

Unpack over the old one. Config, licence and certificate are untouched,
as before. Nothing in the API changed except the added field.

## [0.7.1] — 2026-10-05

A cosmetic release, but the cosmetics were lying about which build you
were running.

### The version shown was not the version running

The web interface displayed **v1.4.2**. Windows file properties said
**0.3.0.0**. The relay itself reported 0.7.0 at startup. All three
were true at once, and had been for several releases.

The interface's number was written into the page by hand and never
touched again — and it could not have been anything else, because the
version was never given to the web server in the first place. The page
had nothing to read. It now comes from the relay, so it cannot
disagree with it. If the page cannot reach the API it shows an
ellipsis rather than a number, which is the honest answer to "which
build is this".

The file properties came from a resource compiled in May and carried
into every build since.

Both are fixed, and a test now compares every place the version is
written down and fails when they drift apart. That is the part worth
having: the numbers were wrong because nothing checked them.

If you are on 0.7.0 and it reads **v1.4.2** in the corner, that is
this bug and nothing else — the relay is 0.7.0 and works.

### Operator credentials fits on the screen

Four stacked full-width fields pushed **Save** below the fold on a
laptop, which is the one control the section exists for. Two columns
now: who you are on the left, what you are changing it to on the
right.

### Nothing else changed

No change to the relay, the switching, replication, encryption or the
API beyond one added field. Upgrading from 0.7.0 is optional unless
the wrong version number bothers you.

## [0.7.0] — 2026-10-05

Push replication — the last thing the free/paid table promised and the
relay did not do. Plus two fixes that had been waiting for a release.

### Sending a copy to a second relay

If your viewers are spread across sites, one link across the network
beats one per viewer. KarRelay now dials a second KarRelay and sends it
a single copy of what the local subscribers get; the far relay fans it
out on its own network.

```
              ┌──SRT──▶ viewer
  encoder ──▶ │──SRT──▶ viewer        site A
   KarRelay   │
              └──SRT──▶ KarRelay ──┬──SRT──▶ viewer
                         site B    └──SRT──▶ viewer
```

It is also a standby path: if site A becomes unreachable, site B is
already holding the stream for its own viewers.

On the far relay, add a source in **listener** mode and note its input
port. On this one, press **Replicate** on the source or channel you
want sent and add a target with that host and port. The card then shows
how many remotes are up.

- A **channel's** targets receive the switched programme, not one of
  its candidate sources. A **source's** targets receive what its own
  subscribers receive, timecode injection included.
- Targets are edited while everything is running. Adding, removing or
  changing one leaves the others' connections untouched and does not
  restart the source, so nobody watching notices.
- A target **reconnects on its own** if the far relay restarts or the
  link drops, backing off between attempts. Local viewers are not
  disturbed while a remote is down, and the byte count does not reset
  when it comes back.
- **Disabled** keeps a target in the list without contacting it, for
  pausing a remote without losing its settings.
- Each target can carry **its own SRT passphrase**, independent of the
  source's, matching the passphrase on the far relay's source.
- Per-target state in the interface: connected or retrying, uptime,
  bytes sent, drops, RTT, and the reason the last attempt failed.
- A target pointing back at one of this relay's own ports is
  **refused**, in whichever order the two settings were saved. That
  would feed the stream into itself, and nothing about the symptoms
  would say so — just a bitrate and a viewer count that make no sense.
  Two *separate* relays pointed at each other cannot be detected from
  one end and are not caught.

Free on every licence, and it will stay that way.

Measured between two relays: the hop comes up in well under a
millisecond on a local network, the far relay reports no loss, and a
viewer on it gets the stream unchanged. Killing the far relay leaves
the local viewers untouched; the link returns by itself a few seconds
after the remote comes back.

### Fixed

**A cached licence worked on any machine it was copied to.** An
activation is issued for one install, and the file was not being
checked against the machine it was loaded on — so copying it to a
second relay activated that one too. The limit on machines per licence
was applied when activating and bypassed entirely by the file. Now
verified every time a cached activation is loaded.

Found while writing tests, not from a report, and the check was
confirmed against a real activation before being turned on: getting it
subtly wrong would lock paying customers out of their own relay.

**A crash when stopping a source while it was receiving.** Two parts of
the relay could touch the same stream state at once during a stop.
Rare, and only on the way down, but it took the process with it.

### Also

The retry delay after a failed input connection now stops growing at
five seconds, as the documentation always said. It had been climbing to
eight.

`sei_metadata`, webhooks and interface branding are still not built.
They are no longer on a roadmap either — see the note under the table
in the README. Everything the free/paid table lists now exists.

## [0.6.0] — 2026-10-04

A security release. 0.5.0 shipped an operator password that crossed the
network in the clear, and said so. This closes that, and two more gaps
that were open alongside it. None of it is behind a paid tier.

### Transport encryption for the web interface

Off by default — a generated certificate makes a browser complain, and
an operator who did not ask for that should not meet it on a relay that
worked yesterday. One click on the **About** tab to turn on.

HTTPS and plain HTTP share the service port, told apart by the first
byte of the connection. That is deliberate: KarPlayer is configured with
one port, speaks plain HTTP and has no certificate store, so any other
arrangement would have meant every installed player needing a settings
change on upgrade day. The player's clock sync keeps working untouched;
everything else over plain HTTP is redirected to https, and anything
carrying credentials is refused rather than redirected, because
redirecting it would invite the caller to send a password a second time
when the first already crossed in the clear.

The certificate is generated for you, valid ten years, and covers every
name the interface might be opened under. **Compare the fingerprint**
the UI and log show against what your browser shows — with a
self-signed certificate that comparison is the whole of the security.
Your own certificate and key can be used instead.

### A listen address

The relay bound every interface, so the port answered on every network
the machine could see. It can now be restricted to one interface, or to
the relay machine itself, which removes the port from the others rather
than defending it.

Chosen from a list of the machine's own addresses rather than typed, and
an address the machine does not have is refused while the interface is
still reachable to say so. If the address stops being bindable later — a
DHCP lease moves, a machine is cloned — the relay falls back to loopback
and logs it, rather than leaving you with no way in.

### SRT link encryption

The option had existed in the code since 0.1 and was read by nothing, so
streams were always in the clear while the documentation said otherwise.
Now real: a passphrase per direction on a source and one for a channel's
subscribers, AES-128, 192 or 256. A peer with the wrong passphrase, or
none, is rejected rather than quietly served an unencrypted link.

Verified end to end with KarPlayer on a tablet, including source
switching on an encrypted channel.

### Also

- Any libsrt receiver — KarPlayer, ffmpeg, vMix — no longer has its log
  filled with `EPE: incoming UMSG: 6 INVALID SIZE: 0` several times a
  second. 405 lines in 8 seconds before, zero now. The fix has been
  [proposed upstream](https://github.com/datarhei/gosrt/pull/161).

---

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
