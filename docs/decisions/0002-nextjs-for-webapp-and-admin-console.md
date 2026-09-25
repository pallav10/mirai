# 0002 — Next.js for the patient/doctor webapp and the admin console

- **Status:** Accepted
- **Date:** 2026-08-29
- **Specs:** [01-design-system](../specs/01-design-system.md), [04-admin-app](../specs/04-admin-app.md)

## Context

Two web surfaces are required: a webapp mirroring the patient/doctor mobile experience, and the organization admin console. Spec 04 already proposes the admin console's production form factor as a responsive desktop-first web app. Spec 01 requires pixel-consistency of the Meridian design system across every surface, built by potentially different teams.

## Decision

Both web surfaces are **Next.js (React, TypeScript)** apps in the shared monorepo:

- **Two separate apps** — `apps/web` (patient/doctor) and `apps/admin` — with independent deploys and independent auth surfaces. The admin console never shares a session domain with the patient/doctor webapp.
- Both consume the shared `@atlas/ui` Meridian tokens-and-components package and the generated API clients (ADR 0011).
- Server-side rendering only where it earns its keep (public/onboarding pages); the record views are authenticated client-rendered flows against the API gateway — no PHI in build output, caches, or edge functions.

## Consequences

- Meridian is implemented exactly once for the web; admin and webapp cannot drift apart visually.
- One framework and one hiring profile across all web surfaces.
- The admin console's desktop-first layouts (spec 04 §1 "Proposed") become the canonical implementation; the mobile-width spec layouts are its small breakpoint.
- PHI-handling rules constrain Next.js features: no ISR/static generation of authenticated views, no third-party analytics on record pages, strict CSP.

## Alternatives rejected

- **React Native Web (single codebase with mobile):** tempting for the patient/doctor webapp, but produces a second-class web experience for content-heavy record views and drags the admin console into mobile-first constraints it doesn't have.
- **Separate SPA framework (Vue/Angular) for admin:** splits the design system and the hiring profile for zero benefit.
