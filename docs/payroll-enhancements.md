# Payroll Enhancement Features

This document describes the lightweight enhancements added to improve payroll workflow validation and audit trail without exposing sensitive data.

## Per-Period Payout Count Guard (#545)

**Purpose**: Track and validate the number of payouts within a single payroll period to prevent excessive transactions and detect anomalies.

**Implementation**:
- Added `payout_count` field to `PeriodUsage` struct
- Guard enforced in `enforce_and_record_capacity()` function
- Max payout count configurable in `CapacityLimits`
- Privacy-safe: count increments without revealing individual salary amounts

**Error Handling**:
- Returns privacy-safe error code `CapacityLimitKind::PayoutCount` when exceeded
- Does not expose employee names or salary values in error messages

**Usage**:
```rust
// Configure limits including payout count
payroll.set_capacity_limits(admin, CapacityLimits {
    max_batches: 50,
    max_employees: 1000,
    max_total_value: 10_000_000,
    max_payouts: 5000,  // New field
});
```

## Payout Destination Blacklist Validation (#550)

**Purpose**: Block payments to flagged addresses (e.g., sanctioned wallets, compromised accounts).

**Implementation**:
- New `blacklisted_destinations` storage map
- Validation occurs before payment execution in `batch_process_payroll()`
- Admin-only management via `add_blacklisted_destination()` / `remove_blacklisted_destination()`
- Privacy-safe: rejection does not reveal which employee triggered the block

**Error Handling**:
- Returns generic `PayrollError::PayoutDestinationBlocked` error
- Event emits only boolean flag, not employee address

**Usage**:
```rust
// Add address to blacklist
payroll.add_blacklisted_destination(admin, flagged_address);

// Check if address is blacklisted
if payroll.is_destination_blacklisted(employee_address) {
    // Handle blocked payout
}
```

## Audit Event for Payroll Draft Expiration (#540)

**Purpose**: Provide audit trail when drafts expire to support compliance and operational monitoring.

**Implementation**:
- Enhanced `emit_draft_expired` event with additional context
- New `emit_draft_expiration_audit()` function in events module
- Includes: draft_id, expiration_reason_code, admin_actor, timestamp
- Privacy-safe: no salary amounts or employee data exposed

**Event Schema**:
```rust
emit_draft_expiration_audit(
    env,
    draft_id,          // u64
    reason_code,       // Symbol (e.g., "timeout", "admin_action")
    admin,             // Address
    created_timestamp, // u64
    expired_timestamp  // u64
);
```

**Reason Codes**:
- `timeout` - Draft exceeded configured expiration window
- `admin_action` - Manually expired by admin
- `replaced` - Superseded by new draft for same period

## Payroll Period Amendment Audit Trail (#505)

**Purpose**: Record all authorized amendments to payroll periods with actor, timestamp, and privacy-safe reason codes.

**Implementation**:
- New `PeriodAmendment` struct stored per amendment
- Tracks: amendment_id, period_label, admin_actor, reason_code, amended_at
- Emits `period_amended` event for off-chain indexing
- Cumulative counter `amendment_count` per period for quick audit checks

**Privacy Protection**:
- Uses coded reason symbols (e.g., "correction", "late_addition", "rate_change")
- Does not store or emit salary amounts, employee names, or commitment values
- Audit log queryable only by admin role

**Event Schema**:
```rust
emit_period_amended(
    env,
    period,            // Symbol
    amendment_id,      // u64
    admin,             // Address
    reason_code,       // Symbol
    amendment_count    // u32 - total amendments for this period
);
```

**Usage**:
```rust
// Amend a period (internally records audit trail)
payroll.amend_run_draft(admin, draft_id, new_total, new_count);

// Query amendment history (admin only)
let amendments = payroll.get_period_amendments(period);
for amendment in amendments {
    log!(
        "Period {} amended by {} at {} (reason: {})",
        amendment.period,
        amendment.admin,
        amendment.amended_at,
        amendment.reason_code
    );
}
```

## Testing

All features include:
- Unit tests for happy path
- Edge case validation (e.g., boundary conditions, unauthorized access)
- Privacy verification (sensitive data not exposed in errors/events)

Test files:
- `tests/payout_count_guard_test.rs`
- `tests/blacklist_validation_test.rs`
- `tests/draft_expiration_audit_test.rs`
- `tests/period_amendment_audit_test.rs`

## Compliance Notes

These enhancements support compliance and audit requirements while maintaining zero-knowledge privacy properties:
- No salary amounts exposed in events or errors
- No employee identities revealed in validation failures
- Audit trails use privacy-safe codes instead of descriptive text
- All changes follow repository security conventions
