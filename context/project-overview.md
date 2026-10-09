# Lendify

## Overview

Lendify is a decentralized Buy Now Pay Later (BNPL) protocol on the Stellar network, designed specifically for learners, interns, and early-career developers in emerging markets. Users finance laptops, courses, and dev tools, and repay in small installments — powered by Soroban smart contracts, with no banks and no passwords.

## Tagline

> Step into your future, pay small small.

## Repository Structure

Lendify is split across three repositories under the `Lendify-app` GitHub organization:

| Repo | Stack | Role |
|---|---|---|
| `Lendify-API` | NestJS + Fastify + TypeScript | Off-chain orchestration layer — auth, loans, reputation, jobs |
| `Lendify-Contracts` | Rust + Soroban SDK | On-chain smart contracts — credit line, reputation, liquidity pool, vendor registry |
| `Lendify-App` | React Native + Expo | Mobile application — learner and sponsor interfaces |

## Goals

1. Let learners register with a Stellar wallet and build an on-chain reputation score.
2. Let learners apply for financing on learning tools and repay in small installments.
3. Let mentors vouch for learners to boost their credit limits.
4. Let sponsors and companies fund the learner lending pool.
5. Let verified learning vendors receive conditional disbursements.
6. Expose all of this through a clean mobile app and a well-documented API.

## Core User Flow — Learner

1. Learner downloads Lendify App and connects their Stellar wallet.
2. Learner completes their learner profile (school, program, income type).
3. Learner views their on-chain reputation score and credit limit.
4. Learner browses verified vendors (bootcamps, electronics, online courses).
5. Learner applies for a loan — selects vendor, amount, and installment schedule.
6. API builds unsigned Soroban transaction — learner signs with wallet.
7. Signed XDR submitted to Stellar network — loan is created on-chain.
8. Learner receives payment reminders and repays in installments.
9. On-time repayments increase reputation score and unlock higher credit limits.

## Core User Flow — Sponsor

1. Sponsor registers and deposits into the learner liquidity pool.
2. Pool funds learner loans — sponsor earns yield from repayments.
3. Sponsor tracks pool stats, their position, and interest earned.

## Core User Flow — Mentor

1. Mentor registers as a verified voucher.
2. Learner requests a vouch from a mentor.
3. Mentor approves — vouch recorded on-chain.
4. Learner's credit limit increases by 10% per vouch (max 3 vouches).

## Features

### Authentication
- Wallet-based authentication — Ed25519 signature verification via Stellar SDK.
- No passwords. No OAuth. Just wallet signatures → JWT access + refresh tokens.

### Learner Profiles
- Off-chain learner metadata: school, program, income type, country, city.
- Device ownership tracking.

### Reputation System
- On-chain reputation score (0–100) stored in Soroban.
- Three-tier cache: Redis (hot, 5min) → Supabase (warm, 60min) → Blockchain (source of truth).
- Score increases on on-time repayment, decreases on default.

### Loan Lifecycle
- Quote → Create → Repay (per-installment) → Complete or Default.
- Per-installment tracking with due dates, paid flags, and late fee accrual.
- Grace periods configurable per loan type.

### Vendor Registry
- Admin-managed whitelist of verified learning vendors.
- Vendor types: School, Bootcamp, Electronics, Books, Subscriptions.
- Conditional disbursement: funds released to vendor on enrollment confirmation.

### Liquidity Pool
- Share-based accounting — deposit → shares → withdrawal.
- Interest distribution: 85% LPs / 10% protocol / 5% vendor fund.
- Sponsor pool type separate from individual LP investor pool.

### Vouching System
- On-chain vouch record per mentor-learner pair.
- Vouch expiry, revocation, and history.
- Credit limit boost: +10% per vouch, max 3 vouches.

### Background Jobs
- Blockchain indexer: polls Soroban events every 30s → Supabase.
- Transaction status checker: polls Horizon every 15s for pending txs.
- Loan payment reminder: daily 9AM UTC — 3-day, 1-day, and overdue alerts.
- Nonce cleanup: hourly — deletes expired nonces from DB.

### Notifications
- In-app notification system — loan reminders, tx status, reputation changes.

## Scope

### In Scope
- Wallet-based auth and learner registration
- Learner profile management
- On-chain reputation scoring with tiered cache
- Loan lifecycle (quote, create, per-installment repay, default)
- Vendor registry and conditional disbursement
- Liquidity pool with share accounting and interest distribution
- Vouching system (mentor → learner)
- Background indexing of Soroban events
- Notification system
- Mobile app (React Native + Expo)
- Testnet deployment of all Soroban contracts
- Open-source contributor workflow (issues, PRs, CONTRIBUTING.md)

### Out Of Scope
- Fiat on/off ramp integration
- KYC/AML compliance flows (schema exists, logic deferred)
- Enterprise admin dashboard
- Push notifications (schema exists, implementation deferred)
- Mainnet deployment (Phase 10)
- Web app (mobile-first only)

## Success Criteria

1. A learner can register, sign in, and view their reputation score end to end on testnet.
2. A learner can apply for a loan, sign the Soroban transaction, and see it confirmed on-chain.
3. A sponsor can deposit into the liquidity pool and track their position.
4. A mentor can vouch for a learner and the credit limit increases.
5. Background jobs index on-chain events and deliver payment reminders.
6. All Soroban contracts are deployed on testnet with verified contract IDs.
7. Lendify-API is live on Render with a public health check and Swagger UI.
8. Lendify-App has a working Expo preview build.
9. The project has 20+ well-labeled GitHub issues ready for contributors.
10. The project is submitted to Stellar Drips Wave with live URLs.
