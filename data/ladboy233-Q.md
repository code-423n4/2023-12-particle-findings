| Number | Title |
|--------|-------|
| 1 | Gas Inefficiency in WithdrawTreasury Function |
| 2 | Limited Fee Tiers Assumption in Uniswap V3 |
| 3 | Lack of view function to render Liquidation status in ParticleInfoReader.sol |
| 4 | Hardcoded Uniswap V3 Position Manager Address |
| 5 | Absence of NFT Transfer/Burn Methods in ParticlePositionManager.sol |


# consider batch withdraw protocol fee

the current [withdraw fee](https://github.com/code-423n4/2023-12-particle/blob/a3af40839b24aa13f5764d4f84933dbfa8bc8134/contracts/protocol/ParticlePositionManager.sol#L595) function is too gas inefficient

it is likely a wide range of Uniswap  V3 NFT will be deposit into the Position manager and the Position Manager will accumulate a lots of fee but settled in different type of ERC20 token

if the protocol has to call the function 

```solidity
function withdrawTreasury(address token, address recipient) external override onlyOwner nonReentrant {
	uint256 withdrawAmount = _treasury[token];
	if (withdrawAmount > 0) {
		if (recipient == address(0)) {
			revert Errors.InvalidRecipient();
		}
		_treasury[token] = 0;
		TransferHelper.safeTransfer(token, recipient, withdrawAmount);
		emit WithdrawTreasury(token, recipient, withdrawAmount);
	}
}
```

every time to withdraw the token

it is not gas efficient

it is recommend to take the token address array as parameter so the protocol can withdraw multiple token fee in one single transaction

# uniswap v3 can add more tiers

the code assumes that there will only be three fee tiers

however, the uniswap protocol governance is capable of adding more fee tiers, 

https://docs.uniswap.org/concepts/protocol/fees#pool-fees-tiers

> Uniswap v3 introduces multiple pools for each token pair, each with a different swapping fee. Liquidity providers may initially create pools at three fee levels: 0.05%, 0.30%, and 1%. More fee levels may be added by UNI governance, e.g. the 0.01% fee level added by this governance proposal in November 2021, as executed here.

then after the new fee tier is added, the function [getDeepPool](https://github.com/code-423n4/2023-12-particle/blob/a3af40839b24aa13f5764d4f84933dbfa8bc8134/contracts/protocol/ParticleInfoReader.sol#L103) may not return the pool with deepest liquidity

```
    function getDeepPool(address token0, address token1) external view returns (address deepPool) {
        uint24[3] memory feeTiers = [uint24(500), uint24(3000), uint24(10000)];
        uint128 maxLiquidity = 0;

        for (uint256 i = 0; i < feeTiers.length; i++) {
            address poolAddress = Base.UNI_FACTORY.getPool(token0, token1, feeTiers[i]);
            if (poolAddress != address(0)) {
                IUniswapV3Pool pool = IUniswapV3Pool(poolAddress);
                uint128 liquidity = pool.liquidity();
                if (liquidity > maxLiquidity) {
                    maxLiquidity = liquidity;
                    deepPool = poolAddress;
                }
            }
        }
    }

```

it is recommend to make the feeTiers configurable


# Should expose whether position the is liquidable as a view function in ParticleIinfoReader.sol

In [ParticleInfoReader.sol](https://github.com/code-423n4/2023-12-particle/blob/a3af40839b24aa13f5764d4f84933dbfa8bc8134/contracts/protocol/ParticleInfoReader.sol#L21)

there is no view function to validate and expose if a single position is liquidable 

it is recommendated to wrap the liquidation check to a view function and expose whether the position is liquidable for potential liquidator

```solidity
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
```


# Avoid hardcode address

THe Uniswap V3 Position manager address is hardcoded ,

however, if the protocol wants to deploy in multiple EVM network

the Uniswap V3 Position Manager address maybe different

https://docs.uniswap.org/contracts/v3/reference/deployments

for example, the Uniswap V3 Position Manager address is different from ethereum mainnet blockchain and base network blockchain

it is recommend to pass in the position manager as a parameter and pass the address into the constructor

# Lack of method to transfer NFT out in ParticlePositionManager.sol

In [ParticlePositionManager.sol](https://github.com/code-423n4/2023-12-particle/blob/a3af40839b24aa13f5764d4f84933dbfa8bc8134/contracts/protocol/ParticlePositionManager.sol#L19)

User can mint and create Uniswap V3 Position NFT directly in Position Manager,

user can increase liquidity, decrease liquidity or harvest the claim the reawrd

however once the Uniswap V3 NFT is transferred to the contract, 

there is lack of function to transfer the NFT out 

the bigger problem is as long as user's nft is in the particle position contract, anyone can borrow the fund out even while original owner does not want the fund to be borrowed

so if the original nft does not want others to borrow fund, he cannot do that so he will choose to only remove liquidity but not add liquidity once he does not want to borrow any more

and there is lack of function to burn the NFT as well

it is recommended to implement the function to transfer V3 NFT out in PositionManager.sol