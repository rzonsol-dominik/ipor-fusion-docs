# Fuse Pattern

## What is a Fuse?
A fuse is a stateless contract that implements a specific DeFi protocol interaction. It is executed via `delegatecall` from PlasmaVault, meaning it runs in the vault's storage context.

## Lifecycle
```
1. Registration: Atomist calls PlasmaVaultGovernance.addFuses([fuseAddress])
2. Market Setup: Atomist grants market substrates for the fuse's market
3. Balance Fuse: Atomist calls addBalanceFuse(marketId, balanceFuseAddress)
4. Execution: Alpha calls PlasmaVault.execute([FuseAction{fuse, data}])
5. Post-check: Vault calls _updateMarketsBalances() → delegatecalls balance fuses
```

## FuseAction Struct
```solidity
struct FuseAction {
    address fuse;   // registered fuse contract address
    bytes data;     // ABI-encoded function call (enter/exit data)
}
```

## Fuse Types

### Supply Fuse (most common)
- `enter(EnterData)` — deposit/supply assets into external protocol
- `exit(ExitData)` — withdraw assets back from external protocol
- Example: AaveV3SupplyFuse, CompoundV3SupplyFuse, Erc4626SupplyFuse

### Borrow Fuse
- `enter(EnterData)` — borrow assets from protocol
- `exit(ExitData)` — repay borrowed assets
- Example: AaveV3BorrowFuse, MorphoBorrowFuse, EulerV2BorrowFuse

### Collateral Fuse
- `enter(EnterData)` — enable asset as collateral
- `exit(ExitData)` — disable collateral
- Example: AaveV3CollateralFuse, MorphoCollateralFuse

### Liquidity Fuse (DEX)
- `enter(EnterData)` — add liquidity / open position
- `exit(ExitData)` — remove liquidity / close position
- Example: AerodromeLiquidityFuse, BalancerLiquidityProportionalFuse

### Position Fuse (concentrated liquidity)
- `newPosition(Data)` — mint new LP position (returns tokenId)
- `modifyPosition(Data)` — add/remove from existing position
- `collect(Data)` — collect fees from position
- Example: UniswapV3NewPositionFuse, RamsesV2NewPositionFuse

### Swap Fuse
- `enter(Data)` — execute token swap
- Example: UniswapV3SwapFuse, OdosSwapperFuse, VeloraSwapperFuse

### Balance Fuse
- `balanceOf()` → uint256 (WAD USD value)
- Executed via delegatecall from vault to read position value
- One per market. Returns total position value in 18-decimal USD.
- Example: AaveV3BalanceFuse, Erc20BalanceFuse

### Reward Fuse
- Used via RewardsClaimManager, not PlasmaVault.execute()
- Claims protocol incentives (COMP, MORPHO, CRV, etc.)
- Separate registry from main fuses

## Common Interface
```solidity
interface IFuseCommon {
    function MARKET_ID() external view returns (uint256);
}

interface IFuse {
    function enter(bytes calldata data_) external;
    function exit(bytes calldata data_) external;
}
```

All fuses expose `MARKET_ID()` (links to IporFusionMarkets constant). `enter`/`exit` accept raw `bytes calldata` which is ABI-decoded into fuse-specific structs internally. Most fuse implementations also have `address public immutable VERSION = address(this)` for transient storage identification (not part of the interface).

## Substrate Validation
Before a fuse can operate, the market must have granted substrates:
```
PlasmaVaultGovernance.grantMarketSubstrates(marketId, bytes32[] substrates)
```
Substrates are protocol-specific (pool addresses, token addresses, config params). Each fuse validates its operations against granted substrates.

## Security Model
- Fuses run via delegatecall → they access vault's storage and token balances
- Only registered fuses can be called (FusesLib.isFuseSupported check)
- Post-execution balance check catches invariant violations
- Market substrates restrict which external contracts/pools a fuse can interact with
