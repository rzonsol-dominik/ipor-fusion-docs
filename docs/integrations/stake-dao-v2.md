# Stake DAO V2 Integration

## Quick Reference
- Fuse contracts: `StakeDaoV2SupplyFuse.sol`, `StakeDaoV2BalanceFuse.sol`
- Market ID: STAKE_DAO_V2=34
- External contracts: Reward Vault (ERC4626), LP Token Vault (ERC4626), IRewardVault, IAccountant

## Architecture
Stake DAO V2 has a nested ERC4626 structure:
```
Underlying Asset (e.g. crvUSD)
  -> LP Token Vault (ERC4626, e.g. cvcrvUSD)
    -> Reward Vault (ERC4626, e.g. sd-cvcrvUSD-vault)
```

## Fuse Interface

### StakeDaoV2SupplyFuse
```
enter(rewardVault, lpTokenUnderlyingAmount, minLpTokenUnderlyingAmount)
  -> returns (rewardVault, rewardVaultShares, lpTokenAmount, finalLpTokenUnderlyingAmount)
  - Two-step deposit:
    1. Deposit underlying into LP Token Vault: lpTokenAmount = lpTokenVault.deposit(amount)
    2. Deposit LP tokens into Reward Vault: shares = rewardVault.deposit(lpTokenAmount)
  - Reverts if final amount < minLpTokenUnderlyingAmount
  - rewardVault must be granted substrate

exit(rewardVault, rewardVaultShares, minRewardVaultShares)
  -> returns (rewardVault, finalRewardVaultShares, lpTokenAmount, lpTokenUnderlyingAmount)
  - Two-step withdrawal:
    1. Redeem reward vault shares: lpTokenAmount = rewardVault.redeem(shares)
    2. Redeem LP token shares: underlying = lpTokenVault.redeem(lpTokenAmount)
  - Caps shares to available balance
  - Reverts if final shares < minRewardVaultShares

instantWithdraw(params[amount, rewardVaultAddress])
  - Converts underlying amount -> LP shares -> reward vault shares
  - Uses try/catch on both redeem steps
  - Caps to available reward vault balance
```

## Balance Tracking
`StakeDaoV2BalanceFuse.balanceOf()`:
- For each reward vault substrate:
  - `rewardVaultAssets = rewardVault.balanceOf(plasmaVault)` (1:1 shares in StakeDAO V2)
  - `lpTokenAddress = rewardVault.asset()`
  - `lpTokenAssets = lpTokenVault.convertToAssets(rewardVaultAssets)` (LP token shares -> underlying)
  - `lpTokenUnderlying = lpTokenVault.asset()`
  - Gets underlying price from PriceOracleMiddleware
  - Converts to USD in WAD

Important: In StakeDAO V2, deposited assets are 1:1 with reward vault shares, so no convertToAssets needed at the reward vault level.

## Key Invariants
- Substrates are reward vault addresses
- Reward vault shares are 1:1 with reward vault assets (no conversion needed)
- LP token vault IS ERC4626 compatible (uses convertToAssets for balance)
- Price feed required for the LP token's underlying asset
- Enter requires underlying asset pre-prepared in the vault

## Risks & Edge Cases
- Double try/catch in instantWithdraw: if LP token redeem fails, reward vault shares are still burned
- LP token vault may have withdrawal limits affecting instant withdrawals
- Reward vault balance may not perfectly reflect underlying value during reward distribution
- minRewardVaultShares/minLpTokenUnderlyingAmount provide slippage protection

## Related Files
- Source: `contracts/fuses/stake_dao_v2/`
- Ext: `ext/IRewardVault.sol`, `ext/IAccountant.sol`
