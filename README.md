# smtp-nv

**Status: NOT IMPLEMENTED — interface only.**

Every public function below is published with its signature and its
effect row, and every body is `todo()`.  Installing this package works;
calling it panics with `not implemented`.

## What this is

SMTP submission (RFC 6409): handing a message you composed to the
server that will deliver it.  EHLO and the capability list, STARTTLS,
AUTH PLAIN / LOGIN / CRAM-MD5, the envelope, DATA with its dots
stuffed, pipelining, and an RFC 5322 message builder with MIME
multipart bodies.

It is not a mail server, not a relay and not a mail reader.  It is what
a program that needs to send its own mail uses, on port 587 or 465,
against a server that will ask it who it is.

## Adding it, and checking it

```bash
novo pkg add smtp-nv        # into your novo.toml
novo pkg build              # type- and effect-check the package
novo test --isolate tests/smtpwire_tests.nv
```

`novo test` is red today and that is the point of the release: every
assertion fails with `not implemented: smtp-nv.<module>.<fn>`.  They
turn green one at a time as bodies land.

## The one example that will work

```novo
use smtpmsg
use smtpsend
use smtptrans
use smtpwire

// Submit one message over STARTTLS.  The second EHLO is not optional
// and the section below says why.
fn submit(host: Str, m: SmtpMessage) -> Result<Unit, SmtpSessionError> [io, net, time]
    let tcp = smtptrans.dial_tcp(host, smtpwire.SMTP_SUBMISSION_PORT)!
    var s = smtpsend.session(smtpsend.default_options())

    s = smtpsend.read_greeting(s, tcp, smtpsend.now_ms())!.session
    s = smtpsend.ehlo(s, tcp, smtpsend.now_ms())!.session

    // Everything the server said above this line is now discarded.
    let up = smtpsend.starttls(s, tcp, host, smtpsend.now_ms())!
    let tls = up.transport
    s = smtpsend.ehlo(up.session, tls, smtpsend.now_ms())!.session

    s = smtpsend.authenticate(s, tls, "user@example.com", secret(), smtpsend.now_ms())!.session
    s = smtpsend.send_envelope(s, tls, m, smtpsend.now_ms())!.session
    s = smtpsend.send_data(s, tls, m, smtpsend.now_ms())!.session
    let _ = smtpsend.quit(s, tls, smtpsend.now_ms())!
    Ok()
```

## The layer, and why

`host`, and most of the package declares nothing.

| module | row | why |
| --- | --- | --- |
| `smtpwire` | `[]` throughout | the command and reply codec; sans-IO by construction |
| `smtpauth` | `[]` throughout | a mechanism is a function from a challenge to a response |
| `smtpmsg` | `[]` throughout | a message is a value, and rendering it is arithmetic |
| `smtptrans.dial_tcp` | `[io, net]` | `std.net`'s own row |
| `smtptrans.dial_tls` | `[net]` | `std.tls`'s own row, which is narrower |
| `smtpsend.now_ms` | `[time]` | the one function in the package that reads a clock |
| every session function in `smtpsend` | `[e]` | effect-POLYMORPHIC: whatever the transport costs |

`layer = "host"` and **not** `layer = "core"` with `host_modules`.  The
manifest can declare the narrower layer and name the wider modules, and
that shape is for a package whose *subject* is the pure half — a
matcher with a directory walker beside it.  This package's subject is
submitting a message over a socket; the codec is there because a
session needs one.

## The load-bearing interface

**Everything the server said before STARTTLS is discarded**, and
`smtpsend.starttls` is where that happens.

Before the TLS handshake, every byte the server sent arrived over a
wire an attacker in the middle was writing on.  So the capability list
is the *attacker's* list, and three things follow from treating it as
the server's:

- they remove `STARTTLS`, the client sees a server that cannot
  encrypt, and the whole session is in the clear;
- they add `AUTH PLAIN` a real server never offered, and the client
  sends a password in base64 over that clear wire;
- they shrink `SIZE`, and a message that would have gone through is
  refused.

So `starttls` resets the session to `smtpwire.no_capabilities()`, moves
it back to `SmtpGreeted`, and requires a second EHLO.
`smtpsend.capabilities_of` answers the post-upgrade list and no other
— there is no accessor for the earlier one, because there is no correct
use for it.

The same function handles the other half of the same attack.  A server
may not send anything between its `220 Go ahead` and the TLS
ClientHello; a client that had buffered those bytes would execute them
as replies to commands it sends **inside** the TLS session.  That is
CVE-2011-0411 and its descendants, found again in several clients as
recently as 2021.  `starttls` discards the read buffer along with the
capabilities, and a server that sent anything gets
`SmtpPipelinedAfterStartTls` rather than having its bytes obeyed.

The second decision is `SmtpTransport[e]`, and it carries a method no
other transport in this cohort has: **`smtp_upgrade`**.  STARTTLS is
not a second connection — it is the same TCP connection becoming a TLS
one after a command. mqtt-nv and websocket-nv decide `ws://` or `wss://`
before they dial; SMTP decides after the server has already spoken.  So
`smtp_upgrade` answers a *new* transport rather than mutating one, and
a caller that held the old one afterwards would be writing plaintext
into a TLS session — which the type system now makes awkward rather
than silent.

## The envelope is not the headers

`MAIL FROM` and `RCPT TO` are what the server routes on; `From:`,
`To:` and `Cc:` are text inside the message that nothing routes on at
all.  A blind recipient is in the envelope and in **no header** — that
is the whole of what blind means — and a library that derived the
envelope from the headers either drops the blind copies or reveals
them.

So `SmtpMessage` carries both, `with_bcc` grows the envelope and
touches no header, and `envelope_recipients` is what `RCPT TO` is
issued for — deduplicated, because an address that is both a `To` and a
`Cc` is one delivery.

## The date and the message identifier are arguments

Both are what a `core` package would have had to take as parameters — a
clock and randomness — and this package takes them for the same reason
one layer up: a rendered message becomes byte-for-byte reproducible, so
a test asserts the bytes rather than a regular expression over them.
`smtpsend` is where a caller who wants "now" gets it.

The `Date:` header is RFC 5322 § 3.3's spelling — `Tue, 1 Jul 2003
10:52:37 +0200` — and not RFC 3339's.  calendar-nv renders the latter,
and a `Date:` in ISO spelling is one some readers show as the epoch, so
the rendering is `smtpmsg.date_header`'s and what comes from calendar-nv
is the arithmetic under it.

## Injection, in two places

A CR or LF in a header value ends the header and begins another one the
caller never wrote — which is how a web form's "your name" field
becomes a `Bcc:`.  `smtpmsg.header_fault` is that check and `render`
calls it, so the injected header cannot be built.

The same character in a command argument ends the command, which is how
the same bug becomes an open relay.  `smtpwire.encode` refuses with
`SmtpControlCharacterInArgument`.

Both are refusals inside the encoder rather than rules in this README,
because a rule in a README is one a calling program can believe it has
followed.

## Dot-stuffing, and the truncation that reports success

A line beginning with a dot is doubled, and the terminator is
`\r\n.\r\n`.  A message whose body has a line that is just `.` ends the
DATA early with no error at all: the server accepts half a message, and
the rest is interpreted as commands.

`smtpwire.data_body` stuffs and terminates in one function, because a
caller that stuffed and forgot the terminator hangs and one that
terminated without stuffing truncates — and only the first is visible.
`transmitted_size` is what a `SIZE=` parameter carries, because the
stuffed length is what actually travels and a client that declared the
unstuffed one can be refused for a message that would have fitted.

## Whether this earns an `smtp-codec-nv` row later

Not yet, and the case for it is thin.

`smtpwire` is the whole codec: a line, a three-digit number, a
continuation character in column four, and dot-stuffing.  Perhaps two
hundred lines of arithmetic when it is written, with none of the
protocol's interesting behaviour in it — that is all in the
conversation, which is `smtpsend`.  A separate package would be one
whose README said "this parses `250 OK`", and nobody browses to that.

Compare the three codecs this cohort sits over.  mqtt-codec-nv exists
because a device runs the codec and not the client.  websocket-codec-nv
exists because the framing is genuinely intricate — masking,
fragmentation, sixteen close codes — and because a server, a proxy and
a fuzzer all want it without a socket.  grpc-codec-nv exists because
the call state machine is the protocol.  SMTP has none of those three
properties: no device speaks SMTP, the framing is a line, and the state
machine is nine states of "did the server say 2yz".

**What would change it** is a second consumer that wants the codec and
not the session: an SMTP *server*, a milter, or a proxy that rewrites
envelopes.  If one of those appears on the grid, `smtpwire` lifts out
unchanged — it is sans-IO already and its module comment says so — and
the row is worth adding then.  Until then it would be a package split
for symmetry, which is the reason this cohort's other splits exist and
this one does not.

## What this does not do, on purpose

- **It is not a server.**  No listener, no queue, no relay decision.
- **It does not deliver.**  Submission hands a message to a server that
  will; MX lookup, retry schedules and bounce generation are that
  server's.
- **It does not sign.**  DKIM is a canonicalisation, a hash and an
  RSA or Ed25519 signature over selected headers, and it belongs in its
  own `core` package that `smtpmsg` would then depend on — an honest
  missing row, and one nothing on the grid names yet.
- **It does not parse a received message.**  Reading mail is IMAP and
  a different RFC 5322 half — a parser rather than a builder — and a
  package that did both would be two packages.
- **It does not do SASL beyond three mechanisms.**  XOAUTH2 is what a
  large provider wants now, and it is a token this package would only
  base64: worth adding when a consumer needs it, and not worth a SASL
  framework.
- **It does not resolve names.**  A submission server is configured,
  not discovered.
- **No device claim.**  The package is `host`.

## The reference implementation

`lettre` (Rust) for the session shape and the transport split, and
Python's `smtplib` for the command surface.  RFC 5321, RFC 5322, RFC
6409, RFC 2045–2047, RFC 2195 and RFC 4616 are the specifications, and
RFC 2195's own CRAM-MD5 example is a test vector.

Three things change in the port.  `lettre`'s transport is an enum of
the connections it knows about; here it is a trait, because STARTTLS
needs a transport that can replace itself and a caller's own transport
should be a first-class case.  Its `Message` builder is typed with a
phantom state machine so a message without a recipient will not
compile; here the check is `smtpmsg.render` answering
`SmtpNoRecipients`, because the phantom-typed builder makes the common
case harder to read and this package has only one place the check is
needed.  And its `Tokio`/`async-std` split does not exist at all: there
is one set of functions, effect-polymorphic over the transport, and a
caller that wants them on a runtime supplies a transport that is.

## Status

| item | implemented |
| --- | --- |
| `smtpwire` — `SmtpCommand`, `SmtpReplyClass`, `SmtpReply`, `SmtpScan`, `SmtpReader`, `SmtpStep`, `SmtpCapabilities`, `SmtpWireError` | types only |
| `smtpwire.SMTP_LINE_MAX`, `.SMTP_COMMAND_MAX`, `.SMTP_SUBMISSION_PORT`, `.SMTP_SUBMISSIONS_PORT`, `.SMTP_PORT` | yes — they are constants |
| `smtpwire.reader`, `.reader_with`, `.scan`, `.feed`, `.take`, `.pending_len`, `.parse_reply` | no |
| `smtpwire.class_of`, `.is_positive`, `.is_temporary`, `.encode` | no |
| `smtpwire.capabilities_of`, `.no_capabilities`, `.may_pipeline` | no |
| `smtpwire.data_body`, `.undo_dot_stuffing`, `.transmitted_size`, the `message` impl | no |
| `smtpauth` — `SmtpMechanism`, `SmtpAuthStep`, `SmtpAuthState`, `SmtpAuthTurn`, `SmtpAuthError` | types only |
| `smtpauth.mechanism_name`, `.mechanism_of`, `.best_mechanism`, `.is_safe_without_tls` | no |
| `smtpauth.begin`, `.initial_response`, `.answer`, `.is_done` | no |
| `smtpauth.plain_response`, `.cram_md5_response`, the `message` impl | no |
| `smtpmsg` — `SmtpHeader`, `SmtpBody`, `SmtpMessage`, `SmtpMessageError` | types only |
| `smtpmsg.HEADER_LINE_SOFT_MAX`, `.HEADER_LINE_HARD_MAX` | yes — they are constants |
| `smtpmsg.message`, `.with_bcc`, `.with_to`, `.with_cc`, `.with_header`, `.in_thread` | no |
| `smtpmsg.with_alternative`, `.with_attachment`, `.with_inline` | no |
| `smtpmsg.render`, `.envelope_recipients`, `.date_header`, `.boundary` | no |
| `smtpmsg.encoded_word`, `.needs_encoding`, `.header_fault`, `.fold`, the `message` impl | no |
| `smtptrans` — `SmtpTransport[e]`, `SmtpTcp`, `SmtpTls` | types only |
| `smtptrans.dial_tcp`, `.dial_tls`, `.tls_for`, `.stream_of` | no |
| `smtptrans` — both `SmtpTransport` impls | no |
| `smtpsend` — `SmtpSessionState`, `SmtpSessionOptions`, `SmtpSession`, `SmtpRefusal`, `SmtpSessionEvent`, `SmtpStepResult`, `SmtpUpgrade`, `SmtpSessionError` | types only |
| `smtpsend.default_options`, `.with_ehlo_domain`, `.allow_cleartext`, `.session` | no |
| `smtpsend.state_of`, `.capabilities_of`, `.now_ms`, `.is_encrypted` | no |
| `smtpsend.read_greeting`, `.ehlo`, `.starttls` | no |
| `smtpsend.authenticate`, `.authenticate_with` | no |
| `smtpsend.send_envelope`, `.send_data`, `.reset`, `.quit` | no |
| `smtpsend.accepted_of`, `.refused_of`, `.is_temporary`, the `message` impl | no |
