# Market Substrates System

## What are Substrates?
Substrates are **whitelisted configuration parameters** granted to a market that control what a fuse is allowed to interact with. Before a fuse can operate on a market, the Atomist must grant the appropriate substrates via governance.

Without granted substrates, fuse operations will revert — this is the primary security mechanism preventing unauthorized protocol interactions.

## How Substrates Work

### Granting
```
PlasmaVaultGovernance.grantMarketSubstrates(uint256 marketId, bytes32[] substrates)
  — ATOMIST_ROLE only
  — Atomic: revokes ALL existing substrates, then grants new ones
  — Emits MarketSubstratesGranted(marketId, substrates)

Note: PlasmaVaultConfigLib has an internal helper `grantSubstratesAsAssetsToMarket()` that auto-converts
addresses to bytes32, but this is NOT exposed as an external governance function.
```

### Checking (called by fuses during execution)
```
PlasmaVaultConfigLib.isSubstrateAsAssetGranted(marketId, address) → bool
  — For simple address substrates (tokens, pools, vaults)

PlasmaVaultConfigLib.isMarketSubstrateGranted(marketId, bytes32) → bool
  — For typed/encoded substrates (with type flags)

PlasmaVaultConfigLib.getMarketSubstrates(marketId) → bytes32[]
  — Returns all granted substrates for iteration (used by balance fuses)
```

### Storage
```
mapping(marketId => MarketSubstratesStruct) where:
  MarketSubstratesStruct {
    mapping(bytes32 => uint256) substrateAllowances;  // 1 = granted, 0 = revoked
    bytes32[] substrates;                               // ordered list for iteration
  }
```

## Encoding Patterns

### Pattern 1: Plain Address (most common)
Used by: Aave V2/V3, Compound V2/V3, Morpho, Euler, ERC4626, Moonwell, Silo, Gearbox, Fluid, Liquity, Pendle, Curve, StakeDAO, Ramses, Uniswap, Aerodrome (AMM), Velodrome (AMM)

```
bytes32 substrate = bytes32(uint256(uint160(address)))
```

Grant via `grantSubstratesAsAssetsToMarket(marketId, [tokenAddr1, tokenAddr2, ...])`.

Fuse checks: `PlasmaVaultConfigLib.isSubstrateAsAssetGranted(marketId, tokenAddress)`

**What to grant:** token addresses, pool addresses, vault addresses, cToken addresses — depends on protocol.

### Pattern 2: Type-Flagged Address (MSB = type byte)
Layout: `[type 8 bits][padding 88 bits][address 160 bits]`

Used by: **Aave V4**, **Balancer**, **Midas**, **Ebisu**, **Velodrome Slipstream**

#### Aave V4 (AaveV4SubstrateLib)
```
Types: Asset (1), Spoke (2)
Encode: AaveV4SubstrateLib.encodeAsset(token)     → flag 0x01 | address
        AaveV4SubstrateLib.encodeSpoke(spoke)     → flag 0x02 | address
```
Must grant BOTH asset tokens AND spoke contract addresses.

#### Balancer (BalancerSubstrateLib)
```
Types: GAUGE (1), POOL (2), TOKEN (3)
Encode: BalancerSubstrateLib.substrateToBytes32({substrateType, substrateAddress})
Layout: [type 96 bits | address 160 bits]
```
Must grant pool addresses, gauge addresses, AND individual token addresses.

#### Midas (MidasSubstrateLib)
```
Types: M_TOKEN (1), DEPOSIT_VAULT (2), REDEMPTION_VAULT (3), INSTANT_REDEMPTION_VAULT (4), ASSET (5)
Encode: MidasSubstrateLib.substrateToBytes32({substrateType, substrateAddress})
Layout: [type 96 bits | address 160 bits]
```
Must grant mToken, deposit vault, redemption vault(s), AND asset (USDC etc.).

#### Ebisu (EbisuZapperSubstrateLib)
```
Types: ZAPPER (1), REGISTRY (2)
Encode: EbisuZapperSubstrateLib.substrateToBytes32({substrateType, substrateAddress})
Layout: [type 96 bits | address 160 bits]
```
Balance fuse iterates only ZAPPER-type substrates.

#### Velodrome Slipstream (VelodromeSuperchainSlipstreamSubstrateLib)
```
Types: Gauge (1), Pool (2)
Encode: VelodromeSuperchainSlipstreamSubstrateLib.substrateToBytes32({substrateType, substrateAddress})
Layout: [type 96 bits | address 160 bits]
```

### Pattern 3: Type + Data (MSB = type byte, rest = mixed data)
Layout: `[type 8 bits][data 248 bits]`

Used by: **Odos**, **Velora**, **Universal Token Swapper**

#### Odos (OdosSubstrateLib)
```
Types: Token (1), Slippage (2)
Token:    OdosSubstrateLib.encodeTokenSubstrate(address)      → flag 0x01 | address
Slippage: OdosSubstrateLib.encodeSlippageSubstrate(wadValue)  → flag 0x02 | uint248
```
Slippage in WAD (1e18 = 100%). Grant allowed swap tokens + max slippage.

#### Velora (VeloraSubstrateLib)
```
Types: Token (1), Slippage (2)
Identical pattern to Odos. Grant allowed swap tokens + max slippage.
```

#### Universal Token Swapper (UniversalTokenSwapperSubstrateLib)
```
Types: Token (1), Target (2), Slippage (3)
Token:    encodeTokenSubstrate(address)   → flag 0x01 | address
Target:   encodeTargetSubstrate(address)  → flag 0x02 | address
Slippage: encodeSlippageSubstrate(wad)    → flag 0x03 | uint248
```
Must grant: allowed tokens, allowed DEX/target addresses, AND slippage limit.

### Pattern 4: Address + Selector (for call validation)
Layout: `[address 160 bits][selector 32 bits][padding 64 bits]`

Used by: **Enso** (EnsoSubstrateLib)

```
Encode: EnsoSubstrateLib.encode({target_: address, functionSelector_: bytes4})
        EnsoSubstrateLib.encodeRaw(address, bytes4)
```
Each granted substrate whitelists a specific (contract, function) pair. Fuse validates that all calls in the execution batch match granted substrates.

### Pattern 5: Complex Structs (AsyncAction)
Market ID: ASYNC_ACTION = 40

Three substrate sub-types (see IporFusionMarkets.sol comments):
- **ALLOWED_AMOUNT_TO_OUTSIDE**: `{asset, amount(uint88)}` — max transfer to AsyncExecutor
- **ALLOWED_TARGETS**: `{target, selector}` — permitted call targets
- **ALLOWED_EXIT_SLIPPAGE**: `{slippage(uint248)}` — max balance drift (WAD, 1e18=100%)

Encoded via `AsyncActionFuseLib.encodeAsyncActionFuseSubstrate(...)`.

## Per-Market Substrate Reference

| Market ID | Market | Substrate Type | What to Grant |
|-----------|--------|---------------|---------------|
| 1 | AAVE_V3 | Plain address | Token addresses (USDC, WETH, etc.) |
| 2,13,26 | COMPOUND_V3 | Plain address | Comet contract addresses |
| 3 | GEARBOX_POOL_V3 | Plain address | dToken addresses |
| 4 | GEARBOX_FARM_V3 | Plain address | farmDToken addresses |
| 5,6 | FLUID_INSTADAPP | Plain address | fToken / staking addresses |
| 7 | ERC20_VAULT_BALANCE | Plain address | Token addresses |
| 8 | UNISWAP_V3_POSITIONS | Plain address | Token pair addresses |
| 9 | UNISWAP_V2 | Plain address | Token addresses |
| 10 | UNISWAP_V3 | Plain address | Token addresses |
| 11 | EULER_V2 | Plain address | eToken vault addresses |
| 12 | UNIVERSAL_TOKEN_SWAPPER | Type+Data | Token + Target + Slippage |
| 14 | MORPHO | Plain address | Morpho market IDs (as bytes32) |
| 15 | SPARK | Plain address | Token addresses |
| 16 | CURVE_POOL | Plain address | Pool addresses |
| 17 | CURVE_LP_GAUGE | Plain address | Gauge addresses |
| 18 | RAMSES_V2 | Plain address | Token pair addresses |
| 21 | MOONWELL | Plain address | mToken addresses |
| 23 | PENDLE | Plain address | Market / PT addresses |
| 28 | TAC_STAKING | Plain address | Validator addresses |
| 29 | LIQUITY_V2 | Plain address | StabilityPool addresses |
| 30 | AERODROME | Plain address | Pool / gauge addresses |
| 31 | VELODROME_SUPERCHAIN | Plain address | Pool / gauge addresses |
| 32 | VELODROME_SLIPSTREAM | Type-flagged | Gauge + Pool typed substrates |
| 33 | AERODROME_SLIPSTREAM | Plain address | Pool / gauge addresses |
| 34 | STAKE_DAO_V2 | Plain address | Reward vault addresses |
| 35 | SILO_V2 | Plain address | SiloConfig addresses |
| 36 | BALANCER | Type-flagged | GAUGE + POOL + TOKEN typed substrates |
| 37 | YIELD_BASIS_LT | Plain address | LT token addresses |
| 38 | ENSO | Addr+Selector | (target, functionSelector) pairs |
| 39 | EBISU | Type-flagged | ZAPPER + REGISTRY typed substrates |
| 40 | ASYNC_ACTION | Complex | Amount + Targets + Slippage |
| 42 | ODOS_SWAPPER | Type+Data | Token + Slippage |
| 43 | VELORA_SWAPPER | Type+Data | Token + Slippage |
| 45 | AAVE_V4 | Type-flagged | Asset + Spoke typed substrates |
| 45 | MIDAS | Type-flagged | M_TOKEN + DEPOSIT_VAULT + REDEMPTION_VAULT + INSTANT_REDEMPTION_VAULT + ASSET |
| 100_001+ | ERC4626 | Plain address | ERC4626 vault addresses |
| 200_001+ | META_MORPHO | Plain address | MetaMorpho vault addresses |

## Key Invariants
1. Substrates are the primary authorization mechanism — fuse operations without matching substrates revert
2. `grantMarketSubstrates` is atomic: ALL old substrates revoked before new ones applied
3. Balance fuses iterate `getMarketSubstrates()` to find all positions to value
4. Substrate encoding must exactly match what the fuse checks — wrong type flag = silent denial
5. Only ATOMIST_ROLE can grant/revoke substrates

## Related Files
- `contracts/libraries/PlasmaVaultConfigLib.sol` — core substrate logic
- `contracts/libraries/PlasmaVaultStorageLib.sol` — storage structs
- Each `*SubstrateLib.sol` — encoding/decoding for typed substrates
