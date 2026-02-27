# Market IDs Reference

Source: `contracts/libraries/IporFusionMarkets.sol`

## Protocol Markets

| ID | Constant | Protocol |
|----|----------|----------|
| 1 | AAVE_V3 | Aave V3 |
| 2 | COMPOUND_V3_USDC | Compound V3 (USDC) |
| 3 | GEARBOX_POOL_V3 | Gearbox V3 Pool |
| 4 | GEARBOX_FARM_DTOKEN_V3 | Gearbox V3 Farm (depends on 3) |
| 5 | FLUID_INSTADAPP_POOL | Fluid Instadapp Pool |
| 6 | FLUID_INSTADAPP_STAKING | Fluid Instadapp Staking |
| 7 | ERC20_VAULT_BALANCE | ERC20 vault balance tracking |
| 8 | UNISWAP_SWAP_V3_POSITIONS | Uniswap V3 LP positions (depends on 7) |
| 9 | UNISWAP_SWAP_V2 | Uniswap V2 swaps (depends on 7) |
| 10 | UNISWAP_SWAP_V3 | Uniswap V3 swaps (depends on 7) |
| 11 | EULER_V2 | Euler V2 |
| 12 | UNIVERSAL_TOKEN_SWAPPER | Token swapper (depends on 7) |
| 13 | COMPOUND_V3_USDT | Compound V3 (USDT) |
| 14 | MORPHO | Morpho Blue |
| 15 | SPARK | Spark (Ethereum) |
| 16 | CURVE_POOL | Curve StableSwap NG |
| 17 | CURVE_LP_GAUGE | Curve LP Gauge |
| 18 | RAMSES_V2_POSITIONS | Ramses V2 LP positions |
| 19 | MORPHO_FLASH_LOAN | Morpho flash loans (depends on 7) |
| 20 | AAVE_V3_LIDO | Aave V3 Lido market |
| 21 | MOONWELL | Moonwell |
| 22 | MORPHO_REWARDS | Morpho rewards |
| 23 | PENDLE | Pendle |
| 24 | FLUID_REWARDS | Fluid rewards |
| 25 | CURVE_GAUGE_ERC4626 | Curve Gauge ERC4626 |
| 26 | COMPOUND_V3_WETH | Compound V3 (WETH) |
| 27 | HARVEST_HARD_WORK | Harvest hard work |
| 28 | TAC_STAKING | TAC staking |
| 29 | LIQUITY_V2 | Liquity V2 |
| 30 | AERODROME | Aerodrome |
| 31 | VELODROME_SUPERCHAIN | Velodrome Superchain |
| 32 | VELODROME_SUPERCHAIN_SLIPSTREAM | Velodrome Superchain Slipstream |
| 33 | AREODROME_SLIPSTREAM | Aerodrome Slipstream |
| 34 | STAKE_DAO_V2 | Stake DAO V2 |
| 35 | SILO_V2 | Silo V2 |
| 36 | BALANCER | Balancer |
| 37 | YIELD_BASIS_LT | Yield Basis LT |
| 38 | ENSO | Enso Finance |
| 39 | EBISU | Ebisu |
| 40 | ASYNC_ACTION | Async Action |
| 41 | MORPHO_LIQUIDITY_IN_MARKETS | Morpho liquidity |
| 42 | ODOS_SWAPPER | Odos |
| 43 | VELORA_SWAPPER | Velora |
| 45 | AAVE_V4 / MIDAS | Aave V4 / Midas RWA (shared ID!) |

## ERC4626 Vault Markets
| ID Range | Constant Pattern | Count |
|----------|-----------------|-------|
| 100_001 - 100_020 | ERC4626_0001 - ERC4626_0020 | 20 slots |

## Meta Morpho Markets
| ID Range | Constant Pattern | Count |
|----------|-----------------|-------|
| 200_001 - 200_010 | META_MORPHO_0001 - META_MORPHO_0010 | 10 slots |

## Special Markets
| ID | Constant | Purpose |
|----|----------|---------|
| max-2 | EXCHANGE_RATE_VALIDATOR | Pre-hook exchange rate validation |
| max-1 | ASSETS_BALANCE_VALIDATION | Balance validation only |
| max | ZERO_BALANCE_MARKET | No balance tracking needed |

## Dependency Graph
Some markets depend on ERC20_VAULT_BALANCE (ID 7) being registered:
- UNISWAP_SWAP_V3_POSITIONS (8)
- UNISWAP_SWAP_V2 (9), UNISWAP_SWAP_V3 (10)
- UNIVERSAL_TOKEN_SWAPPER (12)
- MORPHO_FLASH_LOAN (19)

GEARBOX_FARM_DTOKEN_V3 (4) depends on GEARBOX_POOL_V3 (3).
