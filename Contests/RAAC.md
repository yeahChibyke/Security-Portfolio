![](../logo.png)
<img src="../img/raac.png" alt="RAAC" height="320" />

# Security Report for the RAAC Contest

https://codehawks.cyfrin.io/c/2025-02-raac

### [H-1] Precision error in `Auction::buy()` means buyers will always pay astronomically higher than they should when they want to buy `ZENO` tokens.

**Description:**

The `::buy()` function calculates the `cost` a buyer should pay by multiplying the amount of tokens they want to buy, and the current slippage price per token, and then requires that this amount is transferred from the buyer to the `businessAddress`.

```solidity
    /**
     * Bid on the ZENO auction
     *     User will able to buy ZENO tokens in exchange for USDC
     */
    function buy(uint256 amount) external whenActive {
        require(amount <= state.totalRemaining, "Not enough ZENO remaining");
        uint256 price = getPrice();
        uint256 cost = price * amount;  👈🏾👈🏾
​
        require(usdc.transferFrom(msg.sender, businessAddress, cost), "Transfer failed");  👈🏾👈🏾
​
        bidAmounts[msg.sender] += amount;
        state.totalRemaining -= amount;
        state.lastBidTime = block.timestamp;
        state.lastBidder = msg.sender;
​
        // zeno.mint(msg.sender, amount);
        emit ZENOPurchased(msg.sender, amount, price);
    }
```

But because both values (amount of tokens to be bought and current slippage price) are in `e18`, the cost ends up being in `e36`.

**Impact:**

One of three things will happen:

- If sufficient approval is given by the buyer to the Auction contract and the buyer has such amount in usdc worth (very unlikely), the buyer pays the astronomical cost

- If sufficient approval is given by the buyer to the Auction contract and the buyer does not have such amount in usdc worth (very likely), the transfer reverts with an ERCInsuffientBalance() error

- If the buyer doesn't give sufficient approval, the transfer reverts with an ERCInsufficientAllowance() error

*P.S. I commented out `zeno.mint(msg.sender, amount);` in order to post the traces of my PoC*

**Proof of Code:**

Run the following test in Foundry:

```solidity
    function testBuyerPaysAstronomicalCost() public {
        usdc.mint(buyer, 1e36); // Minted the buyer some crazy amount of usdc for the purpose of this test
​
        vm.warp(startTime + 1 seconds);
​
        console2.log("This is the usdc balance of the buyer before he buys any ZENO: ", usdc.balanceOf(buyer));
​
        vm.startPrank(buyer);
        usdc.approve(address(auction), type(uint256).max); // Buyer approves for type(uint256).max
        auction.buy(1e18); // Buyer wants to buy 1 ZENO token
        vm.stopPrank();
​
        console2.log("This is the usdc balance of the buyer after he buys any ZENO: ", usdc.balanceOf(buyer));
        console2.log(
            "This is the amount of usdc that was transferred from the buyer's account: ", (amount * auction.getPrice())
        );
    }
```

And we can read the logs:

```solidity
    [PASS] testBuyerPaysAstronomicalCost() (gas: 202811)
Logs:
  This is the usdc balance of the buyer before he buys any ZENO:  1000000000000000000000000000000000000
  This is the usdc balance of the buyer after he buys any ZENO:  138888888888888000000000000000000
  This is the amount of usdc that was transferred from the buyer's account:  999861111111111112000000000000000000
```

**Recommended Mitigation:**

Refactor the `buy()` function so that the cost is divided by `1e18` before it is being transferred from the `buyer`

```diff
    function buy(uint256 amount) external whenActive {
        ...
        ...
        
-       uint256 cost = price * amount;
+       uint256 cost = (price * amount) / 1e18;
​
        require(usdc.transferFrom(msg.sender, businessAddress, cost), "Transfer failed");
​
        ...
        ...
    }
```

### [M-1] `locks` mapping in `veRAACToken` contract is not updated in key functions

**Description:**

Whenever users lock tokens, increase their locked tokens amount, or extend their lock duration, it is expected that these details are updated in the `locks` mapping.

```solidity
    /**
     * @notice Mapping of user addresses to their lock positions
     */
    mapping(address => Lock) public locks;
```

But, this mapping is never updated.

The `veRAACToken` updates user details into the private `_lockState` mapping instead. Which is a library state.

```solidity
    /**
     * @notice State for managing lock positions
     */
    LockManager.LockState private _lockState;
```

**Impact:**

Reading from the veRAACToken to verify user details will always return wrong details and results.

This is particular harmful because, if an external contract needs to verify user details by calling getter functions, it will get back wrong details.

**Proof of Code:**

Run this test in Foundry:

```soliidty
    // SPDX-License-Identifier: MIT
pragma solidity ^0.8.19;
​
import {Test, console2} from "forge-std/Test.sol";
import {veRAACToken} from "../contracts/core/tokens/veRAACToken.sol";
import {RAACMockERC20} from "../contracts/mocks/core/tokens/RAACMockERC20.sol";
​
contract TestVeRAACToken is Test {
    veRAACToken vrc;
    RAACMockERC20 mrt; // mock RAACERC20 token
    address owner = address(0x1);
    address user = address(0x2);
​
    function setUp() public {
        vm.startPrank(owner);
        mrt = new RAACMockERC20(owner);
​
        vrc = new veRAACToken(address(mrt));
​
        mrt.mintTo(user, 10_000e18); // mint 10k mrt to user
​
        vm.stopPrank();
    }
​
    function testUserDetailsNotUpdated() public {
        uint256 amountToLock = 1_000e18;
        uint256 timeToLock = 31_536_000 seconds; // 1 year
​
        // user locks tokens
        vm.startPrank(user);
        mrt.approve(address(vrc), amountToLock);
        vrc.lock(amountToLock, timeToLock);
        vm.stopPrank();
​
        uint256 userLockedAmount = vrc.getLockedBalance(user);
        uint256 userLockedTime = vrc.getLockEndTime(user);
​
        console2.log("This is the locked balance of the user: ", userLockedAmount);
        console2.log("This is the duration the user locked their funds for: ", userLockedTime);
​
        assert(amountToLock == userLockedAmount);
        assert((timeToLock + block.timestamp) == userLockedTime);
    }
}
```

And these are the traces:

```solidity
    [509948] TestVeRAACToken::testUserDetailsNotUpdated()
    ├─ [0] VM::startPrank(SHA-256: [0x0000000000000000000000000000000000000002])
    │   └─ ← [Return] 
    ├─ [25296] RAACMockERC20::approve(veRAACToken: [0x535B3D7A252fa034Ed71F0C53ec0C6F784cB64E1], 1000000000000000000000 [1e21])
    │   ├─ emit Approval(owner: SHA-256: [0x0000000000000000000000000000000000000002], spender: veRAACToken: [0x535B3D7A252fa034Ed71F0C53ec0C6F784cB64E1], value: 1000000000000000000000 [1e21])
    │   └─ ← [Return] true
    ├─ [455613] veRAACToken::lock(1000000000000000000000 [1e21], 31536000 [3.153e7])
    │   ├─ [31614] RAACMockERC20::transferFrom(SHA-256: [0x0000000000000000000000000000000000000002], veRAACToken: [0x535B3D7A252fa034Ed71F0C53ec0C6F784cB64E1], 1000000000000000000000 [1e21])
    │   │   ├─ emit Transfer(from: SHA-256: [0x0000000000000000000000000000000000000002], to: veRAACToken: [0x535B3D7A252fa034Ed71F0C53ec0C6F784cB64E1], value: 1000000000000000000000 [1e21])
    │   │   └─ ← [Return] true
    │   ├─ emit LockCreated(user: SHA-256: [0x0000000000000000000000000000000000000002], amount: 1000000000000000000000 [1e21], unlockTime: 31536001 [3.153e7])
    │   ├─ emit PeriodCreated(startTime: 1, duration: 604800 [6.048e5], initialValue: 0)
    │   ├─ emit VotingPowerUpdated(user: SHA-256: [0x0000000000000000000000000000000000000002], oldPower: 0, newPower: 250000000000000000000 [2.5e20])
    │   ├─ emit CheckpointCreated(user: SHA-256: [0x0000000000000000000000000000000000000002], blockNumber: 1, power: 250000000000000000000 [2.5e20])
    │   ├─ emit Transfer(from: 0x0000000000000000000000000000000000000000, to: SHA-256: [0x0000000000000000000000000000000000000002], value: 250000000000000000000 [2.5e20])
    │   ├─ emit LockCreated(user: SHA-256: [0x0000000000000000000000000000000000000002], amount: 1000000000000000000000 [1e21], unlockTime: 31536001 [3.153e7])
    │   └─ ← [Stop] 
    ├─ [0] VM::stopPrank()
    │   └─ ← [Return] 
    ├─ [2989] veRAACToken::getLockedBalance(SHA-256: [0x0000000000000000000000000000000000000002]) [staticcall]
    │   └─ ← [Return] 0
    ├─ [2947] veRAACToken::getLockEndTime(SHA-256: [0x0000000000000000000000000000000000000002]) [staticcall]
    │   └─ ← [Return] 0
    ├─ [0] console::log("This is the locked balance of the user: ", 0) [staticcall]
    │   └─ ← [Stop] 
    ├─ [0] console::log("This is the duration the user locked their funds for: ", 0) [staticcall]
    │   └─ ← [Stop] 
    └─ ← [Revert] panic: assertion failed (0x01)
​
Suite result: FAILED. 0 passed; 1 failed; 0 skipped; finished in 6.74ms (2.49ms CPU time)
```

**Recommended Mitigation:**

There are two approaches to be considered for mitigation:

- Read directly from the library mapping `_lockState` to get user lock details

```diff
     /**
     * @notice Gets the amount of RAAC tokens locked by an account
     * @dev Returns the raw locked token amount without time-weighting
     * @param account The address to check
     * @return The amount of RAAC tokens locked by the account
     */
    function getLockedBalance(address account) external view returns (uint256) {
-       return locks[account].amount;
+       LockManager.Lock memory userLock = _lockState.locks[account]; // read directly from _lockState mapping
+       return userLock.amount;
    }
​
    /**
     * @notice Gets the lock end time for an account
     * @dev Returns the timestamp when the lock expires
     * @param account The address to check
     * @return The unix timestamp when the lock expires
     */
    function getLockEndTime(address account) external view returns (uint256) {
-       return locks[account].end;
+       LockManager.Lock memory userLock = _lockState.locks[account]; // read directly from _lockState mapping
+       return userLock.end;
    }
```

or,

- Update `veRAACToken` `locks` mapping in `::lock()`, `::increase()`, and `::extend()` functions

```diff
    /**
     * @notice Creates a new lock position for RAAC tokens
     * @dev Locks RAAC tokens for a specified duration and mints veRAAC tokens representing voting power
     * @param amount The amount of RAAC tokens to lock
     * @param duration The duration to lock tokens for, in seconds
     */
    function lock(uint256 amount, uint256 duration) external nonReentrant whenNotPaused {
        ...
        ...
​
        // Calculate unlock time
        uint256 unlockTime = block.timestamp + duration;
​
        // Create lock position
        _lockState.createLock(msg.sender, amount, duration);
        _updateBoostState(msg.sender, amount);
​
+       locks[msg.sender] = Lock({amount: amount, end: unlockTime}); // Update the locks mapping in the veRAACToken contract
​
        ...
        ...
    }
​
   /**
     * @notice Increases the amount of locked RAAC tokens
     * @dev Adds more tokens to an existing lock without changing the unlock time
     * @param amount The additional amount of RAAC tokens to lock
     */
    function increase(uint256 amount) external nonReentrant whenNotPaused {
        ...
        ...
​
        // Update voting power
        LockManager.Lock memory userLock = _lockState.locks[msg.sender];
+       locks[msg.sender].amount = userLock.amount; // Update locks mapping in the veRAACToken contract
​
        ...
        ...
    }
​
   /**
     * @notice Extends the duration of an existing lock
     * @dev Increases the lock duration which results in updated voting power
     * @param newDuration The new total duration extension for the lock, in seconds
     */
    function extend(uint256 newDuration) external nonReentrant whenNotPaused {
        // Extend lock using LockManager
        uint256 newUnlockTime = _lockState.extendLock(msg.sender, newDuration);
​
        // Update voting power
        LockManager.Lock memory userLock = _lockState.locks[msg.sender];
+       locks[msg.sender].end = newUnlockTime; // Update locks mapping in the veRAACToken contract
        ...
        ...
    }
```

Either of these approaches will mitigate this issue and ensure that user details are correctly read from the `veRAACToken` contract.

### [L-1] `veRAACToken` contract lacks pause functionality

**Description:**

Per the purpose for which the `veRAACToken` was built, and the presence of a `whenNotPaused()` modifier, this particular contract is meant to have a means for it to be paused. But in its current state, this functionality is missing.

**Impact:**

In the events that the contract needs to be paused - withdrawals, upgrades, in the case of an hack, etc.- it is impossible to do so.

Plus, it makes the `whenNotPaused()` modifier currently useless.

**Recommended Mitigation:**

Add a `pause()` function that is callable by `onlyOwner`

```solidity
    function pause() external onlyOwner {
        paused = true;
    }
```

Here is a test in Foundry to prove this mitigation:

```solidity
    // SPDX-License-Identifier: MIT
pragma solidity ^0.8.19;
​
import {Test, console2} from "forge-std/Test.sol";
import {veRAACToken} from "../contracts/core/tokens/veRAACToken.sol";
import {RAACMockERC20} from "../contracts/mocks/core/tokens/RAACMockERC20.sol";
​
contract TestVeRAACToken is Test {
    veRAACToken vrc;
    RAACMockERC20 mrt; // mock RAACERC20 token
    address owner = address(0x1);
    address user = address(0x2);
​
    function setUp() public {
        vm.startPrank(owner);
        mrt = new RAACMockERC20(owner);
​
        vrc = new veRAACToken(address(mrt));
​
        mrt.mintTo(user, 10_000e18); // mint 10k mrt to user
​
        vm.stopPrank();
    }
​
    function testPause() public {
        assert(vrc.paused() == false);
​
        vm.prank(owner);
        vrc.pause();
​
        assert(vrc.paused() == true);
​
        // User tries to lock funds when contract is currently paused
        vm.startPrank(user);
        mrt.approve(address(vrc), 10_000e18);
        vm.expectRevert();
        vrc.lock(1_000e18, 365 days);
        vm.stopPrank();
    }
}
```

### [I-1] Roundup logic in `Auction::getPrice()` will see users pay quite higher on negligible prices

**Description:**

The `getPrice()` function is used to get the current price of `ZENO` tokens in `USDC`. But per its current logic, it will always round up to the nearest whole number, even on values that are very negligible.

**Impact:**

`ZENO` tokens could be listed at prices that would result in slippage prices becoming quite high. And because the `getPrice()` logic rounds up every amount, the buyer could end up paying significantly higher than expected.

**Proof of Code:**

This is the current logic used to getthe slippage price of tokens:

```solidity
    function getPrice() public view returns (uint256) {
        ...
        ...
​
        return state.startingPrice
            - (
                (state.startingPrice - state.reservePrice) * (block.timestamp - state.startTime)
                    / (state.endTime - state.startTime)
            );
  }
```

If we run the following test in Foundry we can see the current slippage price:

```solidity
    // SPDX-License-Identifier: MIT
pragma solidity ^0.8.19;
​
import {Test, console2} from "forge-std/Test.sol";
import {Auction} from "../contracts/zeno/Auction.sol";
import {ZENO} from "../contracts/zeno/ZENO.sol";
import {MockUSDC} from "../contracts/mocks/core/tokens/MockUSDC.sol";
​
contract AuctionTest is Test {
    Auction auction;
    ZENO zeno;
    MockUSDC usdc;
​
    address zenoOwner = address(0x1);
    address businessAddress = address(0x2);
    address buyer = address(0x3);
    address auctionOwner = address(0x4);
​
    uint256 startTime = 3600 seconds;
    uint256 endTime = startTime + 3600 seconds; // i.e. auction lasts for 2 hours
    uint256 startingPrice = 1e18; // 1 USDC per ZENO
    uint256 reservePrice = 5e17; // 0.5 USDC per ZENO
    uint256 totalAllocated = 1000e18; // 1000 ZENO tokens allocated for the auction
    uint256 initialSupply = 10_000e18;
    uint256 zenoMaturityDate = 1 hours;
​
    function setUp() public {
        usdc = new MockUSDC(initialSupply);
        zeno = new ZENO(address(usdc), zenoMaturityDate, "ZTOKEN", "ZTK", zenoOwner);
        auction = new Auction(
            address(zeno),
            address(usdc),
            businessAddress,
            startTime,
            endTime,
            startingPrice,
            reservePrice,
            totalAllocated,
            auctionOwner
        );
​
        // Mint usdc to buyer for bidding
        usdc.mint(buyer, 1000e18);
    }
​
    function testGetPrice() public {
        console2.log(auction.getPrice()); // 1_000_000_000_000_000_000
        vm.warp(4600 seconds);
        console2.log(auction.getPrice()); // 861_111_111_111_111_112
    }
}
```

But if we calculate the current slippage price using the logic in the getPrice() function:

`1e18 - ((1e18 - 5e17) * (4600 - 3600) / (7200 - 3600))`

We get:

`861,111,111,111,111,111.11111111111111`

The current logic rounds up even on very negligible amounts.

**Recommended Mitigation:**

I added an internal function that only rounds up on amounts greater than `.4`. Amounts in the range of `.1` to `.4` round down.

```solidity
    function divideAndRound(uint256 numerator, uint256 denominator) internal pure returns (uint256) {
        require(denominator > 0, "Division by zero or negative values not allowed!!!");
​
        // Calculate the result of the division
        uint256 quotient = numerator / denominator;
​
        // Calculate the remainder
        uint256 remainder = numerator % denominator;
​
        // Check if the remainder is greater than or equal to half of the denominator by comparing 'remainder * 2' to the denominator
        if ((remainder * 2) >= denominator) {
            // Round up
            quotient += 1;
        }
​
        return quotient;
    }
```

Then this internal function is used to calculate the current slippage price in `getPrice()`.

```diff
    function getPrice() public view returns (uint256) {
        ...
        ...
-       return state.startingPrice
-           - (
-               (state.startingPrice - state.reservePrice) * (block.timestamp - state.startTime)
-                   / (state.endTime - state.startTime)
-           );
​
+       uint256 priceDifference = state.startingPrice - state.reservePrice;
+       uint256 timeElapsed = block.timestamp - state.startTime;
+       uint256 totalDuration = state.endTime - state.startTime;
+
+       uint256 product = priceDifference * timeElapsed;
+       uint256 quotient = divideAndRound(product, totalDuration);
+       uint256 price = state.startingPrice - quotient;
​
+       return price;
    }
```

And if we run our tests we get a rounded down result

```solidity
    function testGetPrice() public {
        console2.log(auction.getPrice()); // 1_000_000_000_000_000_000
        vm.warp(4600 seconds);
        console2.log(auction.getPrice()); // 861_111_111_111_111_111
    }
```

This also works for rounded up results when necessary.

### [I-2] Redundant code in `RAACNFT::mint()`

**Description:**

The current logic of the `::mint()` sources the price of a `RAACNFT` from the `RAACNFT::getHousePrice()` and then uses this figure to handle a refund if necessary.

But, there is absolutely no need for a refund if redundant code is removed in the `::mint()` function.

**Recommended Mitigation:**

Rewrite the `::mint()` function such that the price of a `RACCNFT` is sourced from the `::getHousePrice()` function, and this figure is transferred from the user.

```solidity
    function mint(uint256 _tokenId) public {
        uint256 price = raac_hp.tokenToHousePrice(_tokenId);
​
        // transfer erc20 from user to contract - requires pre-approval from user
        token.safeTransferFrom(msg.sender, address(this), price);
​
        // mint tokenId to user
        _safeMint(msg.sender, _tokenId);
​
        emit NFTMinted(msg.sender, _tokenId, price);
    }
```

This way, there will never be an incident where a refund is necessary because only the exact price of a `RAACNFT` is always transferred from the user.

*P.S. The `RAACNFT` inherits the `::mint()` function from the `IRAACNFT` interface. So I commented out the `IRAACNFT::mint()` function. SInce it is only used in the `RAACNFT` contract, it is safe to do so.*
