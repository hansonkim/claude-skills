# Phase Plan: User Authentication (example)

This is a sample `phase-plan.md` showing all phase-harness fields in
use. It implements JWT-based email/password authentication on a
Node/Express + React stack.

Copy this as a starting point, then adapt to your stack and feature.

## Overview

JWT-based user authentication added to an existing Express + React app.
Email + password registration, login, and a protected-route guard.

## Tech Context

- Build: `pnpm build`
- Test: `pnpm test`
- Lint: `pnpm lint`
- Browser verification: chrome-devtools MCP

## Execution Settings

- plan_review_interval: 2
- patterns_distill_interval: 3

## Phases

### Phase 1: DB schema

- **Objective**: users table with email/password_hash/created_at
- **Inputs**: existing migration infra at `migrations/`
- **Outputs**: `migrations/0042_users.sql`
- **Domain**: db-migration
- **Depends**: []
- **Cost tier**: minimal
- **Criticality**: normal
- **Generator model**: simple
- **Acceptance Criteria**:
  - [ ] users table exists with `email UNIQUE` and `password_hash NOT NULL`
        Verify: `psql -c "\d users" | grep -E 'email.*UNIQUE|password_hash.*NOT NULL'`
  - [ ] migration up/down/up round-trip works
        Verify: `pnpm migrate up && pnpm migrate down && pnpm migrate up`
- **Commit message**: `feat(db): add users table`

### Phase 2: Auth service

- **Objective**: email/password signup + login endpoints issuing JWT
- **Inputs**: Phase 1 schema
- **Outputs**: `src/auth/*`, `tests/auth/*`
- **Domain**: [backend-typescript, security]
- **Depends**: [1]
- **Cost tier**: standard
- **Criticality**: high
- **Generator model**: complex
- **Acceptance Criteria**:
  - [ ] `POST /auth/signup` creates a user and returns a JWT
        Verify: `pnpm test tests/auth/signup.test.ts`
  - [ ] `POST /auth/login` returns a JWT for valid credentials, 401 otherwise
        Verify: `pnpm test tests/auth/login.test.ts`
  - [ ] Passwords are bcrypt-hashed, never stored or logged in plaintext
        Verify: `pnpm test tests/auth/hashing.test.ts`
- **Commit message**: `feat(auth): signup and login endpoints`

### Phase 3: Frontend forms

- **Objective**: signup and login forms with error states
- **Inputs**: Phase 2 endpoints
- **Outputs**: `src/ui/auth/*`
- **Domain**: frontend-web
- **Depends**: [2]
- **Cost tier**: standard
- **Criticality**: normal
- **Generator model**: simple
- **Acceptance Criteria**:
  - [ ] Signup form submits and redirects to `/` on success
        Verify: chrome-devtools MCP — navigate to `/signup`, fill, submit, assert URL
  - [ ] Login form shows an inline error on wrong password
        Verify: chrome-devtools MCP — fill wrong creds, assert error element visible
- **Commit message**: `feat(ui): signup and login forms`

### Phase 4: E2E tests

- **Objective**: full signup → login → protected route flow
- **Inputs**: Phases 1–3
- **Outputs**: `e2e/auth-flow.spec.ts`
- **Domain**: testing
- **Depends**: [3]
- **Cost tier**: minimal
- **Criticality**: normal
- **Generator model**: simple
- **Acceptance Criteria**:
  - [ ] Full flow passes end-to-end
        Verify: `pnpm test:e2e e2e/auth-flow.spec.ts`
- **Commit message**: `test(auth): add E2E signup→login flow`
