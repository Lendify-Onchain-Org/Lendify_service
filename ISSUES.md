# Lendify Contributor Issues

35 detailed issues spanning Contracts (Rust/Soroban), API (NestJS), Mobile (Expo/React Native), and DevOps.

Each issue is self-contained and ready for a contributor to pick up. Read the linked context files before starting.

---

## Issue 1: Add dedicated tests for `repay_installment()`
**Repo:** Lendify-app/Lendify-Contracts
**Labels:** contracts, testing, hard
**Difficulty:** hard

### Problem
The `repay_installment()` function in `contracts/creditline-contract/src/lib.rs` lacks isolated unit tests. The only existing coverage is incidental via end-to-end flow tests, leaving error paths (double-pay, out-of-bounds, unauthorized, zero-amount) untested. Without these, regressions in repayment logic can silently break borrower balances.

### Context
Repayment correctness is the most safety-critical operation in Lendify — a bug here can either lock learners out of repaying or allow them to pay twice. Sponsors lose trust if installments are mis-accounted, and the reputation contract derives scoring from these calls. This must be airtight before mainnet.

### Before Starting
Read these context files first:
- context/architecture-context.md
- context/code-standards.md
- context/progress-tracker.md
- contracts/creditline-contract/src/lib.rs
- contracts/creditline-contract/src/tests.rs

### What To Build
1. Add a helper `setup_loan_with_schedule(env, borrower, amount, n_installments)` at the top of `tests.rs` that initializes the contract, creates a loan, and approves it.
2. Test `repay_installment_happy_path`: pay installment 0, assert `Installment.paid == true`, assert outstanding balance decremented by exact amount.
3. Test `repay_installment_double_pay_rejected`: pay installment 0, then call again — assert `ContractError::InstallmentAlreadyPaid` is returned.
4. Test `repay_installment_out_of_bounds`: call with index `installments.len()` — assert `ContractError::InvalidInstallmentIndex`.
5. Test `repay_installment_non_borrower_rejected`: call from a wallet other than the loan borrower — assert auth panic via `env.mock_auths()` mismatch.
6. Test `repay_installment_zero_amount_rejected`: ensure zero-amount payments are explicitly rejected with `ContractError::InvalidAmount`.

### Files To Touch
- `contracts/creditline-contract/src/tests.rs`
- `contracts/creditline-contract/src/errors.rs` (if `InvalidInstallmentIndex` / `InstallmentAlreadyPaid` not yet defined)

### Acceptance Criteria
- [ ] 5 new `#[test]` functions covering each scenario above
- [ ] `cargo test -p creditline-contract` passes with all new tests green
- [ ] Each test uses `Env::default()` and `mock_all_auths()` where appropriate
- [ ] No existing test is modified or weakened
- [ ] Test names follow the `repay_installment_<scenario>` convention
- [ ] Coverage report shows `repay_installment()` branch coverage ≥ 90%

### Mandatory Checks Before PR
- [ ] cargo build passes with zero errors
- [ ] cargo test — all 93 existing tests still pass
- [ ] require_auth() is FIRST line of every mutating function
- [ ] extend_ttl() called after EVERY persistent storage write
- [ ] New unit tests written for every new function
- [ ] context/progress-tracker.md updated

---

## Issue 2: Implement `approve_loan()` — Pending to Active transition
**Repo:** Lendify-app/Lendify-Contracts
**Labels:** contracts, feature, medium
**Difficulty:** medium

### Problem
Loans created by `create_loan()` in `contracts/creditline-contract/src/lib.rs` are stuck in `LoanStatus::Pending` because no transition function exists. Without `approve_loan()`, no learner can ever receive funds — the protocol is effectively read-only end-to-end.

### Context
The Pending → Active gate is what separates "applied for credit" from "owes money". Sponsors need to see only Active loans count toward pool utilization, and learners only accrue obligations after admin approval. This unblocks the entire mobile-app borrowing flow.

### Before Starting
Read these context files first:
- context/architecture-context.md
- context/code-standards.md
- context/progress-tracker.md
- contracts/creditline-contract/src/lib.rs
- contracts/creditline-contract/src/events.rs

### What To Build
1. Add `pub fn approve_loan(env: Env, loan_id: u64) -> Result<(), ContractError>` to the contract `impl`.
2. First line: `require_auth()` on the admin address loaded from storage (`StorageKey::Admin`).
3. Load loan via `storage::get_loan(&env, loan_id)`; return `ContractError::LoanNotFound` if missing.
4. Verify `loan.status == LoanStatus::Pending`; otherwise return `ContractError::InvalidLoanState`.
5. Mutate to `LoanStatus::Active`, write back with `storage::set_loan(&env, &loan)`, then `extend_ttl()`.
6. Emit a `LOANAPPROVED` event via `events::emit_loan_approved(&env, loan_id, loan.borrower)`. Add corresponding emit helper in `events.rs`.
7. Add 3 unit tests: happy path, non-admin rejection, non-pending state rejection.

### Files To Touch
- `contracts/creditline-contract/src/lib.rs`
- `contracts/creditline-contract/src/events.rs`
- `contracts/creditline-contract/src/errors.rs`
- `contracts/creditline-contract/src/tests.rs`

### Acceptance Criteria
- [ ] `approve_loan()` is the second public entry point after `create_loan()` and follows the same code structure
- [ ] All 3 unit tests pass
- [ ] `LOANAPPROVED` event is documented in events.rs with a comment
- [ ] No silent state transitions — every error path returns a typed `ContractError`
- [ ] Loan TTL is extended after status mutation
- [ ] Admin auth is the FIRST line of the function

### Mandatory Checks Before PR
- [ ] cargo build passes with zero errors
- [ ] cargo test — all 93 existing tests still pass
- [ ] require_auth() is FIRST line of every mutating function
- [ ] extend_ttl() called after EVERY persistent storage write
- [ ] New unit tests written for every new function
- [ ] context/progress-tracker.md updated

---

## Issue 3: Add per-installment late fee accrual
**Repo:** Lendify-app/Lendify-Contracts
**Labels:** contracts, feature, hard
**Difficulty:** hard

### Problem
The `RepaymentInstallment` struct in `contracts/creditline-contract/src/types.rs` carries no `due_date`, and `repay_installment()` does not penalize late payments. Borrowers can effectively pay months late with zero consequence, which breaks the reputation score's underlying assumption that on-time payment matters.

### Context
Late fees are the protocol's only economic enforcement mechanism short of liquidation. Without them, sponsors price all risk into the base APR, hurting on-time learners. The fee must flow back into the liquidity pool so sponsors are made whole.

### Before Starting
Read these context files first:
- context/architecture-context.md
- context/code-standards.md
- context/progress-tracker.md
- contracts/creditline-contract/src/types.rs
- contracts/creditline-contract/src/lib.rs

### What To Build
1. Add `pub due_date: u64` to `RepaymentInstallment` in `types.rs`. Populate at loan creation: `due_date = approval_ts + (n+1) * installment_period_secs`.
2. Add `pub late_fee_bps: u32` to `ProtocolParameters` in `types.rs`, default 500 bps (5%).
3. In `repay_installment()`, compute `now = env.ledger().timestamp()`. If `now > installment.due_date`, compute `late_fee = installment.amount * late_fee_bps / 10_000` and require the caller to transfer `installment.amount + late_fee`.
4. Route the late fee to the liquidity pool via cross-contract call, not the borrower's loan balance.
5. Emit a `LATEFEEPAID` event with `(loan_id, installment_index, fee_amount)`.
6. Add 4 unit tests: on-time path (no fee), 1-day-late, exact-due-date boundary, custom-bps test.

### Files To Touch
- `contracts/creditline-contract/src/types.rs`
- `contracts/creditline-contract/src/lib.rs`
- `contracts/creditline-contract/src/storage.rs`
- `contracts/creditline-contract/src/events.rs`
- `contracts/creditline-contract/src/tests.rs`

### Acceptance Criteria
- [ ] `due_date` is set on every installment at loan creation
- [ ] `late_fee_bps` is configurable and persisted in `ProtocolParameters`
- [ ] Late fees route to the liquidity pool, not the loan balance
- [ ] On-time payments incur zero fee (exact `now == due_date` is on-time)
- [ ] 4 new tests pass
- [ ] Migration note added: existing loans without `due_date` are gracefully handled (assume 0 = on-time)

### Mandatory Checks Before PR
- [ ] cargo build passes with zero errors
- [ ] cargo test — all 93 existing tests still pass
- [ ] require_auth() is FIRST line of every mutating function
- [ ] extend_ttl() called after EVERY persistent storage write
- [ ] New unit tests written for every new function
- [ ] context/progress-tracker.md updated

---

## Issue 4: Add mentor vouching contract
**Repo:** Lendify-app/Lendify-Contracts
**Labels:** contracts, feature, hard
**Difficulty:** hard

### Problem
There is no on-chain mechanism for verified mentors to vouch for learners. The reputation contract currently has no off-board signal for new wallets with zero loan history, so first-time learners face artificially high interest rates with no path to bootstrap trust.

### Context
Mentor vouching is Lendify's cold-start fix. A verified educator or community lead can stake their own reputation behind a learner, unlocking lower-rate credit for previously unscored users. This is the protocol's social capital layer.

### Before Starting
Read these context files first:
- context/architecture-context.md
- context/code-standards.md
- context/progress-tracker.md
- contracts/reputation-contract/src/lib.rs

### What To Build
1. Create a new crate at `contracts/vouching-contract/` mirroring the layout of `creditline-contract` (`Cargo.toml`, `src/{lib,types,storage,events,errors,tests}.rs`).
2. Storage: `VerifiedMentors: Map<Address, bool>`, `Vouches: Map<(Address mentor, Address learner), VouchRecord { ts, boost_amount, active }>`.
3. `pub fn vouch(env, mentor: Address, learner: Address)`: `require_auth(&mentor)`, check `VerifiedMentors[mentor] == true`, write `VouchRecord { ts: now, boost: protocol.vouch_boost, active: true }`, `extend_ttl()`, cross-call reputation contract `add_boost(learner, boost_amount)`.
4. `pub fn revoke_vouch(env, mentor, learner)`: `require_auth(&mentor)`, set `active=false`, cross-call reputation `remove_boost(learner, boost_amount)`.
5. `pub fn get_vouches(env, learner) -> Vec<VouchRecord>`: read-only.
6. `pub fn set_mentor(env, mentor, verified: bool)`: admin-only.
7. Add events: `MENTORVOUCHED`, `VOUCHREVOKED`, `MENTORVERIFIED`.
8. Write tests for all 4 mutating functions including duplicate-vouch rejection and unverified-mentor rejection.

### Files To Touch
- `contracts/vouching-contract/Cargo.toml`
- `contracts/vouching-contract/src/lib.rs`
- `contracts/vouching-contract/src/types.rs`
- `contracts/vouching-contract/src/storage.rs`
- `contracts/vouching-contract/src/events.rs`
- `contracts/vouching-contract/src/errors.rs`
- `contracts/vouching-contract/src/tests.rs`
- Workspace `Cargo.toml` (add member)
- `contracts/reputation-contract/src/lib.rs` (expose `add_boost` / `remove_boost`)

### Acceptance Criteria
- [ ] New crate builds cleanly with `cargo build -p vouching-contract`
- [ ] All 4 mutating functions begin with `require_auth()`
- [ ] All persistent writes followed by `extend_ttl()`
- [ ] Cross-contract calls to reputation are tested with a mock
- [ ] Minimum 8 unit tests
- [ ] Events are emitted on every state change
- [ ] Workspace `Cargo.toml` includes the new crate

### Mandatory Checks Before PR
- [ ] cargo build passes with zero errors
- [ ] cargo test — all 93 existing tests still pass
- [ ] require_auth() is FIRST line of every mutating function
- [ ] extend_ttl() called after EVERY persistent storage write
- [ ] New unit tests written for every new function
- [ ] context/progress-tracker.md updated

---

## Issue 5: Add `upgrade()` function to all 5 contracts
**Repo:** Lendify-app/Lendify-Contracts
**Labels:** contracts, infra, medium
**Difficulty:** medium

### Problem
None of the 5 contracts (creditline, liquidity-pool, reputation, vendor-registry, token-mock) currently expose an `upgrade()` entry point. Once deployed to testnet, any bug fix requires a full redeploy and address change — breaking every client that references the old address.

### Context
Soroban supports in-place WASM upgrades preserving contract address and storage. Without this, every contract bugfix invalidates the API's contract IDs and the mobile app's hardcoded references. Upgradeability is the difference between a one-shot deploy and a maintainable protocol.

### Before Starting
Read these context files first:
- context/architecture-context.md
- context/code-standards.md
- context/progress-tracker.md
- contracts/*/src/lib.rs

### What To Build
1. Define a shared snippet (in each contract's `lib.rs`):
   ```rust
   pub fn upgrade(env: Env, new_wasm_hash: BytesN<32>) -> Result<(), ContractError> {
       let admin: Address = env.storage().instance().get(&StorageKey::Admin).ok_or(ContractError::NotInitialized)?;
       admin.require_auth();
       env.deployer().update_current_contract_wasm(new_wasm_hash);
       Ok(())
   }
   ```
2. Add to all 5 contracts: creditline, liquidity-pool, reputation, vendor-registry, token-mock.
3. Emit a `CONTRACTUPGRADED` event with `(old_version, new_version, ts)` — add a `get_version() -> u32` function returning current version, bumped by upgrade.
4. Persist `StorageKey::Version` in each contract's instance storage, default 1.
5. Write one unit test per contract: non-admin call rejected, admin call succeeds (mock the wasm hash).
6. Document the upgrade flow in `contracts/README.md`.

### Files To Touch
- `contracts/creditline-contract/src/lib.rs`
- `contracts/liquidity-pool-contract/src/lib.rs`
- `contracts/reputation-contract/src/lib.rs`
- `contracts/vendor-registry-contract/src/lib.rs`
- `contracts/token-mock-contract/src/lib.rs`
- Each contract's `events.rs`, `storage.rs`, `errors.rs`
- `contracts/README.md`

### Acceptance Criteria
- [ ] All 5 contracts expose `upgrade(env, new_wasm_hash)`
- [ ] All 5 contracts expose `get_version()` returning a `u32`
- [ ] Admin auth check is the FIRST executable line of `upgrade()`
- [ ] 5 new tests pass (one per contract)
- [ ] `CONTRACTUPGRADED` event emitted on successful upgrade
- [ ] Upgrade flow documented in README

### Mandatory Checks Before PR
- [ ] cargo build passes with zero errors
- [ ] cargo test — all 93 existing tests still pass
- [ ] require_auth() is FIRST line of every mutating function
- [ ] extend_ttl() called after EVERY persistent storage write
- [ ] New unit tests written for every new function
- [ ] context/progress-tracker.md updated

---

## Issue 6: Harden `storage.rs` `expect()` panics to typed errors
**Repo:** Lendify-app/Lendify-Contracts
**Labels:** contracts, refactor, medium
**Difficulty:** medium

### Problem
Across all contracts, `storage.rs` files use `.expect("loan not found")` and similar patterns when reading persistent storage. These produce opaque VM traps with no error code, making the API unable to distinguish "loan does not exist" from "contract panicked unexpectedly".

### Context
Every contract error must be a typed `ContractError` so the API can translate it to a proper HTTP status code. Right now, any missing storage read returns a 500 to the mobile app instead of a 404, which makes the user-facing error stories incoherent.

### Before Starting
Read these context files first:
- context/architecture-context.md
- context/code-standards.md
- context/progress-tracker.md
- contracts/*/src/storage.rs
- contracts/*/src/errors.rs

### What To Build
1. Audit every `.expect(...)` and `.unwrap()` call across all 5 `storage.rs` files. Use `rg -n "expect\(|unwrap\(\)" contracts/*/src/storage.rs`.
2. For each, change the function signature to return `Result<T, ContractError>`.
3. Replace `.expect("loan not found")` with `.ok_or(ContractError::LoanNotFound)?`.
4. Update callers in `lib.rs` to propagate with `?`.
5. Add missing error variants where needed (e.g. `PoolNotInitialized`, `VendorNotFound`).
6. Add a regression test per contract: call a getter before `initialize()` and assert `NotInitialized` is returned (not a panic).

### Files To Touch
- `contracts/creditline-contract/src/storage.rs`, `errors.rs`, `lib.rs`, `tests.rs`
- `contracts/liquidity-pool-contract/src/storage.rs`, `errors.rs`, `lib.rs`, `tests.rs`
- `contracts/reputation-contract/src/storage.rs`, `errors.rs`, `lib.rs`, `tests.rs`
- `contracts/vendor-registry-contract/src/storage.rs`, `errors.rs`, `lib.rs`, `tests.rs`
- `contracts/token-mock-contract/src/storage.rs`, `errors.rs`, `lib.rs`, `tests.rs`

### Acceptance Criteria
- [ ] Zero `.expect()` or `.unwrap()` calls remain in any `storage.rs`
- [ ] All getter functions return `Result<T, ContractError>`
- [ ] 5 new regression tests prove typed errors are returned instead of panics
- [ ] All 93 existing tests still pass
- [ ] Cargo clippy returns no new warnings

### Mandatory Checks Before PR
- [ ] cargo build passes with zero errors
- [ ] cargo test — all 93 existing tests still pass
- [ ] require_auth() is FIRST line of every mutating function
- [ ] extend_ttl() called after EVERY persistent storage write
- [ ] New unit tests written for every new function
- [ ] context/progress-tracker.md updated

---

## Issue 7: Add initialize check to all contract functions
**Repo:** Lendify-app/Lendify-Contracts
**Labels:** contracts, safety, medium
**Difficulty:** medium

### Problem
Public functions that read from persistent storage will panic with a VM trap if invoked before `initialize()` has been called. There is no `is_initialized()` guard, so any deploy-without-init race condition produces opaque failures rather than a clear `NotInitialized` error.

### Context
The API frontend retries on transient errors. A panic on uninitialized state looks transient — but it isn't. Wrapping every public read with a typed init check lets the API surface a clear "contract not initialized" message and stop retrying.

### Before Starting
Read these context files first:
- context/architecture-context.md
- context/code-standards.md
- context/progress-tracker.md
- contracts/*/src/lib.rs

### What To Build
1. Add `pub fn is_initialized(env: &Env) -> bool` helper in each contract's `lib.rs` that checks `env.storage().instance().has(&StorageKey::Admin)`.
2. At the top of every public function (mutating or read-only) that touches persistent storage, add: `if !Self::is_initialized(&env) { return Err(ContractError::NotInitialized); }`.
3. Exclude `initialize()` itself and any pure helper that doesn't read storage.
4. Add `ContractError::NotInitialized = 1` variant in `errors.rs` if missing.
5. Write one regression test per contract that calls a getter before initialize and asserts `NotInitialized`.
6. Document the convention in `code-standards.md`.

### Files To Touch
- All 5 `contracts/*/src/lib.rs`
- All 5 `contracts/*/src/errors.rs`
- All 5 `contracts/*/src/tests.rs`
- `context/code-standards.md`

### Acceptance Criteria
- [ ] Every public function that reads storage starts with an `is_initialized` check
- [ ] `ContractError::NotInitialized` exists in all 5 error enums
- [ ] 5 regression tests pass
- [ ] No existing tests fail
- [ ] code-standards.md documents the convention

### Mandatory Checks Before PR
- [ ] cargo build passes with zero errors
- [ ] cargo test — all 93 existing tests still pass
- [ ] require_auth() is FIRST line of every mutating function
- [ ] extend_ttl() called after EVERY persistent storage write
- [ ] New unit tests written for every new function
- [ ] context/progress-tracker.md updated

---

## Issue 8: Add TTL `extend_ttl()` to liquidity-pool and vendor-registry contracts
**Repo:** Lendify-app/Lendify-Contracts
**Labels:** contracts, safety, medium
**Difficulty:** medium

### Problem
Only `creditline-contract` consistently calls `extend_ttl()` after persistent storage writes. The liquidity-pool and vendor-registry contracts write to persistent storage but never extend TTL, meaning their data will expire and the contract will appear to "lose" deposits or registered vendors.

### Context
Soroban persistent entries expire if not refreshed. A sponsor's pool balance disappearing because of TTL would be catastrophic — funds become unrecoverable. This is a latent bug that only surfaces after weeks of inactivity on testnet but would be irrecoverable on mainnet.

### Before Starting
Read these context files first:
- context/architecture-context.md
- context/code-standards.md
- context/progress-tracker.md
- contracts/liquidity-pool-contract/src/lib.rs
- contracts/vendor-registry-contract/src/lib.rs

### What To Build
1. Audit liquidity-pool: locate every `env.storage().persistent().set(...)`. After each, call `env.storage().persistent().extend_ttl(&key, MIN_TTL, MAX_TTL)` where `MIN_TTL = 100_000`, `MAX_TTL = 1_000_000` (define in `constants.rs`).
2. Same for vendor-registry: every `set(...)` followed by `extend_ttl()`.
3. Centralize the constants in a shared `contracts/common/src/ttl.rs` if not yet present; otherwise define locally per contract.
4. Add a `storage::write_and_extend(env, key, value)` helper to enforce the pattern at the source.
5. Write a test per contract: deposit to pool, advance ledger by `MIN_TTL - 1`, read — assert value still present.
6. Update `code-standards.md` to make the helper mandatory.

### Files To Touch
- `contracts/liquidity-pool-contract/src/storage.rs`
- `contracts/liquidity-pool-contract/src/lib.rs`
- `contracts/vendor-registry-contract/src/storage.rs`
- `contracts/vendor-registry-contract/src/lib.rs`
- `contracts/liquidity-pool-contract/src/tests.rs`
- `contracts/vendor-registry-contract/src/tests.rs`
- `context/code-standards.md`

### Acceptance Criteria
- [ ] Every persistent `set()` in both contracts is followed by `extend_ttl()`
- [ ] TTL constants are named and centralized
- [ ] 2 new tests prove data survives near-expiry
- [ ] Helper function exists and is used by all writes
- [ ] code-standards.md mandates the pattern

### Mandatory Checks Before PR
- [ ] cargo build passes with zero errors
- [ ] cargo test — all 93 existing tests still pass
- [ ] require_auth() is FIRST line of every mutating function
- [ ] extend_ttl() called after EVERY persistent storage write
- [ ] New unit tests written for every new function
- [ ] context/progress-tracker.md updated

---

## Issue 9: Wire `LiquidityContractClient` into `LiquidityService`
**Repo:** Lendify-app/Lendify-API
**Labels:** backend, blockchain, hard
**Difficulty:** hard

### Problem
`src/modules/liquidity/liquidity.service.ts` currently returns hardcoded placeholder XDR strings for `deposit()` and `withdraw()` operations. The real `LiquidityContractClient` exists in `src/blockchain/contracts/` but is not injected, so the mobile app's sponsor flow cannot actually deposit on-chain.

### Context
This is the sponsor-side blocker for end-to-end testing. Without real XDR generation, sponsors can sign but the resulting transaction is unsubmittable garbage. Every other sponsor feature (portfolio, APY, withdrawals) depends on this being real.

### Before Starting
Read these context files first:
- context/architecture-context.md
- context/code-standards.md
- context/progress-tracker.md
- src/modules/liquidity/liquidity.service.ts
- src/blockchain/contracts/liquidity-contract.client.ts

### What To Build
1. Inject `LiquidityContractClient` into `LiquidityService` constructor.
2. Read `LIQUIDITY_POOL_CONTRACT_ID` from `ConfigService` and pass to the client.
3. Replace placeholder in `buildDepositXdr(walletAddress, amount)`: call `client.buildUnsignedXdr('deposit', [walletAddress, amount])` and return the base64 XDR string.
4. Same for `buildWithdrawXdr(walletAddress, shares)`: call `client.buildUnsignedXdr('withdraw', [walletAddress, shares])`.
5. Add error handling: contract simulation errors should return a 400 with the typed Soroban error code mapped to a user-facing message.
6. Update e2e test `test/e2e/liquidity.e2e-spec.ts` to assert the returned XDR is a valid base64-encoded Stellar transaction (use `TransactionBuilder.fromXDR()`).

### Files To Touch
- `src/modules/liquidity/liquidity.service.ts`
- `src/modules/liquidity/liquidity.module.ts`
- `src/blockchain/contracts/liquidity-contract.client.ts`
- `src/blockchain/blockchain.module.ts`
- `test/e2e/liquidity.e2e-spec.ts`
- `.env.example`

### Acceptance Criteria
- [ ] No placeholder XDR strings remain in the liquidity service
- [ ] Returned XDR parses successfully via `TransactionBuilder.fromXDR`
- [ ] `LIQUIDITY_POOL_CONTRACT_ID` is read via `ConfigService`, not `process.env`
- [ ] Contract simulation errors map to typed HTTP errors
- [ ] E2E test passes against a running Soroban testnet RPC
- [ ] Unit tests for the service mock the client and assert the right method is invoked

### Mandatory Checks Before PR
- [ ] npm run build passes with zero TypeScript errors
- [ ] No new `any` types introduced anywhere
- [ ] Full Swagger @ApiOperation + @ApiResponse decorators on any new endpoints
- [ ] New migration file created for any schema changes
- [ ] context/progress-tracker.md updated

---

## Issue 10: Implement `POST /vendors` admin endpoint
**Repo:** Lendify-app/Lendify-API
**Labels:** backend, feature, medium
**Difficulty:** medium

### Problem
The `VendorsModule` has a repository and service skeleton but exposes no creation endpoint. The mobile app's loan wizard relies on a vendor list, but admins currently have no API path to register a new vendor — vendor data has to be inserted directly via SQL.

### Context
Vendors are the only entities a learner can transact with — they're the "merchants" of Lendify. Onboarding the first 10–20 schools and bootcamps requires admins to be able to add vendors through Postman or the admin tool, not psql.

### Before Starting
Read these context files first:
- context/architecture-context.md
- context/code-standards.md
- context/progress-tracker.md
- src/modules/vendors/

### What To Build
1. Create `dto/create-vendor.dto.ts` with class-validator decorators: `name: string @IsString @MinLength(2)`, `type: VendorType (enum: school|bootcamp|certification|tool)`, `country: string @Length(2,2)`, `website: string @IsUrl`, `description: string @IsOptional @MaxLength(500)`.
2. Add `POST /api/v1/vendors` to `vendors.controller.ts`, guarded by `@UseGuards(JwtAuthGuard, AdminGuard)`, decorated with `@ApiTags('vendors')`, `@ApiOperation`, `@ApiResponse(201/400/401/403)`.
3. In `vendors.service.ts` add `createVendor(dto: CreateVendorDto): Promise<Vendor>` — calls Supabase repo, returns the inserted row.
4. Add `GET /api/v1/vendors` (paginated, public) and `GET /api/v1/vendors/:id` (public, 404 if missing).
5. Write a migration `supabase/migrations/NNN_create_vendors_table.sql` if the vendors table does not yet exist.
6. Add e2e test covering: admin can create, non-admin gets 403, invalid DTO gets 400, GET returns paginated list.

### Files To Touch
- `src/modules/vendors/vendors.controller.ts`
- `src/modules/vendors/vendors.service.ts`
- `src/modules/vendors/vendors.repository.ts`
- `src/modules/vendors/dto/create-vendor.dto.ts`
- `src/modules/vendors/dto/vendor.dto.ts`
- `supabase/migrations/NNN_create_vendors_table.sql` (if needed)
- `test/e2e/vendors.e2e-spec.ts`

### Acceptance Criteria
- [ ] POST /vendors is admin-guarded and returns 403 for non-admins
- [ ] DTO validation rejects malformed payloads with 400
- [ ] GET /vendors returns paginated results with `total`, `page`, `limit`
- [ ] GET /vendors/:id returns 404 for unknown IDs
- [ ] All endpoints have Swagger decorators
- [ ] E2E tests cover all 4 scenarios

### Mandatory Checks Before PR
- [ ] npm run build passes with zero TypeScript errors
- [ ] No new `any` types introduced anywhere
- [ ] Full Swagger @ApiOperation + @ApiResponse decorators on any new endpoints
- [ ] New migration file created for any schema changes
- [ ] context/progress-tracker.md updated

---

## Issue 11: Add learner profile creation on first login
**Repo:** Lendify-app/Lendify-API
**Labels:** backend, feature, medium
**Difficulty:** medium

### Problem
When a wallet hits `POST /auth/verify` for the first time, no row is inserted into `learner_profiles`. The mobile app then calls `GET /learners/me` and receives 404, producing a null-pointer error on the home screen rendering.

### Context
Every authenticated wallet should map to exactly one learner profile. The current race means the very first action a new user takes after signing in is to immediately hit an error state. Auto-create-on-first-login is the standard pattern.

### Before Starting
Read these context files first:
- context/architecture-context.md
- context/code-standards.md
- context/progress-tracker.md
- src/modules/auth/auth.service.ts
- src/modules/learners/learners.service.ts

### What To Build
1. In `auth.service.ts#verify()`, after successful signature validation and before issuing JWT, call `learnersService.ensureProfile(walletAddress)`.
2. Add `LearnersService#ensureProfile(wallet: string): Promise<LearnerProfile>`: query by wallet; if found, return; else insert with defaults `{ wallet_address: wallet, display_name: null, role: 'learner', created_at: now }`.
3. The operation must be idempotent — two concurrent verifies for the same wallet must produce exactly one row (use unique constraint on `wallet_address`).
4. Inject `LearnersService` into `AuthService` (handle circular dependency with `forwardRef`).
5. Add migration adding `UNIQUE(wallet_address)` to `learner_profiles` if not present.
6. Add unit test: first verify creates profile; second verify returns existing profile.

### Files To Touch
- `src/modules/auth/auth.service.ts`
- `src/modules/auth/auth.module.ts`
- `src/modules/learners/learners.service.ts`
- `src/modules/learners/learners.module.ts`
- `supabase/migrations/NNN_learner_profiles_unique_wallet.sql` (if missing)
- `test/unit/auth.service.spec.ts`

### Acceptance Criteria
- [ ] First-time verify creates a learner_profiles row
- [ ] Second verify for the same wallet does NOT create a duplicate
- [ ] GET /learners/me succeeds immediately after first verify
- [ ] DB has a unique constraint on `wallet_address`
- [ ] Unit tests cover both first-time and repeat paths
- [ ] Concurrent verifies for the same wallet produce one row

### Mandatory Checks Before PR
- [ ] npm run build passes with zero TypeScript errors
- [ ] No new `any` types introduced anywhere
- [ ] Full Swagger @ApiOperation + @ApiResponse decorators on any new endpoints
- [ ] New migration file created for any schema changes
- [ ] context/progress-tracker.md updated

---

## Issue 12: Add unit tests for `AuthService`
**Repo:** Lendify-app/Lendify-API
**Labels:** backend, testing, good first issue
**Difficulty:** good first issue

### Problem
`src/modules/auth/auth.service.ts` has zero unit tests. All authentication paths (nonce generation, signature verification, token rotation) are untested. Any regression here logs every user out and is undetectable until production.

### Context
Auth is the most security-sensitive module. A silent bug in token rotation could leak session lifetimes or allow refresh-token replay attacks. Unit tests are the cheapest line of defense.

### Before Starting
Read these context files first:
- context/architecture-context.md
- context/code-standards.md
- context/progress-tracker.md
- src/modules/auth/auth.service.ts

### What To Build
1. Create `test/unit/auth.service.spec.ts` with a `Test.createTestingModule` setup mocking `SupabaseService`, `JwtService`, and `ConfigService`.
2. Test `getNonce(wallet)`: returns a 32-char nonce, inserts a row in `nonces` with `wallet`, `nonce`, `expires_at`.
3. Test `verify(wallet, signature)`: valid signature path returns `{ accessToken, refreshToken }`; invalid signature throws `UnauthorizedException`.
4. Test `refresh(refreshToken)`: valid token returns new pair; refresh token is rotated (old one marked revoked).
5. Test nonce dedup: two `getNonce(wallet)` calls within expiry window return the same nonce.
6. Achieve ≥ 90% branch coverage on `auth.service.ts`.

### Files To Touch
- `test/unit/auth.service.spec.ts`

### Acceptance Criteria
- [ ] At least 5 distinct `describe` blocks covering each method
- [ ] All Supabase calls are mocked — no real DB hit
- [ ] Test runs in under 2 seconds
- [ ] `npm test -- auth.service` passes
- [ ] Branch coverage ≥ 90% on the service
- [ ] No `any` types in the test file

### Mandatory Checks Before PR
- [ ] npm run build passes with zero TypeScript errors
- [ ] No new `any` types introduced anywhere
- [ ] Full Swagger @ApiOperation + @ApiResponse decorators on any new endpoints
- [ ] New migration file created for any schema changes
- [ ] context/progress-tracker.md updated

---

## Issue 13: Add unit tests for `LearnersService`
**Repo:** Lendify-app/Lendify-API
**Labels:** backend, testing, good first issue
**Difficulty:** good first issue

### Problem
`src/modules/learners/learners.service.ts` has no unit tests. Profile read/write paths are unverified, so a typo in the Supabase select chain would silently return empty profiles.

### Context
Profile data drives the home screen and reputation display. A broken `getProfile` makes the entire learner experience appear empty. Cheap unit tests prevent silent breakage.

### Before Starting
Read these context files first:
- context/architecture-context.md
- context/code-standards.md
- context/progress-tracker.md
- src/modules/learners/learners.service.ts

### What To Build
1. Create `test/unit/learners.service.spec.ts` using `Test.createTestingModule` with mocked Supabase client.
2. Test `getProfile(wallet)`: returns a profile for a valid wallet; returns 404 (`NotFoundException`) for an unknown wallet.
3. Test `updateProfile(wallet, dto)`: persists changes and returns the updated row; rejects unknown wallet with 404.
4. Test `ensureProfile(wallet)`: returns existing profile if present; creates and returns new if missing.
5. Mock the supabase chain via `from().select().eq().single()` chained returns.
6. Achieve ≥ 85% branch coverage.

### Files To Touch
- `test/unit/learners.service.spec.ts`

### Acceptance Criteria
- [ ] All 3 service methods tested in distinct describes
- [ ] Both success and failure paths covered
- [ ] No real DB calls
- [ ] `npm test -- learners.service` passes
- [ ] Coverage ≥ 85% branches
- [ ] No `any` types

### Mandatory Checks Before PR
- [ ] npm run build passes with zero TypeScript errors
- [ ] No new `any` types introduced anywhere
- [ ] Full Swagger @ApiOperation + @ApiResponse decorators on any new endpoints
- [ ] New migration file created for any schema changes
- [ ] context/progress-tracker.md updated

---

## Issue 14: Add unit tests for `VendorsService`
**Repo:** Lendify-app/Lendify-API
**Labels:** backend, testing, good first issue
**Difficulty:** good first issue

### Problem
`src/modules/vendors/vendors.service.ts` has no unit tests. Listing, filtering, and creation logic is unverified, so pagination off-by-one bugs or filter mismatches go undetected.

### Context
The vendor list powers the loan application wizard — if it returns wrong results, learners borrow against the wrong merchant. Tests anchor expected behavior before the controller in Issue 10 is exercised.

### Before Starting
Read these context files first:
- context/architecture-context.md
- context/code-standards.md
- context/progress-tracker.md
- src/modules/vendors/vendors.service.ts

### What To Build
1. Create `test/unit/vendors.service.spec.ts` with `Test.createTestingModule` and mocked repo.
2. Test `listVendors({ page, limit, type? })`: returns paginated `{ data, total, page, limit }`; respects page+limit; filters by `type` when provided.
3. Test `getVendor(id)`: returns the vendor; throws `NotFoundException` for unknown id.
4. Test `createVendor(dto)`: persists and returns inserted row with generated id and `created_at`.
5. Edge case: `listVendors` with `page=0` or `limit=0` falls back to defaults.
6. Achieve ≥ 85% branch coverage.

### Files To Touch
- `test/unit/vendors.service.spec.ts`

### Acceptance Criteria
- [ ] All 3 service methods covered
- [ ] Pagination math is correct (off-by-one verified)
- [ ] Type filter is exercised with valid + invalid values
- [ ] `npm test -- vendors.service` passes
- [ ] Coverage ≥ 85%
- [ ] No `any` types

### Mandatory Checks Before PR
- [ ] npm run build passes with zero TypeScript errors
- [ ] No new `any` types introduced anywhere
- [ ] Full Swagger @ApiOperation + @ApiResponse decorators on any new endpoints
- [ ] New migration file created for any schema changes
- [ ] context/progress-tracker.md updated

---

## Issue 15: Add unit tests for `VouchingService`
**Repo:** Lendify-app/Lendify-API
**Labels:** backend, testing, good first issue
**Difficulty:** good first issue

### Problem
The `VouchingService` (mirror of the new on-chain vouching contract) has no unit tests. Vouch creation, retrieval, duplicate rejection, and expiry are all unverified at the service layer.

### Context
Vouching is a trust-bootstrap mechanism — a silent bug here could let one mentor inflate a learner's reputation unboundedly. Tests anchor the invariants before this goes live.

### Before Starting
Read these context files first:
- context/architecture-context.md
- context/code-standards.md
- context/progress-tracker.md
- src/modules/vouching/vouching.service.ts

### What To Build
1. Create `test/unit/vouching.service.spec.ts`.
2. Test `createVouch(mentor, learner)`: inserts a `vouches` row with status `active`; rejects with `ConflictException` if an active vouch already exists for the same pair.
3. Test `getVouches(learner)`: returns the list of active vouches for the learner; returns empty array if none.
4. Test `expireVouches()`: scans for rows with `expires_at < now()` and sets `status = 'expired'`; idempotent on re-run.
5. Mock Supabase client throughout.
6. Achieve ≥ 85% coverage.

### Files To Touch
- `test/unit/vouching.service.spec.ts`

### Acceptance Criteria
- [ ] 3 distinct describe blocks
- [ ] Duplicate vouch rejection verified
- [ ] Expiry cleanup verified for the boundary case `expires_at == now`
- [ ] `npm test -- vouching.service` passes
- [ ] Coverage ≥ 85%
- [ ] No `any` types

### Mandatory Checks Before PR
- [ ] npm run build passes with zero TypeScript errors
- [ ] No new `any` types introduced anywhere
- [ ] Full Swagger @ApiOperation + @ApiResponse decorators on any new endpoints
- [ ] New migration file created for any schema changes
- [ ] context/progress-tracker.md updated

---

## Issue 16: Add unit tests for `SponsorsService`
**Repo:** Lendify-app/Lendify-API
**Labels:** backend, testing, good first issue
**Difficulty:** good first issue

### Problem
`src/modules/sponsors/sponsors.service.ts` has no unit tests. Pool stats aggregation, deposit XDR construction, and withdrawal validation are all unverified.

### Context
Sponsors are the funding side of the protocol — any silent miscalculation in pool stats undermines trust. Tests are essential before sponsors put real funds in.

### Before Starting
Read these context files first:
- context/architecture-context.md
- context/code-standards.md
- context/progress-tracker.md
- src/modules/sponsors/sponsors.service.ts

### What To Build
1. Create `test/unit/sponsors.service.spec.ts`.
2. Test `getPool()`: returns aggregated `{ totalDeposited, totalShares, utilizationBps, apyBps }`; mocks repo + contract client.
3. Test `buildDepositXdr(wallet, amount)`: calls the liquidity client with correct args; returns the XDR; rejects amount ≤ 0.
4. Test `buildWithdrawXdr(wallet, shares)`: rejects shares below `MIN_WITHDRAWAL_SHARES`; otherwise returns valid XDR.
5. Test that pool utilization is computed as `(borrowed / totalDeposited) * 10_000` rounded to bps.
6. Achieve ≥ 85% coverage.

### Files To Touch
- `test/unit/sponsors.service.spec.ts`

### Acceptance Criteria
- [ ] All 3 service methods covered
- [ ] Minimum-withdrawal validation tested
- [ ] Pool math verified against a known fixture
- [ ] `npm test -- sponsors.service` passes
- [ ] Coverage ≥ 85%
- [ ] No `any` types

### Mandatory Checks Before PR
- [ ] npm run build passes with zero TypeScript errors
- [ ] No new `any` types introduced anywhere
- [ ] Full Swagger @ApiOperation + @ApiResponse decorators on any new endpoints
- [ ] New migration file created for any schema changes
- [ ] context/progress-tracker.md updated

---

## Issue 17: Implement vouch expiry cleanup BullMQ job
**Repo:** Lendify-app/Lendify-API
**Labels:** backend, jobs, medium
**Difficulty:** medium

### Problem
Vouches have an `expires_at` column but no background process marks them inactive once that timestamp passes. Stale vouches accumulate and continue inflating reputation scores indefinitely.

### Context
Vouches are time-bounded by design — a mentor vouching today shouldn't still affect a learner's score two years later without renewal. Without expiry, the trust signal decays into noise.

### Before Starting
Read these context files first:
- context/architecture-context.md
- context/code-standards.md
- context/progress-tracker.md
- src/jobs/nonce-cleanup/

### What To Build
1. Create `src/jobs/vouch-cleanup/vouch-cleanup.module.ts`, `vouch-cleanup.processor.ts`, `vouch-cleanup.service.ts` mirroring the nonce-cleanup layout.
2. Register a hourly repeatable job with BullMQ: `every: 60 * 60 * 1000` ms, named `vouch-cleanup`.
3. Processor `handleCleanup()`: calls `vouchingService.expireVouches()` and logs the affected row count.
4. Register the module in `app.module.ts`.
5. Add structured log line via `Logger`: `{ job: 'vouch-cleanup', expired: N, took_ms: T }`.
6. Add integration test that seeds 3 vouches (1 expired, 2 active) and asserts only 1 is marked expired.

### Files To Touch
- `src/jobs/vouch-cleanup/vouch-cleanup.module.ts`
- `src/jobs/vouch-cleanup/vouch-cleanup.processor.ts`
- `src/jobs/vouch-cleanup/vouch-cleanup.service.ts`
- `src/app.module.ts`
- `test/integration/vouch-cleanup.spec.ts`

### Acceptance Criteria
- [ ] Job runs every 60 minutes via BullMQ scheduler
- [ ] Processor logs the count of expired vouches per run
- [ ] Module is registered in `app.module.ts`
- [ ] Integration test seeds + verifies cleanup
- [ ] Job is idempotent (a second immediate run touches zero rows)
- [ ] No business logic in the processor — delegates to service

### Mandatory Checks Before PR
- [ ] npm run build passes with zero TypeScript errors
- [ ] No new `any` types introduced anywhere
- [ ] Full Swagger @ApiOperation + @ApiResponse decorators on any new endpoints
- [ ] New migration file created for any schema changes
- [ ] context/progress-tracker.md updated

---

## Issue 18: Add `POST /auth/refresh` endpoint tests
**Repo:** Lendify-app/Lendify-API
**Labels:** backend, testing, good first issue
**Difficulty:** good first issue

### Problem
The `POST /auth/refresh` endpoint exists in `auth.controller.ts` but `test/e2e/auth.e2e-spec.ts` only covers nonce + verify. Token-rotation behavior (single-use refresh tokens) is unverified end-to-end.

### Context
Refresh-token rotation is a security-critical pattern: a reused refresh token must be rejected, otherwise stolen tokens give attackers indefinite access. Without an e2e test, this regression could slip into production silently.

### Before Starting
Read these context files first:
- context/architecture-context.md
- context/code-standards.md
- context/progress-tracker.md
- src/modules/auth/auth.controller.ts
- test/e2e/auth.e2e-spec.ts

### What To Build
1. Extend `test/e2e/auth.e2e-spec.ts` with a new `describe('POST /auth/refresh')` block.
2. Test happy path: sign in, receive `{ accessToken, refreshToken }`, call refresh with the refresh token, receive a new pair, assert tokens differ from previous.
3. Test expired refresh token: forge an expired token (sign with past `exp`), assert 401.
4. Test rotation/reuse detection: use refresh token once successfully, then attempt to reuse the same one — assert 401 and that the user's session is revoked entirely.
5. Test malformed token: send garbage, assert 401.
6. Use the e2e test harness that already exists (Supabase test schema + Nest test app).

### Files To Touch
- `test/e2e/auth.e2e-spec.ts`

### Acceptance Criteria
- [ ] 4 new test cases in a single describe block
- [ ] Rotation reuse case verifies session revocation
- [ ] All cases run against a real (test) Supabase schema
- [ ] `npm run test:e2e` passes
- [ ] No flake on 10 consecutive runs
- [ ] No `any` types

### Mandatory Checks Before PR
- [ ] npm run build passes with zero TypeScript errors
- [ ] No new `any` types introduced anywhere
- [ ] Full Swagger @ApiOperation + @ApiResponse decorators on any new endpoints
- [ ] New migration file created for any schema changes
- [ ] context/progress-tracker.md updated

---

## Issue 19: Add reputation score caching with Redis
**Repo:** Lendify-app/Lendify-API
**Labels:** backend, performance, medium
**Difficulty:** medium

### Problem
`GET /api/v1/reputation/:wallet` calls the reputation Soroban contract on every request. Each call costs an RPC round trip (~300–800ms). Mobile screens that show reputation on render produce noticeable lag.

### Context
Reputation scores change only on loan events (creation, repayment, late fee). Between events, the score is stable for hours or days — perfect for a short Redis cache. This is the single biggest UX win for the home screen.

### Before Starting
Read these context files first:
- context/architecture-context.md
- context/code-standards.md
- context/progress-tracker.md
- src/modules/reputation/reputation.service.ts

### What To Build
1. Inject `Redis` client into `ReputationService`.
2. Add `REPUTATION_CACHE_TTL_SEC` to `.env.example` with default `300`.
3. In `getScore(wallet)`: first `GET reputation:{wallet}` from Redis; if hit, return parsed; if miss, call contract, write to Redis with TTL, return.
4. Add `invalidate(wallet)` method that `DEL reputation:{wallet}`.
5. In `LoansService.repay()` and `LoansService.create()`, call `reputationService.invalidate(borrowerWallet)` after every contract write.
6. Add unit tests covering cache-hit, cache-miss, and invalidation paths.

### Files To Touch
- `src/modules/reputation/reputation.service.ts`
- `src/modules/reputation/reputation.module.ts`
- `src/modules/loans/loans.service.ts`
- `.env.example`
- `test/unit/reputation.service.spec.ts`

### Acceptance Criteria
- [ ] Cache hit returns in < 10 ms
- [ ] Cache miss populates Redis with the configured TTL
- [ ] Loan creation + repayment invalidate the cache for that wallet
- [ ] `REPUTATION_CACHE_TTL_SEC` is configurable via env
- [ ] 3 unit tests cover hit/miss/invalidate
- [ ] No stale read after invalidation in any test

### Mandatory Checks Before PR
- [ ] npm run build passes with zero TypeScript errors
- [ ] No new `any` types introduced anywhere
- [ ] Full Swagger @ApiOperation + @ApiResponse decorators on any new endpoints
- [ ] New migration file created for any schema changes
- [ ] context/progress-tracker.md updated

---

## Issue 20: Add Sentry error tracking to all modules
**Repo:** Lendify-app/Lendify-API
**Labels:** backend, observability, good first issue
**Difficulty:** good first issue

### Problem
`@sentry/nestjs` is installed in `package.json` but `SentryModule.forRoot()` is not registered in `app.module.ts`, and `main.ts` does not initialize Sentry. Unhandled exceptions are logged to stdout only — invisible in production.

### Context
Render's free-tier logs roll over fast. Without an external error sink, real production issues vanish before the team sees them. Sentry is the table-stakes minimum.

### Before Starting
Read these context files first:
- context/architecture-context.md
- context/code-standards.md
- context/progress-tracker.md
- package.json
- src/main.ts
- src/app.module.ts

### What To Build
1. Call `Sentry.init({ dsn: process.env.SENTRY_DSN, environment: process.env.NODE_ENV, tracesSampleRate: 0.1 })` at the very top of `main.ts`, before `NestFactory.create`.
2. Register `SentryModule.forRoot()` in `app.module.ts`.
3. Add `SentryGlobalFilter` to `APP_FILTER` so all unhandled exceptions report to Sentry.
4. Add `SENTRY_DSN=` and `NODE_ENV=development` to `.env.example`.
5. Verify integration locally with a deliberate `throw new Error('sentry test')` in a test endpoint — confirm event lands in Sentry.
6. Add a README section "Error tracking" describing how to enable Sentry.

### Files To Touch
- `src/main.ts`
- `src/app.module.ts`
- `.env.example`
- `README.md`

### Acceptance Criteria
- [ ] Sentry initializes before Nest bootstrap
- [ ] Unhandled exceptions in any controller reach Sentry
- [ ] App still starts when `SENTRY_DSN` is empty (no-op fallback)
- [ ] `traces_sample_rate` is configurable via env
- [ ] README documents the env vars
- [ ] No `any` types introduced

### Mandatory Checks Before PR
- [ ] npm run build passes with zero TypeScript errors
- [ ] No new `any` types introduced anywhere
- [ ] Full Swagger @ApiOperation + @ApiResponse decorators on any new endpoints
- [ ] New migration file created for any schema changes
- [ ] context/progress-tracker.md updated

---

## Issue 21: Add rate limiting to auth endpoints
**Repo:** Lendify-app/Lendify-API
**Labels:** backend, security, medium
**Difficulty:** medium

### Problem
`POST /auth/nonce` and `POST /auth/verify` have no rate limit. An attacker can hammer signature verification (CPU-expensive) or nonce issuance (DB write) to exhaust resources or fish for valid signatures.

### Context
Auth endpoints are the most exposed surface. Free-tier Render has limited CPU and DB connections; one bad actor can degrade service for all real users. `@nestjs/throttler` is the standard mitigation.

### Before Starting
Read these context files first:
- context/architecture-context.md
- context/code-standards.md
- context/progress-tracker.md
- src/modules/auth/auth.controller.ts

### What To Build
1. Confirm `@nestjs/throttler` is installed; if not, add it and register `ThrottlerModule.forRoot([{ ttl: 60_000, limit: 5 }])` in `app.module.ts`.
2. Add `@UseGuards(ThrottlerGuard)` to `AuthController` if not globally enabled.
3. Decorate `POST /auth/nonce` with `@Throttle({ default: { limit: 5, ttl: 60_000 } })`.
4. Decorate `POST /auth/verify` with same.
5. Configure throttler keyGenerator to use client IP + wallet address combined, so one IP cannot exhaust limits for multiple wallets.
6. Add e2e test: 6 rapid requests from the same IP receive 429 on the 6th.

### Files To Touch
- `src/modules/auth/auth.controller.ts`
- `src/app.module.ts`
- `test/e2e/auth.e2e-spec.ts`

### Acceptance Criteria
- [ ] 5 requests within 60 seconds succeed; 6th returns 429
- [ ] Limits are configurable via env (`AUTH_THROTTLE_LIMIT`, `AUTH_THROTTLE_TTL_MS`)
- [ ] Throttler key includes IP — multiple wallets from one IP share the limit
- [ ] E2E test passes
- [ ] No regression in other endpoints
- [ ] Swagger response 429 documented on both endpoints

### Mandatory Checks Before PR
- [ ] npm run build passes with zero TypeScript errors
- [ ] No new `any` types introduced anywhere
- [ ] Full Swagger @ApiOperation + @ApiResponse decorators on any new endpoints
- [ ] New migration file created for any schema changes
- [ ] context/progress-tracker.md updated

---

## Issue 22: Add pagination to `GET /loans` endpoint
**Repo:** Lendify-app/Lendify-API
**Labels:** backend, performance, good first issue
**Difficulty:** good first issue

### Problem
`GET /api/v1/loans` in `loans.controller.ts` returns every row in the loans table with no `LIMIT`. As the protocol grows, this becomes a slow query and a payload bomb on the mobile client.

### Context
The mobile loan history screen needs paginated access — infinite scroll, not a single 50MB JSON response. Adding pagination now is cheap; retrofitting it after the UI exists is expensive.

### Before Starting
Read these context files first:
- context/architecture-context.md
- context/code-standards.md
- context/progress-tracker.md
- src/modules/loans/loans.controller.ts

### What To Build
1. Create `dto/loan-list-query.dto.ts` with `page: number @IsInt @Min(1) @Default(1)`, `limit: number @IsInt @Min(1) @Max(50) @Default(10)`.
2. Update `LoansController.list(@Query() q: LoanListQueryDto)` signature.
3. Update `LoansService.list(q)` to apply `range((page - 1) * limit, page * limit - 1)` on Supabase, and return `{ data, total, page, limit }`.
4. Add `Total-Count` response header for clients that prefer headers over body.
5. Update Swagger: `@ApiQuery({ name: 'page', required: false })`, etc.
6. Add e2e test: insert 12 loans, request page=2 with limit=10 — assert returns 2 items and `total=12`.

### Files To Touch
- `src/modules/loans/loans.controller.ts`
- `src/modules/loans/loans.service.ts`
- `src/modules/loans/dto/loan-list-query.dto.ts`
- `test/e2e/loans.e2e-spec.ts`

### Acceptance Criteria
- [ ] Default page=1, limit=10 when query params omitted
- [ ] `limit > 50` rejected with 400
- [ ] Response body shape: `{ data, total, page, limit }`
- [ ] Swagger documents query params
- [ ] E2E test verifies pagination math
- [ ] No regression on existing consumers (data field still an array)

### Mandatory Checks Before PR
- [ ] npm run build passes with zero TypeScript errors
- [ ] No new `any` types introduced anywhere
- [ ] Full Swagger @ApiOperation + @ApiResponse decorators on any new endpoints
- [ ] New migration file created for any schema changes
- [ ] context/progress-tracker.md updated

---

## Issue 23: Implement `stellar.toml` endpoint
**Repo:** Lendify-app/Lendify-API
**Labels:** backend, integrations, good first issue
**Difficulty:** good first issue

### Problem
There is no `GET /.well-known/stellar.toml` endpoint. Wallets and indexers that auto-discover Stellar projects (Lobstr directory, StellarExpert) cannot find Lendify's federation file from the API domain.

### Context
`stellar.toml` is the discoverability standard for Stellar projects — it advertises org info, contract IDs, and curated asset metadata. Wallets read it to display the project name and verify contract addresses.

### Before Starting
Read these context files first:
- context/architecture-context.md
- context/code-standards.md
- context/progress-tracker.md
- src/modules/health/health.controller.ts

### What To Build
1. Add a `StellarTomlController` (or extend `HealthController`) exposing `GET /.well-known/stellar.toml`.
2. Set `Content-Type: text/plain; charset=utf-8` and `Access-Control-Allow-Origin: *`.
3. Body content: org name, website, github, contracts block listing `CREDITLINE_CONTRACT_ID`, `LIQUIDITY_POOL_CONTRACT_ID`, `REPUTATION_CONTRACT_ID`, `VENDOR_REGISTRY_CONTRACT_ID`, all read from `ConfigService`.
4. Cache the response in memory with 1-hour TTL (cheap to regenerate, but no need to do it per request).
5. Add Swagger decorators marking it as a static metadata endpoint.
6. Add e2e test asserting status 200, content-type text/plain, and contract IDs present.

### Files To Touch
- `src/modules/health/stellar-toml.controller.ts`
- `src/modules/health/health.module.ts`
- `test/e2e/stellar-toml.e2e-spec.ts`

### Acceptance Criteria
- [ ] GET /.well-known/stellar.toml returns 200 with text/plain body
- [ ] Body includes all 4 contract IDs
- [ ] CORS allows `*` for this endpoint specifically
- [ ] Cached in memory for 60 minutes
- [ ] E2E test verifies content
- [ ] Swagger lists the endpoint

### Mandatory Checks Before PR
- [ ] npm run build passes with zero TypeScript errors
- [ ] No new `any` types introduced anywhere
- [ ] Full Swagger @ApiOperation + @ApiResponse decorators on any new endpoints
- [ ] New migration file created for any schema changes
- [ ] context/progress-tracker.md updated

---

## Issue 24: Build WalletConnect v2 integration
**Repo:** Lendify-app/Lendify-App
**Labels:** mobile, blockchain, hard
**Difficulty:** hard

### Problem
The mobile app has no wallet integration. Users cannot connect Lobstr, xBull, or Freighter to sign transactions. Every flow that requires signing (loan apply, repay, deposit) is currently dead-end.

### Context
WalletConnect v2 is the Stellar-wallet standard. Without it, the entire transactional surface of Lendify-App is non-functional. This is the gate to running any e2e flow.

### Before Starting
Read these context files first:
- context/architecture-context.md
- context/code-standards.md
- context/progress-tracker.md
- context/ui-context.md
- services/auth.service.ts

### What To Build
1. Install `@walletconnect/web3wallet` and Stellar adapter; configure project ID via `EXPO_PUBLIC_WALLETCONNECT_PROJECT_ID`.
2. Create `services/wallet.service.ts` exposing: `initWalletConnect()`, `connectWallet(): Promise<{ address, sessionId }>`, `signXdr(xdr: string): Promise<string>`, `disconnectWallet()`.
3. Show a QR-code modal in the connect flow using `react-native-qrcode-svg`; on mobile, deep-link to wallet app if installed.
4. Create `stores/wallet.store.ts` (Zustand): `{ address, sessionId, isConnected, connect, disconnect }`. Persist to SecureStore.
5. Handle session expiry — listener that calls `disconnect()` on session_delete event.
6. Add a `useWallet()` hook wrapping the store for use in screens.

### Files To Touch
- `services/wallet.service.ts`
- `stores/wallet.store.ts`
- `hooks/useWallet.ts`
- `components/wallet/ConnectModal.tsx`
- `app.config.ts` (env var)
- `package.json`

### Acceptance Criteria
- [ ] User can connect Lobstr/xBull/Freighter via QR scan
- [ ] Session persists across app restarts (SecureStore)
- [ ] `signXdr` returns a signed XDR or throws on user rejection
- [ ] Disconnect clears all wallet state
- [ ] Session expiry auto-clears state without crash
- [ ] No hardcoded hex colors in any new UI

### Mandatory Checks Before PR
- [ ] No hardcoded hex colors — use constants/colors.ts only
- [ ] No API calls in screen files — use services/ only
- [ ] Loading, error, AND empty states handled
- [ ] Lucide React Native used for ALL icons
- [ ] npx expo export --platform web passes
- [ ] context/progress-tracker.md updated

---

## Issue 25: Build sign-in screen with sliding onboarding
**Repo:** Lendify-app/Lendify-App
**Labels:** mobile, ui, medium
**Difficulty:** medium

### Problem
`app/(auth)/sign-in.tsx` is a placeholder. There is no onboarding flow introducing Lendify's value proposition, features, or reputation tiers before asking the user to connect a wallet — leading to immediate drop-off.

### Context
First-time users have no context for "connect wallet" — they need a 30-second onboarding explaining what Lendify does and why they should trust it. The sliding 4-step pattern matches industry norms (Robinhood, Cash App).

### Before Starting
Read these context files first:
- context/architecture-context.md
- context/code-standards.md
- context/progress-tracker.md
- context/ui-context.md
- constants/colors.ts

### What To Build
1. `app/(auth)/sign-in.tsx`: a `FlatList` with `pagingEnabled`, `horizontal`, `showsHorizontalScrollIndicator={false}`, 4 slides.
2. Slide 1 (Welcome): logo + tagline "Climb your credit stairs" + staircase illustration. Use `colors.background.primary` and `colors.text.primary`.
3. Slide 2 (Features): 3 rows with Lucide icons (`GraduationCap`, `TrendingUp`, `Shield`) and descriptions.
4. Slide 3 (Reputation Tiers): 4 horizontally scrollable cards for Starter / Bronze / Silver / Gold with tier-color borders from `colors.tiers.*`.
5. Slide 4 (Connect Wallet): wallet option buttons (Lobstr, xBull, Freighter) that call the `useWallet` hook from Issue 24.
6. Page indicator dots at the bottom; "Skip" button top-right jumps to slide 4.

### Files To Touch
- `app/(auth)/sign-in.tsx`
- `components/onboarding/OnboardingSlide.tsx`
- `components/onboarding/PageIndicator.tsx`
- `components/onboarding/WalletOptions.tsx`

### Acceptance Criteria
- [ ] 4 slides render and paginate smoothly
- [ ] All colors come from `constants/colors.ts` — zero hardcoded hex
- [ ] All icons are Lucide React Native
- [ ] Skip button works from any slide
- [ ] Connecting a wallet on slide 4 navigates to `role-select`
- [ ] Works on `npx expo export --platform web`

### Mandatory Checks Before PR
- [ ] No hardcoded hex colors — use constants/colors.ts only
- [ ] No API calls in screen files — use services/ only
- [ ] Loading, error, AND empty states handled
- [ ] Lucide React Native used for ALL icons
- [ ] npx expo export --platform web passes
- [ ] context/progress-tracker.md updated

---

## Issue 26: Build role selection screen
**Repo:** Lendify-app/Lendify-App
**Labels:** mobile, ui, good first issue
**Difficulty:** good first issue

### Problem
After wallet connection, there is no role-selection step. The app currently assumes everyone is a learner, which breaks the sponsor experience entirely.

### Context
Learner and sponsor flows are fundamentally different — different home screens, different actions, different tab bars. The role must be picked once at onboarding and persisted.

### Before Starting
Read these context files first:
- context/architecture-context.md
- context/code-standards.md
- context/progress-tracker.md
- context/ui-context.md
- constants/colors.ts

### What To Build
1. `app/(auth)/role-select.tsx`: two large tappable cards, vertically stacked, 80% screen width, 240px tall each.
2. Learner card: `GraduationCap` icon (Lucide, 48px), title "I'm a Learner", subtitle "Build credit for school", border color `colors.tiers.silver` (blue).
3. Sponsor card: `TrendingUp` icon, title "I'm a Sponsor", subtitle "Fund learners, earn yield", border color `colors.semantic.success` (green).
4. Tap stores `role` in `stores/user.store.ts` and animates a checkmark badge on the selected card.
5. CTA button at bottom: "Continue" — disabled until a role is selected, navigates to `app/(auth)/register.tsx`.
6. Loading / error / empty states unused here, but ensure no flicker on mount.

### Files To Touch
- `app/(auth)/role-select.tsx`
- `components/auth/RoleCard.tsx`
- `stores/user.store.ts`

### Acceptance Criteria
- [ ] Two cards render with correct icons and borders from colors.ts
- [ ] Tapping a card marks it selected with checkmark
- [ ] Continue is disabled until selection
- [ ] Selection persists in user.store.ts
- [ ] No hardcoded hex
- [ ] `npx expo export --platform web` passes

### Mandatory Checks Before PR
- [ ] No hardcoded hex colors — use constants/colors.ts only
- [ ] No API calls in screen files — use services/ only
- [ ] Loading, error, AND empty states handled
- [ ] Lucide React Native used for ALL icons
- [ ] npx expo export --platform web passes
- [ ] context/progress-tracker.md updated

---

## Issue 27: Build learner home dashboard screen
**Repo:** Lendify-app/Lendify-App
**Labels:** mobile, ui, hard
**Difficulty:** hard

### Problem
`app/(tabs)/index.tsx` is a placeholder. The learner has no home dashboard showing credit available, active loans, or upcoming payments — the core screen of the app.

### Context
The home screen is the first thing a learner sees every session. It must surface: credit ceiling, current debt, next payment due, recent activity. Without it, the app has no anchor screen.

### Before Starting
Read these context files first:
- context/architecture-context.md
- context/code-standards.md
- context/progress-tracker.md
- context/ui-context.md
- services/loans.service.ts
- services/reputation.service.ts

### What To Build
1. Header row: greeting "Hi, {first-letters-of-wallet}" + truncated wallet address (e.g. `GABC...XYZ`) with copy icon.
2. Credit available card (full width): big number "Available: $X", progress bar (used / ceiling), tier label from reputation.
3. Quick actions row: 4 round buttons with Lucide icons (`Plus` Apply, `CreditCard` Pay, `History` History, `Users` Vouches), each navigates to its respective screen.
4. Active loans horizontal scroll: cards showing vendor logo, amount, next due date.
5. Upcoming payments list: 3-row vertical list with date, amount, "Pay now" button.
6. Pull-to-refresh re-fetches via `hooks/useLoans` and `hooks/useReputation`. Loading skeletons + empty state ("No active loans yet — apply for credit") + error state with retry.

### Files To Touch
- `app/(tabs)/index.tsx`
- `hooks/useLoans.ts`
- `hooks/useReputation.ts`
- `components/home/CreditCard.tsx`
- `components/home/QuickActionsRow.tsx`
- `components/home/ActiveLoanCard.tsx`
- `components/home/UpcomingPaymentRow.tsx`

### Acceptance Criteria
- [ ] All 4 sections render with real data from services
- [ ] No fetch logic in the screen file — only hooks
- [ ] Loading skeletons render during fetch
- [ ] Empty state shows when no loans
- [ ] Error state shows retry button
- [ ] No hardcoded hex; all Lucide icons

### Mandatory Checks Before PR
- [ ] No hardcoded hex colors — use constants/colors.ts only
- [ ] No API calls in screen files — use services/ only
- [ ] Loading, error, AND empty states handled
- [ ] Lucide React Native used for ALL icons
- [ ] npx expo export --platform web passes
- [ ] context/progress-tracker.md updated

---

## Issue 28: Build loan application wizard
**Repo:** Lendify-app/Lendify-App
**Labels:** mobile, ui, hard
**Difficulty:** hard

### Problem
There is no loan application flow. Learners cannot actually borrow — `app/(tabs)/apply.tsx` is empty.

### Context
This is the second most important screen after home. Without it, the app is read-only. The wizard pattern (multi-step bottom sheet) matches the user mental model of "filling out an application".

### Before Starting
Read these context files first:
- context/architecture-context.md
- context/code-standards.md
- context/progress-tracker.md
- context/ui-context.md
- services/loans.service.ts
- services/vendors.service.ts

### What To Build
1. Step 1 — Vendor: search bar + category filter chips (school/bootcamp/cert/tool). Tapping a vendor advances to step 2.
2. Step 2 — Details: amount slider (min/max from credit ceiling), installment-schedule selector (3/6/12 months), live preview of monthly payment + total interest using `useReputation` interest rate.
3. Step 3 — Review: summary card listing vendor, amount, schedule, interest, total; warning banner about late-fee policy; CTA "Sign with Wallet" calls `wallet.signXdr` from Issue 24.
4. Step 4 — Success: green checkmark, tx-hash row (tap to copy / open in StellarExpert), next payment summary, CTA "Back to home".
5. Step transitions animated horizontally; back button preserves prior state.
6. Loading + error states on step 3 sign action; empty vendor list state on step 1.

### Files To Touch
- `app/(tabs)/apply.tsx`
- `components/loan-wizard/Step1Vendor.tsx`
- `components/loan-wizard/Step2Details.tsx`
- `components/loan-wizard/Step3Review.tsx`
- `components/loan-wizard/Step4Success.tsx`
- `hooks/useVendors.ts`
- `hooks/useLoanQuote.ts`

### Acceptance Criteria
- [ ] All 4 steps render and transition
- [ ] Back button preserves prior step state
- [ ] Step 2 preview recalculates on slider drag
- [ ] Step 3 sign error shows toast + keeps user on step 3
- [ ] Empty vendor list shows empty state
- [ ] All colors from colors.ts; all icons Lucide

### Mandatory Checks Before PR
- [ ] No hardcoded hex colors — use constants/colors.ts only
- [ ] No API calls in screen files — use services/ only
- [ ] Loading, error, AND empty states handled
- [ ] Lucide React Native used for ALL icons
- [ ] npx expo export --platform web passes
- [ ] context/progress-tracker.md updated

---

## Issue 29: Build reputation score screen
**Repo:** Lendify-app/Lendify-App
**Labels:** mobile, ui, medium
**Difficulty:** medium

### Problem
There is no UI for a learner to see their reputation score, tier, or path to the next tier. The score is opaque, undermining the gamified "climb the stairs" core mechanic.

### Context
Reputation is the protocol's loyalty loop. Making it visible and progressive (showing distance to next tier) drives behavior — on-time payments, vouching, building history. Hidden, it's just a number.

### Before Starting
Read these context files first:
- context/architecture-context.md
- context/code-standards.md
- context/progress-tracker.md
- context/ui-context.md
- services/reputation.service.ts

### What To Build
1. Animated circular progress ring (200×200 px) with the score 0–100 in the center, color from `colors.tiers.<current>`.
2. Tier badge below ring: tier name + tier color, e.g. "🥈 Silver Tier".
3. Stats row (3 columns): interest rate, credit limit, loans repaid.
4. "How to reach next tier" section: progress bar showing distance to next threshold and bullet list of actions ("Pay 2 more loans on time", "Get 1 mentor vouch").
5. Score history line chart (last 6 months) using `react-native-chart-kit` or `victory-native` — y-axis 0–100, x-axis month labels.
6. Pull-to-refresh, loading + empty + error states.

### Files To Touch
- `app/(tabs)/reputation.tsx`
- `components/reputation/ScoreRing.tsx`
- `components/reputation/TierBadge.tsx`
- `components/reputation/NextTierProgress.tsx`
- `components/reputation/ScoreHistoryChart.tsx`
- `hooks/useReputation.ts`

### Acceptance Criteria
- [ ] Ring animates from 0 to score on mount
- [ ] Tier color matches `colors.tiers.<tier>`
- [ ] Chart renders 6 data points
- [ ] No tier hardcoded — read from data
- [ ] All icons Lucide; no hardcoded hex
- [ ] Empty state when wallet has no history

### Mandatory Checks Before PR
- [ ] No hardcoded hex colors — use constants/colors.ts only
- [ ] No API calls in screen files — use services/ only
- [ ] Loading, error, AND empty states handled
- [ ] Lucide React Native used for ALL icons
- [ ] npx expo export --platform web passes
- [ ] context/progress-tracker.md updated

---

## Issue 30: Build settings screen with role switcher
**Repo:** Lendify-app/Lendify-App
**Labels:** mobile, ui, medium
**Difficulty:** medium

### Problem
`app/(tabs)/settings.tsx` is empty. Users cannot edit their profile, switch roles, manage notifications, or disconnect their wallet.

### Context
Settings is the universal escape hatch — fix profile typos, log out, switch sides. Many users will spend less than 1 minute total here but it must exist or the app feels half-finished.

### Before Starting
Read these context files first:
- context/architecture-context.md
- context/code-standards.md
- context/progress-tracker.md
- context/ui-context.md
- stores/auth.store.ts
- stores/user.store.ts

### What To Build
1. Profile card: avatar (initials), display name, wallet address with copy icon, edit button.
2. Learner profile card: school, program, income type — editable fields stored via `learners.service`.
3. Role switcher: pill toggle (Learner ↔ Sponsor) bound to `user.store`. Switch triggers a re-route to the correct tab layout.
4. App section: notification toggle, language picker (English/Spanish/French).
5. Security section: connected wallet (with disconnect button calling `walletService.disconnectWallet`), session expiry timer.
6. About section: version, terms, privacy policy, support email.

### Files To Touch
- `app/(tabs)/settings.tsx`
- `components/settings/ProfileCard.tsx`
- `components/settings/RoleSwitcher.tsx`
- `components/settings/SettingRow.tsx`
- `hooks/useUserProfile.ts`

### Acceptance Criteria
- [ ] Profile fields persist via the learners service
- [ ] Role switcher updates the tab layout immediately
- [ ] Disconnect clears wallet state and routes to sign-in
- [ ] Notification + language preferences persist locally
- [ ] All icons Lucide; no hardcoded hex
- [ ] Loading/error states on profile fetch

### Mandatory Checks Before PR
- [ ] No hardcoded hex colors — use constants/colors.ts only
- [ ] No API calls in screen files — use services/ only
- [ ] Loading, error, AND empty states handled
- [ ] Lucide React Native used for ALL icons
- [ ] npx expo export --platform web passes
- [ ] context/progress-tracker.md updated

---

## Issue 31: Build sponsor portfolio screen
**Repo:** Lendify-app/Lendify-App
**Labels:** mobile, ui, medium
**Difficulty:** medium

### Problem
Sponsor users have no portfolio view. There's no screen showing deposit value, share price, APY, or pool utilization. Sponsors can't see what they've funded.

### Context
Sponsors need a Robinhood-style portfolio view — total value, daily change, action buttons. Without it, sponsoring is invisible and they have no reason to come back.

### Before Starting
Read these context files first:
- context/architecture-context.md
- context/code-standards.md
- context/progress-tracker.md
- context/ui-context.md
- services/sponsors.service.ts

### What To Build
1. Total Deposited card: big number "$X", APY badge ("8.4% APY"), 24h change indicator (+/- color).
2. Pool stats row (3 columns): your shares, share price, interest earned to date.
3. Pool health bar: visualization of locked vs available liquidity, with utilization percentage.
4. Recent activity list: deposits + withdrawals + interest accruals with timestamps.
5. Floating action button (FAB): "Deposit" — opens a bottom sheet flow that calls `sponsorsService.buildDepositXdr` then `walletService.signXdr`.
6. Pull-to-refresh, loading + empty + error states.

### Files To Touch
- `app/(tabs)/sponsor-home.tsx`
- `components/sponsor/TotalDepositedCard.tsx`
- `components/sponsor/PoolStatsRow.tsx`
- `components/sponsor/PoolHealthBar.tsx`
- `components/sponsor/ActivityList.tsx`
- `components/sponsor/DepositSheet.tsx`
- `hooks/useSponsorPortfolio.ts`

### Acceptance Criteria
- [ ] Portfolio renders with mocked-then-real data via the sponsors service
- [ ] Deposit FAB opens a working signing flow
- [ ] Pool health bar visualizes utilization correctly
- [ ] Activity list paginates beyond 10 entries
- [ ] All icons Lucide; no hardcoded hex
- [ ] Empty state shown for new sponsor (zero deposits)

### Mandatory Checks Before PR
- [ ] No hardcoded hex colors — use constants/colors.ts only
- [ ] No API calls in screen files — use services/ only
- [ ] Loading, error, AND empty states handled
- [ ] Lucide React Native used for ALL icons
- [ ] npx expo export --platform web passes
- [ ] context/progress-tracker.md updated

---

## Issue 32: Add GitHub Actions CI for Lendify-API
**Repo:** Lendify-app/Lendify-API
**Labels:** devops, ci, good first issue
**Difficulty:** good first issue

### Problem
The Lendify-API repo has no GitHub Actions workflow. PRs can merge with broken TypeScript builds or failing tests because nothing enforces them.

### Context
Open-source contributors will submit PRs blind. CI is the gate that prevents broken builds from landing in main. Without it, the maintainer has to manually run `npm run build` on every PR.

### Before Starting
Read these context files first:
- context/architecture-context.md
- context/code-standards.md
- context/progress-tracker.md

### What To Build
1. Create `.github/workflows/ci.yml`.
2. Triggers: `pull_request` to main + `push` to main.
3. Job `build-test`: ubuntu-latest, Node 20 via `actions/setup-node@v4` with `cache: 'npm'`.
4. Steps: checkout → setup node → `npm ci` → `npm run build` → `npm test`.
5. Cache `node_modules` keyed on `package-lock.json` hash.
6. Add CI status badge to README pointing at the workflow.

### Files To Touch
- `.github/workflows/ci.yml`
- `README.md`

### Acceptance Criteria
- [ ] Workflow runs on every PR to main
- [ ] Failing build blocks merge
- [ ] node_modules cached between runs
- [ ] Total run time under 5 minutes
- [ ] Status badge visible at top of README
- [ ] No secrets exposed in workflow logs

### Mandatory Checks Before PR
- [ ] CI passes on a test PR before merging
- [ ] Zero secrets committed to the repo
- [ ] context/progress-tracker.md updated

---

## Issue 33: Add GitHub Actions CI for Lendify-App
**Repo:** Lendify-app/Lendify-App
**Labels:** devops, ci, good first issue
**Difficulty:** good first issue

### Problem
The mobile app repo has no CI workflow. A typo in any component can break the Expo build and only be discovered when EAS build fails hours later.

### Context
Expo builds are slow and rate-limited. Catching compile errors at PR time via a cheap `expo export --platform web` is dramatically faster than waiting for an EAS run.

### Before Starting
Read these context files first:
- context/architecture-context.md
- context/code-standards.md
- context/progress-tracker.md

### What To Build
1. Create `.github/workflows/ci.yml` in Lendify-App.
2. Triggers: `pull_request` and `push` to main.
3. Job `web-build`: ubuntu-latest, Node 20, cache npm.
4. Steps: checkout → setup-node → `npm ci` → `npx expo export --platform web`.
5. Upload web build as artifact for inspection.
6. Status badge in README.

### Files To Touch
- `.github/workflows/ci.yml`
- `README.md`

### Acceptance Criteria
- [ ] Workflow runs on every PR to main
- [ ] Failed Expo export blocks merge
- [ ] node_modules cached
- [ ] Web bundle uploaded as artifact
- [ ] Status badge in README
- [ ] Run time under 7 minutes

### Mandatory Checks Before PR
- [ ] CI passes on a test PR before merging
- [ ] Zero secrets committed to the repo
- [ ] context/progress-tracker.md updated

---

## Issue 34: Add EAS production build workflow
**Repo:** Lendify-app/Lendify-App
**Labels:** devops, ci, medium
**Difficulty:** medium

### Problem
There is no automated EAS build pipeline. Every release requires the maintainer to manually run `eas build --platform android --profile production` and upload the APK, which delays releases and risks human error.

### Context
Releases gated by manual steps don't happen on schedule. Automating production builds on `v*` tag push means tagging a commit produces a downloadable APK with zero further action.

### Before Starting
Read these context files first:
- context/architecture-context.md
- context/code-standards.md
- context/progress-tracker.md
- eas.json

### What To Build
1. Create `.github/workflows/eas-build.yml`.
2. Trigger: `push` with `tags: ['v*']`.
3. Job `production-android`: ubuntu-latest, Node 20.
4. Steps: checkout → setup-node → `npm ci` → install `eas-cli` → `eas build --platform android --profile production --non-interactive --no-wait`.
5. Auth via `EXPO_TOKEN` secret (must be added in repo settings — document in workflow comment).
6. After job, run `eas build:list --json` and download APK, upload as release asset via `actions/upload-release-asset`.

### Files To Touch
- `.github/workflows/eas-build.yml`
- `eas.json` (verify production profile exists)
- `README.md` (release process section)

### Acceptance Criteria
- [ ] Pushing tag `v0.1.0` triggers an EAS production build
- [ ] APK is attached to the corresponding GitHub release
- [ ] `EXPO_TOKEN` secret is documented as required
- [ ] No secrets leaked in logs
- [ ] eas.json has a production profile with `buildType: apk` or `aab`
- [ ] Release process documented in README

### Mandatory Checks Before PR
- [ ] CI passes on a test PR before merging
- [ ] Zero secrets committed to the repo
- [ ] context/progress-tracker.md updated

---

## Issue 35: Add Render deployment health check workflow
**Repo:** Lendify-app/Lendify-API
**Labels:** devops, observability, good first issue
**Difficulty:** good first issue

### Problem
The Render free-tier instance hosting Lendify-API spins down after inactivity. First request after sleep takes 30+ seconds, and outages are silent — no one knows the service is down until a user complains.

### Context
A simple periodic ping keeps the instance warm and acts as a heartbeat monitor. If the ping fails, a GitHub issue is auto-created so the maintainer sees the outage even without external monitoring tools.

### Before Starting
Read these context files first:
- context/architecture-context.md
- context/code-standards.md
- context/progress-tracker.md

### What To Build
1. Create `.github/workflows/health-check.yml`.
2. Trigger: `schedule: cron '0 */6 * * *'` (every 6 hours) + `workflow_dispatch` (manual).
3. Step 1: `curl -sf -o /dev/null -w "%{http_code}" https://lendify-api.onrender.com/api/v1/health` — capture status.
4. Step 2: if status != 200, use `actions/github-script` to create an issue titled "Health check failed at {timestamp}" with the HTTP status in the body. Include label `incident`.
5. Step 3: dedupe — if an open `incident` issue already exists, comment on it instead of creating a duplicate.
6. Document the ping endpoint in README.

### Files To Touch
- `.github/workflows/health-check.yml`
- `README.md`

### Acceptance Criteria
- [ ] Workflow runs every 6 hours via cron
- [ ] Manual trigger works
- [ ] Non-200 response opens a GitHub issue with label `incident`
- [ ] Duplicate issues are not created — comments on existing ones
- [ ] Ping URL and label documented in README
- [ ] Workflow uses no committed secrets

### Mandatory Checks Before PR
- [ ] CI passes on a test PR before merging
- [ ] Zero secrets committed to the repo
- [ ] context/progress-tracker.md updated

---

_End of issues — 35 total._
