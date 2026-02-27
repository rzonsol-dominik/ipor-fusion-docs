# IporFusionAccessManager

## Quick Reference
- Source: `contracts/managers/access/IporFusionAccessManager.sol`
- Extends: OpenZeppelin AccessManager + Initializable
- Roles defined in: `contracts/libraries/Roles.sol`

## Role Hierarchy

| ID | Name | Managed By | Purpose |
|----|------|-----------|---------|
| 0 | ADMIN_ROLE | Self | Highest role, manages all. Use MultiSig. |
| 1 | OWNER_ROLE | Admin | Manages Guardians, Atomists. Use MultiSig. |
| 2 | GUARDIAN_ROLE | Owner | Emergency pause, cancel ops |
| 3 | TECH_PLASMA_VAULT_ROLE | System (init only) | PlasmaVault contract calls only |
| 4 | IPOR_DAO_ROLE | System | DAO operations (fee recipient) |
| 5 | TECH_CONTEXT_MANAGER_ROLE | System (init only) | ContextManager calls only |
| 6 | TECH_WITHDRAW_MANAGER_ROLE | System (init only) | WithdrawManager calls only |
| 7 | TECH_VAULT_TRANSFER_SHARES_ROLE | System | Transfer/transferFrom gating |
| 100 | ATOMIST_ROLE | Owner | Vault config, fees, fuse mgmt |
| 200 | ALPHA_ROLE | Atomist | Execute strategies |
| 300 | FUSE_MANAGER_ROLE | Atomist | Add/remove fuses |
| 301 | PRE_HOOKS_MANAGER_ROLE | Atomist | Add/remove pre-hooks |
| 400 | TECH_PERFORMANCE_FEE_MANAGER_ROLE | System (init only) | FeeManager perf ops |
| 500 | TECH_MANAGEMENT_FEE_MANAGER_ROLE | System (init only) | FeeManager mgmt ops |
| 600 | CLAIM_REWARDS_ROLE | Atomist | Claim rewards |
| 601 | TECH_REWARDS_CLAIM_MANAGER_ROLE | System (init only) | RewardsClaimManager calls |
| 700 | TRANSFER_REWARDS_ROLE | Atomist | Transfer rewards |
| 800 | WHITELIST_ROLE | Atomist | Deposit/withdraw access |
| 900 | CONFIG_INSTANT_WITHDRAWAL_FUSES_ROLE | Atomist | Configure instant withdrawal order |
| 901 | WITHDRAW_MANAGER_REQUEST_FEE_ROLE | Atomist | Update withdrawal request fees |
| 902 | WITHDRAW_MANAGER_WITHDRAW_FEE_ROLE | Atomist | Update withdrawal withdraw fees |
| 1000 | UPDATE_MARKETS_BALANCES_ROLE | Atomist | Update market balances |
| 1100 | UPDATE_REWARDS_BALANCE_ROLE | Atomist | Update rewards balance |
| 1200 | PRICE_ORACLE_MIDDLEWARE_MANAGER_ROLE | Atomist | Manage price oracle |
| max uint64 | PUBLIC_ROLE | — | No restrictions |

## Key Functions
```
initialize(InitializationData) — ADMIN only, one-time. Sets up all role→function mappings, grants, delays.
canCallAndUpdate(caller, target, selector) → (bool immediate, uint32 delay)
  — Checks permission + updates redemption delay state.
updateTargetClosed(target, closed) — GUARDIAN. Emergency pause.
convertToPublicVault(vault) — Sets deposit/mint/depositWithPermit to PUBLIC_ROLE.
enableTransferShares(vault) — Sets transfer/transferFrom to PUBLIC_ROLE.
grantRole(roleId, account, executionDelay) — Validates delay >= minimalExecutionDelay.
setMinimalExecutionDelaysForRoles(roleIds[], delays[])
proxyInitialize(address initialAdmin, uint256 redemptionDelayInSeconds) — clone-based init
getMinimalExecutionDelayForRole(uint64 roleId) → uint256
getAccountLockTime(address account) → uint256
isConsumingScheduledOp() → bytes4
MAX_REDEMPTION_DELAY_IN_SECONDS = 7 days (constant)
```

## Security Features
- Role execution delays: minimum delay enforced per role
- Redemption delay: `REDEMPTION_DELAY_IN_SECONDS` (max 7 days), locks account after certain operations
- Guardian can cancel scheduled operations and pause targets
- TECH_ roles locked to contracts at init — cannot be reassigned

## Related Files
- RoleExecutionTimelockLib.sol — minimal execution delay storage
- RedemptionDelayLib.sol — account lock time tracking
- IporFusionAccessManagerInitializationLib.sol — initialization data structures
