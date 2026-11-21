# Migrating off a deployed OurDAO

OurDAO is deliberately immutable once deployed (see [Known limitations](../README.md#known-limitations)) — there is no upgrade path, and there never will be one. That's a trust-minimization choice, not an oversight. But it leaves one real question unanswered until now: **if a bug is found in a deployed OurDAO, what exactly do the members do?**

This document is that procedure. It is a design and process document — it changes no contract code. The one piece of contract surface it assumes, a `seed_state` entrypoint on the *new* contract, is specified precisely enough to implement but is **not implemented here**; that's deliberately a separate, follow-up issue, same as any other new entrypoint.

There is no way around the core trust tradeoff this creates: **whoever runs a migration is trusted to seed the new contract truthfully.** Nothing on-chain proves the seeded state matches the old contract's real history. What makes that checkable after the fact is (a) `ourdao-backend`'s append-only event log, which every member can independently replay, and (b) the old contract's own views, which stay live and publicly readable for the entire migration window. A migration is legitimate to the extent members can — and did — check it against those two things themselves, not because the procedure was followed.

## Table of contents

- [State inventory](#state-inventory)
- [Procedure](#procedure)
- [Edge cases: mid-migration loans and proposals](#edge-cases-mid-migration-loans-and-proposals)
- [TTL / state-archival risk](#ttl--state-archival-risk)
- [The `seed_state` entrypoint (design only)](#the-seed_state-entrypoint-design-only)
- [Trust assumption](#trust-assumption)

## State inventory

Every `DataKey` variant in `contracts/dao/src/storage.rs` is accounted for
below. "View" means read live from the old contract before it is decommissioned.
"Event" means reconstructible from `ourdao-backend`'s indexed event log. Where
the contract has no public view, the table says so and identifies whether a
view is needed or the state is derived/excluded. Where an event supplies the
address/id set and a view supplies its current value, both are listed.

| `DataKey` variant(s) | Read source | Notes |
|---|---|---|
| `Admins` | View: `get_admins()` | Fully enumerable. |
| `Threshold` | View: `get_consensus_threshold()` | |
| `MembershipFee` | Event: `init` | Included in initialization event, but there is no public getter; add `get_membership_fee()` for a direct state read. |
| `Token` | View: `get_token()` | The configured base asset; decide whether the new deployment keeps it. |
| `Policy` | View: `get_loan_policy()` | Includes all `LoanPolicy` fields. |
| `Paused` | View: `is_paused()` or `get_dao_summary()` | The old deployment remains paused after migration. |
| `Pauser` | Events: `pset`, `prev` | No public getter; replay events for the current assignment and add `get_pauser()` for direct verification. The new contract's seed configuration must also include the assignment. |
| `PendingPolicyUpdate` | View: `get_pending_policy_update()` | Capture pending policy and execution time, or cancel/resolve it before migration. |
| `Members`, `TotalMembers`, `ActiveMembers` | Views: `list_members(start, limit)`, `get_total_members()`, `get_active_members()`; events: `joined`, `exited` | Page the list and cross-check its address set against events. `list_members` skips records whose storage expired. |
| `Member` | View: `get_member(address)`; also `get_member_loan_stats(address)` / `get_member_overview(address)` | Per member. The member record includes lifetime loan counts and active-loan status; use the overview when collecting related member fields. |
| `TotalContributions` | No public view; recompute from member records if needed | This legacy counter is not used by current exit calculations and is not maintained as an authoritative total; exclude it from the seed. |
| `ActiveContributions` | No public view; derive from active `Member.contribution` values | Required for exit-share calculations. Recompute it from seeded active members; add a getter so the result can be verified before unlocking. |
| `NextProposalId`, `NextTreasuryId` | Views: `get_loan_proposal_count()`, `get_treasury_proposal_count()` | These counts are the next ids because ids are allocated from zero. |
| `LoanProposal` | View: `get_loan_proposal(id)`; events: `loan_req` and proposal lifecycle events | In-flight proposals are recommended to lapse and be resubmitted, not seeded. |
| `Loan` | View: `get_loan(id)`; ids from `loan_req` / `loan_appr` events | `amount_repaid` and `status` give the outstanding balance. |
| `TreasuryProposal` | View: `get_treasury_proposal(id)`; ids from `tre_prop` events | In-flight proposals are recommended to lapse and be resubmitted, not seeded. |
| `LoanAsset` | Event: `loan_asst`; base token is the default when no asset was recorded | No public asset getter; add `get_loan_asset(id)`. The current `seed_loans` design also needs to carry the asset if active loans are migrated. |
| `TreasuryAsset` | Event: `tre_asset`; base token is the default when no asset was recorded | No public asset getter; add `get_treasury_asset(id)`. If a treasury proposal is ever migrated, its seed format must carry the asset too. |
| `LoanVoted`, `TreasuryVoted` | View: `has_voted(kind, proposal_id, voter)`; events: `loan_vote`, `tre_vote` | Do not seed vote records when the associated in-flight proposals are allowed to lapse. |
| `Commit` | Event: `committed`; no public view of the commitment value | Exclude with its in-flight private treasury proposal; do not migrate partial commit-reveal state. |
| `Stake`, `TotalStaked` | Views: `get_stake(address)`, `get_dao_summary().total_staked`; events: `staked`, `unstaked` | Stake is base-token-only. Read each member's amount and reconcile the sum with the total. |
| `StakeTime` | Event: latest `staked` event for the member (use its ledger timestamp) | No public getter; add `get_stake_time(address)` and include the timestamp in stake seeding. It gates unstaking cooldowns and cannot be recovered from `get_stake()` alone. |
| `PendingYield` | View: `get_pending_yield(address)`; `exited` event as a historical cross-check | Read the computed claim live; do not copy this legacy key independently. If storage has expired, document any unrecoverable claim rather than guessing. |
| `YieldAccumulator`, `MemberYieldSnapshot` | View: `get_pending_yield(address)` per member | Internal implementation detail; seed the computed pending amount with `seed_pending_yield` and a fresh accumulator as described below, rather than copying these keys. |
| `YieldRemainder` | No public view or event value | Base-token interest remainder rolls into later distributions. Add a getter and seed support to preserve it exactly; it is not included in any member's current pending yield. |
| `OpenProposals` | Derived by scanning proposal records; no public getter | Recompute on the new contract. Since in-flight proposals are not migrated, do not seed this cache. |
| `Doc` | View: `get_document(kind, id)`; ids from `doc_attn` events | Only the hash is on-chain; the underlying content needs no migration. |
| `Name`, `NameOf` | Views: `resolve_name(name)`, `name_of(owner)`; names/owners from `name_reg` events | Neither view enumerates all names. Include names held by non-members from the event index. |
| `Delegation` | View: `get_delegate(delegator)` or `get_member_overview(address)`; event: `delegate` | A direct view exists; read per member. Add a `seed_delegations` step to the migration design. |
| `TokenWhitelisted`, `WhitelistedTokens` | Views: `is_token_whitelisted(token)`, `get_whitelisted_tokens()`; events: `tok_add`, `tok_rem` | The base token is implicitly whitelisted; the list view returns it first. Keep removed assets in the historical candidate set. Add a whitelist seeding step for the extra assets. |
| `PendingTokenAddition` | View: `get_pending_token_addition(token)`; candidate addresses from `tok_prop` events | Read for every token proposed for addition; do not carry pending additions over. Re-propose after migration if still wanted. |
| `MigrationSeedingLocked` | View: `is_seeding_locked()` | Record the old contract's lock state; the new deployment must remain unlocked until verification is complete. |
| `OutstandingPrincipal` | View: `get_treasury_accounting(asset)` per asset | Reconcile against active loans in that asset; zero after all loans in that asset are repaid or defaulted. |
| External token balance (not a `DataKey`) | View: `get_treasury_accounting(asset).token_balance` | Read for every current or historically removed asset; use whitelist (`tok_add`, `tok_rem`) and loan/proposal asset events to enumerate candidates. The view also reports stake and outstanding principal. |

This table intentionally distinguishes public views from internal storage
accessors. In particular, delegation already has `get_delegate()`; it does not
need a new view, although the seed design needs to include it. Add public views
for `MembershipFee`, `Pauser`, `ActiveContributions`, loan/treasury proposal
assets, `StakeTime`, and `YieldRemainder` before relying on direct on-chain
reads of those values. The seed design must also carry pauser assignment,
whitelisted assets, delegation, stake timestamps, loan/proposal asset metadata,
and yield remainder. `TotalContributions`, vote/commit records, and
`OpenProposals` are explicitly excluded or derived as noted above.

One structural note: **names can only be read per name or per owner address**,
so a name held by an address that is no longer a member has to come from the
event log. Member addresses and loan, proposal and document ids can be
enumerated from the live contract (`list_members` and the proposal counters).
The event log is still the independent cross-check on all of them. If
`ourdao-backend` has not been running continuously since deployment, that
cross-check is missing and this procedure cannot be completed — see the RPC
caveat below.

> Public Soroban RPC retains only ~24h of ledger history. "Just replay the event log" is only possible because `ourdao-backend` has (assumed) been indexing continuously since deployment into its own durable store — RPC itself is not that store.

## Procedure

1. **Freeze the address/id sets.** Query `ourdao-backend` for every member address, loan id, proposal id, name, and `(kind, id)` doc pair as of a chosen cutoff ledger. This defines exactly what the live-view reads in the next steps need to cover.
2. **`pause()` the old contract.** This is an admin call and the first on-chain action of the migration. Know precisely what it does and doesn't stop (see the table below) — it is not a full freeze.
3. **Read every category in the [state inventory](#state-inventory) live**, for every address/id from step 1. Do this as soon as possible after pausing — see [TTL risk](#ttl--state-archival-risk) for why waiting is dangerous. `scripts/export-state.sh <contract-id> --out state.json` runs these reads in one pass and warns when members with expired storage are missing from the export.
4. **Settle active loans.** Pausing does *not* block `repay_loan`, `repay_loan_partial`, or the permissionless `mark_loan_defaulted`. Give borrowers a defined settlement window (e.g. matching the existing `default_grace_period`) to repay in full or in part; any loan still active and overdue at the end of the window gets called via `mark_loan_defaulted` before state is re-read for the final time. This is the recommended path over carrying a still-active loan's `due_time`/`status` across into the new deployment, which would otherwise force the new contract to reproduce default-timing continuity for state it didn't originate. See [edge cases](#edge-cases-mid-migration-loans-and-proposals) for the fallback if a clean settlement window isn't acceptable.
5. **Resolve in-flight proposals.** Recommended: let them lapse (loan proposals auto-expire via `expire_loan_proposal`; treasury proposals auto-expire via `expire_treasury_proposal` once `VOTING_PERIOD` has elapsed — call it on each outstanding proposal id after the voting period passes). Do not carry partial vote tallies into the new contract. See [edge cases](#edge-cases-mid-migration-loans-and-proposals).
6. **Deploy the new contract** and `initialize()` it normally (admins, threshold, membership fee, token, policy — from the values read in step 3).
7. **Seed the new contract's state** via the (not-yet-implemented, see below) `seed_state` entrypoint(s), from the data captured in step 3.
8. **Independent verification window.** Before normal operation resumes, publish the full seeded dataset (or its hashes) so members can diff it against their own replay of the event log and their own last-known balances. This is the only real check on the trust assumption below — don't skip it or rush it.
9. **Lock seeding** on the new contract (see [below](#the-seed_state-entrypoint-design-only)) and resume normal operation. Publicly retire the old contract's frontend/indexing so members aren't misled into interacting with a decommissioned deployment; the old contract itself cannot be "shut off" beyond `pause()`, so its `pause()` state stays permanent.

### Version Reporting Post-Seeding / Pre-Lock (#532)
Immediately post-seeding and prior to `lock_seeding`, `get_dao_summary().version` and `get_version()` always report the *new deployment's* own `CARGO_PKG_VERSION` from its build binary, with no reference to the prior contract's version. This is the intended behavior: the newly deployed contract identifies itself by its own codebase version rather than masquerading as the deprecated deployment. Any future need for migration lineage tracking (e.g. recording the predecessor contract's address or prior migration hash) would require an explicit lineage field in storage and ABI, tracked as a separate follow-up task.

Step 2's `pause()` blocks: `register_member`, `propose_treasury_withdrawal`, `vote_on_treasury_proposal`, `commit_treasury_vote`, `reveal_treasury_vote`, `request_loan`, `vote_on_loan_proposal`. It does **not** block: `repay_loan`, `repay_loan_partial`, `exit_dao`, `claim_rewards`, `mark_loan_defaulted`, `expire_loan_proposal`, `stake`, `unstake`, `register_name`, `attach_document`. Steps 3 onward have to account for state legitimately continuing to change in the unblocked paths until the live reads in step 3 are taken as final.

## Edge cases: mid-migration loans and proposals

**A member with an active loan mid-migration.** The recommended path (step 4 above) settles every loan — via repayment or default — before the final state read, so no active loan ever needs to be represented in the seed data at all. If a project chooses not to impose a settlement window (e.g. the DAO wants migration to complete faster than borrowers can reasonably repay), the fallback is to carry the `Loan` struct across verbatim, including `due_time`. That due date was set against the old contract's ledger timestamps, which are directly portable (Soroban ledger time is a real Unix timestamp, not contract-relative), so `mark_loan_defaulted`'s existing time comparison keeps working unmodified on the new contract post-seed. The cost is that the new contract's very first blocks of activity include a borrower who never interacted with it, which is unusual but not unsound.

**A proposal mid-vote.** Recommended: require resubmission after migration rather than carrying over partial tallies. Carrying a proposal over would mean seeding not just the proposal but every individual vote record (`LoanVoted`/`TreasuryVoted` per voter) to prevent double-voting on the new contract, which meaningfully complicates the seed format for a comparatively low-value class of state — proposals are ephemeral by design, and members who already discussed and voted for something will not find re-proposing it costly. Note this in whatever member communication accompanies the migration, since it does mean real, if minor, lost momentum on anything mid-vote at cutoff.

## TTL / state-archival risk

Every persistent storage entry (members, loans, proposals, votes, stakes, yield snapshots, docs, name records) extends its TTL on every read *or* write (90-day bump — see `storage.rs`). An entry nobody has touched in 90 days is eligible for state archival: it becomes unreadable through the normal `get_*` views without an explicit restore operation first, and depending on how long it's been archived, restoration may not be possible at all.

This matters specifically for step 3 above: if the migration is triggered by a bug discovery rather than routine housekeeping, some members' entries — particularly low-activity members who haven't interacted with the contract in a while — may already be close to or past their TTL by the time the migration actually runs. Concretely:

- **Read every category as early as possible**, ideally as the very next action after `pause()`, not after the settlement window in step 4. A member whose entry expires between pause and the final read is a member whose pending yield (in particular — see the inventory table) becomes genuinely unrecoverable.
- If any entries are already archived when the migration starts, they need an explicit restore (`stellar contract restore` or equivalent) before they're readable again — budget for this as a real possibility, not an edge case, for any DAO that's been live for several months with low-activity members.
- Instance-storage singletons (admin set, threshold, policy, token, the yield accumulator itself) bump on a shorter 30-day cycle but are touched by nearly every call, so in practice they're at far lower risk than a specific inactive member's persistent entries.

## The `seed_state` entrypoint (design only)

Not implemented here — this is a specification for a follow-up issue, on the **new** contract only (the old, already-deployed contract's ABI cannot change).

**Shape.** Chunked rather than a single call: a live DAO's member/loan/proposal counts can exceed what fits in one transaction's resource limits, so seeding needs to be splittable across multiple calls without any ordering requirement between chunks (each entry keyed by its own address/id, so a partial batch is safe to retry or reorder):

```
fn seed_members(env: Env, caller: Address, members: Vec<Member>) -> Result<(), Error>
fn seed_stakes(env: Env, caller: Address, stakes: Vec<(Address, i128)>) -> Result<(), Error>
fn seed_pending_yield(env: Env, caller: Address, yields: Vec<(Address, i128)>) -> Result<(), Error>
fn seed_loans(env: Env, caller: Address, loans: Vec<Loan>) -> Result<(), Error>
fn seed_names(env: Env, caller: Address, names: Vec<(String, Address)>) -> Result<(), Error>
fn seed_docs(env: Env, caller: Address, docs: Vec<(ProposalKind, u32, Bytes)>) -> Result<(), Error>
fn lock_seeding(env: Env, caller: Address) -> Result<(), Error>
```

**Pending yield seeding, specifically.** The pull-based accumulator (`YieldAccumulator` + per-member `MemberYieldSnapshot`, see `loans::distribute_interest`) makes "set this member's pending yield to X" not a direct field write. The clean way to seed it without touching `claim_rewards`/`distribute_interest` at all: leave the new contract's `YieldAccumulator` at its natural starting value (0), and for a member with pending yield `X`, set their `MemberYieldSnapshot` to `-X`. `get_pending_yield` computes `(accumulator - snapshot).max(0)`, so `(0 - (-X)).max(0) == X` — the existing formula reconstructs the seeded value with no special-casing.

**Authorization model.** Admin-only (`require_admin`, same as every other privileged entrypoint), and one-shot in aggregate: every seed function checks a `MigrationSeedingLocked` instance-storage flag and rejects if set, and `lock_seeding` is the only way to set that flag — irreversibly, no `unlock`. Call `lock_seeding` only once independent verification (procedure step 8) is done. Before that lock, seeding is idempotent per key (re-seeding the same address/id overwrites, so a bad chunk can be corrected by re-sending it) but is otherwise unrestricted — this is exactly the trust-concentration point the next section is about.

**Why not fold this into `initialize`?** Splitting it out means the normal `initialize` path (a fresh, non-migrated DAO) doesn't carry any migration-only surface area, and it lets seeding happen in as many transactions as the real member count requires instead of being bounded by what a single `initialize` call could hold.

## Trust assumption

Say this plainly, because burying it is how it stops being checked: **admins seeding a migrated contract are trusted to seed it truthfully.** The contract has no way to verify that a seeded `Member` record, loan, or stake actually matches what the old contract held — `seed_state` accepts whatever the caller passes. Nothing about the design in this document changes that; it can't, because verifying arbitrary historical state on-chain against a contract that's being decommissioned isn't a problem Soroban gives us primitives for.

What makes a migration accountable, then, isn't the contract — it's procedure step 8. The seeded dataset (or a hash of it, with the raw data published off-chain) has to be checkable by any member against `ourdao-backend`'s event log and against their own last interaction with the old contract, *before* `lock_seeding` closes the window. A migration nobody checked is a migration that was trusted on faith, regardless of how carefully this document was followed.
