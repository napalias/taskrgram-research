# Telegram Web A audit

Audit target: <https://web.telegram.org/a/>  
Audit date: 2026-08-14  
Observed client version: `12.0.38 A`

This folder contains a passive architecture and UI/UX audit of Telegram Web A. The audit used normal browser interaction, public response inspection, and the public `Ajaxy/telegram-tt` repository. The repository package version matched the deployed version observed in runtime.

No messages were sent, no settings were changed, no access controls were bypassed, and no account-specific login codes are retained in these files.

## Files

- [report.md](./report.md) — executive report, evidence tables, maps, architecture diagram, findings, and recommendations.
- [features.md](./features.md) — feature inventory and interaction model.
- [tech-stack.md](./tech-stack.md) — framework, build, transport, storage, external services, and confidence.
- [ui-ux.md](./ui-ux.md) — design system, responsive behavior, reusable components, and likely rationale.
- [ui-ux-psychology.md](./ui-ux-psychology.md) — evidence-backed analysis of the psychological principles behind the observed desktop UI/UX decisions.
- [desktop-first.md](./desktop-first.md) — recommended desktop MVP, shell dimensions, keyboard model, and phased implementation.
- [architecture.md](./architecture.md) — runtime request flow, state, workers, caching, authentication, and deployment inference.
- [performance-accessibility.md](./performance-accessibility.md) — measured bundle/header observations, console checks, accessibility, SEO, and mobile quality.
- [evidence.md](./evidence.md) — compact observation log and methodology.
- [screenshots/](./screenshots/) — desktop UI evidence and a screenshot catalog. Some authenticated captures contain account-visible details; QR images are not retained because they encode temporary login tokens.

## Confidence scale

- **Confirmed** — directly visible in source, headers, requests, or runtime behavior.
- **Strong inference** — supported by multiple independent signals.
- **Possible** — plausible but not sufficiently verified.
- **Unknown** — cannot be determined from available access.

## Important interpretation boundary

The deployed client and public repository both reported version `12.0.38`, which makes the public source unusually strong corroboration. It still does not reveal Telegram's private server implementation, production configuration, data-center topology, observability, or deployment pipeline.
