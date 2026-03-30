# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What Is Meetly

Embeddable vanilla JS booking widget for scheduling consultant meetings. Multi-brand, bilingual (de/en). Deployed as a single `dist/widget.js` bundle that host pages include via `<script>` tag. See `docs/booking-widget-spec.md` for full spec, `docs/booking_flow.svg` for swim lane diagram.

## Commands

```bash
npm install      # install deps (only esbuild)
npm run build    # production bundle -> dist/widget.js (minified, no sourcemap)
npm run dev      # watch mode (unminified, inline sourcemap)
```

No test suite exists yet. No linter configured.

## Architecture

### Build

esbuild bundles `src/widget.js` as IIFE into `dist/widget.js`. CSS is imported as text (`loader: { '.css': 'text' }`) and injected at runtime. Single file output, zero runtime dependencies.

### 5-Step Booking Flow

1. **Select consultant** (step 1) -- loads consultant JSON from brand website or URL
2. **Pick date** (step 2) -- calendar UI, skips past dates and days with no schedule
3. **Pick time** (step 3) -- slots computed client-side from consultant `schedule` JSON, then async-verified against app2gcal backend for booked conflicts
4. **Visitor details** (step 4) -- name, email, company, topic (required), GDPR consent, honeypot
5. **Confirmation** (step 5) -- ICS download, email confirmations sent

### Module Responsibilities

| File | Role |
|------|------|
| `widget.js` | Entry point. Registers `window.SimplifyBooking` API, manages state machine, modal lifecycle |
| `ui.js` | Pure DOM construction via `el()` helper. Renders each step. Generates ICS content |
| `availability.js` | Client-side slot computation from consultant `schedule` + `exceptions` JSON. Merges with backend booked-slot data |
| `api.js` | HTTP layer. Fetches consultant JSON per-brand, talks to app2gcal backend for availability/bookings |
| `email.js` | EmailJS REST API integration (no SDK). Sends visitor confirmation + consultant notification |
| `context.js` | Auto-collects page URL, referrer, UTM params, `data-context-*` attrs for attribution |
| `i18n.js` | All user-visible strings keyed by `de`/`en` |
| `theme.js` | Detects host page CSS variables for visual integration |
| `styles.css` | Widget styles, all classes prefixed `sb-` |

### External Dependencies

- **app2gcal** (sibling repo `../app2gcal/`): Google Calendar backend. Widget calls its `/api/v1/availability` and `/api/v1/bookings` endpoints. Widget works without it (fallback: EmailJS-only booking).
- **EmailJS**: REST API for email delivery. Config via `data-emailjs-*` attributes on script tag. Docs in `docs/emailjs_setup.md`.
- **Brand websites**: Host consultant JSON files. Widget fetches `{consultantsBase}/{id}.json` per consultant.

### Deployment

Kubernetes manifests in `k8s/` (deployment, service, ingress). Widget JS is served as a static file.

### Configuration (Script Tag Data Attributes)

Key config read from `data-*` attributes on the `<script>` tag:
- `data-api` -- app2gcal backend URL
- `data-consultants-base` + `data-consultants` -- per-brand consultant loading
- `data-consultants-url` -- legacy: single JSON URL with all consultants
- `data-lang` -- `de` (default) or `en`
- `data-brand` -- brand name for context/emails
- `data-trigger="floating"` -- adds floating CTA button
- `data-emailjs-*` -- EmailJS service/template/key overrides

### Public JS API

```
window.SimplifyBooking.open({ consultant, topic, context, consultantsUrl, lang })
window.SimplifyBooking.close()
window.SimplifyBooking.on('booking:confirmed', callback)
window.SimplifyBooking.on('booking:started', callback)
window.SimplifyBooking.on('widget:closed', callback)
```

Host pages can also use `data-booking-trigger` attribute on any element to auto-bind click-to-open.

## Rules

- Vanilla JS only. No frameworks, no runtime deps.
- All CSS classes prefixed with `sb-`.
- All user-visible strings in `src/i18n.js` (de + en).
- Every HTML element gets a unique `id` attribute.
- Build produces single `dist/widget.js` with CSS inlined.
- Consultant JSON files hosted per-brand, not in this repo.
- Emails sent via EmailJS from widget, not from app2gcal backend.
