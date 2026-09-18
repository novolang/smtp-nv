# smtp-nv

Message submission is the act of handing a message you composed to the
server that will deliver it. The commands and replies are the Simple
Mail Transfer Protocol, specified in
[RFC 5321](https://www.rfc-editor.org/rfc/rfc5321); the restrictions a
submission server places on them are
[RFC 6409](https://www.rfc-editor.org/rfc/rfc6409); the message itself
is [RFC 5322](https://www.rfc-editor.org/rfc/rfc5322). This package
brings all three to novo-lang, for a program that sends its own mail.
It builds on four packages on the registry:
[mime-nv](https://novo-lang.org/packages/mime-nv) for the media types,
[base64-nv](https://novo-lang.org/packages/base64-nv) for the three
places SMTP needs base64,
[crypto-nv](https://novo-lang.org/packages/crypto-nv) for HMAC-MD5, and
[calendar-nv](https://novo-lang.org/packages/calendar-nv) for the date
arithmetic. [dkim-nv](https://novo-lang.org/packages/dkim-nv) signs a
message this package sends.

**Status: NOT IMPLEMENTED — interface only.** Every function is declared
with its full signature, but every body is a `todo()` that panics when
called. The package is published so its design can be reviewed and
depended on before it is implemented. Version 0.1.0 will be the first
working release.

## What message submission is

A mail transfer agent (MTA) is a server that accepts a message and
passes it on. Submission is the first hop: a program hands its own
message to one MTA, which authenticates the program and takes
responsibility for the rest. Relay is every hop after that, and this
package does not do it. RFC 6409 section 1 draws the line.

The conversation is lines of text. The client sends a command and the
server answers with a **reply**, which is a three-digit code and some
text. A reply may run to several lines. `250-SIZE` continues and
`250 SIZE` ends, and the difference is one character in column four.
RFC 5321 section 4.2.1 defines both, and it also fixes what the first
digit means.

| First digit | Meaning | What a client does |
| --- | --- | --- |
| 2 | Accepted | Send the next command |
| 3 | Send more | Send the body, or the next authentication line |
| 4 | Temporary failure | Try the same thing later |
| 5 | Permanent failure | Do not try the same thing again |

The client opens with **EHLO**, which names the client and asks what the
server can do. The server answers with its **capability list**: the
extensions it supports, one per line. `STARTTLS` offers encryption,
`AUTH` lists the authentication mechanisms, `SIZE` gives a byte limit,
and `PIPELINING` permits several commands to be sent without waiting
for each reply. RFC 5321 section 4.1.1.1 defines EHLO.

Three ports carry SMTP, and they are not interchangeable.

| Port | What it carries | Reference |
| --- | --- | --- |
| 587 | Submission, encrypted by STARTTLS after the first EHLO | RFC 6409 section 3.1 |
| 465 | Submission, encrypted from the first byte | RFC 8314 section 3.3 |
| 25 | Relay between servers, with no authentication | RFC 5321 section 2.3.4 |

The **envelope** is `MAIL FROM` and one `RCPT TO` per recipient. It is
what the server routes on. The **headers** are `From:`, `To:` and `Cc:`,
which are text inside the message that nothing routes on at all. A blind
recipient is in the envelope and in no header, which is the whole of
what blind means. RFC 5321 section 2.3.1 keeps the two apart.

The body follows a `DATA` command and ends with a line holding one dot.
A line of the message that already begins with a dot therefore has a
second dot put in front of it, which is called **dot-stuffing**. RFC
5321 section 4.5.2 defines it. The sizes below are the limits every
implementation is held to.

| Limit | Value | Reference |
| --- | --- | --- |
| A command line | 512 bytes | RFC 5321 section 4.5.3.1.4 |
| Any line, CRLF included | 1000 bytes | RFC 5321 section 4.5.3.1.6 |
| A header line, recommended | 78 characters | RFC 5322 section 2.1.1 |
| A header line, maximum | 998 characters | RFC 5322 section 2.1.1 |
| Waiting for an ordinary reply | 5 minutes | RFC 5321 section 4.5.3.2 |
| Waiting for the reply after the final dot | 10 minutes | RFC 5321 section 4.5.3.2 |

Three authentication mechanisms are implemented, and they differ in what
travels over the connection.

| Mechanism | Reference | What travels |
| --- | --- | --- |
| PLAIN | RFC 4616 | The password, in base64, in one line |
| LOGIN | No standard; an expired draft | The password, in base64, after two prompts |
| CRAM-MD5 | RFC 2195 | An HMAC-MD5 of a challenge the server chose |

## Install

```
novo pkg add smtp-nv
```

## Example

```novo
use civil
use smtpmsg
use smtpsend
use smtptrans
use smtpwire

// Submit one message on port 587, upgrading the connection to TLS.
fn submit(host: Str, user: Str, secret: [u8],
          m: SmtpMessage) -> Result<Unit, SmtpSessionError> [io, net, time]
    // Open a plain connection and read the server's 220 greeting.
    let tcp: SmtpTcp = match smtptrans.dial_tcp(host, smtpwire.SMTP_SUBMISSION_PORT)
        Ok(t)  => t
        Err(e) => return Err(SmtpConnectionFailed(e))
    var s = smtpsend.session(smtpsend.default_options())
    s = smtpsend.read_greeting(s, tcp, smtpsend.now_ms())!.session

    // The first EHLO. Its capability list is discarded at the upgrade.
    s = smtpsend.ehlo(s, tcp, smtpsend.now_ms())!.session

    // STARTTLS. It answers the transport that replaces the plain one.
    let up: SmtpUpgrade = smtpsend.starttls(s, tcp, host, smtpsend.now_ms())!
    let tls: SmtpTls = up.transport

    // The second EHLO, over TLS. This capability list is the one that counts.
    s = smtpsend.ehlo(up.session, tls, smtpsend.now_ms())!.session
    s = smtpsend.authenticate(s, tls, user, secret, smtpsend.now_ms())!.session

    // MAIL FROM and one RCPT TO per recipient, then DATA and the body.
    s = smtpsend.send_envelope(s, tls, m, smtpsend.now_ms())!.session
    s = smtpsend.send_data(s, tls, m, smtpsend.now_ms())!.session
    let _ = smtpsend.quit(s, tls, smtpsend.now_ms())!
    Ok()

fn main() [io, net, time]
    match civil.date(2026, 9, 15)
        Err(_) => println("not a date")
        Ok(d)  =>
            // The date and the message identifier are arguments, so the
            // same message renders to the same bytes every time.
            let when = civil.datetime(d, civil.midnight())
            let m = smtpmsg.message("alice@example.com", "bob@example.net",
                                    "hello", [], "id-1@example.com", when, 0)
            match submit("smtp.example.com", "alice@example.com", [], m)
                Ok(_)  => println("the server accepted the message")
                Err(e) => println(e.message())
```

Build and test with `novo pkg build` and `novo test`. Today `novo test`
fails on purpose: every test reaches a
`not implemented: smtp-nv.<module>.<fn>` panic. The tests are the
specification the implementation will have to satisfy.

## What the package contains

| Module | Contents |
| --- | --- |
| `smtpwire` | The command and reply codec. Commands as values, a multiline reply as one reply, the enhanced status code beside the three digits, the capability list, dot-stuffing with its terminator, and the pipelining rule. Nothing in it performs input or output. |
| `smtpauth` | The three authentication mechanisms, each as a function from a server challenge to a response, and the rule for choosing between them. |
| `smtpmsg` | The message: RFC 5322 headers, the envelope kept apart from them, MIME multipart bodies, base64 attachments, and the encoded-word form of a header that is not ASCII. |
| `smtptrans` | The transport the session runs over. A plain TCP connection, a TLS connection dialled with the standard library's `TlsConfig`, and the upgrade from the first to the second. |
| `smtpsend` | The session. The greeting, EHLO, STARTTLS, authentication, the envelope, DATA, RSET and QUIT. |

## How to choose an entry point

**`smtpsend` is the whole conversation.** Call it when you want to send
a message and have the package drive the exchange. Every session
function takes the transport as an argument and costs whatever that
transport costs, so the same code runs over TCP, over TLS, and over a
recorded transcript in a test.

**`smtpwire` is the codec on its own.** It turns bytes into replies and
commands into bytes, and it performs nothing. Call it when you hold the
connection yourself, or when you are writing something that reads SMTP
without speaking it.

**`smtpmsg` builds a message without sending one.** Call it when the
message is going somewhere other than a socket.

There are also two ways to open a connection. `smtptrans.dial_tcp` opens
port 587 in the clear, and the session then upgrades it with STARTTLS.
`smtptrans.dial_tls` opens port 465, which is encrypted from the first
byte and has no STARTTLS at all. RFC 8314 section 3.3 prefers the
second, because an offer of STARTTLS can be removed in transit and an
already-encrypted connection cannot.

## The rules a user needs

1. **Everything the server said before STARTTLS is discarded.** RFC 3207
   section 4.2 requires it. The capability list read over the plain
   connection was the attacker's list if there was one, and acting on it
   means a stripped `STARTTLS`, an invented `AUTH PLAIN`, or a shrunken
   `SIZE`. `starttls` resets the session to
   `smtpwire.no_capabilities()`, and `capabilities_of` answers the list
   read after the upgrade and no other.
2. **A second EHLO is required after the upgrade.** RFC 3207 section
   4.2. `starttls` moves the session back to `SmtpGreeted`, so EHLO is
   the only command it will then accept.
3. **The server may send nothing between its `220` and the handshake.**
   A client that buffered those bytes would obey them as replies to
   commands it sends inside the TLS session, which is CVE-2011-0411.
   `starttls` discards the read buffer, and a server that sent anything
   gets `SmtpPipelinedAfterStartTls`.
4. **Only the first digit of a reply code may be branched on.** RFC 5321
   section 4.2.1. A client that retried a 550 sends the same rejection
   until somebody stops it, and one that gave up on a 451 loses mail
   that would have gone through in a minute. `class_of` and
   `is_temporary` answer the question.
5. **The envelope is not the headers.** RFC 5321 section 2.3.1.
   `with_bcc` grows the envelope and touches no header, `with_to` and
   `with_cc` grow both, and `envelope_recipients` is what `RCPT TO` is
   issued for. It deduplicates, because an address that is both a `To`
   and a `Cc` is one delivery.
6. **A refused recipient is a value, not an error.** A message to twelve
   people with one wrong address is eleven deliveries and one report.
   `send_envelope` records the refusal and continues, and
   `SmtpNoRecipientAccepted` arrives only when every recipient was
   refused.
7. **`data_body` stuffs the dots and appends the terminator together.**
   RFC 5321 section 4.5.2. A caller that stuffed and forgot the
   terminator hangs. A caller that terminated without stuffing sends
   half a message, the server accepts it, and the rest is read as
   commands.
8. **A `SIZE=` parameter carries the transmitted size.** That is the
   length after stuffing, which `transmitted_size` answers. A client
   that declared the unstuffed length can be refused for a message that
   would have fitted. RFC 1870 defines the parameter.
9. **`EHLO`, `DATA`, `STARTTLS`, `QUIT` and every AUTH exchange are
   pipelining barriers.** RFC 2920 section 3.1. Everything else may be
   sent in a group, which turns a twelve-recipient message from thirteen
   round trips into two. `may_pipeline` answers for a group.
10. **PLAIN is preferred on an encrypted connection and refused without
    one.** `best_mechanism` answers PLAIN, then CRAM-MD5, then LOGIN when
    the connection is encrypted. Without encryption it answers CRAM-MD5
    or nothing, because PLAIN over a cleartext wire is the password in
    base64 and base64 is not encryption. `allow_cleartext` is how a
    caller overrides that for a relay on the loopback interface.
11. **The `Date:` header is RFC 5322 section 3.3's spelling, not RFC
    3339's.** `Tue, 1 Jul 2003 10:52:37 +0200` is the form.
    `date_header` produces it. A date in ISO spelling is one some readers
    show as the epoch.
12. **The date, the message identifier and the MIME boundary are
    arguments.** None of them is drawn inside the package, so the same
    message renders to the same bytes every time. `now_ms` is the one
    function here that reads a clock.
13. **A CR or an LF is refused where it would inject.** A newline in a
    header value ends the header and begins another one the caller never
    wrote, and `header_fault` refuses it before `render` writes it. The
    same character in a command argument ends the command, and `encode`
    refuses it with `SmtpControlCharacterInArgument`.

## What is not included

- **A server.** There is no listener, no queue and no relay decision.
  This package is the client half.
- **Delivery.** Submission hands a message to a server that will deliver
  it. Looking up the recipient's MX records, retrying on a temporary
  failure and generating a bounce are that server's work.
- **A signature.** DKIM is a canonicalisation, a hash and a signature
  over selected headers. It is
  [dkim-nv](https://novo-lang.org/packages/dkim-nv), which takes the
  bytes `smtpmsg.render` produces.
- **Reading a received message.** That is a parser rather than a
  builder, and fetching the message is IMAP or POP3. Neither is here.
- **Authentication beyond three mechanisms.** XOAUTH2 is what a large
  provider asks for now. It is a bearer token this package would only
  base64, and it will be added when a consumer needs it.
- **Name resolution.** A submission server is configured by the person
  running the program, not discovered.
- **A microcontroller build.** The package opens sockets and speaks TLS,
  so it runs on a host.

## Related packages

- [dkim-nv](https://novo-lang.org/packages/dkim-nv) signs a message
  before it is submitted. It takes the rendered headers and body, and it
  answers the `DKIM-Signature` field value to put above them. It has no
  socket and no resolver, which is why it is a separate package.
- [mime-nv](https://novo-lang.org/packages/mime-nv) is the media type
  grammar. `with_attachment` looks a filename up in its extension table.
- [base64-nv](https://novo-lang.org/packages/base64-nv) is RFC 4648.
  SMTP needs it in three places: the AUTH exchange, an attachment's
  `Content-Transfer-Encoding`, and the encoded-word form of a header.
- [crypto-nv](https://novo-lang.org/packages/crypto-nv) supplies the
  HMAC-MD5 that CRAM-MD5 is built on, and nothing else.
- [calendar-nv](https://novo-lang.org/packages/calendar-nv) supplies the
  civil date arithmetic under the `Date:` header. It renders RFC 3339,
  which is a different spelling, so the header is rendered here.
- There is no `smtp-codec-nv` on the registry. `smtpwire` is that codec,
  and it performs no input or output, so a program that wants the bytes
  without the conversation can use it alone.
- `std.net` in the standard library is the socket this package dials
  through. The TLS session beside it is an opaque handle, and its
  options are `std.tls`'s own `TlsConfig`, named bare after
  `use std.tls`. This package declared an `SmtpTlsConfig` of its own in
  0.0.2, when `std.tls` had no page and no `use` that resolved; 0.0.3
  drops it for the standard library's record, which carries every field
  it had. A caller that has its own transport implements
  `SmtpTransport` over it instead.

## Tests

```bash
novo test tests/smtpwire_tests.nv    # 22 tests over the codec and the mechanisms
novo test tests/smtpsend_tests.nv    # 27 tests over the session and the message
```

The reference data is the specifications' own. RFC 2195's CRAM-MD5
example, with its challenge, its shared secret and the digest it
produces, is an assertion. RFC 4616's PLAIN example is another, and its
leading empty authorization identity is the byte a naive implementation
leaves out. The transcripts the codec is fed are EHLO capability lists in RFC
5321's own shape, and the ports, the line limits and the two timeouts
are asserted against the numbers the RFCs publish.

The session suite runs an entire submission over `SmtpTape`, a transport
whose bytes are already in memory. It implements `SmtpTransport[]`, so
the compiler checks that a whole conversation, the STARTTLS upgrade
included, costs no effects at all. The suite asserts that the upgrade
empties the capability list, that a server speaking before the handshake
is refused, that a blind recipient appears in the envelope and in no
rendered header, and that the same message renders to the same bytes
twice.

The tests compile today and fail at run, each on the
`not implemented: smtp-nv.<module>.<fn>` panic that is its body. That is
the expected state of an interface release. They turn green one at a
time as bodies land.

## Implementation status

| Item | Implemented |
| --- | --- |
| The `SMTP_LINE_MAX`, `SMTP_COMMAND_MAX` and the three port constants | yes (they are constants) |
| `smtpmsg.HEADER_LINE_SOFT_MAX`, `.HEADER_LINE_HARD_MAX` | yes (they are constants) |
| `smtpwire.reader`, `.reader_with`, `.scan`, `.feed`, `.take`, `.pending_len`, `.parse_reply` | no |
| `smtpwire.class_of`, `.is_positive`, `.is_temporary`, `.encode` | no |
| `smtpwire.capabilities_of`, `.no_capabilities`, `.may_pipeline` | no |
| `smtpwire.data_body`, `.undo_dot_stuffing`, `.transmitted_size` | no |
| `smtpauth.mechanism_name`, `.mechanism_of`, `.best_mechanism`, `.is_safe_without_tls` | no |
| `smtpauth.begin`, `.initial_response`, `.answer`, `.is_done` | no |
| `smtpauth.plain_response`, `.cram_md5_response` | no |
| `smtpmsg.message`, `.with_bcc`, `.with_to`, `.with_cc`, `.with_header`, `.in_thread` | no |
| `smtpmsg.with_alternative`, `.with_attachment`, `.with_inline` | no |
| `smtpmsg.render`, `.envelope_recipients`, `.date_header`, `.boundary` | no |
| `smtpmsg.encoded_word`, `.needs_encoding`, `.header_fault`, `.fold` | no |
| `smtptrans.dial_tcp`, `.dial_tls`, `.tls_for`, `.stream_of` | no |
| `smtptrans`: both `SmtpTransport` implementations | no |
| `smtpsend.default_options`, `.with_ehlo_domain`, `.allow_cleartext`, `.session` | no |
| `smtpsend.state_of`, `.capabilities_of`, `.now_ms`, `.is_encrypted` | no |
| `smtpsend.read_greeting`, `.ehlo`, `.starttls` | no |
| `smtpsend.authenticate`, `.authenticate_with` | no |
| `smtpsend.send_envelope`, `.send_data`, `.reset`, `.quit` | no |
| `smtpsend.accepted_of`, `.refused_of`, `.is_temporary` | no |
| The four `Error` implementations, one per module that has an error | no |

## Licence

Apache-2.0. See `LICENSE`.

<!-- docs/writing-a-readme.md is the style guide for this page. -->
