# RewardsClaimManager

## Quick Reference
- Source: `contracts/managers/rewards/RewardsClaimManager.sol`
- Purpose: Claim, vest, and distribute protocol rewards to PlasmaVault
- Has its own fuse registry (separate from vault's main fuses)

## Interface

### Public
```
balanceOf() → uint256
  — Current vested balance available. Linear vesting from updateBalanceTimestamp.
  — If vestingTime=0: returns full underlying balance.
  — If ratio >= 1e18 (fully vested): lastUpdateBalance - transferredTokens.

isRewardFuseSupported(address fuse) → bool
getVestingData() → VestingData { vestingTime, updateBalanceTimestamp, lastUpdateBalance, transferredTokens }
getRewardsFuses() → address[]

transferVestedTokensToVault() — PUBLIC. Transfers vested tokens to PlasmaVault.
```

### CLAIM_REWARDS_ROLE
```
claimRewards(FuseAction[] calls)
  — Validates each fuse is registered, then calls PlasmaVault.claimRewards().
```

### TRANSFER_REWARDS_ROLE
```
transfer(address asset, address to, uint256 amount)
  — Transfer non-underlying tokens out. Cannot transfer underlying token.
```

### UPDATE_REWARDS_BALANCE_ROLE
```
updateBalance()
  — Transfers vested balance to vault, resets vesting with current balance.
```

### FUSE_MANAGER_ROLE
```
addRewardFuses(address[] fuses)
removeRewardFuses(address[] fuses)
```

### ATOMIST_ROLE
```
setupVestingTime(uint256 vestingTimeInSeconds)
```

## Vesting Mechanism
```
1. updateBalance() called → transfers vested tokens to vault, snapshots new balance
2. Vesting starts from updateBalanceTimestamp
3. ratio = (block.timestamp - updateBalanceTimestamp) * 1e18 / vestingTime
4. Available = lastUpdateBalance * min(ratio, 1e18) / 1e18 - transferredTokens
5. transferVestedTokensToVault() can be called anytime to move vested portion
```

## Key Invariants
- Underlying token can ONLY go to PlasmaVault (never external)
- Non-underlying tokens can be transferred via TRANSFER_REWARDS_ROLE
- Default vesting time = 1 second (effectively instant)
- Reward fuses are separate from vault execution fuses

## Related Files
- RewardsClaimManagersStorageLib.sol — VestingData, underlying token, plasma vault address
