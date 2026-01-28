# Bug Submission: Bug #2: Integer Overflow & Rounding Loss in StablecoinDEX

## Bug Description
The `StablecoinDEX` implementation contained two distinct classes of issues:
1.  **Integer Overflow Vulnerabilities**: Unsafe arithmetic on `u128` and `i16` types in state transitions and error reporting.
2.  **Rounding Loss (TIP-1005)**: A precision loss bug in `swapExactAmountIn` for ask orders where the maker could receive slightly fewer tokens than the taker paid due to double-rounding during partial fills.

### 1. Integer Arithmetic Issues:
- **Order ID Overflow**: `increment_next_order_id` used naked `+ 1`.
- **Partial Fill Underflow**: `partial_fill_order` used naked `- fill_amount`.
- **Error Reporting Overflow**: `price_to_tick` used lossy casts that could overflow for extremely large prices, leading to corrupted error messages.

### 2. Rounding Loss (TIP-1005):
In the ask path of `swapExactAmountIn`, the system calculates how much base to give the taker based on their quote input. If the fill is partial, the maker's credit is re-calculated from the base amount. Because of the integer division floor in the first step and the ceiling in the second, a token could be lost (zero-sum invariant violation).

## Steps to Reproduce (Rounding Loss)
1.  **Place Ask Order**: Place an ask order at price 1.02 (tick 2000).
2.  **Swap Exact Input**: Swap `102,001` quote tokens for base.
3.  **Result**: 
    - `base_out = floor(102001 / 1.02) = 100,000`.
    - `maker_receives = ceil(100000 * 1.02) = 102,000`.
    - **1 token is permanently lost from the system.**

## Implementation Fix
### Rust Precompile (`crates/precompiles/src/stablecoin_dex/mod.rs`)
- Hardened all arithmetic with `checked_` operations.
- Added `quote_override` to `partial_fill_order`.
- Updated `fill_orders_exact_in` to pass the original `amount_in` as the override for partial fills on ask orders, ensuring the maker receives exactly what the taker paid.

### Reference Specification (`tips/ref-impls/src/StablecoinDEX.sol`)
- Performed the same rounding fix by adding a `quoteOverride` parameter to the internal `_fillOrder` function.
- Hardened the `priceToTick` error reporting path to handle extreme `u32` values safely.

## Verification Results
- **Unit Tests**: All StablecoinDEX tests (127/127) passed on localnet.
- **Invariant Check**: Verified that `taker_paid == maker_received` for the edge case mentioned in TIP-1005.

## Conclusion
The StablecoinDEX now correctly enforces the zero-sum invariant and is hardened against all identified arithmetic edge cases. Both the production Rust code and the reference Solidity implementation have been synchronized with these fixes.
