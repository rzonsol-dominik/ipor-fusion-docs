# Protocol Invariants

## Economic
1. `totalAssets() >= totalSupply() * minSharePrice` — vault cannot lose value through execute()
2. Post-execute invariant check: `_updateMarketsBalances()` recalculates all positions; reverts if asset distribution limits violated
3. Performance fee only charged on gains above High Water Mark (HWM)
4. Management fee minted per-second, proportional to totalAssets
5. Fee percentages immutable for DAO portion (set at FeeManager construction); recipient fees updatable by ATOMIST
6. Deposit fee (WAD precision) deducted from shares before minting
7. Total supply cannot exceed `getTotalSupplyCap()` (denominated in shares, not assets)
8. Performance fee max 50% (`PERFORMANCE_MAX_FEE_IN_PERCENTAGE`), management fee max 5% (`MANAGEMENT_MAX_FEE_IN_PERCENTAGE`)

## Access Control
9. Only ALPHA_ROLE can call `execute(FuseAction[])` — strategy execution
10. Only ATOMIST_ROLE can: configure markets, update fees, set withdraw windows
11. Only FUSE_MANAGER_ROLE can: add/remove fuses and balance fuses via governance
12. Only GUARDIAN_ROLE can: pause vault (`updateTargetClosed`), cancel scheduled operations
13. Only ADMIN_ROLE can: initialize AccessManager, manage OWNER_ROLE
14. TECH_ roles (3,5,6) are system-only — assigned to contracts at init, not changeable at runtime
15. Role execution delays enforced — `grantRole` validates `executionDelay >= minimalExecutionDelay`
16. Redemption delay: after certain actions, account locked for `REDEMPTION_DELAY_IN_SECONDS` (max 7 days)

## Fuse System
17. Only registered fuses (via `FusesLib.isFuseSupported`) can be executed — prevents unauthorized protocol calls
18. Each market has exactly 0 or 1 balance fuse — no duplicate balance tracking
19. Balance fuse cannot be removed if position balance > dust threshold
20. Fuse operations use delegatecall — fuse code runs in vault's storage context
21. Market substrates must be granted before fuse can operate on a market

## Withdrawal System
22. Scheduled withdrawal: request → releaseFunds (by ALPHA) → withdraw within window
23. Instant withdrawal from unallocated: `vault balance - sharesToRelease >= withdrawal amount`
24. Pending requests NOT released by governance do NOT reduce unallocated balance
25. Withdraw window enforced: `requestTimestamp < releaseFundsTimestamp && block.timestamp <= endWindowTimestamp`
26. Request fee (WAD) deducted from shares at request time; withdraw fee (WAD) deducted at withdrawal time
27. Max fee rate = 100% (1e18) — enforced in updateWithdrawFee/updateRequestFee

## Rewards
28. Underlying token cannot be transferred out of RewardsClaimManager (only to PlasmaVault)
29. Vesting: rewards vest linearly over `vestingTime` seconds from `updateBalanceTimestamp`
30. Only registered reward fuses can be used in `claimRewards()`

## Price Oracle
31. All prices returned in 18-decimal WAD USD
32. Custom price feed checked first; Chainlink Feed Registry as fallback (Ethereum only)
33. Price must be > 0; revert on zero/negative
34. Zero address asset always reverts with UnsupportedAsset
35. CHAINLINK_FEED_REGISTRY = address(0) disables Chainlink fallback entirely
