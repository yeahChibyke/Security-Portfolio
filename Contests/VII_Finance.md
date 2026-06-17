![](../logo.png)
<img src="../img/vii.png" alt="VII-Finance-Contracts" height="320" />

# Security Report for the VII-Finance-Contracts Contest

https://cantina.xyz/code/eb93d215-e328-4d19-99ab-6c510acbb5aa/overview

### [L-1] Default Deadline Disables Slippage Protection in `UniswapV4Wrapper`

**Summary:**

The `UniswapV4Wrapper` defaults the deadline to `block.timestamp` if `extraData` is empty or malformed, effectively disabling transaction expiration checks. This disables time-bound execution guarantees, exposing users to stale market conditions during partial unwraps.

**Finding Description:**

In `_decreaseLiquidity`, the wrapper decodes slippage and deadline parameters from `extraData`:

```solidity
    (amount0Min, amount1Min, deadline) =
        extraData.length == 96 ? abi.decode(extraData, ...) : (0, 0, block.timestamp);
```

If a user provides no `extraData`, the deadline is set to the current block timestamp. Later, a standard Uniswap-style check is assumed:

```solidity
        require(block.timestamp <= deadline, "EXPIRED");
```

This means the transaction will *always* satisfy the deadline check, even if it has been sitting in the mempool for hours (pending validation). This exposes the user to adverse market conditions that might have developed since they signed the transaction.

While Uniswap V4 core does not enforce deadlines natively, the wrapper is responsible for enforcing them if it accepts deadline parameters. By defaulting to `block.timestamp`, it creates a silent failure mode where users believe they have time protection, but do not.

**Recomended Mitigation:**

Require a valid deadline or revert if `extraData` is missing. At minimum, do not default to `block.timestamp` without explicit warning; consider a default relative timeout (though that requires `block.timestamp` + delta which is also checked at execution time, effectively same as `block.timestamp`). Better to force the user to provide it.


### [L-2] Theft of User Deposits via `skim()` Race Condition

**Summary:**

The `skim()` function in both `UniswapV3Wrapper` and `UniswapV4Wrapper` allows any user to wrap a pending NFT deposit. Due to a predictable or race-prone mechanism for selecting which token to wrap, an attacker can steal a legitimate user's deposit.

**Finding Description:**

The wrappers provide a `skim()` function intended to wrap NFTs that were sent directly to the contract. The mechanism for identifying the "stray" token is insecure:

- **UniswapV3Wrapper**: Calls `tokenOfOwnerByIndex(address(this), balance - 1)`. This returns the last token added to the Wrapper's balance.
- **UniswapV4Wrapper**: Calls `nextTokenId() - 1`. This assumes the last minted token is the one to wrap.

In a scenario where User A and User B both send tokens to the wrapper (or mint to it) in close proximity:

1. User A transfers Token A to Wrapper.
2. User B transfers Token B to Wrapper.
3. User A calls skim(UserA).

In the V3 case, `skim` selects the last token (Token B). User A receives the ERC6909 wrapper for Token B. In the V4 case, if User B minted last, `getTokenIdToSkim` returns Token B. User A receives the ERC6909 wrapper for Token B.

User A effectively steals User B's position. User B is left with nothing (Token A is stuck or User B can claim it, but their original asset is gone).

**Recommended Mitigation:**

Implement `onERC721Received` in the Wrapper to handle safe transfers atomically. This ensures that the user sending the token is the one credited with the wrapped balance. Deprecate or remove `skim()`, or strictly warn against its use in public mempools.

