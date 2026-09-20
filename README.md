# RaceOS

**Live application: [https://raceosapp.com/](https://raceosapp.com/)**

RaceOS is a production B2C application that generates complete race-execution plans for Ironman and 70.3 triathletes: discipline-by-discipline pacing targets, an aid-station nutrition timeline, and transition and special-needs bag contents, delivered as a printable race card. It is deployed, payment-enabled, and used by real athletes.

This repository is the unified root of the project and contains both halves of the system:

```
RaceOS/
├── frontend/     React + TypeScript client application
├── backend/      Data layer, edge functions, and generation services
└── README.md     This document
```

Each of the two folders carries its own README with component-level documentation, setup instructions, and implementation detail. This root document describes the system as a whole: what problem it solves, how it is structured, and which engineering decisions shaped it.

---

## 1. Problem and Motivation

Long-course triathlon preparation is a planning problem that athletes currently solve by hand. Pacing calculators, nutrition spreadsheets, and paper bag checklists live in separate tools, and the resulting plan is neither internally consistent nor responsive to conditions. The most frequently asked question in triathlon communities, which items belong in the T2 bag versus the special-needs bag, has no intelligent answer in any existing product, because it depends jointly on the athlete's goal time, planned nutrition, and race-day weather.

RaceOS treats the whole of race execution as a single dependent planning problem rather than four disconnected calculators. Pacing determines energy expenditure; energy expenditure determines nutrition requirements; nutrition requirements plus forecast conditions determine what must be physically placed in each bag at each point on the course. Each stage constrains the next, which is what makes the problem interesting technically and what makes the output useful practically.

---

## 2. System Architecture

RaceOS is a three-tier system with a deliberate separation between generative and deterministic components.

```
Client (React + TypeScript + Vite)
   │
   │  authenticated requests, row-level-scoped
   ▼
Supabase Postgres  <--  Row Level Security on every table
   │
   ▼
Edge Functions (server-side only)
   ├── generation service   ->  Anthropic Claude API (3 sequential calls)
   ├── validation layer     ->  deterministic physiological range checks
   ├── payment service      ->  Stripe (intents, webhooks, idempotency)
   └── weather service      ->  OpenWeatherMap, cached, key held server-side
```

**Frontend.** React 18 with TypeScript and Vite, Tailwind for styling against a fixed design-token set, and a seven-step plan-builder wizard with draft persistence and a live summary panel that recomputes as inputs change. Course visualisation uses Leaflet with OpenStreetMap tiles; elevation profiles are rendered from parsed GPX and cross-linked to the map so that hovering a point on the elevation chart highlights the corresponding course position. Race cards are rendered to PDF client-side with `@react-pdf/renderer`, which removes any dependency on a hosted rendering service.

**Data layer.** Supabase Postgres with Row Level Security enforced on every table from the first deployment rather than retrofitted. Plans, athlete profiles, purchases, subscriptions, and notifications are all user-scoped by policy at the database level, with an explicit additional policy permitting token-based read access for the coach-sharing flow. Generated artefacts are stored as validated JSON columns, plans are soft-deleted so that purchase and financial history remains intact, and the schema includes tables for roadmap features that are not yet surfaced in the UI so that later releases require no breaking migration.

**Service layer.** All third-party calls run in edge functions. No API key for Anthropic, Stripe, or OpenWeatherMap is ever exposed to the client. Every write is validated server-side against the same schema definitions used client-side, on the principle that client validation is a usability feature and server validation is the actual control.

---

## 3. The Generation Pipeline

The core of the system is a dependent three-stage pipeline in which each stage consumes the validated output of the previous one.

| Stage | Inputs | Output |
|---|---|---|
| 1. Pacing | Race distance, course profile from GPX, functional threshold power, swim critical speed, run threshold pace, body mass, goal, risk tolerance | Target pace and power per discipline, expected splits, zone distribution, execution cues |
| 2. Nutrition | Pacing output, sweat rate, gut tolerance, carbohydrate preference, aid-station positions | Hourly carbohydrate and fluid targets, aid-station-by-aid-station action table |
| 3. Bag contents | Nutrition output, weather snapshot, goal time, first-time-athlete flag | Categorised T1, T2, and special-needs contents with a justification per item |

Two design decisions define this pipeline and are the parts of the project I would most want to discuss in a technical conversation.

**The language model does not decide anything that can be checked.** Claude is used for structured generation and for explanatory text. Every numeric output it produces passes through a deterministic validation layer before it reaches the user: paces, powers, calorie rates, and fluid volumes are checked against physiological ranges, and implausible goal times are cross-validated against the athlete's declared fitness data. When validation fails, the system surfaces a warning and conservative targets rather than silently presenting a fabricated number. Generation failures never fall back to invented data; they fail visibly and offer a retry. In a domain where a wrong number is acted on for eleven hours under physical stress, the distinction between "the model produced it" and "the system can stand behind it" is the whole product.

**Bag contents are computed from constraints, not prompted.** The rule layer governing what goes into each bag is deterministic and inspectable: forecast temperature above a threshold adds electrolyte and sun-protection items; an aggressive goal time combined with declared gut tolerance restricts solid food on the run leg; a first-time athlete with an aggressive risk setting is silently moved to a moderate profile with an explanatory note rather than being blocked. The model writes the justification text; the rules decide the contents. This makes the output reproducible and auditable, which a purely prompt-driven implementation would not be.

Stage dependency is enforced in the application logic as well. Editing pacing after nutrition and bag contents have been generated prompts regeneration of everything downstream rather than leaving stale, mutually inconsistent sections in a plan that an athlete will trust on race morning.

---

## 4. Engineering Decisions

| Decision | Reasoning |
|---|---|
| Sequential dependent generation rather than one large prompt | Each stage is independently inspectable, cacheable, and re-runnable. A failure or an edit affects one stage and its descendants, not the entire plan. |
| Deterministic validation layer over model output | Physiological plausibility is a checkable property. Anything checkable should be checked rather than trusted. |
| Row Level Security from the first deployment | Authorisation expressed as database policy rather than application code cannot be bypassed by a missing check in a new endpoint. |
| All third-party calls server-side | Keys stay out of the browser, and rate limiting and idempotency are enforced at a single point. |
| Leaflet with OpenStreetMap instead of a keyed map provider | No API key, no request quota, no vendor account in the critical path. A richer provider remains a documented drop-in upgrade. |
| Client-side PDF rendering | Removes a hosted rendering dependency and keeps generation cost proportional to usage rather than fixed. |
| Soft deletion on plans | Financial and purchase records must survive user-initiated deletion. |
| Schema includes not-yet-built features | Future releases become additive rather than breaking migrations. |
| Free-tier-first infrastructure | Fixed monthly cost is approximately zero; the only two costs, model inference and payment processing, are proportional to actual usage. |

---

## 5. Payments, Access Control, and Deployment

Access is tier-based, with tier resolved from a single source of truth in the database and consumed through one centralised hook so that gated interface states stay consistent. Purchases are made through Stripe with idempotency keys on every attempt, and entitlement is granted by webhook rather than by client-side confirmation. A purchased plan is permanently accessible, including when the underlying race is later cancelled. Subscription tiers bypass the transactional checkout path entirely through a separate entitlement check.

The application is deployed at [https://raceosapp.com/](https://raceosapp.com/) with environment-separated configuration, database migrations committed to the repository, seeded reference data for major events so that search returns real results from first use, and accessibility and Core Web Vitals targets (WCAG 2.1 AA, LCP under 2.5s, CLS under 0.1) treated as build requirements rather than post-launch cleanup.

---

## 6. Practical Impact

RaceOS replaces four to five disconnected tools with one coherent artefact, and it answers the bag-packing question that no competing product addresses, because answering it requires reasoning across pacing, nutrition, and forecast conditions simultaneously. It is monetised transactionally per plan, with season and coach tiers, and it runs at near-zero fixed infrastructure cost, which means the system is economically viable at small scale rather than dependent on volume it does not yet have.

For a reader assessing this repository as engineering work rather than as a product: the aspects I consider most substantive are the dependent generation pipeline with enforced downstream invalidation, the deterministic validation boundary placed around a probabilistic component, the constraint-driven rule layer for bag contents, and database-level authorisation applied uniformly from the first deployment.

---

## 7. Repository Contents

| Path | Contents |
|---|---|
| `frontend/` | React client: routing, design system, plan-builder wizard, map and elevation visualisation, PDF templates, state and permission hooks. See the README in that folder. |
| `backend/` | Schema and migrations, Row Level Security policies, edge functions for generation, validation, payments and weather, prompt construction, and seed data. See the README in that folder. |

Both folder-level READMEs document local setup, required environment variables, and the internal structure of their respective halves in more detail than is appropriate here.

---

## 8. Technology Summary

**Frontend:** React, TypeScript, Vite, Tailwind CSS, Leaflet, Recharts, @react-pdf/renderer
**Backend and data:** Supabase (Postgres, Auth, Storage, Edge Functions), Row Level Security, SQL migrations
**AI and services:** Anthropic Claude API, deterministic validation layer, OpenWeatherMap, Stripe
**Practices:** Shared client and server schema validation, structured error handling, soft deletion, idempotent payment handling, seeded reference data, accessibility and performance budgets

---

*RaceOS is an independently designed, built, and deployed project. The live application is available at [https://raceosapp.com/](https://raceosapp.com/).*
