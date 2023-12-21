## Findings Summary

| Label | Description |
| - | - |
| [L-01](#) | It's unreasonable that LP's token0Owed can only be less than or equal to Lien.tokenToPremium, especially when there is a profit|
| [L-02](#) | openPosition() maybe underflow|
| [L-03](#) | In `mint()`, it is suggested to judge `amount0ToMint/amount1ToMint` >0 before executing `transferFrom()` |
| [L-04](#) | suggested ParticlePositionManager use ReentrancyGuardUpgradeable instead of ReentrancyGuard |
| [L-05](#) |add a minimum limit to premium0 and premium1 in `addPremium()` |

## [L-01] It's unreasonable that LP's token0Owed can only be less than or equal to Lien.tokenToPremium, especially when there is a profit

When calculating the Yield from Borrowed Liquidity that should be paid to `LP` during `closePostion()`, the current protocol restricts it to be less than or equal to `Lien.tokenToPremium`.

This is not reasonable when the borrower is making a profit.

For example:
lien.collateral = 100  
lien.premium = 20
amountFromAdd = 80 (only repay 80, the borrower still profits 20)
token0Owed =  30

According to the current protocol calculation method:
LP.token0Owed = 20   (because token0Owed > premium, only get premium, lost 10)
borrower profit = 100 + 20 - 80 -20 =20

This way, the document's description that needs to ensure LP's greater than `Yield from Borrowed Liquidity` cannot be satisfied.

Because the `borrower` always has a profit, a part of the profit should be deducted to give `LP` to satisfy greater than `Yield from Borrowed Liquidity`.

Note: Liquidation will cause `premium` to be less, `LP.token0Owed` will be less.

Suggestion:
If there is a profit and the Premium is insufficient, deduct a part from the profit to supplement `token0Owed`.

```
      token0Owed = Math.min(cache.token0Owed,(collateralFrom + cache.tokenFromPremium - cache.amountSpent - cache.amountFromAdd ))
```

## [L-02] openPosition() maybe underflow

in `openPosition()` -> `Base.swap()`
```solidity
    function openPosition(
        DataStruct.OpenPositionParams calldata params
    ) public override nonReentrant returns (uint96 lienId, uint256 collateralTo) {
...
        (cache.amountSpent, cache.amountReceived) = Base.swap(
            cache.tokenFrom,
            cache.tokenTo,
            params.amountSwap,
@>          collateralTo - cache.amountToBorrowed - params.marginTo, // amount needed to meet requirement
            DEX_AGGREGATOR,
            params.data
        );
```

The formula `collateralTo - cache.amountToBorrowed - params.marginTo` may underflow.

This is especially true if the user wants to increase `token0PremiumPortion`, thereby transferring to `marginTo`.

For example: (price outOfRange)
collateralTo = 100
amountToBorrowed = 100
marginTo = 10 (want to set token0PremiumPortion >10%)

`collateralTo - amountToBorrowed - marginTo` will underflow.

Suggestions:
```diff
        (cache.amountSpent, cache.amountReceived) = Base.swap(
            cache.tokenFrom,
            cache.tokenTo,
            params.amountSwap,
-           collateralTo - cache.amountToBorrowed - params.marginTo, // amount needed to meet requirement
+          (cache.amountToBorrowed + params.marginTo) > collateralTo ? 0 : (collateralTo - cache.amountToBorrowed - params.marginTo),
            DEX_AGGREGATOR,
            params.data
        );
```
## [L-03] In `mint()`, it is suggested to judge `amount0ToMint/amount1ToMint` >0 before executing `transferFrom()`
In `mint()`, regardless of whether the specified `amount0ToMint/amount1ToMint` is greater than 0, `transferFrom()` is executed. But when the price is `OutOfRange`, `amount0ToMint` or `amount1ToMint` is not needed. 
In this case, the user will input `0`. 
This will cause the transfer of some tokens to fail if `0` is transferred.
https://github.com/d-xo/weird-erc20/?tab=readme-ov-file#revert-on-zero-value-transfers

suggest:
```diff
    function mint(
        mapping(uint256 => Info) storage self,
        DataStruct.MintParams calldata params
    ) internal returns (uint256 tokenId, uint128 liquidity, uint256 amount0Minted, uint256 amount1Minted) {
        // transfer in the tokens
+      if(params.amount0ToMint > 0) {
        TransferHelper.safeTransferFrom(params.token0, msg.sender, address(this), params.amount0ToMint);
+       }
+      if(params.amount0ToMint > 0) {
        TransferHelper.safeTransferFrom(params.token1, msg.sender, address(this), params.amount1ToMint);
+      }
```

## [L-04] suggested ParticlePositionManager use ReentrancyGuardUpgradeable instead of ReentrancyGuard
`ParticlePositionManager.sol` uses `UUPSUpgradeable`. It is suggested to use `ReentrancyGuardUpgradeable` to avoid subsequent upgrade storage collisions. Currently, `ReentrancyGuard.sol` is used.

```diff
contract ParticlePositionManager is
    IParticlePositionManager,
    Ownable2StepUpgradeable,
    UUPSUpgradeable,
    IERC721Receiver,
-   ReentrancyGuard,
+  ReentrancyGuardUpgradeable,
    Multicall
{
```

    function initialize(
        address dexAggregator,
        uint256 feeFactor,
        uint128 liquidationRewardFactor,
        uint256 loanTerm,
        uint256 treasuryRate
    ) external initializer {
+    __ReentrancyGuard_init()

## [L-05] add a minimum limit to premium0 and premium1 in `addPremium()`
`addPremium()` is used to increase `premium`, and at the same time, it will modify `lien.startTime`. 
`lien.startTime` will extend the expiration time of the `loan`. 
At present, there is no restriction on premium0 and premium1, and both can be 0. 
In this way, users can extend the expiration time at no cost. 
It is suggested to add restrictions, at least one of them should have a growth percentage greater than `MIN_ADD_PREMIUM_PORTION=10%`.




