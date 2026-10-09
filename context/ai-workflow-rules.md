# AI Development Workflow Rules

## Purpose

These rules govern how AI agents (Claude, Cursor, Copilot, etc.) interact with the Lendify codebase. They exist to prevent context drift, architectural violations, and code breaks — especially as the project opens to open-source contributors and multiple AI sessions.

---

## Core Principle

**Always implement against the context files. Never infer or invent behavior from scratch.**

The six context files are the single source of truth for this project:

| File | What it defines |
|---|---|
| `project-overview.md` | What Lendify is, what it does, what's in scope |
| `architecture-context.md` | Stack, system boundaries, storage model, invariants |
| `code-standards.md` | Naming, patterns, what not to do |
| `progress-tracker.md` | Current state, what's done, what's next, open questions |
| `ai-workflow-rules.md` | This file — how to work |
| `ui-context.md` | Lendify-App design system, colors, component patterns |

Read all relevant context files before starting any implementation task.

---

## Scoping Rules

- Work on one feature unit or subsystem at a time.
- Prefer small, verifiable increments over large speculative changes.
- Do not combine unrelated system boundaries in a single implementation step.
- If a task touches both API and contracts, split it into two steps.
- If a task touches both UI and state management, split it into two steps.

## When To Split Work

Split an implementation step if it combines:

- API module changes and contract changes
- UI screen changes and background job changes
- Multiple unrelated API routes
- Behavior that is not clearly defined in the context files

If a change cannot be verified end to end with a single `npm run build` or `cargo build`, the scope is too broad — split it.

---

## Before Starting Any Task

1. Read `progress-tracker.md` — confirm the current state matches what you expect.
2. Read `architecture-context.md` — identify which system boundary you are working in.
3. Read `code-standards.md` — confirm the naming and patterns for the language/layer.
4. If the task is UI-related, read `ui-context.md` for colors, tokens, and component patterns.
5. If any requirement is ambiguous, add it as an open question in `progress-tracker.md` before implementing.

---

## System Boundary Rules

### Lendify-API
- All Stellar SDK / Soroban RPC calls must stay inside `src/blockchain/`.
- All Supabase calls in modules must go through `SupabaseService` or a repository — never raw client calls.
- No business logic in controllers — controllers validate input and delegate to services.
- No long-lived work in request handlers — use BullMQ jobs.
- Every new module must be registered in `app.module.ts`.
- Every new module must have `@ApiTags`, `@ApiOperation`, and `@ApiResponse` on all endpoints.

### Lendify-Contracts
- All storage operations go through `storage.rs`.
- All events go through `events.rs`.
- Every mutating function starts with `require_auth()`.
- Every persistent storage write calls `extend_ttl()` immediately after.
- Every new contract must have `get_version()` and `upgrade()`.
- Tests use `soroban_sdk::Env::default()` and `mock_all_auths()`.

### Lendify-App
- No API calls in screen files or components — use hooks.
- No global state mutations outside Zustand stores.
- No JWT token handling outside `services/auth.service.ts` and `stores/auth.store.ts`.
- No hardcoded hex values — use `constants/colors.ts` tokens.
- All navigation uses Expo Router — no manual `navigation.navigate()`.

---

## Handling Missing Requirements

- Do not invent product behavior not defined in the context files.
- If a requirement is ambiguous, add it to the **Open Questions** section of `progress-tracker.md` before implementing.
- If a requirement is missing entirely, pause and ask the maintainer.

---

## Protected Foundation Files

Do not modify these files unless explicitly instructed:

### Lendify-API
- `src/common/guards/jwt-auth.guard.ts` — all protected routes depend on this
- `src/common/decorators/current-user.decorator.ts` — used across all protected controllers
- `src/blockchain/soroban/soroban.service.ts` — foundation for all contract clients
- `src/database/supabase.client.ts` — singleton pattern must not be broken

### Lendify-Contracts
- Any contract that is already deployed to testnet — do not modify without incrementing version and deploying an upgrade

### Lendify-App
- `constants/colors.ts` — token definitions used across all components
- `stores/auth.store.ts` — auth state used across all screens

---

## Verification Checklist

Before marking any unit as complete:

- [ ] `npm run build` passes with zero TypeScript errors (Lendify-API)
- [ ] `cargo build` passes with zero Rust errors (Lendify-Contracts)
- [ ] New module is registered in `app.module.ts` (Lendify-API)
- [ ] New migration file exists for any new table (Lendify-API)
- [ ] Swagger decorators exist on all new endpoints (Lendify-API)
- [ ] TTL extension is called on all new persistent storage writes (Lendify-Contracts)
- [ ] No invariant from `architecture-context.md` was violated
- [ ] `progress-tracker.md` is updated to reflect completed work

---

## Keeping Docs In Sync

Update the relevant context file whenever implementation changes:

- New module added → update `progress-tracker.md` completed section
- Architecture decision made → add to **Architecture Decisions** in `progress-tracker.md`
- New open question → add to **Open Questions** in `progress-tracker.md`
- System boundary changes → update `architecture-context.md`
- New naming convention or pattern → update `code-standards.md`
- New screen or component pattern → update `ui-context.md`

Progress state must reflect the actual implementation — not the intended state.

---

## What AI Must Never Do

- Invent product behavior not defined in the context files
- Call `stellar-sdk` outside `src/blockchain/`
- Call Supabase directly in NestJS modules
- Write business logic in controllers
- Skip `require_auth()` in Soroban contract functions
- Skip `extend_ttl()` after persistent storage writes
- Store raw refresh tokens (must be SHA-256 hashed)
- Add hardcoded hex colors in Lendify-App components
- Modify protected foundation files without explicit instruction
- Mark a unit complete without verifying the build passes
