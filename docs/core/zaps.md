# Zaps

## Overview

Zap contracts enable single-transaction deposit workflows into ERC4626 vaults. They execute arbitrary call sequences (token swaps, wrapping, etc.), then deposit the resulting asset balance into a target vault. Two variants exist: one for ERC20-only flows and one that additionally handles native ETH.

## Architecture

```
User approves tokens to ZAP_IN_ALLOWANCE_CONTRACT
  -> zapIn(ZapInData) on ERC4626ZapIn
    -> executes Call[] (swaps, wraps, etc.)
    -> deposits resulting balance into vault
    -> refunds leftover tokens to sender
```

The `ERC4626ZapInAllowance` helper contract is deployed by the zap constructor. Users approve their tokens to this helper, and during `zapIn`, the helper transfers tokens from the user to the zap contract. This separation ensures the zap contract only pulls tokens during an active zap operation (`currentZapSender != address(0)`).

## Contracts

### ERC4626ZapIn

**Source:** `contracts/zaps/ERC4626ZapIn.sol`

| Field | Type | Description |
|-------|------|-------------|
| `ZAP_IN_ALLOWANCE_CONTRACT` | `address immutable` | Deployed in constructor |
| `currentZapSender` | `address` | Set during `zapIn`, cleared after |

```solidity
function zapIn(ZapInData calldata) external nonReentrant returns (bytes[] memory results)
```

`ZapInData` struct:
- `vault`: target ERC4626 vault
- `receiver`: share recipient
- `minAmountToDeposit`: minimum asset balance required before deposit
- `minSharesOut`: slippage protection on minted shares
- `assetsToRefundToSender`: tokens to return if leftover balance exists
- `calls`: array of `Call{target, data}` to execute

### ERC4626ZapInWithNativeToken

**Source:** `contracts/zaps/ERC4626ZapInWithNativeToken.sol`

Same pattern as `ERC4626ZapIn` but:
- Accepts `payable` calls with native ETH via `msg.value`
- Each `Call` includes a `nativeTokenAmount` field for `functionCallWithValue`
- Refunds leftover native ETH to sender after deposit
- Supports optional referral codes via `ReferralPlasmaVault` integration
- Owner can set referral contract address once, then ownership is renounced

```solidity
function zapIn(ZapInData calldata) public payable returns (bytes[] memory)
function zapIn(ZapInData calldata, bytes32 referralCode) public payable nonReentrant returns (bytes[] memory)
```

### ERC4626ZapInAllowance

**Source:** `contracts/zaps/ERC4626ZapInAllowance.sol`

Helper that transfers pre-approved tokens from the user to the zap contract.

```solidity
function transferApprovedAssets(asset, amount) external onlyERC4626ZapIn
```

Reverts if `currentZapSender` is `address(0)` (no active zap), preventing unauthorized pulls. Rejects all direct ETH transfers.

## Key Invariants

- `currentZapSender` is only non-zero during an active `zapIn` call (set in modifier, cleared after).
- `transferApprovedAssets` can only be called by the parent zap contract.
- `minAmountToDeposit` and `minSharesOut` provide two layers of slippage protection.
- All leftover tokens and native ETH are refunded to the original sender, not `receiver`.
- Reentrancy guard on all `zapIn` entry points.

## Related Files

- `contracts/zaps/` -- all zap contracts; `contracts/vaults/extensions/ReferralPlasmaVault.sol` -- referral integration
