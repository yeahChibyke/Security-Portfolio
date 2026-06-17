![](../logo.png)
<img src="../img/alignerz.png" alt="AlignerZ" height="320" />

# Security Report for the AlignerZ Contest

https://github.com/dualguard/2025-11-alignerz

### [H-1] Uninitialized Memory Arrays in `AlignerzVesting::_computeSplitArrays()` Causes Complete Failure of NFT Split Functionality

**Summary:**

The `::_computeSplitArrays()` function attempts to write to uninitialized dynamic arrays in memory, causing all calls to `AlignerzVesting::splitTVS()` to revert. This completely breaks the NFT splitting feature, preventing users from dividing their vesting positions into multiple smaller positions. Any user attempting to split their NFT will experience a transaction failure.

**Root Cause:**

The function declares a return variable Allocation memory alloc but never initializes its dynamic array fields before attempting to write to them:

```solidity
    function _computeSplitArrays(Allocation storage allocation, uint256 percentage, uint256 nbOfFlows)
      internal
      view
      returns (Allocation memory alloc)  // ← alloc is declared but arrays are uninitialized
   {
      uint256[] memory baseAmounts = allocation.amounts;
      uint256[] memory baseVestings = allocation.vestingPeriods;
      uint256[] memory baseVestingStartTimes = allocation.vestingStartTimes;
      uint256[] memory baseClaimed = allocation.claimedSeconds;
      bool[] memory baseClaimedFlows = allocation.claimedFlows;
      alloc.assignedPoolId = allocation.assignedPoolId;
      alloc.token = allocation.token;
      
      for (uint256 j; j < nbOfFlows;) {
        alloc.amounts[j] = (baseAmounts[j] * percentage) / BASIS_POINT;  //  REVERT!
        alloc.vestingPeriods[j] = baseVestings[j];                       //  Array length is 0
        alloc.vestingStartTimes[j] = baseVestingStartTimes[j];
        alloc.claimedSeconds[j] = baseClaimed[j];
        alloc.claimedFlows[j] = baseClaimedFlows[j];
        unchecked {
            ++j;
        }
    }
   }
```

**Impact:**

100% failure rate for all split attempts.

**Recommended Mitigation:**

Initialize all dynamic arrays in the returned struct before writing to them.

### [H-2] Cumulative Fee Subtraction causes Users to be Overcharged in Multi-Flow Fee Calculation

**Summary:**

The `FeesManager::calculateFeeAndNewAmountForOneTVS()` function contains a logic error in how it calculates fees for allocations with multiple token flows. While the function correctly accumulates the total fee amount across all flows, it incorrectly subtracts the cumulative total from each flow instead of subtracting only that individual flow's fee. This causes later flows in the array to have increasingly excessive fees deducted, resulting in users receiving less tokens than they should after fee deduction.

**Root Cause:**

The function calculates fees correctly, but applies them incorrectly:

```solidity
    function calculateFeeAndNewAmountForOneTVS(uint256 feeRate, uint256[] memory amounts, uint256 length)
    public
    pure
    returns (uint256 feeAmount, uint256[] memory newAmounts)
{
    for (uint256 i; i < length;) {
        feeAmount += calculateFeeAmount(feeRate, amounts[i]);  // ✓ Accumulates total
        newAmounts[i] = amounts[i] - feeAmount;  // Subtracts cumulative total!
        unchecked { ++i; }
    }
}
```

The `feeAmount` variable serves dual purposes but only one is implemented correctly:

1. Track total fees (for returning to caller) - works correctly
2. Individual flow fee (for calculating newAmount) - uses wrong value

**Impact:**

- Tokens end up missing.
- Users end up getting overcharged on splits and fees, and because this bug is overshadowed by earlier bugs (infinite loop and uninitialized array), protocol never realizes this.

**Recommended Mitigation:**

Use a local variable to store the individual flow's fee, separate from the cumulative total:

```diff
    function calculateFeeAndNewAmountForOneTVS(uint256 feeRate, uint256[] memory amounts, uint256 length)
    public
    pure
    returns (uint256 feeAmount, uint256[] memory newAmounts)
{
    newAmounts = new uint256[](length);  // Fix uninitialized array bug
    
    for (uint256 i; i < length;) {
-       feeAmount += calculateFeeAmount(feeRate, amounts[i]);
-       newAmounts[i] = amounts[i] - feeAmount;
+       uint256 flowFee = calculateFeeAmount(feeRate, amounts[i]);
+       feeAmount += flowFee;  // Accumulate for total
+       newAmounts[i] = amounts[i] - flowFee;  // Subtract only this flow's fee
        unchecked {
            ++i;  // Fix infinite loop bug
        }
    }
}
```

### [H-3] Uninitialized Memory Array Causes Out-of-Bounds Panic in Fee Calculation

**Summary:**

The `FeesManager::calculateFeeAndNewAmountForOneTVS()` function declares a return parameter newAmounts as a memory array, but never initializes it before attempting to write to it.

**Root Cause:**

The return parameter `newAmounts` is declared but never allocated:

```solidity
    function calculateFeeAndNewAmountForOneTVS(uint256 feeRate, uint256[] memory amounts, uint256 length)
    public
    pure
    returns (uint256 feeAmount, uint256[] memory newAmounts)  // ← Declared here
{
    // ← MISSING: newAmounts = new uint256[](length);
    
    for (uint256 i; i < length;) {
        feeAmount += calculateFeeAmount(feeRate, amounts[i]);
        newAmounts[i] = amounts[i] - feeAmount;  // Writing to length-0 array!
        unchecked { ++i; }
    }
}
```

**Impact:**

This bug is masked by the infinite loop bug reported earlier, and it results in split and merge functionalities failing.

**Mitigation:**

Initialize the `newAmounts` array before the loop.

### [H-4] Missing Loop Increment Causes Infinite Loop and Gas Exhaustion in Fee Calculation

**Summary:**

The `FeesManager::calculateFeeAndNewAmountForOneTVS()` function contains a for-loop that lacks an increment statement and this causes the loop counter `i` to remain at 0 indefinitely. This results in an infinite loop that will exhaust all available gas and revert any transaction that calls this function.

**Root Cause:**

The for loop is missing the increment statement that advances the loop counter:

```solidity
    function calculateFeeAndNewAmountForOneTVS(uint256 feeRate, uint256[] memory amounts, uint256 length)
      public
      pure
      returns (uint256 feeAmount, uint256[] memory newAmounts)
   {
    for (uint256 i; i < length;) {
        feeAmount += calculateFeeAmount(feeRate, amounts[i]);
        newAmounts[i] = amounts[i] - feeAmount;
        // ← MISSING: unchecked { ++i; }
    }
   }
```

**The Problem:**

- loop initialzes with `i = 0`
- loop conditions is `i < length`
- loop body executes operations with `i = 0`
- loop counter `i` is never incremented
- control returns to check condition: `0 < length` is still `TRUE`
- loop executes with `i = 0`
- this continues until gas runs out 

**Impact:**

Split and merge functionalities completely broken.

**Recommended Mitigation:**

Add the missing increment statement to the for loop:

```diff
    function calculateFeeAndNewAmountForOneTVS(uint256 feeRate, uint256[] memory amounts, uint256 length)
    public
    pure
    returns (uint256 feeAmount, uint256[] memory newAmounts)
{
+   newAmounts = new uint256[](length);  // Also fix uninitialized array bug
    for (uint256 i; i < length;) {
        feeAmount += calculateFeeAmount(feeRate, amounts[i]);
        newAmounts[i] = amounts[i] - feeAmount;
+       unchecked {
+           ++i;
+       }
    }
}
```

### [H-5] Infinite Loop Causes Denial of Service in Dividend Distribution

**Summary:**

The `A26ZDividendDistributor::getUnclaimedAmounts()` function contains a loop where the increment statement `(++i)` is placed after continue statements. When certain common conditions are met (claimed flows or unclaimed flows with zero claimed seconds), the loop variable i never increments, causing an infinite loop. This makes the `::setUpTheDividends()` function run out of gas and revert, permanently breaking the dividend distribution functionality.

**Root Cause:**

```solidity
    function getUnclaimedAmounts(uint256 nftId) public returns (uint256 amount) {
        if (address(token) == address(vesting.allocationOf(nftId).token)) return 0;
        uint256[] memory amounts = vesting.allocationOf(nftId).amounts;
        uint256[] memory claimedSeconds = vesting.allocationOf(nftId).claimedSeconds;
        uint256[] memory vestingPeriods = vesting.allocationOf(nftId).vestingPeriods;
        bool[] memory claimedFlows = vesting.allocationOf(nftId).claimedFlows;
        uint256 len = vesting.allocationOf(nftId).amounts.length;
    
    for (uint256 i; i < len;) {
        if (claimedFlows[i]) continue;  // Skips increment!
        
        if (claimedSeconds[i] == 0) {
            amount += amounts[i];
            continue;  // Also skips increment!
        }
        
        uint256 claimedAmount = claimedSeconds[i] * amounts[i] / vestingPeriods[i];
        uint256 unclaimedAmount = amounts[i] - claimedAmount;
        amount += unclaimedAmount;
        
        unchecked {
            ++i;  // Only reached if NEITHER continue is hit!
        }
    }
    unclaimedAmountsIn[nftId] = amount;
    }
```

- the loop increment `++i` is inside the `unchecked` block at the end
- two `continue` statements jump back to the loop start before reaching the increment
- loop variable `i` stays the same value forever
- loop never exits, consumes all gas

**Impact:**

Complete DOS as `::_setUpDividends()` always reverts with out-of-gas.

### [M-1] Duplicate Dividend Allocation Leads To Contract Insolvency 

**Summary:**

The `A26ZDividendDistributor::_setDividends()` function uses the `+=` operator to allocate dividends to users. This causes allocations to accumulate instead of being replaced when the function is called multiple times. If `::setDividends()` or `::setUpTheDividends()` is called more than once, users will be allocated multiples of their intended dividend amounts, making the contract insolvent. The contract will owe more stablecoins than it actually holds, causing later claimants to receive nothing.

**Root Cause:**

```solidity
    function _setDividends() internal {
        uint256 len = nft.getTotalMinted();
        for (uint256 i; i < len;) {
            (address owner, bool isOwned) = safeOwnerOf(i);
            if (isOwned) {
                dividendsOf[owner].amount += ((unclaimedAmountsIn[i] * stablecoinAmountToDistribute)
                        / totalUnclaimedAmounts);
            }
            unchecked {
                ++i;
            }
        }
        emit dividendsSet();
    }
```

The `+=` operator adds to the existing allocation instead of setting a new value. Each time the function runs, it adds the same allocation again.

**Impact:**

- Total allocations exceed contract balance
- First claimers get paid, last claimers get nothing

