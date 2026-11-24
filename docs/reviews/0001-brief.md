# Review 0001: the brief

2026-11-24. Reviewer: Gemini 4 Neon (High), through agy, about 37k tokens.
Gemini 4 Argon was cooling down. The reviewer had the brief, plus excerpts
of auth's `Sender` contract, i18n's context API and neozp's preferences.
17 findings. Each is answered below, and the brief was revised the same day.

| # | Severity | Finding | Answer |
|---|---|---|---|
| 1 | critical | The outbox stores codes and link tokens on disk. bbolt keeps deleted pages in the file, so they persist, and the brief had no key rotation | Accepted. Secret kinds stay in memory only (D5, §5.4); Q3 asks whether to encrypt instead |
| 2 | critical | Sharing auth's bbolt file means write contention, and a deadlock if an app sends inside its own write transaction | Accepted. The outbox has its own file (D10, Q4), and `Send` documents the rule |
| 3 | critical | `New` took both built transports and the transports' configuration; a 12-getter `Config` | Accepted. Small configs per part, `WithTransport` per channel, and `messages/wire` for selecting a transport by configuration (D11, §8) |
| 4 | critical | `Permanent` conflated "address gone" with "bad credentials", so apps would drop valid addresses | Accepted. `Undeliverable` is separate, with `Is…` helpers (§5.2, §5.3) |
| 5 | critical | "A missing translation fails `Send`" contradicts i18n, which falls back to the code's text; also a nil translator and an empty locale | Accepted. D6 is corrected, `New` refuses a nil translator, and an empty locale means the default |
| 6 | critical | Header injection through names and subjects; `ParseAddress` accepts display-name forms; `tls: none` with credentials | Accepted (D12, §5.2 `messages/smtp`) |
| 7 | major | `authmessages`: zero `Expires` on notices, a locale lookup that can't fail, nil links, links without a context, and the queued-send behaviour on sign-in | Accepted (§9). The sign-in behaviour is stated, and Q1 is where to change it |
| 8 | major | The loop: strict order versus concurrency, double dispatch, retry age versus `NotBefore`, attempts before or after `Deliver`, timeouts, shutdown, head-of-line blocking across channels, unreadable records | Accepted (§5.3) |
| 9 | major | `Status` returned no kind or times, and records grew forever | Accepted. `Record`, with retention (§5.1, §8) |
| 10 | major | Kind and channel rules contradicted each other; a global registry; `Part[D]` generics; the HTML function had no context; the core imported templ; push doesn't fit | Accepted. `Spec[D]` with `Channels`, no global registry, HTML through a `Renderer` interface, and push out with no promise (§4, §10) |
| 11 | major | `Recorder` was unspecified | Accepted (§5.5) |
| 12 | major | `Envelope` had no `From` and no `Message-ID` domain | Accepted. Both are fixed at `Send` (§5.2), and the domain is configured (§8) |
| 13 | major | `messagestest.Flush` couldn't reach the loop, and time-based behaviour was untestable | Accepted. `Messages.Flush`, `WithClock` and `messagestest.Clock` |
| 14 | major | The layout hard-coded `branding.title`, and i18n has no text-direction API | Half. The app now passes the title in. i18n's `Locale` does carry `RTL` (interface §6), so direction needs nothing new |
| 15 | minor | Transport errors and `dir` files can leak codes; masking was undefined; a wrong cross-reference | Accepted (D13, §5.2) |
| 16 | minor | `Message-ID` doesn't make SMS idempotent | Accepted. The envelope id is an idempotency key for providers (D15) |
| 17 | minor | No cancel, no size limit, no E.164 rule, and `Run`'s lifecycle was undefined; some settled items belong in §11 | Accepted. `Cancel`, `max_size`, the E.164 pattern and `Run`'s rules are added. Storage moved to Q4, and the declaration shape stays Q8. The logger keeps the siblings' default, `slog.Default()` |
