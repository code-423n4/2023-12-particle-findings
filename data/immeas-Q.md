# QA Report

## Summary

| id | title |
| --- | --- |
| [L-01](#l-01-particlepositionmanageronerc721received-allows-all-erc721-tokens) | `ParticlePositionManager::onERC721Received` allows all ERC721 tokens |
| [L-02](#l-02-particlepositionmanager-can-be-initialized-with-bad-params) | `ParticlePositionManager` can be initialized with bad params |
| [L-03](#l-03-fee-tier-100-is-also-supported-by-uniswap) | Fee tier `100` is also supported by Uniswap |
| [L-04](#l-04-use-of-non-upgradable-reentrancyguard) | Use of non-upgradable `ReentrancyGuard` |
| [L-05](#l-05-some-tokens-revert-on-0-amount-transfer) | Some tokens revert on `0` amount transfer |
| [L-06](#l-06-uniswap-positions-minted-to-particlepositionmanager-are-stuck) | Uniswap positions minted to `ParticlePositionManager` are stuck |
| [NC-01](#nc-01-liquidation-criteria-very-complicated) | Liquidation criteria very complicated |
| [NC-02](#nc-02-confusing-parameter-naming-in-particlepositionmanagerswap) | Confusing parameter naming in `ParticlePositionManager::swap` |
| [NC-03](#nc-03-erroneous-comment) | Erroneous comment |

## Low

### L-01 `ParticlePositionManager::onERC721Received` allows all ERC721 tokens

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

#### Recommendation
Consider only allowing `UNI_POSITION_MANAGER` to send ERC721 tokens to the contract and reverting otherwise.

### L-02 `ParticlePositionManager` can be initialized with bad params

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

#### PoC
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

#### Recommendation
Consider using the setter functions already declared, `update*(...)`, to initialize the contract parameters. That would guarantee that the initialization is done using the same validation as changing the parameters. 

### L-03 Fee tier `100` is also supported by Uniswap

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

If you look at the contract:
https://etherscan.io/address/0x1f98431c8ad98523631ae4a59f267346ea31f984#readContract
it returns tick spacing `1` for fee tier `100`

Hence this might not return the pool with the deepest liquidity.

#### Recommendation
Consider adding `100` to the list of fee tiers

### L-04 Use of non-upgradable `ReentrancyGuard`

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

#### Recommendation
Consider using `ReentrancyGuardUpgradeable` from OZ.

### L-05 Some tokens revert on `0` amount transfer

[`ParticlePositionManager::liquidatePosition`](https://github.com/code-423n4/2023-12-particle/blob/main/contracts/protocol/ParticlePositionManager.sol#L376-L378):
```solidity
File: protocol/ParticlePositionManager.sol

376:        // reward liquidator
377:        TransferHelper.safeTransfer(closeCache.tokenFrom, msg.sender, liquidateCache.liquidationRewardFrom);
378:        TransferHelper.safeTransfer(closeCache.tokenTo, msg.sender, liquidateCache.liquidationRewardTo);
```

[Some tokens, like `LEND`](https://github.com/d-xo/weird-erc20?tab=readme-ov-file#revert-on-zero-value-transfers) revert on `0` amount transfer. If the liquidation amount is `0` for this token this would make liquidation impossible for this position.

#### Recommendation
Consider checking if the `liquidationAmount > 0` before transferring liquidation reward.

### L-06 Uniswap positions minted to `ParticlePositionManager` are stuck

Using the `NonfungiblePositionManager` directly a user could potentially mint a position directly to `ParticlePositionManager` by having `ParticlePositionManager` as the `receiver` when calling [`NonfungiblePositionManager::mint`](https://github.com/Uniswap/v3-periphery/blob/main/contracts/NonfungiblePositionManager.sol#L156).

Since `NonfungiblePositionManager` doesn't use `safeMint` this would just transfer the token and liquidity to `ParticlePositionManager` without calling `ParticlePositionManager::onERC721Received`, thus the liquidity would be lost.

#### Recommendation
Consider adding a rescue function callable by `owner` that can transfer any accidentally minted tokens out of the contract. This could check that `ParticlePositionManager` is the owner of the token but has `lps.owner == address(0)` to prevent misuse.

## Informational / Non crit

### NC-01 Liquidation criteria very complicated

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

### NC-02 Confusing parameter naming in `ParticlePositionManager::swap`

It's called `token0`/`1` but later in the swap library its `tokenFrom`/`To`.

The parameters would be easier to understand if they were from/to in `ParticlePositionManager::swap` as well.

### NC-03 Erroneous comment

[`ParticlePositionManager::_closePosition`](https://github.com/code-423n4/2023-12-particle/blob/main/contracts/protocol/ParticlePositionManager.sol#L422):
```solidity
File: contracts/protocol/ParticlePositionManager.sol

422:        // add liquidity back to borrower
```

`borrower` is not correct here, liquidity is added back to the lender/liquidity provider.