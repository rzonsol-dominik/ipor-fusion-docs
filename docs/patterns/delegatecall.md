# Delegatecall Architecture

## How Fuses Execute
PlasmaVault uses `delegatecall` to execute fuse logic in its own storage context:

```
PlasmaVault.execute(FuseAction[] actions)
  for each action:
    action.fuse.functionDelegateCall(action.data)
```

This means:
- Fuse code runs with vault's `address(this)`, `msg.sender`, and storage
- Fuse can read/write vault's token balances
- Fuse can approve external protocols on behalf of vault
- Fuse's own storage is NOT accessed

## Why Delegatecall?
1. **Single contract holds all assets** — no token transfers between vault and fuse
2. **Composability** — fuse can call `IERC20(asset).approve(protocol, amount)` as the vault
3. **Atomic execution** — multiple fuse actions in one tx
4. **Upgradeability** — new fuses can be added without migrating assets

## PlasmaVaultBase Delegatecall
PlasmaVault also delegatecalls to PlasmaVaultBase for ERC20/Permit/Votes functionality:
```
PlasmaVault._update() → PlasmaVaultBase.updateInternal() via delegatecall
```
This splits the vault into two contracts to stay under the 24KB contract size limit.

## Plugin System
```
PlasmaVaultVotesPlugin — optional voting power tracking
  Enabled by setting votesPlugin address in storage
  Called via delegatecall on every transfer
  Saves ~2800-9800 gas per transfer when disabled
```

## Security Implications
- Fuse code has FULL access to vault storage and funds
- Only registered fuses can be called (FusesLib check)
- Post-execution invariant check catches malicious/buggy fuses
- Substrate validation restricts which external calls a fuse can make
- Fuse registration is governance-only (FUSE_MANAGER_ROLE)
