# PreHooks & Callback Handlers

## Overview

PreHooks are interceptors that run **before** specific PlasmaVault functions via `delegatecall`, enforcing invariants, updating state, or blocking execution. Callback Handlers are stateless contracts that process protocol-specific callbacks (e.g., flash loans) during Fuse execution, transforming callback data into `FuseAction` arrays the vault can execute.

## Architecture

```
Vault function called
  -> PreHooksHandler._runPreHook(msg.sig)
    -> PreHooksLib.getPreHookImplementation(selector) // storage lookup
    -> if implementation != address(0):
         delegatecall implementation.run(selector)  // runs in vault's storage context
  -> main function body executes
```

Pre-hooks execute in the vault's storage context (delegatecall), so they can read/write vault state. Each function selector maps to at most one pre-hook implementation. Hooks are optional -- if no implementation is registered for a selector, execution proceeds normally.

### Storage Layout (PreHooksConfig)

| Field | Type | Purpose |
|---|---|---|
| `hooksImplementation` | `mapping(bytes4 => address)` | Selector to hook implementation |
| `selectors` | `bytes4[]` | All registered selectors |
| `indexes` | `mapping(bytes4 => uint256)` | Selector position in array (O(1) removal) |
| `substrates` | `mapping(bytes32 => bytes32[])` | Per-hook config, keyed by `keccak256(implementation, selector)` |

## IPreHook Interface

```solidity
interface IPreHook {
    function run(bytes4 selector_) external;
}
```

All pre-hook contracts implement this single-function interface. The `selector_` parameter identifies which vault function triggered the hook.

## PreHooksHandler

Abstract contract inherited by PlasmaVault. Exposes one internal function:

```solidity
function _runPreHook(bytes4 selector_) internal
```

Looks up the implementation for `selector_`, skips if `address(0)`, otherwise calls `implementation.run(selector_)` via delegatecall.

## Available Pre-Hooks

### ExchangeRateValidatorPreHook

**Purpose:** Prevents exchange rate manipulation by validating the vault's share-to-asset rate stays within a configurable threshold band.

- **Constructor:** `ExchangeRateValidatorPreHook(uint256 marketId_)` -- binds to a specific market.
- **Configuration:** Substrates for the market encode a list of typed entries (HookType enum: PREHOOKS, POSTHOOKS, VALIDATOR). Supports up to 10 pre-hooks and 10 post-hooks that run around the validator.
- **Validation logic:**
  - Computes `current = convertToAssets(10^decimals)`.
  - Compares against stored `expected` rate with `threshold` (1e18 = 100%).
  - **Reverts** `ExchangeRateOutOfRange` if `|current - expected| > threshold * expected / 1e18`.
  - If deviation is between `threshold/2` and `threshold`, auto-updates the stored expected rate in substrates.
  - If within `threshold/2`, no update needed.
- **Substrate source:** Uses the market's substrates (via `PlasmaVaultLib.getMarketSubstrates(marketId)`), not the PreHooksConfig substrates mapping.

### PauseFunctionPreHook

**Purpose:** Emergency kill-switch for individual vault functions.

- Stateless, no constructor arguments, no configuration.
- `run()` is a `pure` function that always reverts with `FunctionPaused(selector)`.
- Register it for any selector to block that function; remove it to unpause.

### UpdateBalancesPreHook

**Purpose:** Refreshes all active market balances before a vault operation.

- Calls `FusesLib.getActiveMarketsInBalanceFuses()` to get market IDs.
- Calls `PlasmaVault(address(this)).updateMarketsBalances(marketIds)`.
- No configuration needed. Updates every active market regardless of balance size.

### UpdateBalancesIgnoreDustPreHook

**Purpose:** Same as UpdateBalancesPreHook but skips markets with balances below a dust threshold.

- **Configuration:** Requires exactly one substrate value -- the dust threshold in the vault's underlying token decimals (e.g., `1e6` for 1 USDC).
- Filters markets: only those with `totalAssetsInMarket >= dustThreshold` are updated.
- Reduces gas cost when many markets hold negligible amounts.
- Reverts `InvalidSubstratesLength` if substrates are misconfigured.

### ValidateAllAssetsPricesPreHook

**Purpose:** Validates prices for all configured assets via the PriceOracleMiddlewareManager before an operation.

- Reads `PlasmaVaultLib.getPriceOracleMiddleware()` and calls `validateAllAssetsPrices()`.
- Reverts `PriceOracleMiddlewareManagerNotConfigured` if the oracle manager is not set.
- No constructor args, no substrates.

### EIP7702DelegateValidationPreHook

**Purpose:** Validates that EIP-7702 delegated accounts have whitelisted delegate targets.

- **EIP-7702 detection:** Checks if the effective sender (context sender or `msg.sender`) has exactly 23 bytes of code starting with `0xef0100`. If so, extracts the 20-byte delegate target address.
- **Whitelist:** Substrates for this hook contain allowed delegate target addresses (as `bytes32`-encoded addresses).
- Regular EOAs and contracts with non-23-byte code pass without checks.
- Reverts `InvalidDelegateTarget(sender, delegateTarget)` if the delegate target is not whitelisted.
- Uses `ContextClientStorageLib.getSenderFromContext()` to support calls through ContextManager.

## Callback Handlers

Callback handlers are registered per `(sender, functionSig)` pair via `CallbackHandlerLib.updateCallbackHandler()`. During Fuse execution, the vault's fallback function routes callbacks to the registered handler using a regular `call` (not delegatecall -- handlers are stateless and cannot access vault storage).

The handler returns a `CallbackData` struct:

```solidity
struct CallbackData {
    address asset;              // Token to approve
    address addressToApprove;   // Spender address
    uint256 amountToApprove;    // Approval amount
    bytes actionData;           // Encoded FuseAction[] to execute
}
```

After the handler returns, the vault decodes `actionData` into `FuseAction[]`, executes them via `executeInternal()`, and sets the token approval.

### CallbackHandlerMorpho

Handles two Morpho protocol callbacks:

- `onMorphoSupply(uint256 assets, bytes data)` -- triggered during supply operations.
- `onMorphoFlashLoan(uint256 assets, bytes data)` -- triggered during flash loans.

Both decode `data` as `CallbackData` and return it.

### CallbackHandlerEuler

Handles one Euler protocol callback:

- `onEulerFlashLoan(bytes data)` -- triggered during Euler flash loans.

Decodes `data` as `CallbackData` and returns it.

## Registration

Pre-hooks are registered via governance calling:

```solidity
PreHooksLib.setPreHookImplementations(
    bytes4[] selectors,       // Function selectors to hook
    address[] implementations, // Hook contract addresses (address(0) to remove)
    bytes32[][] substrates     // Per-hook configuration data
)
```

**State transitions per selector:**
- **Add:** `oldImpl == 0, newImpl != 0` -- pushes selector to array, stores substrates.
- **Update:** `oldImpl != 0, newImpl != 0` -- replaces implementation and substrates.
- **Remove:** `oldImpl != 0, newImpl == 0` -- swap-and-pop removal from selector array, cleans up storage.

Callback handlers are registered via:

```solidity
CallbackHandlerLib.updateCallbackHandler(address handler, address sender, bytes4 sig)
```

Both registration functions are restricted to the ATOMIST role through PlasmaVaultGovernance.

## Key Invariants

1. **Execution context:** Pre-hooks run via `delegatecall` and share the vault's storage. Callback handlers run via `call` and cannot access vault storage.
2. **Single hook per selector:** Each function selector maps to exactly one pre-hook implementation (not a chain).
3. **Optional by default:** If no hook is registered for a selector, the function proceeds without interruption.
4. **Governance-only registration:** Only ATOMIST role can add, update, or remove hooks and callback handlers.
5. **Atomic removal:** Removing a hook cleans up implementation mapping, selector array, index mapping, and substrates in one transaction.
6. **Exchange rate banding:** The ExchangeRateValidator uses a two-tier threshold (full band for revert, half band for auto-update) to prevent rate manipulation while allowing organic drift.

## Related Files

| File | Path |
|---|---|
| IPreHook | `contracts/handlers/pre_hooks/IPreHook.sol` |
| PreHooksHandler | `contracts/handlers/pre_hooks/PreHooksHandler.sol` |
| PreHooksLib | `contracts/handlers/pre_hooks/PreHooksLib.sol` |
| ExchangeRateValidatorPreHook | `contracts/handlers/pre_hooks/pre_hooks/ExchangeRateValidatorPreHook.sol` |
| ExchangeRateValidatorConfigLib | `contracts/handlers/pre_hooks/pre_hooks/ExchangeRateValidatorConfigLib.sol` |
| PauseFunctionPreHook | `contracts/handlers/pre_hooks/pre_hooks/PauseFunctionPreHook.sol` |
| UpdateBalancesPreHook | `contracts/handlers/pre_hooks/pre_hooks/UpdateBalancesPreHook.sol` |
| UpdateBalancesIgnoreDustPreHook | `contracts/handlers/pre_hooks/pre_hooks/UpdateBalancesIgnoreDustPreHook.sol` |
| ValidateAllAssetsPricesPreHook | `contracts/handlers/pre_hooks/pre_hooks/ValidateAllAssetsPricesPreHook.sol` |
| EIP7702DelegateValidationPreHook | `contracts/handlers/pre_hooks/pre_hooks/EIP7702DelegateValidationPreHook.sol` |
| CallbackHandlerMorpho | `contracts/handlers/callbacks/CallbackHandlerMorpho.sol` |
| CallbackHandlerEuler | `contracts/handlers/callbacks/CallbackHandlerEuler.sol` |
| CallbackHandlerLib | `contracts/libraries/CallbackHandlerLib.sol` |
| PlasmaVaultStorageLib (PreHooksConfig) | `contracts/libraries/PlasmaVaultStorageLib.sol` |
