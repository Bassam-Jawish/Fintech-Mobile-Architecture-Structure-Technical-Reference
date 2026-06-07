# Fintech Mobile Architecture — Structure & Technical Reference

> **Portfolio reference** — This repository documents how I design, structure, and scale production-grade fintech mobile applications. It reflects architectural decisions and engineering practices applied across real-world wallet, payments, and digital banking products. **No proprietary source code is published here.**

---

## Overview

Building a fintech app is not just about screens and API calls. It requires a codebase that can absorb regulatory change, support multiple brands and environments, enforce security by default, and stay maintainable as the feature surface grows from onboarding to transfers, bill payments, QR, vouchers, and merchant services.

This document captures the **technical foundation** I use when leading or contributing to fintech Flutter projects: layered architecture, feature isolation, shared component libraries, typed error handling, and operational concerns like OTA updates and crash reporting.

---

## Design Principles

| Principle | Rationale |
|-----------|-----------|
| **Feature-first modularity** | Each financial capability (auth, wallets, transfers, payments, etc.) is a self-contained vertical slice. Teams can ship independently without cross-feature coupling. |
| **Clean Architecture** | Domain logic stays framework-agnostic. UI and infrastructure are replaceable without rewriting business rules. |
| **Explicit data flow** | `Either<Failure, T>` at the use-case boundary — no silent failures in money-moving flows. |
| **Security by default** | Biometrics, secure storage, screenshot blocking, device integrity checks, and token refresh are infrastructure concerns, not per-screen afterthoughts. |
| **White-label readiness** | Shared UI kit + per-brand assets/config so one codebase serves consumer and business variants across dev/staging/production. |
| **Codegen over boilerplate** | Routing, DI, mappers, localization keys, and immutable models are generated to reduce human error in repetitive layers. |

---

**Dependency rule:** outer layers depend inward. Domain never imports Flutter, Dio, or platform SDKs.

---

## Project Structure

```
lib/
├── main.dart                    # App bootstrap, zone error handling, localization init
├── app.dart                     # Root widget, theme, router, global overlays
├── injection_container.dart     # Service locator entry point
│
├── core/                        # Cross-cutting infrastructure
│   # Reusable state/UI patterns (cubits, builders)
│   # Environment bindings, startup sequence
│   # Global enums, settings, feature flags
│   # Failure types and mapping
│   # Client, interceptors, connectivity
│   # Declarative routes + auth/session guards
│   # Logging, notifications, OTA, navigation helpers
│   # Secure prefs, local DB abstractions
│   # Base UseCase contract
│   # Validators, formatters, platform helpers
│
├── components/                  # Shared, brand-agnostic UI & utilities
│   # Design-system primitives (buttons, fields, receipts…)
│   # Theming
│   # Cross-feature services (maps, notifications…)
│   # Small reusable feature fragments
│
├── features/                    # Business verticals — one folder per capability
│   └── <feature_name>/
│       ├── data/
│       │   ├── data_sources/    # Remote / local API access
│       │   ├── models/          # Serializable DTOs + generated mappers
│       │   └── repositories/    # Repository implementations
│       ├── domain/
│       │   ├── entities/        # Pure business objects
│       │   ├── repositories/    # Abstract contracts
│       │   └── use_cases/       # Single-responsibility operations
│       ├── presentation/
│       │   ├── pages/           # Route-level screens
│       │   ├── widgets/         # Feature-specific UI
│       │   ├── cubits/          # Feature state (or managers)
│       │   └── utils/           # Presentation-only helpers
│       └── di/                  # Feature-scoped DI module
│
├── navigation/                  # Deep-linking, notification-driven routing
└── auto_generated/              # Codegen output (routes, assets, i18n keys)
```

### Typical feature domains in a fintech product

- **Identity & access** — onboarding, registration, login, OTP/PIN, device management, password policies
- **Account & profile** — KYC state, personal info, alias management, default account
- **Wallets & balances** — multi-wallet views, top-up, cash-in/out
- **Money movement** — P2P transfer, IBAN, request-to-pay, bulk transfers, fees preview
- **Payments & commerce** — QR, manual payment, billing aggregator, e-vouchers, donations
- **History & insights** — transaction ledger, filters, receipts, analytics dashboards
- **Platform** — notifications, settings, FAQ, policies, contact, about

Each domain follows the **same folder contract**, so onboarding a new engineer or adding a new payment rail is predictable.

---

## Layer Responsibilities

### Domain

- Defines **what** the app does in business terms.
- Use cases expose a single `call()` returning `Future<Either<Failure, T>>`.
- No JSON, no HTTP status codes, no `BuildContext`.
- Entities are immutable value objects; repository interfaces live here.

### Data

- Implements repository contracts.
- Maps API models ↔ domain entities via compile-time mappers.
- Splits **remote** and **local** data sources for cache-first or offline-tolerant reads where appropriate.
- Translates transport errors into domain `Failure` types at the boundary.

### Presentation

- **Pages** own navigation and layout composition.
- **Cubits** orchestrate use cases and emit loading / success / error states.
- **Widgets** stay as dumb as possible; money formatting and validation rules are centralized.
- Shared flows (OTP entry, payment summary, receipt, PIN verification) live in the components library to avoid duplication across features.

---

## State Management

I standardize on **flutter_bloc** with two core abstractions:

1. **`UseCaseCubit`** — wraps any `UseCase`, handles `UseCaseInitial → Loading → Loaded | Error` lifecycle, and keeps request params for retry.
2. **`StateBuilder`** — thin `BlocConsumer` wrapper for local/ephemeral UI state that does not warrant a full feature cubit.

This keeps async money flows consistent: every financial action has an explicit loading state, a typed success payload, and a mapped failure that the UI can render without parsing raw exceptions.

---

## Dependency Injection

**get_it + injectable** with per-feature `@module` classes:

- Singletons for repositories, HTTP client, storage.
- Factory methods for cubits (fresh instance per screen where needed).
- Codegen (`build_runner`) produces the registration graph — no manual service-locator maintenance at scale.

Feature modules register only their own dependencies. Core registers shared infrastructure once at startup.

---

## Networking

| Concern | Approach |
|---------|----------|
| HTTP client | Dio with layered interceptors |
| Authentication | Queued token-refresh interceptor — concurrent 401s await a single refresh, then replay |
| Observability | Structured logging interceptor (disabled in production builds) |
| Resilience | Connection checker, cache interceptor for idempotent reads |
| Error mapping | HTTP/business errors → `ServerFailure`, `BusinessFailure`, `UnauthorizedFailure`, etc. |

Business failures (e.g. insufficient balance, limit exceeded) are first-class types with server-provided conflict metadata — not generic toast messages.

---

## Navigation & Access Control

**auto_route** provides:

- Type-safe route definitions and deep links
- **Route guards** for session state (first-time user, authenticated, KYC complete)
- Nested stacks for multi-step financial wizards (amount → review → OTP → receipt)

Notification payloads and external links are normalized in a dedicated navigation layer so push taps land on the correct feature without tight coupling to Firebase handlers.

---

## Security & Compliance-Oriented Practices

Fintech apps handle credentials, balances, and PII. Infrastructure-level controls I bake in:

- **Secure storage** for tokens and sensitive prefs (flutter_secure_storage)
- **Biometric authentication** with change-detection hooks
- **Screenshot / screen-recording prevention** on sensitive screens
- **Device integrity** checks (root/jailbreak, emulator detection)
- **Certificate pinning** support via per-flavor asset bundles
- **Crypto primitives** for signing/encryption where the backend contract requires it
- **Session hygiene** — logout clears in-memory and persisted auth state via guards and storage abstractions

Security policies (password rules, counterparty limits, username constraints) are fetched from the backend and modeled as domain entities — not hardcoded strings.

---

## Local Persistence & Caching

- **ObjectBox** for structured offline entities (contacts, cached lists, draft state)
- **Hive-backed HTTP cache** for stable reference data
- **Shared preferences abstraction** for lightweight flags (language, theme, first-launch)
- Clear separation between *cache* (replaceable) and *source of truth* (server)

---

## Multi-Brand & Multi-Environment

Production fintech products rarely ship as a single binary. I structure for:

```
assets/
└── <brand>/
    ├── certs/          # TLS / pinning material
    ├── locales/        # en, ar, … (RTL-first where required)
    ├── images/
    ├── icons/
    ├── fonts/
    └── lotties/
```

**flutter_flavorizr** (or equivalent) drives:

- `dev` / `staging` / `production` environments per brand
- Distinct app IDs, display names, and API base URLs
- Consumer vs business app variants from one codebase

The `components/` theming layer consumes brand tokens so feature code stays brand-agnostic.

---

## Internationalization

- **easy_localization** with codegen loaders — no magic string keys in widgets
- Locale persisted across sessions; RTL layout tested at the design-system level
- Currency, date, and phone number formatting centralized in validators/utils
- Receipt and statement templates support bilingual output where regulations require it

---

## Code Generation Stack

| Tool | Purpose |
|------|---------|
| `injectable_generator` | DI registration |
| `auto_route_generator` | Typed routes |
| `freezed` | Immutable unions / copy-with for states and entities |
| `auto_mappr` | DTO ↔ entity mapping |
| `objectbox_generator` | Local DB schemas |
| `flutter_gen` | Asset and font references |

Codegen reduces drift between layers — especially critical when API schemas change frequently in active fintech products.

---

## Observability & Delivery

- **Firebase Crashlytics** inside a guarded zone — fatal errors recorded with stack traces
- **Firebase Cloud Messaging** + local notifications with channel abstraction
- **Shorebird** (or similar) for code-push of non-native fixes between store releases
- Structured app logging with environment-gated verbosity

---

## Shared UI Kit (Components Layer)

Financial UX repeats the same patterns: amount entry, OTP, payment review, transaction receipt, wallet cards, stepper onboarding, empty/error states.

I extract these into a **components library** inside the monorepo:

- Consistent money display (`AppMoneyText`, amount formatters)
- Transaction summary and receipt layouts
- PIN / OTP verification flows
- Payment code and QR widgets
- Shimmer/skeleton loading for ledger views
- Slide-to-confirm for irreversible actions

Features compose primitives; they do not rebuild payment UI from scratch.

---

## Error Handling Taxonomy

```text
Failure
├── ConnectionFailure          # No network
├── ServerFailure              # 5xx / unexpected server error
├── UnauthorizedFailure        # Session expired
├── BusinessFailure            # Domain rule violation (with server detail)
├── BadRequestFailure          # Validation / 400
├── NotFoundFailure
├── CacheFailure
├── ParsingFailure
└── ServerSecurityFailure      # Security policy rejection
```

Presentation maps failures to user-visible copy, retry actions, or forced re-authentication — never raw exception strings.

---

## Testing Strategy

| Layer | Focus |
|-------|-------|
| **Domain** | Use case unit tests with mocked repositories; assert `Either` outcomes |
| **Data** | Mapper tests, repository integration with mocked Dio |
| **Presentation** | Cubit state transitions; golden tests for critical financial screens |

Financial calculations, fee previews, and validation rules are tested in domain/presentation utils — not only on device.

---

## Scalability Considerations

- **40+ feature modules** can coexist because each respects the same `data / domain / presentation / di` contract.
- New payment rails (e.g. QR, NFC, external PSP webviews) plug in as new features without refactoring existing transfer code.
- Shared `core/` and `components/` prevent duplication while keeping feature boundaries strict.
- Barrel exports (`core_exports.dart`) reduce import noise without hiding layer boundaries from reviewers.

---

## Tech Stack Summary

| Category | Choices |
|----------|---------|
| Framework | Flutter (Dart 3.x) |
| Architecture | Clean Architecture + Feature-first modules |
| State | flutter_bloc (Cubit) |
| DI | get_it, injectable |
| Navigation | auto_route |
| Networking | Dio, dio_cache_interceptor |
| Functional helpers | fpdart (`Either`) |
| Immutability | freezed, equatable |
| Local DB | ObjectBox |
| Secure storage | flutter_secure_storage |
| i18n | easy_localization |
| Maps / charts | Syncfusion, Google Maps (where needed) |
| CI / quality | flutter_lints, build_runner pipelines |

---

## What This Repo Represents

This is an **architecture and engineering practices reference** for fintech mobile development — not an open-source product drop. It communicates:

- How I partition a large financial app into testable, replaceable layers
- How I enforce consistent patterns for money-moving operations
- How I prepare a single codebase for multiple brands, locales, and environments
- How I integrate security, compliance-oriented policies, and operational tooling from day one

If you are reviewing this for collaboration or hiring context, the emphasis is on **system design, maintainability, and production discipline** in regulated, high-stakes mobile domains.

---

## License & Confidentiality

Architecture patterns described here are shared for professional reference. **Source code, API contracts, and brand assets from commercial products are proprietary and not included in this publication.**
