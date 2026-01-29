# Bug #5: TIP-20 Overflow Detection Missing Detailed Logging

## Severity
Medium - Debugging difficulty

## Affected Component
TIP-20 Token Precompile (`crates/precompiles/src/tip20/mod.rs`)

## Description
The TIP-20 token precompile detects overflow/underflow conditions in critical operations (mint, transfer, burn) but does not provide sufficient logging information when these errors occur. This makes debugging overflow scenarios extremely difficult in production environments.

When an overflow is detected, the precompile returns a generic `PanicKind::UnderOverflow` error without logging:
- Which operation failed (mint/transfer/burn)
- Which account was involved
- Current balance/supply values
- The amount that caused the overflow
- Supply cap constraints

## Root Cause
The overflow detection uses `ok_or(TempoPrecompileError::under_overflow())` which creates the error inline without context:

```rust
// BEFORE (insufficient logging)
let new_supply = total_supply
    .checked_add(amount)
    .ok_or(TempoPrecompileError::under_overflow())?;
```

This should be replaced with `ok_or_else(|| { ... })` to enable logging at the error site:

```rust
// AFTER (with detailed logging)
let new_supply = total_supply
    .checked_add(amount)
    .ok_or_else(|| {
        tracing::error!(
            total_supply = %total_supply,
            amount = %amount,
            supply_cap = %self.supply_cap(),
            "TIP20 mint: total supply overflow detected"
        );
        TempoPrecompileError::under_overflow()
    })?;
```

## Affected Operations

1. **Mint - Total Supply Overflow**: When `total_supply + amount` exceeds u128::MAX or supply_cap
2. **Mint - Balance Overflow**: When `recipient_balance + amount` would overflow
3. **Transfer - Sender Underflow**: When `sender_balance < amount` (insufficient balance)
4. **Transfer - Recipient Overflow**: When `recipient_balance + amount` would overflow
5. **Burn - Opted-in Supply Underflow**: When opted-in supply accounting underflows
6. **Fee Refund - Opted-in Supply Overflow**: When opted-in supply accounting overflows during refund

## Steps to Reproduce

### Scenario 1: Mint Total Supply Overflow

1. Deploy a TIP-20 token with a supply cap set to a specific value
2. Mint tokens up to the supply cap
3. Attempt to mint additional tokens that would exceed the cap
4. Observe that the transaction reverts with `PanicKind::UnderOverflow`
5. Check logs - no detailed information about:
   - Current total supply
   - Amount being minted
   - Supply cap value

**Expected**: Logs showing the exact values that caused overflow
**Actual**: Generic error with no diagnostic information

```bash
# Test case
cargo test test_mint_increases_balance_and_supply -- --nocapture
```

### Scenario 2: Mint Balance Overflow

1. Create a TIP-20 token
2. Mint `u128::MAX` tokens to an address
3. Attempt to mint additional tokens to the same address
4. Transaction reverts with overflow
5. No logs indicate which address or what balance/amount values were involved

### Scenario 3: Transfer Recipient Overflow

1. Mint `u128::MAX` tokens to address A
2. Transfer tokens from A to address B until B also has `u128::MAX`
3. Attempt to transfer more tokens to B
4. Transaction reverts with overflow
5. No logs identify which recipient overflowed or what the values were

### Scenario 4: Burn with Opted-in Supply Tracking

1. User opts into rewards with `setRewardRecipient(self)`
2. User has tokens and receives rewards
3. User transfers out or burns tokens
4. Opted-in supply accounting underflows/overflows
5. No logs indicate which user's reward accounting failed

## Testing Steps

### 1. Verify Current Behavior (Before Fix)

```bash
cd tempo

# The tests will pass but won't show detailed error logs
cargo test --package tempo-precompiles --lib tip20::tests::test_mint_increases_balance_and_supply

# Check for missing logging
grep -n "tracing::error" crates/precompiles/src/tip20/mod.rs
# Expected: Only find a few or no matches
```

### 2. Trigger Overflow in Localnet

```bash
# Start localnet
just genesis 1000 ./localnet
just localnet

# In a separate terminal, use cast to trigger overflow
# Get an account with tokens first
cast rpc <rpc_url> eth_blockNumber

# Deploy and run a contract that attempts to overflow
# (See test/overflow_trigger.sol for example)
```

### 3. Foundry Test

Create a test file `test/TIP20Overflow.t.sol`:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity >=0.8.13;

import { TIP20 } from "../src/TIP20.sol";
import { BaseTest } from "./BaseTest.t.sol";

contract TIP20OverflowTest is BaseTest {
    TIP20 token;

    function setUp() public override {
        super.setUp();
        token = TIP20(
            factory.createToken(
                "Test", "TST", "USD", TIP20(_PATH_USD), admin, bytes32("token")
            )
        );
        vm.prank(admin);
        token.grantRole(token.ISSUER_ROLE(), admin);
    }

    function testMintOverflow() public {
        // Mint max value
        vm.prank(admin);
        token.mint(alice, type(uint128).max);

        // This should overflow and log details
        vm.prank(admin);
        vm.expectRevert();
        token.mint(alice, 1);
    }

    function testTransferOverflow() public {
        // Setup both accounts with max balance
        vm.prank(admin);
        token.mint(alice, type(uint128).max);
        vm.prank(admin);
        token.mint(bob, type(uint128).max);

        // Try to transfer to bob - should overflow
        vm.prank(admin);
        token.mint(charlie, 1000e18);
        vm.prank(charlie);
        vm.expectRevert();
        token.transfer(bob, 1e18);
    }
}
```

Run with:
```bash
forge test --match-contract TIP20OverflowTest -vvvv
```

### 4. Verify Logging Output

After applying the fix, overflow scenarios should produce logs like:

```
ERROR TIP20 mint: total supply overflow detected
  total_supply=340282366920938463463374607431768211455
  amount=1000000000000000000
  supply_cap=340282366920938463463374607431768211455

ERROR TIP20 transfer: recipient balance overflow detected
  to=0x742d35Cc6634C0532925a3b844Bc9e7595f0bEb
  current_balance=340282366920938463463374607431768211455
  amount=1000000000000000000
```

## Impact

### Without Fix (Current State)
- Operators cannot diagnose overflow issues from logs alone
- Requires redeploying with debug instrumentation to investigate
- Increased Mean Time To Resolution (MTTR) for production issues
- Difficult to identify which user/operation is causing problems

### With Fix (Proposed State)
- All overflow conditions emit structured logs with full context
- Operators can immediately identify:
  - Which operation failed
  - Which accounts were involved
  - The exact values that caused overflow
  - Supply cap constraints (if applicable)
- Enables faster debugging and resolution
- Better observability for production monitoring

## Fix References

- **Commit**: ee9973e20c1f2f585679ea19316c66a6d4e63f0a
- **File**: `crates/precompiles/src/tip20/mod.rs`
- **Lines Modified**: 350-380, 742-768, 796-866
- **Lines Added**: 54

## Verification Commands

```bash
# Verify fix is applied
git show ee9973e2 --stat

# Check for logging at all overflow points
grep -c "tracing::error" crates/precompiles/src/tip20/mod.rs
# Should return: 6

# Verify specific log messages exist
grep "TIP20 mint: total supply overflow detected" crates/precompiles/src/tip20/mod.rs
grep "TIP20 mint: balance overflow detected" crates/precompiles/src/tip20/mod.rs
grep "TIP20 transfer: sender balance overflow detected" crates/precompiles/src/tip20/mod.rs
grep "TIP20 transfer: recipient balance overflow detected" crates/precompiles/src/tip20/mod.rs
grep "TIP20 burn: opted-in supply underflow detected" crates/precompiles/src/tip20/mod.rs
grep "TIP20 transfer: opted-in supply overflow detected" crates/precompiles/src/tip20/mod.rs
```

## Related Documentation

- TIP-20 Token Standard: `/home/agent/tempo/tips/TIP-20.md`
- Precompile Implementation: `/home/agent/tempo/crates/precompiles/src/tip20/mod.rs`
- Foundry Tests: `/home/agent/tempo/tips/ref-impls/test/TIP20.t.sol`
