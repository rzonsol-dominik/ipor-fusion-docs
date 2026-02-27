# IPOR Fusion

> Yield optimization framework for automated on-chain asset management. Solidity 0.8.30, Foundry, BUSL-1.1.

## Build & Test
```
forge build
forge test
forge doc
```

## Architecture

```
User/Alpha
    │
    ▼
PlasmaVault (ERC4626 upgradeable)
    ├── deposit/withdraw/redeem (user)
    ├── execute(FuseAction[]) (ALPHA_ROLE) ──► delegatecall to Fuses
    ├── claimRewards (via RewardsClaimManager)
    └── governance functions (ATOMIST_ROLE)
         │
         ├── FuseManager ── addFuse/removeFuse/addBalanceFuse
         ├── FeeManager ── performance + management fees (HWM-based)
         ├── WithdrawManager ── scheduled withdrawals + instant from unallocated
         ├── RewardsClaimManager ── claim + vest + distribute rewards
         └── PriceOracleMiddleware ── Chainlink + custom feeds, 18-dec USD
              │
IporFusionAccessManager (OpenZeppelin AccessManager extended)
    ├── ADMIN_ROLE (0) ── highest, manages all roles
    ├── OWNER_ROLE (1) ── manages Guardians, Atomists
    ├── GUARDIAN_ROLE (2) ── emergency pause, cancel ops
    ├── ATOMIST_ROLE (100) ── vault config, fees, fuse management
    ├── ALPHA_ROLE (200) ── execute strategies
    ├── FUSE_MANAGER_ROLE (300) ── add/remove fuses
    ├── CLAIM_REWARDS_ROLE (600) ── claim rewards
    ├── WHITELIST_ROLE (800) ── deposit/withdraw access
    └── PUBLIC_ROLE (max uint64) ── unrestricted
```

## Key Concepts
- **PlasmaVault**: ERC4626 vault. Users deposit underlying token, receive shares. Alpha executes strategies via fuses.
- **Fuse**: Contract implementing enter()/exit() for a specific DeFi protocol. Executed via delegatecall from vault.
- **FuseAction**: `struct { address fuse; bytes data }` — passed to `PlasmaVault.execute()`.
- **Market**: Numeric ID representing a protocol integration (see IporFusionMarkets.sol). Each market has a balance fuse.
- **Balance Fuse**: Tracks position value in USD (18 decimals WAD) for a specific market via `balanceOf()`.
- **Substrate**: Protocol-specific configuration data granted to a market (e.g., allowed pool addresses).

## Contract Registry

| Contract | Source | Role |
|----------|--------|------|
| PlasmaVault | vaults/PlasmaVault.sol | Main ERC4626 vault |
| PlasmaVaultBase | vaults/PlasmaVaultBase.sol | ERC20/Permit/Votes via delegatecall |
| PlasmaVaultGovernance | vaults/PlasmaVaultGovernance.sol | Config, fuses, fees, markets |
| IporFusionAccessManager | managers/access/IporFusionAccessManager.sol | RBAC + timelocks + redemption delay |
| FeeManager | managers/fee/FeeManager.sol | Performance (HWM) + management fees |
| WithdrawManager | managers/withdraw/WithdrawManager.sol | Scheduled + instant withdrawals |
| RewardsClaimManager | managers/rewards/RewardsClaimManager.sol | Reward claiming + vesting |
| PriceOracleMiddleware | price_oracle/PriceOracleMiddleware.sol | Asset pricing, Chainlink + custom |
| PreHooksHandler | handlers/pre_hooks/PreHooksHandler.sol | Pre-execution hooks (rate validation, pause) |
| FusionFactory | factory/FusionFactory.sol | Atomic vault deployment orchestrator |
| ContextManager | managers/context/ContextManager.sol | Cross-contract sender identity |
| UniversalReader | universal_reader/UniversalReader.sol | Off-chain delegatecall queries |
| FuseWhitelist | fuses/whitelist/FuseWhitelist.sol | Global fuse registry + metadata |

## Conventions
- Solidity 0.8.30, OpenZeppelin v5 (upgradeable)
- ReentrancyGuard on PlasmaVault state-changing externals
- NatSpec on all public/external functions
- Fees: 2-decimal percentage (10000 = 100%, 100 = 1%) for perf/mgmt; WAD (1e18 = 100%) for withdraw/request fees
- Prices: always 18-decimal WAD in USD
- Market IDs: see IporFusionMarkets.sol
- Role IDs: see Roles.sol

## Documentation
See `docs/INDEX.yaml` for the full documentation manifest. Load only the files you need.
