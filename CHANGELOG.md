# Changelog

All notable changes to smtp-nv are recorded here. The format is
[Keep a Changelog](https://keepachangelog.com/en/1.1.0/), and this
package follows [Semantic Versioning](https://semver.org/spec/v2.0.0.html)
with the pre-1.0 rule that a breaking change bumps the MINOR number.

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
