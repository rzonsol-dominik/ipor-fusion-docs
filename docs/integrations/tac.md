# TAC Staking Integration

## Quick Reference
- Fuse contracts: `TacStakingDelegateFuse.sol`, `TacStakingRedelegateFuse.sol`, `TacStakingEmergencyFuse.sol`, `TacStakingBalanceFuse.sol`
- Delegator: `TacStakingDelegator.sol`
- Libraries: `TacStakingStorageLib.sol`, `TacValidatorAddressConverter.sol`
- Market ID: `TAC_STAKING = 28`
- **Note**: These contracts are marked as deprecated.

## Overview

TAC is a Cosmos EVM chain with native staking via validator delegation. The integration wraps wTAC (ERC20) to native TAC and delegates to validators via the Cosmos staking precompile. A TacStakingDelegator contract acts as the delegation intermediary, stored in ERC-7201 namespaced storage.

## Fuse Interface

### TacStakingDelegateFuse

```
enter(data: {validatorAddresses[], wTacAmounts[]}):
  1. Validate arrays non-empty and same length
  2. Create delegator if not exists
  3. Validate each validator address against substrates (two bytes32 slots)
  4. Transfer total wTAC to delegator
  5. Call delegator.delegate(validators, amounts)
     -> Unwraps wTAC to native TAC
     -> Calls IStaking.delegate() for each validator

exit(data: {validatorAddresses[], tacAmounts[]}):
  1. Validate arrays and substrates
  2. Call delegator.undelegate(validators, amounts)
     -> Calls IStaking.undelegate() for each validator
     -> Wraps remaining native TAC and transfers back

instantWithdraw(params):
  -> Withdraws available wTAC and native TAC from delegator
```

### TacStakingRedelegateFuse

```
enter(data: {validatorSrcAddresses[], validatorDstAddresses[], wTacAmounts[]}):
  -> Moves stake from source validators to destination validators
  -> Validates both source and destination validators as substrates
```

### TacStakingEmergencyFuse

```
exit():
  -> Wraps all native TAC in delegator to wTAC
  -> Transfers all wTAC back to PlasmaVault
```

## Substrate Configuration

Validator addresses are encoded as two consecutive bytes32 slots (the validator string is split across them). Both slots must be granted for a validator to be considered allowed. Substrates must have an even count.

## Balance Tracking

`TacStakingBalanceFuse.balanceOf()`:
- Iterates validator substrates (pairs of bytes32)
- For each validator, queries `IStaking.delegation()` for staked balance
- Also queries `IStaking.unbondingDelegation()` for unbonding entries
- Adds delegator's native TAC balance
- Converts total TAC to USD via PriceOracleMiddleware using wTAC price

## Key Invariants

- Validator addresses require two substrate slots (must be even total)
- Delegator is created once and stored in ERC-7201 storage
- Only PlasmaVault can call delegator functions
- Native TAC is always wrapped to wTAC before transfer back to vault
- All contracts in this module are marked as deprecated

## Related Files
- Source: `contracts/fuses/tac/TacStakingDelegateFuse.sol`
- Source: `contracts/fuses/tac/TacStakingRedelegateFuse.sol`
- Source: `contracts/fuses/tac/TacStakingEmergencyFuse.sol`
- Source: `contracts/fuses/tac/TacStakingBalanceFuse.sol`
- Source: `contracts/fuses/tac/TacStakingDelegator.sol`
- Source: `contracts/fuses/tac/lib/TacStakingStorageLib.sol`
