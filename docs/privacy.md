# Privacy & Telemetry

This document describes exactly what data wirego can send to its
maintainer ([vernizus](https://github.com/vernizus)), under what
circumstances, and what it never sends. It covers the **application
software itself** — not the VPN traffic of the peers connected to a
wirego instance, which is a separate concern addressed at the end.

If anything here is unclear or you believe the software behaves
differently from what's described, please open an issue — see
[Contact](#contact).

## Scope: this is about the software, not your VPN traffic

wirego's telemetry exists to help the maintainer find bugs and understand
which features are actually used — nothing here inspects, logs, or
transmits the network traffic of the peers connected through your VPN.
DNS queries, connection flows, and audit events stay in your own local
SQLite database, on your own server, and are never part of any telemetry
payload described below.

## Two separate consent mechanisms

wirego has two independent ways data can reach the maintainer. They are
not related, and enabling or disabling one has no effect on the other.

### 1. Automatic telemetry — opt-in, off by default

A toggle in the web panel (**System → Settings**), disabled by default.
While enabled, wirego periodically sends, in the background:

- **Crash/error reports** — triggered by a Go panic in the backend or an
  unhandled error in the browser. Contains: an error-type code, an error
  message, a stack trace, the app version, and the OS/architecture
  (e.g. `linux/amd64`).
- **Usage metrics** — a count of how often certain built-in event types
  occur (e.g. "a peer was created", "a login failed"), batched and sent
  periodically. Each metric is just an event name and a number — never
  which peer, which user, or any other detail about the event.

Both include an anonymous, randomly generated installation ID, used only
to group repeated reports from the same instance together — it identifies
an install, not a person.

**Disabling the toggle stops every automatic network call immediately** —
not just future reporting; no new report is even constructed once the
toggle is off.

### 2. Voluntary support report — always available

The **Support** page in the panel lets you send a bug report or
suggestion at any time, whether or not automatic telemetry is enabled.
Sending one is itself your explicit, one-time consent for that specific
report. It can include:

- A category and a free-text description you write.
- An optional contact email, if you want a reply.
- An optional screenshot.

Unlike automatic telemetry, your typed description and contact email are
sent as you wrote them (the whole point is that you're deliberately
providing that information to get help) — the automatic-sanitization
rules below don't strip your own contact email.

## What's sanitized before anything is sent

Every free-text field in an *automatic* crash report (error message, stack
trace) is scanned and scrubbed before it ever leaves your server:
IPv4/IPv6 addresses, email addresses, Windows and Unix file paths, PEM
private-key blocks, WireGuard-shaped keys, and JWTs are all replaced with
a redacted placeholder. This runs even if the underlying error message
happened to embed one of these by accident.

## What's never sent, under any circumstance

- Real IP addresses of your peers or your server (see the one exception
  for country-level location, below).
- WireGuard public or private keys, or any other peer secret.
- Peer names, group names, or any part of your network topology.
- Local file paths.
- Anything from your VPN traffic itself (DNS queries, connection flows,
  audit log entries) — those never leave your server for this purpose.

## About IP addresses and location

Any outbound network connection — from wirego to the telemetry collector,
same as from any other server on the internet to any other service —
inherently reveals the connecting server's IP address to whatever it
connects to, at the network level. This isn't specific to wirego; it's
true of any HTTPS request.

Beyond that unavoidable fact, the maintainer's telemetry collector
resolves that IP to a **country-level location** (e.g. "ES", "DE") for
aggregate statistics — never a city, never a precise location. **The IP
address itself is never stored** — it exists only in the memory of the
single request that resolves it, then is discarded. Only the resulting
two-letter country code is kept, tied to the anonymous install ID
described above.

## Where the data goes

The destination is a telemetry collector the maintainer operates and
hosts. As the person self-hosting wirego, **you never configure or see
this destination** — there is no setting, environment variable, or API
response that exposes it. It's baked into the official release builds at
compile time, and a build made from source without it configured simply
sends nothing at all (every telemetry function becomes a silent no-op).

Your browser never talks to the collector directly — every report is
relayed through your own wirego server first, authenticated the same way
as any other action in the panel.

## How long data is kept

Reports and metrics are automatically purged **90 days** after they're
received. Stale installation records are purged on the same schedule.

## Disabling telemetry entirely

1. Go to **System → Settings** in the panel.
2. Turn off automatic telemetry.

That's it — no restart needed, and it takes effect immediately. You can
still use the Support page to send a one-off report afterward if you
choose to; that's independent of this toggle, by design.

If you build wirego from source yourself without the maintainer's
official build pipeline, telemetry has no destination configured and
never sends anything, regardless of the toggle.

## Contact

Questions about this document, or about how your data is handled:

- [GitHub Issues](https://github.com/vernizus/wirego/issues), or
- alejandro.fernandes@vernizusluna.com
