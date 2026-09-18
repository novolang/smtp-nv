# Changelog

All notable changes to smtp-nv are recorded here. The format is
[Keep a Changelog](https://keepachangelog.com/en/1.1.0/), and this
package follows [Semantic Versioning](https://semver.org/spec/v2.0.0.html)
with the pre-1.0 rule that a breaking change bumps the MINOR number.

## 0.0.2 — 2026-09-15

README rewritten to the package README style guide (docs/writing-a-readme.md).

### Changed

- **`smtptrans.dial_tls` and `.tls_for` name `SmtpTlsConfig`**, a struct
  this package declares, where 0.0.1 named a bare `TlsConfig`.  No such
  type exists in any build: the standard library's TLS surface is not
  published, and the name type-checked in 0.0.1 only because the
  undefined-type check (`E2033`) had not landed.  The fields are the
  ones the two functions always meant — `hostname` for SNI and the
  certificate name match, `verify_peer`, and `ca_bundle_path` for a
  caller that pins its own roots.  `smtp_upgrade` is unchanged: it takes
  a hostname, because STARTTLS has no second set of options to apply.
  A 0.0.1 consumer could not have called either function, so nothing
  that compiled before stops.

## 0.0.1 — 2026-09-11

The **interface**: every signature and every effect row, and no bodies.
`stability = "draft"`, and the release is recorded `implemented = false`.

### Added

- `smtpwire` — the command and reply codec, sans-IO and `[]`
  throughout: commands as values, multiline replies as one reply,
  the enhanced status code beside the three digits, the capability
  list, dot-stuffing with its terminator, and the pipelining rule.
- `smtpsend` — the session: the greeting, EHLO, STARTTLS with the
  discard that is this package's load-bearing rule, AUTH, the
  envelope with a refused recipient as an event rather than an error,
  DATA with its own ten-minute budget, RSET and QUIT.
- `smtptrans` — `SmtpTransport[e]` with `smtp_upgrade`, the method no
  other transport in this cohort has, and `SmtpTcp` / `SmtpTls`.
- `smtpauth` — PLAIN, LOGIN and CRAM-MD5 as functions from a challenge
  to a response, with `best_mechanism` preferring PLAIN on TLS and
  CRAM-MD5 without it, and refusing everything else on a cleartext
  connection.
- `smtpmsg` — the RFC 5322 builder: the envelope kept apart from the
  headers, `with_bcc` that touches no header, MIME multipart bodies
  over mime-nv, base64 attachments, RFC 2047 encoded words, and
  `header_fault`, the injection check `render` calls.

### Known

- **Everything the server said before STARTTLS is discarded**, and
  that is the load-bearing rule.  The pre-upgrade capability list is
  the attacker's if there was one; `starttls` resets it, requires a
  second EHLO, and there is no accessor for the earlier list.
- **The read buffer is discarded too.**  A server that sent anything
  between its `220` and the handshake gets
  `SmtpPipelinedAfterStartTls` rather than having those bytes obeyed
  as replies inside the TLS session — CVE-2011-0411's class.
- **STARTTLS is a transport that replaces itself**, which is why the
  trait has `smtp_upgrade` and why it answers a new transport rather
  than mutating one.
- **The envelope is not the headers.**  A blind recipient is in the
  envelope and in no header; `with_bcc` is the function whose whole
  purpose is that it touches none.
- **The date and the message identifier are fields**, so a rendered
  message is byte-for-byte reproducible.
- **Injection is refused in the encoder, twice**: a newline in a header
  value and a control character in a command argument.
- **A refused recipient is a value.**  Twelve recipients with one bad
  address is eleven deliveries and one report.
- **No `smtp-codec-nv` row yet.**  `smtpwire` is sans-IO and lifts out
  unchanged if a second consumer — a server, a milter, a proxy —
  appears; the README argues why symmetry alone does not earn it.
- **DKIM is a missing row** nothing on the grid names: a
  canonicalisation, a hash and a signature over selected headers,
  belonging in its own `core` package that `smtpmsg` would depend on.
- **Four `core` dependencies**: mime-nv, base64-nv, crypto-nv and
  calendar-nv.
- **No device claim.**  The package is `host`.

### Design notes

- **Why there is no `smtp-codec-nv`.**  `smtpwire` is the whole codec:
  a line, a three-digit number, a continuation character in column
  four, and dot-stuffing.  The three codec packages this cohort sits
  over each exist for a reason SMTP does not have.  mqtt-codec-nv
  exists because a device runs the codec and not the client.
  websocket-codec-nv exists because the framing is intricate — masking,
  fragmentation, sixteen close codes — and because a server, a proxy
  and a fuzzer all want it without a socket.  grpc-codec-nv exists
  because the call state machine is the protocol.  No device speaks
  SMTP, the framing is a line, and the state machine is nine states of
  "did the server say 2yz".  A second consumer that wants the codec and
  not the session — a server, a milter, a proxy that rewrites
  envelopes — is what would change it, and `smtpwire` is sans-IO
  already, so it would lift out unchanged.
- **Why `layer = "host"` rather than `layer = "core"` with
  `host_modules`.**  The narrower layer with the wider modules named is
  the shape for a package whose subject is the pure half.  This
  package's subject is submitting a message over a socket, and the
  codec is there because a session needs one.
- **The effect rows, module by module.**  `smtpwire`, `smtpauth` and
  `smtpmsg` are `[]` throughout.  `smtptrans.dial_tcp` is `[io, net]`,
  which is `std.net`'s own row, and `smtptrans.dial_tls` is `[net]`,
  which is `std.tls`'s and is narrower.  `smtpsend.now_ms` is `[time]`
  and is the one function in the package that reads a clock.  Every
  session function in `smtpsend` is effect-polymorphic over the
  transport.
- **What changed in the port.**  `lettre` (Rust) supplied the session
  shape and the transport split; Python's `smtplib` supplied the
  command surface.  Three things differ.  `lettre`'s transport is an
  enum of the connections it knows about, and here it is a trait,
  because STARTTLS needs a transport that can replace itself and a
  caller's own transport should be a first-class case.  Its `Message`
  builder is typed with a phantom state machine, so a message without a
  recipient does not compile; here the check is `smtpmsg.render`
  answering `SmtpNoRecipients`.  Its `Tokio`/`async-std` split does not
  exist at all: there is one set of functions, effect-polymorphic over
  the transport.
