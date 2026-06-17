![](../logo.png)
<img src="../img/gte.png" alt="GTE" height="320" />

# Security Report for the GTE Perps and Launchpad Contest

https://code4rena.com/audits/2025-08-gte-perps-and-launchpad

### [M-1] Absence of an unchecked block in the cumulative price calculation causes a permanent denial-of-service vulnerability

**Description and Root Cause:** 

The contract implements a Uniswap V2-style cumulative price oracle. These oracles work by continuously adding `price * time_elapsed` to a `price>>>CumulativeLast` variable. To function over long periods, these variables are expected and required to overflow (i.e., wrap around from the maximum `uint256` value back to zero). 

The developer's own comments - `// + overflow is desired` - confirms this was the intention. However, the code does not wrap the addition in an unchecked block.

**Recommended Mitigation:**

The cumulative price additions must be wrapped in an `unchecked` block to allow for the intended safe overflow, which is the standard implementation for Uniswap V2 oracles.


### [M-2] Reward Distribution Misdirection

**Description and Root Cause:**

When users purchase tokens through the Launchpad -triggering stake increases-, pending rewards are incorrectly distributed to the Launchpad contract instead of the actual beneficiary user. This results in permanent loss of all reward incentives for users.

In the `_distributeAssets()` function, rewards are transferred to `msg.sender` during the `increaseStake()` calls. However, when called from the Launchpad contracts's buy flow, `msg.sender` is the Launchpad contract address, not the user who should receive the rewards.

**Recommended Mitigation:**

Update the `_distributeAssets()` function to accept an explicit recipient parameter and transfer rewards to the correct address.


### [L-1] Incorrect payer in _swapRemaining Function

**Description and Root Cause:**

During token graduation, when a "remaining swap" is initiated to cover unfilled base tokens, the _swapRemaining function erroneously deducts quote tokens from `msg.sender` (usually an authorized operator) rather than the original buyer's account, leading to transaction failures in operator processes.

The `_swapRemaining()` function incorrectly uses `msg.sender` for token transfers instead of the `buyData.account` parameter, which identifies the actual buyer.

```solidity
    data.quote.safeTransferFrom(
        msg.sender, // <-- BUG: This should be the user's `account`
        address(this),
        data.quoteAmount
    );
```

**Recommended Mitigation:**

Update `_swapRemaining()` to pull tokens from the correct account:

```solidity
    function _swapRemaining(SwapRemainingData memory data, address payer) internal returns (uint256, uint256) {
        // Pull from specified payer account
        data.quote.safeTransferFrom(payer, address(this), data.quoteAmount);
        // ... rest of implementation
    }
```
