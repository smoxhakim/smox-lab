# gstack

Use the `/browse` skill from gstack for all web browsing. Never use `mcp__claude-in-chrome__*` tools.

Available gstack skills:

- `/office-hours`
- `/plan-ceo-review`
- `/plan-eng-review`
- `/plan-design-review`
- `/design-consultation`
- `/design-shotgun`
- `/design-html`
- `/review`
- `/ship`
- `/land-and-deploy`
- `/canary`
- `/benchmark`
- `/browse`
- `/connect-chrome`
- `/qa`
- `/qa-only`
- `/design-review`
- `/setup-browser-cookies`
- `/setup-deploy`
- `/setup-gbrain`
- `/retro`
- `/investigate`
- `/document-release`
- `/codex`
- `/cso`
- `/autoplan`
- `/plan-devex-review`
- `/devex-review`
- `/careful`
- `/freeze`
- `/guard`
- `/unfreeze`
- `/gstack-upgrade`
- `/learn`

---

# Smox Lab — Project Context

## What we're building
Smox Lab is an immersive retail platform for custom-printed apparel. Customers walk into a physical store, design apparel on a touchscreen kiosk, preview it in AR on their own body, confirm the order, wait in a physical Gaming Lounge while it prints, then pay at the cashier on pickup. The web platform handles the marketplace, creator accounts, and pre-design from home.

**Source-of-truth documents (always read these before planning any feature):**
- `/docs/SmoxLab-PRD.md` — product requirements, features, personas, success metrics
- `/docs/SmoxLab-SAD.md` — system architecture, microservices, database, infra
- `/docs/SmoxLab-AI-Prompt-Architecture.md` — AI pipeline, prompt templates, moderation

## Team & scope
- Solo frontend developer for v1. Backend services in the SAD are out-of-scope unless explicitly scoped in.
- Target markets at launch: Morocco, France, UAE.
- v1 is in-store + web. No mobile app. No shipping. No multi-store sync.

## Tech stack (frontend focus)
- **Framework**: Next.js 15 (App Router), React 19, TypeScript strict mode
- **Styling**: Tailwind CSS, Framer Motion for transitions
- **State**: Zustand (no Redux)
- **3D & AR**: Three.js + @react-three/fiber + @react-three/drei
- **Pose detection**: MediaPipe Holistic (primary), TensorFlow.js (fallback)
- **Design canvas**: Konva.js (preferred) or Fabric.js
- **Real-time**: Socket.io client
- **i18n**: Arabic (RTL), French, English — multilingual from day one
- **PWA**: Service Worker + IndexedDB for kiosk offline mode

## Monorepo layout (Turborepo)
```
apps/
├── kiosk/          → In-store touchscreen app (locked-down, fullscreen, PWA)
├── web/            → Public web platform (marketplace, accounts, creators)
└── queue-display/  → Fullscreen status board for the Gaming Lounge TV
libs/
├── ui/             → Shared component library + design tokens
├── i18n/           → Translation setup (AR/FR/EN, RTL support)
└── types/          → Shared TypeScript types matching backend DTOs
docs/               → PRD, SAD, AI Prompt Architecture
```

## Non-negotiable requirements
- Kiosk UI must render at ≥ 60 fps; AR overlay at ≥ 24 fps minimum
- Prayer Pause system is P0 for Morocco market — kiosk must respect prayer times
- No payment at kiosk — payment happens at the cashier counter on pickup
- Kiosk must work in degraded mode offline (no AI, no AR cloud sync, but core flow works)
- WCAG 2.1 AA accessibility on the web platform
- No PII in logs anywhere
- AR photos auto-delete after 24h unless customer opts in

## Build order (current plan)
1. **Foundation** — monorepo, design tokens, i18n with Arabic RTL, kiosk shell
2. **Kiosk core flow** — product viewer (3D), configurator, design canvas, AR try-on, order confirmation, ticket
3. **Real-time layer** — Socket.io client, queue-display page, prayer pause overlay
4. **AI integration (frontend)** — prompt UI, style profiles, design options gallery, credit counter
5. **Web platform** — marketplace, creator dashboard, accounts, pre-design with QR handoff
6. **Hardening** — QA, perf tuning, security review, docs sync

## Feature workflow (per feature)
Run the full gstack loop for any feature > ~200 lines or anything user-facing:
1. `/office-hours` — challenge the framing
2. `/plan-ceo-review` — is the scope right?
3. `/plan-eng-review` — lock the architecture
4. Build
5. `/design-review` — UI quality 0–10
6. `/review` — code + security audit
7. `/qa` — real browser regression
8. `/ship` — commit, PR, auto-update docs

Skip the heavy ceremony for trivial changes (button color, copy tweaks).

## Lessons learned
_This section grows as the project progresses. Add findings here whenever gstack pushes back on an assumption or QA catches something._