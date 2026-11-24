# Status

## 2026-11-24

- epistemer created the repository and delegated ownership until
  `v1.0.0` on the terms of the other neozp packages
  ([ownership.md](ownership.md)).
- Wrote [brief.md](brief.md) from Confab's outbox, auth's `Sender` and zp's
  email, SMS and push code, read as a record of needs.
- Review 0001 (Gemini 4 Neon (High); Argon was cooling down): 17 findings,
  all answered. The main changes: secret messages never touch disk, the
  outbox has its own bbolt file, small configs per transport,
  `Undeliverable` apart from `Permanent`, and `Spec[D]` with `Channels`.
- Next: epistemer's answers to the brief's open questions (§11, M0.3), then
  spikes S1–S3.
