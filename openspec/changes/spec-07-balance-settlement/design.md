## Context

Spec 05 stores transactions with `amountInMainCurrency` (signed) per member (via BankAccount→HouseholdMember). Spec 01 defined `Settlement` (householdId, billingMonth, status DRAFT|CONFIRMED, unique [householdId, billingMonth]) and `SettlementTransfer` (settlementId, fromMemberId, toMemberId, amount). This spec computes the nets and the minimal transfer set.

The user-specified algorithm: "Dad makes $6,000, Mom makes $2,000. Dad's expenses were $5,000, Mom's expenses were $2,500. Dad's net = +$1,000, Mom's net = −$500. Household net = +$500. Dad transfers Mom $500 to settle." — this is per-member net + min-transfers.

## Goals / Non-Goals

**Goals:**
- Per-member net: sum(income) − sum(expenses) in main currency, per billing month.
- Min-transfers algorithm: minimal set of transfers from surplus members to deficit members.
- Settlement stored as DRAFT; confirmable to CONFIRMED (immutable).
- UI: member nets + suggested transfers + confirm action.
- Full UI states.

**Non-Goals:**
- Automatic transfers, recalculation of confirmed settlements, proportional splits, multi-month history.

## Decisions

### D1: Net calculation
**Decision:** For each ACTIVE household member with transactions in the billing month, compute `net = SUM(amountInMainCurrency)` across their transactions (positive = surplus, negative = deficit). Members with no transactions have a net of 0. The household total net is the sum of all member nets (shows whether the household saved or drew from savings).
**Rationale:** Matches the user's example exactly. Using `amountInMainCurrency` ensures consistent currency.
**Alternatives considered:**
- Equal split of household net: ignores who earned/spent what. Rejected per user decision.

### D2: Min-transfers algorithm
**Decision:** Implement the greedy min-transfers algorithm: sort surplus members descending, deficit members ascending (by absolute deficit). Match the largest surplus to the largest deficit, create a transfer for the smaller of the two, reduce both, repeat. This produces a minimal (not necessarily optimal, but near-optimal and simple) set of transfers. Each transfer has `fromMemberId` (surplus), `toMemberId` (deficit), `amount` (in main currency).
**Rationale:** Simple, well-known, produces fewer transfers than naive pairwise. Optimal min-transfers is NP-hard in general; the greedy approach is standard for settlement apps.
**Alternatives considered:**
- Optimal (DFS/backtracking): NP-hard, overkill for household sizes (2-6 members). Rejected.
- Naive (everyone sends to everyone): too many transfers. Rejected.

### D3: Settlement lifecycle
**Decision:** `GET /api/households/[id]/settlement?billingMonth=YYYY-MM` computes (or returns the stored) settlement. If no `Settlement` row exists for that month, compute on-the-fly and return as DRAFT (do NOT persist until confirmed — the user may add transactions that change the calculation). `POST /api/households/[id]/settlement?billingMonth=YYYY-MM` confirms: persist a `Settlement(CONFIRMED)` + `SettlementTransfer` rows (unique constraint prevents re-confirming). Once CONFIRMED, the settlement is immutable; GET returns the stored version. Any ACTIVE member can view; only ADMIN can confirm.
**Rationale:** DRAFT is computed on demand (reflects latest transactions); CONFIRMED is immutable (historical record). Admin-only confirmation prevents accidental locks.
**Alternatives considered:**
- Persist DRAFT on every GET: creates churn, stale drafts. Rejected.
- Any member can confirm: risk of accidental lock. Rejected.

### D4: Notification on confirm
**Decision:** When a settlement is confirmed, create a `Notification(SETTLEMENT_READY)` row for each **ACTIVE** household member with `userId` = member, `householdId` = household, `emailStatus = PENDING`, and `payload = { householdId, householdName, billingMonth, settlementId }`. The `Settlement`, `SettlementTransfer`, and `Notification` inserts MUST happen in a single DB transaction (atomicity). `Settlement.calculatedAt` is set to `now()` on confirm. Email dispatch is Spec 09.
**Rationale:** Members need to know the settlement is finalized so they can execute their transfers. `householdName` is included so Spec 09's email template can render it without an extra lookup. ACTIVE scoping respects Spec 01's access-revoked rule. Transactional inserts prevent a confirmed settlement with zero emails.

## Risks / Trade-offs

- [Greedy algorithm not always optimal] → Mitigation: for household sizes (2-6 members), greedy produces at most N-1 transfers which is typically optimal. Acceptable.
- [Confirmed settlement immutability] → Mitigation: if transactions are added after confirmation (late sync), they won't be reflected. Document that settlements should be confirmed after the billing month closes. A future spec could support "reopen."
- [Member with no transactions] → Mitigation: net = 0, no transfers suggested for them. They appear in the member list with a 0 net.
