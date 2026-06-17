![](../logo.png)
<img src="../img/revert_hooks.png" alt="Revert Finance - StableSwap Hooks" height="320" />

# Security Report for the Revert Finance - StableSwap Hooks Contest

https://cantina.xyz/competitions/e55ee7b9-6c99-42f8-8338-39f3dd134ef3

### [H-1] Native LP Withdrawals Allow Reentrant Swaps Against Stale Reserves

**Summary:**

The protocol implements LP ownership as ERC20 shares backed by hook-managed reserves and PoolManager ERC6909 claims. This model is not the native Uniswap v4 LP position model. That separation creates a concrete bug in native ETH pools.

During `removeLiquidity`, the hook burns PoolManager claims and transfers assets to the LP before decrementing its internal `reserves` and before burning the LP ERC20 shares. For native ETH, `poolManager.take()` performs a raw ETH transfer to the LP recipient. A contract recipient can use its `receive()` callback to call `PoolManager.swap()` directly while the PoolManager is still unlocked.

The reentrant swap reaches the hook's `beforeSwap` path and is priced using stale `reserves` that still include the ETH already withdrawn. The PoolManager claim has already been burned, but the hook still reports the old reserve. This lets the attacker swap against overstated native ETH liquidity and receive a better price than the same swap after the withdrawal accounting finishes.

**Finding Description:**

`Liquidity.removeLiquidity()` starts a PoolManager unlock:

```solidity
    function removeLiquidity(uint256 _shares, uint256[] calldata _minAmounts) external {
        bytes memory data = abi.encode(Actions.REMOVE_LIQUIDITY, _shares, _minAmounts, msg.sender);

        poolManager.unlock(data);
    }
```

Inside the callback, the hook transfers each withdrawn asset before finalizing its own LP accounting:

```solidity
    poolManager.burn(address(this), currency.toId(), amounts[i]);
    poolManager.take(currency, sender, amounts[i]);

    reserves[i] -= amounts[i];
    ...
    _burn(sender, shares);
```

For native ETH, `PoolManager.take()` transfers ETH to `sender`:

```solidity
    function take(Currency currency, address to, uint256 amount) external onlyWhenUnlocked {
        _accountDelta(currency, -(amount.toInt128()), msg.sender);
        currency.transfer(to, amount);
    }
```

Native currency transfer uses raw `call(gas(), ...)`, so a contract LP receives control immediately:

```solidity
    success := call(gas(), to, amount, 0, 0, 0, 0)
```

The usual nested-`unlock()` defense does not stop this path. `PoolManager.unlock()` blocks another `unlock()` while already unlocked, but `PoolManager.swap()` is explicitly callable only while unlocked:

```solidity
    function swap(...) external onlyWhenUnlocked returns (BalanceDelta swapDelta)
```

The attacker does not call `hook.removeLiquidity()` again. The attacker calls `PoolManager.swap()` directly from the ETH receive callback. That enters the hook through `beforeSwap`:

```solidity
    BeforeSwapDelta delta =
        toBeforeSwapDelta(-SafeCast.toInt128(_params.amountSpecified), _swap(_sender, _poolKey, _params));
```

`Swap._createSwapContext` prices from the hook's internal `reserves`:

```solidity
    ctx.scaledReserves[i] = StableSwapMath.scaleTo(reserves[i], _getRate(i));
```

At this point, the PoolManager ETH claim for the withdrawal has already been burned, but `reserves[ETH]` has not yet been decremented. The hook therefore prices the reentrant swap using more ETH liquidity than it actually still owns.

**Recommended Mitigation:**

Finalize hook accounting before the first external transfer:

- Burn the LP ERC20 shares before transferring withdrawn assets.
- Decrement all `reserves[i]` before calling `poolManager.take()`.
- Avoid per-currency partial finalization where one asset is transferred while the rest of the withdrawal remains unfinalized.
- Add a liquidity-operation reentrancy flag and make `_beforeSwap` revert while add/remove liquidity callbacks are executing.
- Add invariant tests asserting that hook-held PoolManager claims equal `reserves[i] + protocolFees[i] + hookFees[i]` after add, remove, swap, and fee withdrawal flows.

The minimum fix is to remove the stale-reserve window. The stronger fix is to also block swaps during hook-managed LP add/remove callbacks.


### [M-1] Exact-output swaps charge fees on net invariant input instead of the user's gross input

**Summary:**

The exact-output swap path computes the net input required by the invariant, then charges `fee(netInput)`. This is not a gross-input fee. It lets traders pay less than the amount required for a true input-side percentage fee, and the gap increases as LP fees increase. At the maximum 100% LP fee, exact-output swaps still succeed even though a gross-input fee model cannot produce any nonzero net input at 100% fee.

**Finding Description:**

`Swap._swapExactOutput` calculates the invariant-required input first:

```solidity
    rawAmountIn = descale(newTokenInReserves - oldTokenInReserves, inputRate);
```

It then calculates fees on that already-net amount:

```solidity
    fees = _getFees(rawAmountIn);
    result.amountIn = rawAmountIn + fees;
```

This charges:

```solidity
    paid = netInput + fee(netInput)
```

For target-output swaps with input-side fees, the correct gross amount must satisfy:

```solidity
    grossInput - fee(grossInput) >= netInput
```

which is:

```solidity
    grossInput = ceil(netInput * FEE_PRECISION / (FEE_PRECISION - fee))
```

The codebase already uses this target-output gross-up shape in `StableSwapZapIn._calculateSwapAmount`:

```solidity
    rawOutputNeeded = Math.mulDiv(outputNeeded, FEE_PRECISION, FEE_PRECISION - ctx.lpFee);
```

The core swap path does not apply the same fee-base logic. It charges `netInput + fee(netInput)`, which is strictly lower than the required gross input for any nonzero fee below 100%.

**Recommended Mitigation:**

Charge exact-output input fees from the gross input amount:

```solidity
    grossInput = Math.mulDiv(rawInput, FEE_PRECISION, FEE_PRECISION - lpFee, Math.Rounding.Ceil);
    feeAmount = grossInput - rawInput;
```

Reject exact-output swaps when the applicable fee is `>= FEE_PRECISION`.

Apply hook/protocol fee splitting to the resulting gross fee amount, and align `Swap` with the target-output gross-up logic already used by `StableSwapZapIn`.


### {I-1} `sqrtPriceLimitX96` is accepted through the v4 swap interface but is not enforced by the hook

**Summary:**

The hook exposes a Uniswap v4 pool interface, but its swap implementation cancels the native v4 swap amount before PoolManager reaches v4 price-limit enforcement. As a result, direct `PoolManager.swap` callers can pass a `sqrtPriceLimitX96` that v4 would reject as already exceeded, and the StableSwap trade still executes.

This is not a harmless UI mismatch. `sqrtPriceLimitX96` is a core v4 swap safety parameter. The hook accepts the parameter through the canonical v4 swap path and then makes it ineffective.

**Finding Description:**

`Swap._beforeSwap` computes the StableSwap trade and returns a `BeforeSwapDelta` that offsets the caller's full specified amount:

```solidity
    BeforeSwapDelta delta =
        toBeforeSwapDelta(-SafeCast.toInt128(_params.amountSpecified), _swap(_sender, _poolKey, _params));
```

That makes the amount passed into native v4 `Pool.swap()` equal to zero. In v4, the price-limit checks happen after the zero-amount early return:

```solidity
    if (params.amountSpecified == 0) return (BalanceDeltaLibrary.ZERO_DELTA, 0, swapFee, result);

    if (zeroForOne) {
        if (params.sqrtPriceLimitX96 >= slot0Start.sqrtPriceX96()) {
            PriceLimitAlreadyExceeded.selector.revertWith(...);
        }
    } else {
        if (params.sqrtPriceLimitX96 <= slot0Start.sqrtPriceX96()) {
            PriceLimitAlreadyExceeded.selector.revertWith(...);
        }
    }
```

The hook initializes each pair at `sqrtPriceX96 = 1 << 96`. A `zeroForOne` swap with `sqrtPriceLimitX96 == 1 << 96` is already exceeded and should be rejected by the v4 check. A `oneForZero` swap with the same limit is also already exceeded. The PoC calls `PoolManager.swap` directly with those limits and proves both swaps still execute and transfer output.

The reason is simple: v4 never gets to enforce the limit because the hook has already zeroed the native swap amount.

***Recommended Mitigation:**

Do not accept a v4 price-limit parameter and then ignore it.

Either:

- reject any `sqrtPriceLimitX96` that is not the unconstrained extreme value expected by the official v4 router; or
- implement equivalent limit enforcement against the hook's own StableSwap price before returning the swap delta.

