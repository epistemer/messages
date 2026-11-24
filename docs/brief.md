# neozp `messages`: design brief

> Written on 2026-11-24, when epistemer created the repository and
> delegated it (neozp decision D5: message delivery is the first of the new
> packages). Revised the same day after a review by Gemini 4 Neon (High)
> ([review 0001](reviews/0001-brief.md)). Decisions marked **epistemer**
> were made by epistemer. Decisions marked **(Claude)** were settled while
> writing this brief and can be overturned before the interface is locked.
> Open questions (§11) each have a default. zp appears only as counts and
> shapes.

## 0. How to use this brief

This is neozp's ninth package, after cfgx, i18n, genx, models, errors,
auth, handlers and ui. The module path is `github.com/epistemer/messages`.

1. Read the sibling repositories for their conventions: `docs/interface.md`
   (a locked contract with a decisions table and amendments),
   `docs/roadmap.md`, `docs/status.md`, `docs/adr/`, CI. Load the `cfgx`,
   `i18n`, `errors` and `auth` skills.
2. Set the module up like theirs: Go 1.27, MIT, the same CI jobs.
3. Write the design records from this brief: ADR 0001 from §§1–3,
   `docs/interface.md` from §§4–9 for epistemer to lock after the spikes
   (§12), and keep [roadmap.md](roadmap.md) current.
4. zp is private and read-only. Keep its business and its providers' names
   out of this repository. The examples use Confab or a toy shop.

## 1. What zp and Confab teach

zp sends email over SMTP from six templ templates, sends SMS through two
HTTP providers (with delayed sends and a call just for one-time codes), and
sends push notifications through one push service. Each channel has its own
interface and its own message type, and nothing is shared between them.
Confab, the neozp example app, has only a development outbox that writes
auth's messages to files, because no package delivers anything.

The failure shapes this design has to make impossible:

| Failure shape | What this package does about it |
|---|---|
| Subjects and SMS texts are English literals in code | Every text a recipient reads is an i18n key, so it can be translated and edited in the app (§6) |
| A message is rendered in the language of the request that sent it, not the recipient's | A recipient carries a locale; rendering attaches it (`i18n.WithContext` with `Match`) (§6) |
| Sending happens inside the request: a slow or down mail server slows or fails a sign-in, and a failure is lost | `Send` hands the message to a delivery loop and returns; the loop retries with backoff (§5) |
| An unconfigured mail server silently falls back to the local `sendmail` | The transport is chosen by configuration, and a missing setting is an error at startup (§8) |
| Phone numbers get a country prefix added by assumption | An SMS address must be E.164 (`^\+[1-9][0-9]{6,14}$`); anything else is refused at `Send` (§4) |
| An invalid push token is logged with "remove it from your database", and nobody does | A transport says when an address is gone, and the app is told which one (§5.3) |
| One interface per channel, plus a special call for one-time codes | One message model across channels; a code is just data in a message (§4) |
| Emails have an HTML body only | Every message has a text body; HTML is an optional extra for email (§6) |
| Testing means sending a real email from a command | A transport that records, one that writes files for development, and a test clock (§7) |
| Codes and links in messages risk reaching logs or disk | A message that carries a secret never touches disk, and nothing logs a body (§5.4, §9) |

## 2. The idea

**Declare a kind of message once; send it many times.** A kind has a key,
the channels it may go out on, a subject, a text and optionally an HTML
body, each rendered from typed data in the recipient's language. The app
sends a kind with its data to a recipient, and gets an id back.

**Send renders; a loop delivers.** `Send` renders the message and hands it
to a delivery loop. An ordinary message is stored first in an outbox, a
bbolt file of its own, so it survives a restart. A message that carries a
secret (a sign-in code, a recovery link) stays in memory only, and is
retried until its deadline. The loop hands each message to the transport
for its channel, retries temporary failures with backoff, stops on
permanent ones, and drops a message whose deadline has passed.

**Transports are small.** A transport delivers one rendered envelope. The
package ships SMTP for email, a directory transport for development and a
recording transport for tests. SMS providers implement the same small
interface.

**auth gets a sender for free.** A nested module implements `auth.Sender`
with default texts for every message auth sends, so an app on auth stops
writing its own outbox.

## 3. Decisions

| # | Topic | Decision | Who |
|---|---|---|---|
| D1 | Name | `github.com/epistemer/messages`, package `messages` | epistemer |
| D2 | Order | First of the three new packages, from Confab's outbox and auth's `Sender` | epistemer (neozp D5) |
| D3 | Channels in v0.1 | Email and SMS. Push is out, with no promise that today's model fits it | open (§11 Q7) |
| D4 | Delivery | A delivery loop with retries and deadlines; ordinary messages persist in an outbox | open (§11 Q1) |
| D5 | Secrets | A message marked secret is never written to disk: it is retried in memory until its deadline and lost on a crash | open (§11 Q3) |
| D6 | Rendering | At `Send`, in the recipient's locale. A render error reaches the caller, and the envelope doesn't change if templates change before delivery. A missing translation falls back to the code's text, as everywhere in i18n | open (§11 Q2) |
| D7 | Texts | Every text a recipient reads is an i18n key declared in Go (`i18n.Key`); no generator | (Claude) |
| D8 | Bodies | A text body always (the SMS, and the email's text part); HTML optional, through any `Render(ctx, io.Writer) error`, which a `templ.Component` already is, so the core doesn't import templ | open (§11 Q6) |
| D9 | auth | A nested module, `messages/authmessages`, implements `auth.Sender` with default texts for auth's eight message kinds | open (§11 Q5) |
| D10 | Storage | The outbox lives in its own bbolt file, which the app opens and passes in, so its writes never wait on auth's. Still one process (neozp D3) | open (§11 Q4) |
| D11 | Configuration | Small getter interfaces per part: the core (retries, retention), `smtp.Config`, `dir.Config`. `messagesschema` gives each as a cfgx group, and a wiring helper picks the email transport from one setting | (Claude), after review 0001 |
| D12 | Addresses | Email: a bare address (`net/mail` parses it and finds no display name); SMS: E.164. CR, LF and control characters are refused in addresses, names and subjects. Checked at `Send` | (Claude) |
| D13 | Logging | `log/slog`, replaceable by the app, like the siblings. A log names a message's id, kind, channel and a masked address (`a***@example.com`, `+20***4567`), never its body or a transport's raw error text | (Claude) |
| D14 | Errors | `errors` keys under `errors.messages.<name>`, with kinds | (Claude) |
| D15 | Guarantee | At least once for ordinary messages. A crash between delivery and the outbox update can send one twice. Its `Message-ID` stays the same, and an SMS transport gets the envelope's id to pass to its provider as an idempotency key | (Claude) |
| D16 | No log transport | No transport writes messages to the log, ever | (Claude) |
| D17 | Jobs | The delivery loop is this package's own. When neozp's jobs package exists, it may take the loop over; nothing here waits for it | (Claude) |

## 4. The model

```go
package messages

type Channel int

const (
	Email Channel = 1 << iota
	SMS
)

// A Recipient is who receives a message, and in which language.
type Recipient struct {
	Channel Channel
	Address string // a bare email address, or an E.164 phone number
	Name    string // optional: the display name in an email's To
	Locale  string // the recipient's language; the translator's default locale when empty
}

// A Spec declares a kind of message. D is the data its texts are made from.
type Spec[D any] struct {
	Key      string                                 // permanent, unique in the app
	Channels Channel                                // Email, SMS or Email|SMS
	Secret   bool                                   // never written to disk (D5)
	Subject  func(ctx context.Context, d D) string  // required for Email
	Text     func(ctx context.Context, d D) string  // required
	HTML     func(ctx context.Context, d D) Renderer // optional; Email only
}

// Renderer is what an HTML body is; a templ.Component is one.
type Renderer interface {
	Render(ctx context.Context, w io.Writer) error
}

type Kind[D any] struct{ /* unexported */ }

// Declare checks spec and returns its kind. It panics on a missing Key or
// Text, on Email without Subject, and on HTML without Email.
func Declare[D any](spec Spec[D]) *Kind[D]

// With gives a message of this kind with its data.
func (k *Kind[D]) With(d D) Message

type Message struct{ /* unexported */ }

func (m Message) NotBefore(t time.Time) Message // deliver no earlier (a delayed send)
func (m Message) NotAfter(t time.Time) Message  // drop it if it can't be delivered by then
```

A spec is a struct literal, so a missing part reads as a missing field.
The generic parameter only types the data its functions read, and a kind
without data uses `Spec[struct{}]`.

Declaring, in an app:

```go
var (
	L10nTalkAcceptedSubject = i18n.Key("messages.confab.talk_accepted.subject", "Your talk was accepted")
	L10nTalkAcceptedText    = i18n.Key("messages.confab.talk_accepted.text", "%s: your talk \"%s\" is in the programme of %s.")
)

type TalkAccepted struct{ Speaker, Title, Event string }

var TalkAcceptedMsg = messages.Declare(messages.Spec[TalkAccepted]{
	Key:      "confab.talk_accepted",
	Channels: messages.Email,
	Subject:  func(ctx context.Context, d TalkAccepted) string { return L10nTalkAcceptedSubject.T(ctx) },
	Text: func(ctx context.Context, d TalkAccepted) string {
		return L10nTalkAcceptedText.T(ctx, d.Speaker, d.Title, d.Event)
	},
	HTML: func(ctx context.Context, d TalkAccepted) messages.Renderer { return talkAcceptedEmail(d) },
})
```

`Send` refuses:
- a channel outside the kind's `Channels`;
- an address that doesn't fit the channel;
- CR, LF or control characters in an address, name or rendered subject;
- a `NotAfter` already past, or not after `NotBefore`;
- a secret kind without a `NotAfter`;
- an envelope over the size limit (§8);
- an HTML part that renders nothing or fails.

Two kinds with one key sent through the same `Messages` is an error.

## 5. Sending and delivery

### 5.1 The API

```go
func New(db *bbolt.DB, cfg Config, tr *i18n.Translator, opts ...Option) (*Messages, error)

func WithTransport(ch Channel, t Transport) Option // one per channel; Send refuses a channel without one
func WithRecorder(r Recorder) Option
func WithLogger(l *slog.Logger) Option
func WithClock(now func() time.Time) Option // for tests
func WithBucket(name string) Option         // top-level bucket; default "messages"

// Send renders msg for to and hands it to the loop: an ordinary message is
// stored first. It does not wait for delivery. It must not be called inside
// a write transaction on the outbox's bbolt file.
func (m *Messages) Send(ctx context.Context, to Recipient, msg Message) (ID, error)

// Cancel withdraws a message that hasn't been delivered yet.
func (m *Messages) Cancel(ctx context.Context, id ID) error // ErrNotQueued if it's too late

// Run delivers until ctx ends, then lets deliveries in flight finish for a
// grace period, and returns nil. A second Run on the same Messages is an error.
func (m *Messages) Run(ctx context.Context) error

// Flush delivers every due message now and returns. For tests and commands.
func (m *Messages) Flush(ctx context.Context) error

// Record reports what became of a message, without its body or address.
func (m *Messages) Record(ctx context.Context, id ID) (Record, error) // ErrNotFound

type Record struct {
	ID       ID
	Kind     string
	Channel  Channel
	State    State // Queued, Sent, Failed, Expired, Cancelled, Lost
	Attempts int
	Created, Updated time.Time
}
```

`New` refuses a nil translator.

### 5.2 Transports

```go
type Transport interface {
	Deliver(ctx context.Context, e Envelope) error
}

// An Envelope is a rendered message.
type Envelope struct {
	ID        ID // stable across attempts: an idempotency key for providers
	MessageID string // for email: "<id@domain>", fixed at Send
	Kind      string
	From      Sender // fixed at Send, so a retry doesn't change it
	To        Recipient
	Subject   string
	Text      string
	HTML      string // "" when the kind has none
	Attempt   int    // 1 for the first try
}

// Permanent marks a failure retrying can't fix, such as bad credentials.
func Permanent(err error) error
// Undeliverable marks an address that is gone or invalid. It is also permanent.
func Undeliverable(err error) error
// RetryAfter asks for the next attempt no earlier than d (a provider's 429).
func RetryAfter(d time.Duration, err error) error

func IsPermanent(err error) bool
func IsUndeliverable(err error) bool
```

All three return nil for a nil error.

Shipped transports:

| Package | Delivers |
|---|---|
| `messages/smtp` | email over SMTP: STARTTLS or implicit TLS, PLAIN or LOGIN authentication, `multipart/alternative` with text and HTML, RFC 2047 encoded headers. It refuses cleartext to anything but a loopback host, and never sends credentials in the clear |
| `messages/dir` | each message as a file named by its id, `.eml` for email (opens in a mail client), `.txt` for SMS, mode `0600`, written atomically. For development: it logs a warning at start naming its directory, since the files hold codes |
| `messages/messagestest` | records envelopes for tests, with assertions (§7) |

An SMS provider implements `Transport` in the app, or in its own module. A
guide shows how, with a toy provider over HTTP.

### 5.3 The delivery loop

- Each channel has its own workers, so a slow mail server never holds up a
  sign-in code by SMS. Due messages go out oldest-due first, with no
  promise of strict order.
- A message is claimed before it is handed over, so a slow delivery is
  never started twice.
- The attempt count is written before `Deliver`, so a message that crashes
  the process uses up its attempts.
- Each attempt has a timeout, capped by the message's `NotAfter`.
- A temporary failure is retried with exponential backoff and jitter,
  honouring `RetryAfter`. The retry limits run from the time the message
  became due, `max(Send, NotBefore)`. Past them, the message is `Failed`.
- A `Permanent` failure stops at once and is `Failed`. An `Undeliverable`
  one also tells the `Recorder` which address is gone. A generic permanent
  failure never does, so bad credentials never make an app drop a valid
  address.
- A message past its `NotAfter` is `Expired` without being sent.
- An outbox record that can't be read is set aside as `Failed`, logged by
  id, and never stops the loop.
- Once sent, failed, expired or cancelled, a message's body leaves the
  outbox, and its `Record` is kept for the configured retention, then
  pruned.

### 5.4 Secret messages

- A kind declared `Secret` never goes to disk. Neither its envelope nor its
  data does, because bbolt keeps deleted pages in the file.
- It waits in memory and is retried until its `NotAfter`, which it must
  have.
- A crash loses it. On restart nothing knows it existed, so it has no
  `Record`; within the run its state is `Lost` if the loop is stopped
  before it is sent.
- The person asks again, as they would for a code that never arrived. Auth
  codes expire in minutes, so little is lost.

### 5.5 The recorder

```go
type Recorder interface {
	Record(ctx context.Context, e Event)
}

type Event struct {
	At      time.Time
	ID      ID
	Kind    string
	Channel Channel
	State   State
	Address string // the full address, only for an Undeliverable event; else ""
	Reason  string // an error kind, never a transport's text
}
```

The loop calls the recorder outside any bbolt transaction, with a timeout,
and recovers a panic in it. A recorder that needs to be reliable stores the
event itself.

## 6. Language and bodies

- `Send` renders in the recipient's locale: it attaches
  `tr.Match(to.Locale)` with `i18n.WithContext`, so every `T(ctx)` in a
  spec's functions, and in an HTML body, reads the recipient's language.
  An empty locale means the translator's default locale.
- Texts are i18n keys, so the translations page edits them like every
  other text.
- The package offers a minimal email layout with inline styles. It takes
  `dir` and `lang` from the locale (i18n's `Locale.RTL`), and the app's
  name as an `i18n.Ter` the app passes in, such as a white-label
  `branding.title`. An app can use its own layout.
- The text body is the SMS's whole content and the email's text part.

## 7. Development and tests

```go
package messagestest

// Outbox is a transport that records, for tests.
type Outbox struct{ /* unexported */ }

func New() *Outbox
func (o *Outbox) Deliver(ctx context.Context, e messages.Envelope) error
func (o *Outbox) Sent() []messages.Envelope
func (o *Outbox) Fail(err error) // the next deliveries fail with err

// Flush delivers every due message now, failing t on an error.
func Flush(t testing.TB, m *messages.Messages)

// AssertSent fails t unless a message of kind reached to.
func AssertSent(t testing.TB, o *Outbox, kind, to string) messages.Envelope

// Clock is a clock tests move, for NotBefore, NotAfter and backoff.
type Clock struct{ /* unexported */ }

func NewClock(t time.Time) *Clock
func (c *Clock) Now() time.Time
func (c *Clock) Advance(d time.Duration)
```

`i18ntest` shows a translated message: a test sets a translation and
asserts the envelope's text.

## 8. Configuration

Getter interfaces, one per part. `messagesschema` gives each as a cfgx
group, pinned with `GoName`:

| Group | Keys |
|---|---|
| `messages` (core) | `retry.attempts`, `retry.max_age`, `retention` (records), `max_size` (an envelope) |
| `messages.email` | `transport` (enum: `smtp`, `dir`; required, no fallback), `from.address`, `from.name` (`Mutable`), `domain` (for `Message-ID`; defaults to the from address's) |
| `messages.smtp` | `host`, `port`, `username`, `password` (`Secret`), `tls` (enum: `starttls`, `implicit`, `none`; `none` only to loopback), `timeout` |
| `messages.dir` | `path` |

`messages/wire` builds the email transport from `messages.email.transport`,
so an app can switch between `smtp` and `dir` by configuration. SMS
transports take their own configuration, as an app's provider does.

## 9. auth

```go
package authmessages // github.com/epistemer/messages/authmessages, a nested module

// New returns an auth.Sender that sends auth's messages through m, with
// default texts for each auth.MessageKind. It refuses nil functions in links.
func New(m *messages.Messages, links Links, locale Locale) (auth.Sender, error)

// Links builds the URLs auth leaves to the app.
type Links struct {
	Recovery   func(ctx context.Context, token string) (string, error)
	Invitation func(ctx context.Context, token string) (string, error)
}

// Locale returns a user's language; "" means the default. It isn't called
// for a message without a user.
type Locale func(ctx context.Context, u auth.UserID) (string, error)
```

- Each of auth's eight message kinds gets a declared kind with English
  texts under `messages.auth.<kind>.*`, editable in the translations page.
- Codes, recovery links and invitations are `Secret` kinds (§5.4). Their
  `NotAfter` is auth's `Expires`.
- Notices (password changed, new device, factor added or removed) carry no
  secret and no expiry. They go through the outbox.
- The channel comes from `auth.Message.Channel`, and the address from
  `auth.Message.To`.
- Because `Send` doesn't wait for delivery, auth tells a person "a code
  was sent" before it is. A failed delivery reaches the `Recorder`, not the
  sign-in page. This is the same trade as any queued mail, and §11 Q1 is
  where to change it.
- Confab replaces its outbox with `authmessages` and the `dir` transport.

## 10. Out of v0.1

- Push notifications, which need a title, a payload and several devices
  per user, so they get their own design.
- Attachments.
- Bulk and marketing email.
- Inbound mail and bounce webhooks.
- DKIM signing, which is the relay's job.
- Templates edited in the app beyond their texts.
- More than one process (neozp D3).

## 11. Open questions, each with a default

| # | Question | Default |
|---|---|---|
| Q1 | Deliver through a loop, or synchronously inside `Send`? Synchronous gives a sign-in page immediate feedback; the loop keeps a slow relay out of the request | The loop (D4) |
| Q2 | Render at `Send` or at delivery? | At `Send` (D6) |
| Q3 | Keep secret messages in memory only, lost on a crash? The alternative is the outbox with mandatory authenticated encryption (a keyring, so the key can rotate) | Memory only (D5): nothing to encrypt, rotate or leak |
| Q4 | The outbox in its own bbolt file, or a bucket in the app's shared file with auth? | Its own file (D10): no write contention with auth, and no deadlock if an app sends inside its own transaction |
| Q5 | Ship `authmessages`, with default texts for auth's messages? | Yes |
| Q6 | HTML bodies through a `Render` interface that templ satisfies, with a text body always required? | Yes |
| Q7 | Push in v0.1? | No |
| Q8 | The declaration shape: a `Spec[D]` struct literal, generic only at declaration | Yes |
| Q9 | SMTP: the standard library's `net/smtp` with our own MIME, or a maintained library? | (Claude) Decided by spike S1 |

## 12. Spikes, before the interface is locked

- **S1. SMTP.** Send a text-and-HTML email with a stable `Message-ID` and
  an Arabic subject through a local relay (Mailpit in Docker), with
  STARTTLS and with implicit TLS. Do it once with `net/smtp` plus
  `mime/multipart`, and once with a maintained library. Try header
  injection through a name and a subject. Record the choice as an ADR.
- **S2. The loop.** Store, claim, deliver, retry and expire in bbolt, with
  per-channel workers and a test clock. Kill the process mid-delivery to
  confirm at least once, and confirm that a secret message never reaches
  the file. Measure `Send` inside a sign-in.
- **S3. auth in Confab.** `authmessages` with the `dir` transport replaces
  Confab's outbox, and a sign-in code arrives in the reader's language,
  right to left in Arabic.

## 13. Risks to check early

| Risk | Check |
|---|---|
| SMTP servers differ in TLS and authentication | S1 against Mailpit; a guide for common relays |
| A message sent twice after a crash | D15: at least once, with a stable `Message-ID` and an idempotency key for SMS |
| A secret message lost on a crash | D5, accepted: the person asks again |
| Right-to-left emails render wrongly in mail clients | The layout sets `dir` and `lang`; S3 checks it in Arabic |
| Codes leak through logs, files or errors | D5, D13, D16, and `dir`'s `0600` files |
