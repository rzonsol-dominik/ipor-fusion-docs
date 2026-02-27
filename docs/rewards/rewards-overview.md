# Rewards System Overview

## Architecture
```
External Protocol (Aave, Morpho, Curve, etc.)
    │ incentive tokens
    ▼
RewardsClaimManager.claimRewards(FuseAction[])
    │ delegatecall to reward fuse
    ▼
Reward Fuse (e.g. MorphoClaimFuse)
    │ claims tokens to RewardsClaimManager
    ▼
RewardsClaimManager (holds + vests rewards)
    │ vesting over vestingTime
    ▼
PlasmaVault (receives vested underlying token)
```

## Flow
1. CLAIM_REWARDS_ROLE calls `RewardsClaimManager.claimRewards(FuseAction[])`
2. Manager validates each fuse is registered, delegates to PlasmaVault.claimRewards
3. Reward fuse executes protocol-specific claim (e.g., Merkl.claim, CometRewards.claim)
4. Tokens arrive in RewardsClaimManager
5. Non-underlying tokens: TRANSFER_REWARDS_ROLE can send elsewhere (e.g., swap)
6. Underlying tokens: vest linearly, transferVestedTokensToVault() moves to vault

## Reward Fuse Registry
Separate from main vault fuse registry. Managed by FUSE_MANAGER_ROLE:
- `addRewardFuses(address[])`
- `removeRewardFuses(address[])`

## Available Reward Fuses

| Fuse | Source | Protocol | Claim Target |
|------|--------|----------|-------------|
| MerklClaimFuse | rewards_fuses/merkl/ | Merkl (universal) | IDistributor.claim() |
| MorphoClaimFuse | rewards_fuses/morpho/ | Morpho Blue | IUniversalRewardsDistributor.claim() |
| CompoundV3ClaimFuse | rewards_fuses/compound/ | Compound V3 | ICometRewards.claim() |
| CurveGaugeTokenClaimFuse | rewards_fuses/curve_gauges/ | Curve Gauge | gauge.claim_rewards() |
| RewardEulerTokenClaimFuse | rewards_fuses/euler/ | Euler V2 | IREUL.claim() |
| FluidInstadappClaimFuse | rewards_fuses/fluid_instadapp/ | Fluid | Direct claim |
| FluidProofClaimFuse | rewards_fuses/fluid_instadapp/ | Fluid (Merkle) | IFluidMerkleDistributor.claim() |
| GearboxV3FarmDTokenClaimFuse | rewards_fuses/gearbox_v3/ | Gearbox V3 | Farm dToken claim |
| MoonwellClaimFuse | rewards_fuses/moonwell/ | Moonwell | Comptroller claim |
| AerodromeGaugeClaimFuse | rewards_fuses/aerodrome/ | Aerodrome | Gauge.getReward() |
| AreodromeSlipstreamGaugeClaimFuse | rewards_fuses/areodrome_slipstream/ | Aerodrome Slipstream | CL Gauge claim |
| VelodromeSuperchainGaugeClaimFuse | rewards_fuses/velodrome_superchain/ | Velodrome | Gauge claim |
| VelodromeSuperchainSlipstreamGaugeClaimFuse | rewards_fuses/velodrome_superchain/ | Velodrome Slipstream | CL Gauge claim |
| RamsesClaimFuse | rewards_fuses/ramses/ | Ramses V2 | Gauge claim |
| StakeDaoV2ClaimFuse | rewards_fuses/stake_dao_v2/ | Stake DAO V2 | Reward vault claim |
| SyrupClaimFuse | rewards_fuses/syrup/ | Syrup | ISyrup.claim() |
