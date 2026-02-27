# ERC4626 Vault Pattern

## PlasmaVault as ERC4626
PlasmaVault extends ERC4626Upgradeable — standard tokenized vault interface.

## Standard Interface
```
deposit(assets, receiver) → shares    — deposit underlying, receive vault shares
mint(shares, receiver) → assets       — mint exact shares, pull required assets
withdraw(assets, receiver, owner) → shares  — burn shares for exact asset amount
redeem(shares, receiver, owner) → assets    — burn exact shares for assets

totalAssets() → uint256               — total underlying managed by vault
convertToShares(assets) → shares      — preview conversion
convertToAssets(shares) → assets      — preview conversion
maxDeposit/maxMint/maxWithdraw/maxRedeem — capacity limits
```

## PlasmaVault Extensions to ERC4626
1. **Access Control**: deposit/mint gated by WHITELIST_ROLE (or PUBLIC_ROLE after conversion)
2. **Supply Cap**: `cap()` limits total shares (not assets)
3. **Deposit Fee**: deducted from minted shares via FeeManager
4. **Dual Withdrawal Path**: instant (unallocated) or scheduled (WithdrawManager)
5. **Performance Fee**: HWM-based, minted as shares to FeeAccount
6. **Management Fee**: time-based, minted as shares to FeeAccount
7. **totalAssets()**: includes positions across all DeFi protocols (via balance fuses), not just vault's token balance

## Share Price Calculation
```
totalAssets = liquid underlying balance + sum(marketBalances) + RewardsClaimManager.balanceOf()
sharePrice = totalAssets / totalSupply
```

Market balances are read via delegatecall to balance fuses, each returning WAD USD. Converted to underlying via PriceOracleMiddleware.

## Key Difference from Standard ERC4626
- Withdrawal may fail if assets are deployed in protocols (not liquid in vault)
- Alpha must execute fuse exits to make assets available
- Instant withdrawal only from "unallocated" balance (vault's direct token holdings minus sharesToRelease)
