<div align="center">

# senior-mode

**AI-powered skill prompts that turn Claude Code into a production-hardened tech lead for your stack.**

*Distilled from real-world production incidents, senior engineer code reviews, and years of shipping software that actually holds up.*

---

[![License: MIT](https://img.shields.io/badge/License-MIT-blue.svg)](./LICENSE)
[![Skills](https://img.shields.io/badge/Skills-30%2B-green.svg)](#structure)
[![Stacks](https://img.shields.io/badge/Stacks-10%2B-orange.svg)](#version-baselines)
[![Contributions Welcome](https://img.shields.io/badge/Contributions-Welcome-brightgreen.svg)](#contributing)

</div>

---

## What is this?

This repository is a collection of **structured skill prompts** for Claude Code (and other AI coding assistants). Each `.md` file is a complete behavioral contract that tells the AI how to think, what to enforce, and what to reject — for a specific technology in a production context.

These are not tutorials. They are not generic checklists. They are **opinionated, version-specific instructions** written at senior engineer level, covering:

- The latest breaking changes and version-specific patterns
- Domain-first file organization (enforced, not suggested)
- N+1 detection, security patterns, and performance requirements
- Exact anti-patterns to catch and refuse
- Deployment gates that must pass before shipping

> **Origin**: The knowledge in these skills comes from real production codebases, post-mortems, security audits, and the accumulated experience of senior engineers across different stacks. Every rule here exists because someone, somewhere, got burned by ignoring it.

---

## The Philosophy

```
Generic AI advice  →  "use best practices"
These skills       →  "reject response()->json($order) — use OrderResource,
                        and here is exactly why and what to use instead"
```

The difference is specificity. A skill prompt that says *"be secure"* is useless. A skill prompt that says *"every endpoint accepting an ID must call `$this->authorize('view', $model)` — scope queries with `.where('user_id', auth()->id())` — never trust route model binding alone for ownership"* — that is enforceable.

**Claude reads the skill, internalizes the role, and acts as that engineer for the duration of your session.**

---

## How to Use

### Option 1 — CLAUDE.md (Recommended for projects)

Create a `CLAUDE.md` at your project root and paste in the relevant skill(s). Claude Code loads this file automatically on every session — no repetition needed.

```bash
# Example: Laravel API project
cat backend/laravel/api.md backend/laravel/structure.md security/api-security.md > your-project/CLAUDE.md
```

### Option 2 — Inline at session start

Paste the skill content at the beginning of a Claude Code conversation before any coding task. Works well for one-off reviews or when working across multiple projects.

### Option 3 — Stack multiple skills

Skills are designed to be composable. Common combinations:

```
# Full-stack Laravel + React
backend/laravel/api.md
backend/laravel/structure.md
backend/laravel/inertia-react.md
frontend/react/spa.md
security/api-security.md

# Go microservice
backend/go/gin.md
security/api-security.md
devops/docker/compose.md
devops/ci-cd/github-actions.md

# Next.js with security hardening
frontend/nextjs/app-router.md
security/owasp-top10.md
devops/nginx/config.md
```

### Option 4 — Per-task enforcement

Reference a specific skill when asking Claude to do a targeted task:

```
"Using the rules in backend/laravel/419-csrf.md, diagnose why this form returns 419"
"Apply backend/laravel/livewire.md standards to review this component"
```

---

## Structure

```
production-ready-checklist/
│
├── backend/
│   │
│   ├── laravel/                          8 skills — the most comprehensive section
│   │   ├── api.md                        Laravel 11 REST API
│   │   │                                 Versioning (/v1), single-action controllers,
│   │   │                                 Resources (no json()), enums, N+1 strict mode,
│   │   │                                 DB constraints, rate limiting, Actions layer
│   │   │
│   │   ├── structure.md                  Domain-first folder architecture
│   │   │                                 Full app/ layout, naming conventions table,
│   │   │                                 routes/api/v1.php separation, AppServiceProvider setup
│   │   │
│   │   ├── blade.md                      Blade + server-rendered Laravel
│   │   │                                 XSS (@csrf vs {!! !!}), CSRF on every form,
│   │   │                                 Vite assets, component architecture
│   │   │
│   │   ├── inertia-react.md              Inertia v2 + React 19
│   │   │                                 Deferred props, WhenVisible skeleton pattern,
│   │   │                                 lowercase render paths, useForm, TypeScript types
│   │   │
│   │   ├── inertia-vue.md                Inertia v2 + Vue 3.5
│   │   │                                 Deferred props, WhenVisible, script setup,
│   │   │                                 defineProps<T>(), Tailwind v4 components
│   │   │
│   │   ├── livewire.md                   Livewire v3
│   │   │                                 #[Locked] properties, #[Url] state, auth in
│   │   │                                 every action, public property security model
│   │   │
│   │   ├── queues-jobs.md                Laravel Queues & Jobs
│   │   │                                 Idempotency guards, retry/backoff, failed()
│   │   │                                 handler, Horizon, job batching, Supervisor config
│   │   │
│   │   └── 419-csrf.md                   419 Page Expired — complete reference
│   │                                     Every scenario mapped to its exact fix:
│   │                                     Blade / Inertia / SPA+Sanctum / API routes /
│   │                                     proxy config / session timeout / AJAX
│   │
│   ├── go/
│   │   ├── gin.md                        Go 1.22 + Gin 1.10
│   │   │                                 Typed domain enums, slog structured logging,
│   │   │                                 central error mapping, graceful shutdown,
│   │   │                                 context propagation, env validation at startup
│   │   │
│   │   └── echo.md                       Go 1.22 + Echo v4.12
│   │                                     Custom validator, global error handler,
│   │                                     binder pattern, domain error → HTTPError mapping
│   │
│   ├── java/spring-boot/
│   │   └── rest-api.md                   Spring Boot 3.3 + Java 21
│   │                                     Virtual threads, records for DTOs, sealed classes,
│   │                                     Security 6, open-in-view=false, EntityGraph,
│   │                                     @ControllerAdvice, HikariCP config
│   │
│   ├── ruby/rails/
│   │   └── api.md                        Rails 7.2 + Ruby 3.3 (API-only)
│   │                                     Pundit scopes (verify_policy_scoped),
│   │                                     service objects, Alba serializers,
│   │                                     Bullet gem, Rack::Attack rate limiting
│   │
│   ├── dotnet/aspnet-core/
│   │   └── rest-api.md                   ASP.NET Core 8 + C# 12
│   │                                     MediatR CQRS, primary constructors,
│   │                                     FluentValidation, rate limiting (.NET 7+),
│   │                                     nullable reference types enforced
│   │
│   ├── python/fastapi/
│   │   └── rest-api.md                   FastAPI 0.115 + Python 3.12 + Pydantic v2
│   │                                     Async SQLAlchemy 2.0, StrEnum, selectinload,
│   │                                     Pydantic Settings, type aliases, Gunicorn+Uvicorn
│   │
│   ├── rust/axum/
│   │   └── rest-api.md                   Axum 0.7 + Rust 1.80 + SQLx 0.8
│   │                                     Typed AppError with thiserror, no .unwrap(),
│   │                                     compile-time SQL queries, graceful shutdown,
│   │                                     custom extractors, serde deny_unknown_fields
│   │
│   ├── node-express/
│   │   ├── rest-api.md                   Node.js + Express + TypeScript
│   │   │                                 Zod validation, asyncHandler wrapper,
│   │   │                                 central error middleware, pino logger,
│   │   │                                 helmet, env validation at startup
│   │   │
│   │   └── typescript.md                 TypeScript strict configuration
│   │                                     tsconfig strict mode, zero any, zero @ts-ignore,
│   │                                     CI typecheck gate
│   │
│   └── django/
│       └── rest-framework.md             Django REST Framework
│                                         Explicit serializers (no fields='__all__'),
│                                         ViewSet permissions, select_related/prefetch,
│                                         manage.py check --deploy gates
│
├── frontend/
│   │
│   ├── react/
│   │   └── spa.md                        React 19 + TypeScript 5.5 + Tailwind v4
│   │                                     useActionState, useOptimistic, ref-as-prop,
│   │                                     skeleton loading components, all routes lazy,
│   │                                     cva for variants, TanStack Query v5, domain
│   │                                     folder structure: pages/orders/{index,show,create}
│   │
│   ├── vue/
│   │   └── vue3.md                       Vue 3.5 + TypeScript 5.5 + Tailwind v4
│   │                                     Reactive props destructuring, useTemplateRef,
│   │                                     defineModel, WhenVisible pattern, lazy routes,
│   │                                     skeleton components, Pinia composables
│   │
│   └── nextjs/
│       └── app-router.md                 Next.js 15 + React 19 + Tailwind v4
│                                         Async params/searchParams (breaking change),
│                                         loading.tsx skeleton standard, Server Actions
│                                         with Zod + auth, dynamic() for heavy components,
│                                         no-cache-by-default fetch behavior
│
├── mobile/
│   ├── flutter/
│   │   └── production.md                 Flutter + Riverpod + Freezed
│   │                                     Immutable models, SecureStorage for tokens,
│   │                                     release obfuscation, no print() in prod,
│   │                                     certificate pinning, Detox E2E gates
│   │
│   └── react-native/
│       └── production.md                 React Native + Expo + TanStack Query
│                                         expo-secure-store (not AsyncStorage),
│                                         Hermes engine, typed navigation,
│                                         FlatList (not ScrollView+map), MMKV
│
├── devops/
│   ├── docker/
│   │   └── compose.md                    Docker production configuration
│   │                                     Multi-stage builds, non-root user, pinned
│   │                                     base images, healthcheck on all services,
│   │                                     Trivy vulnerability scan in CI, .dockerignore
│   │
│   ├── ci-cd/
│   │   └── github-actions.md             GitHub Actions pipeline
│   │                                     Ordered gates (type→lint→test→scan→build→deploy),
│   │                                     secrets only via GitHub Secrets, action SHA pinning,
│   │                                     image scan before push, environment protection
│   │
│   └── nginx/
│       └── config.md                     Nginx production hardening
│                                         TLS 1.2/1.3 only, HSTS preload, CSP headers,
│                                         rate limiting zones (api/auth/global),
│                                         gzip, static asset caching, block .env/.git
│
└── security/
    ├── api-security.md                   OWASP API Security Top 10 — 2023 edition
    │                                     API1→API10 with exact PHP/JS code fixes,
    │                                     Laravel-specific: nested mass assignment,
    │                                     timing attacks, SSRF, idempotency keys
    │
    ├── auth.md                           Authentication & Authorization patterns
    │                                     JWT expiry + refresh rotation, RBAC with
    │                                     Policies/Gates, session security, MFA/TOTP,
    │                                     constant-time comparison, session fixation
    │
    └── owasp-top10.md                    OWASP Top 10 — 2021 edition
                                          A01–A10 with the exact fix for each,
                                          not just the category name
```

---

## Version Baselines

Each skill targets specific versions. Patterns from older versions are explicitly rejected.

| Stack | Versions Covered |
|-------|-----------------|
| **Laravel** | 11.x · PHP 8.3 · Inertia v2 · Sanctum 4 · Livewire v3 |
| **Go** | 1.22+ · Gin 1.10 · Echo 4.12 · slog (stdlib) |
| **Spring Boot** | 3.3+ · Java 21 LTS · Spring Security 6 · EF Core 8 |
| **Rails** | 7.2 · Ruby 3.3 · Pundit · Alba · Rack::Attack |
| **ASP.NET Core** | .NET 8 LTS · C# 12 · MediatR · FluentValidation |
| **FastAPI** | 0.115+ · Python 3.12 · Pydantic v2 · SQLAlchemy 2.0 |
| **Axum** | 0.7 · Rust 1.80+ (stable) · SQLx 0.8 · thiserror |
| **Express** | Node 20 LTS · Express 4 · Zod · Prisma · pino |
| **Django** | 5.x · Python 3.12 · DRF 3.15 |
| **React** | 19 · TypeScript 5.5 · Tailwind v4 · TanStack Query v5 |
| **Vue** | 3.5 · TypeScript 5.5 · Tailwind v4 · Pinia 2 |
| **Next.js** | 15 · React 19 · TypeScript 5.5 · Tailwind v4 |
| **Flutter** | 3.x · Dart 3 · Riverpod 2 · Freezed |
| **React Native** | 0.75+ · Expo SDK 51 · React Navigation 6 |

---

## Skill Anatomy

Every skill follows the same internal structure so Claude always has the context it needs:

```markdown
## Role
Who Claude becomes for this session — seniority level, specialization, non-negotiables.

## Version Baseline
Exact versions. Patterns from older versions are explicitly flagged as REJECT.

## What Changed in Latest Version
Breaking changes and new patterns that must replace old habits.

## File Structure
The exact folder layout — enforced, not suggested. Deviations are called out.

## Production Standards
The rules. Each one has a code example showing CORRECT and REJECT side by side.

## Common Anti-Patterns to Catch
A list of patterns Claude will refuse and explain why.

## Performance Requirements
Specific, measurable: query count, bundle size, response time thresholds.

## Security Requirements
Non-negotiable security patterns for this technology.

## Testing Requirements
Minimum test coverage and what must be tested before any endpoint is "done".

## Deployment Gates
A checklist. Nothing ships unless every item is checked.
```

---

## Contributing

This project grows through real-world experience. If you've hit a production issue that isn't covered — or found a better pattern — contributions are welcome.

### How to add a skill

**1. Create a folder for the technology**

```bash
# New framework
mkdir -p backend/elixir/phoenix
touch backend/elixir/phoenix/rest-api.md

# New pattern for existing tech
touch backend/laravel/testing.md
```

**2. Follow the skill template**

Every skill must have all sections from the anatomy above. Partial skills that skip sections (e.g., no deployment gates) will not be merged.

**3. Requirements for a skill to be accepted**

- Targets a specific, named version (not "latest")
- All REJECT examples are real anti-patterns — not invented ones
- All CORRECT examples are production-tested, not theoretical
- Deployment gates are concrete and checkable — not vague
- Written in English

**4. Open a PR**

Title format: `skill: add {technology}/{framework} — {version}`

Example: `skill: add backend/elixir/phoenix — Phoenix 1.7 + LiveView`

### What we do not accept

- Generic advice without code examples
- Skills that duplicate existing ones without meaningfully extending them
- Marketing language or vendor preference without technical justification
- Skills targeting EOL versions without a migration note

---

## Origin & Philosophy

> The knowledge in these prompts is distilled from **real production experience** — post-mortems, security audits, performance investigations, and years of code review across different stacks and teams. Every rule exists because a real system was affected when it was ignored.

These skills are not academic. They encode the *exact questions* a senior engineer asks during code review:

- *"Who verified this user owns this resource?"*
- *"What happens when this job fails on the third retry?"*
- *"Where is this N+1 coming from and how many queries does this endpoint make?"*
- *"Is this secret in version control?"*
- *"Can this endpoint be called without authentication?"*

By encoding these questions as behavioral constraints for Claude, we make it possible for any developer — regardless of seniority — to ship code that would survive a senior review.

---

## License

```
MIT License

Copyright (c) 2025 Production-Ready Checklist Contributors

Permission is hereby granted, free of charge, to any person obtaining a copy
of this software and associated documentation files (the "Software"), to deal
in the Software without restriction, including without limitation the rights
to use, copy, modify, merge, publish, distribute, sublicense, and/or sell
copies of the Software, and to permit persons to whom the Software is
furnished to do so, subject to the following conditions:

The above copyright notice and this permission notice shall be included in all
copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE
AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM,
OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE
SOFTWARE.
```

> Skills, patterns, and examples in this repository represent general software engineering knowledge and best practices. They are not affiliated with or endorsed by any framework vendor, company, or organization mentioned within.

---

<div align="center">

**Built by developers, for developers.**
*If a rule is here, someone learned it the hard way.*

</div>
# senior-mode
