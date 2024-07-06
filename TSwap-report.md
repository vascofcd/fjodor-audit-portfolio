## [H-1] In `_swap` function users are rewarded with extra tokens potentially draining the entire protocol

## Summary
- In the `_swap` function, for each 10 swaps executed, the user is rewarded with `1_000_000_000_000_000_000` extra tokens. This extra incentive breaks the protocol invariant of `x * y =  k` meaning that over time the protocol funds will be drained.

## Vulnerability Details
- Block where for each 10 swaps executed, user is rewarded with extra tokens.
```javascript
    swap_count++;
    if (swap_count >= SWAP_COUNT_MAX) {
        swap_count = 0;
        outputToken.safeTransfer(msg.sender, 1_000_000_000_000_000_000);
    }
```

- PoC - Copy this test into `TSwapPool.t.sol`
```javascript
    function testSwapExactOutputBreaksInvariant() public {
        vm.startPrank(liquidityProvider);
        weth.approve(address(pool), 100e18);
        poolToken.approve(address(pool), 100e18);
        pool.deposit(100e18, 100e18, 100e18, uint64(block.timestamp));
        vm.stopPrank();

        uint256 outputAmount = 1e17;

        vm.startPrank(user);
        poolToken.approve(address(pool), type(uint256).max);
        poolToken.mint(user, 100e18);

        for (uint256 i = 0; i < 9; i++) {
            pool.swapExactOutput({
                inputToken: poolToken,
                outputToken: weth,
                outputAmount: outputAmount,
                deadline: uint64(block.timestamp)
            });
        }

        uint256 startingY = weth.balanceOf(address(pool));
        int256 expectedDeltaY = int256(-1) * int256(outputAmount);

        //This is the 10th swap where user receives extra tokens (breaking the invariant)
        pool.swapExactOutput({
            inputToken: poolToken,
            outputToken: weth,
            outputAmount: outputAmount,
            deadline: uint64(block.timestamp)
        });

        vm.stopPrank();

        uint256 endingY = weth.balanceOf(address(pool));
        int256 actualDeltaY = int256(endingY) - int256(startingY);

        // The change of token balance of weth (∆y) is not the expected!
        assertEq(expectedDeltaY, actualDeltaY); // expectedDeltaY -1e17 != actualDeltaY -1.1e18
    }
```
## Impact
-  A user could maliciously drain the protocol's funds by repeatedly executing swaps and collecting the extra incentives provided by the protocol.

## Tools Used
- Manual review

## Recommendations
- Remove the extra rewards implementation (as this mechanism is not described in the documentation)
```diff
-    swap_count++;
-    if (swap_count >= SWAP_COUNT_MAX) {
-        swap_count = 0;
-        outputToken.safeTransfer(msg.sender, 1_000_000_000_000_000_000);
-    }
```
- Implement a system similar to fee collection.
- Change the protocol invariant to incorporate this extra reward.

## [H-2] The `TSwapPool.getInputAmountBasedOnOutput` function inaccurately calculates fees from user swaps, leading to losses

## Summary
- The TSwap protocol is intended to charge a 0.3% fee on swaps by applying a 997/1000 multiplier. However, it is erroneously applying a 997/10_000 multiplier, meaning that the user is paying 10 times what they should be paying. This discrepancy means the protocol is collecting substantially higher fees than designed.

## Vulnerability Details
- PoC - Copy this test into `TSwapPool.t.sol`.
```javascript
    function testShouldReturnTheCorrectInputAmountBasedOnOutputAmount() public {
        uint256 outputAmount = 10e18;
        uint256 inputReserves = 100e18;
        uint256 outputReserves = 100e18;

        //According to documentation: 'Each applies a 997 out of 1000 multiplier.'
        uint256 expectedInputAmount = ((inputReserves * outputAmount) * 1000) / ((outputReserves - outputAmount) * 997);
        
        uint256 actualInputAmount = pool.getInputAmountBasedOnOutput(outputAmount, inputReserves, outputReserves);

        //actualInputAmount is 10 times higher than expectedInputAmount
        assertEq(expectedInputAmount, actualInputAmount);
    }
```
## Impact
- The protocol takes more fees than expected from users, resulting in a significant loss.
  
## Tools Used
- Manual review

## Recommendations
- Change from `10000` to `1_000`.
```diff
    function getInputAmountBasedOnOutput(
        uint256 outputAmount,
        uint256 inputReserves,
        uint256 outputReserves
    )
        public
        pure
        revertIfZero(outputAmount)
        revertIfZero(outputReserves)
        returns (uint256 inputAmount)
    {
-        return ((inputReserves * outputAmount) * 10000) / ((outputReserves - outputAmount) * 997);
+        return ((inputReserves * outputAmount) * 1_000) / ((outputReserves - outputAmount) * 997);
    }
```
## [H-3] Incorrect swapping mechanism in `TSwapPool.sellPoolTokens` leads to user losses

## Summary 
- In the `sellPoolTokens` function users specify the number of pool tokens they want to sell using the `poolTokenAmount` parameter. However, the function currently miscalculates the swapped amount as shown on the PoC below. This issue arises because the `swapExactOutput` function is called instead of the correct `swapExactInput` function. Users provide the exact amount of input tokens, not the output amount.
- `sellPoolTokens` tokens is also missing a deadline check meaning users might inadvertently be selling at an unfavorable rate.
  
## Vulnerability Details
- PoC - Copy this test into `TSwapPool.t.sol` 
- According to the test below, the user's actual balance is lower than the expected balance, indicating the user is selling for less.
```javascript
       function testSellPoolTokens() public {
        vm.startPrank(liquidityProvider);
        weth.approve(address(pool), 100e18);
        poolToken.approve(address(pool), 100e18);
        pool.deposit(100e18, 100e18, 100e18, uint64(block.timestamp));
        vm.stopPrank();

        vm.startPrank(user);
        poolToken.approve(address(pool), type(uint256).max);

        //User has starting 1e19 of pool token  balance
        uint256 startingUserBalance = poolToken.balanceOf(user);
        uint256 poolTokenAmounToSell = 1e10;

        pool.sellPoolTokens(poolTokenAmounToSell);

        uint256 expectedFinalUserBalance = startingUserBalance - poolTokenAmounToSell;
        uint256 actualFinalUserBalance = poolToken.balanceOf(user);
        
        assertEq(expectedFinalUserBalance, actualFinalUserBalance); //expected: 9999999990000000000 vs actual: 9999999899699097282
        //Something is wrong, the actual final balance is lower!
    }
```
## Impact
- Users will swap the wrong amount, resulting in losses.
- Seeling tokens without deadline may occur at an unfavorable rates.
  
## Tools Used
- Manual review
  
## Recommendations
- Add a new parameter, for example, `expectedWethAmount`, which represents the minimum amount of weth to be received
- Add a `deadline` parameter to be passed to `swapExactInput`
```diff
    function sellPoolTokens(
        uint256 poolTokenAmount,
+       uint256 expectedWethAmount,
+       uint64 deadline   
        ) external returns (uint256 wethAmount) {
-        return swapExactOutput(i_poolToken, i_wethToken, poolTokenAmount, uint64(block.timestamp));
+        return swapExactInput(i_poolToken, poolTokenAmount, i_wethToken, expectedWethAmount, deadline);
    }
```

## [M-1] `TSwapPool.deposit` is missing a deadline check potentially result in users receiving fewer liquidity tokens than expected

## Summary
- In `TSwapPool.deposit` function, the `deadline` parameter is passed but not used, resulting in a missing deadline check. Consequently, a user can add liquidity at unexpected times when the deposit rate is unfavorable.

## Vulnerability Details
```javascript
function deposit(
        uint256 wethToDeposit,
        uint256 minimumLiquidityTokensToMint,
        uint256 maximumPoolTokensToDeposit,
    @>  uint64 deadline //unused!
    )
```
## Impact
- Users might inadvertently deposit at an unfavorable rate, resulting users receiving significantly fewer liquidity tokens than they would under normal conditions.

## Tools Used
- Manual review

## Recommendations
- Use the already implemented `revertIfDeadlinePassed`.
```diff
function deposit(
        uint256 wethToDeposit,
        uint256 minimumLiquidityTokensToMint,
        uint256 maximumPoolTokensToDeposit,
        uint64 deadline
    )
        external
+       revertIfDeadlinePassed(deadline)
        revertIfZero(wethToDeposit)
        returns (uint256 liquidityTokensToMint)
    {}
```