# Protocol Invariants

## Economic
1. `totalAssets() >= totalSupply() * minSharePrice` — vault cannot lose value through execute()
2. Post-execute invariant check: `_updateMarketsBalances()` recalculates all positions; reverts if asset distribution limits violated
3. Performance fee only charged on gains above High Water Mark (HWM)
4. Management fee minted per-second, proportional to totalAssets
5. Fee percentages immutable for DAO portion (set at FeeManager construction); recipient fees updatable by ATOMIST
6. Deposit fee (WAD precision) deducted from shares before minting
7. Total supply cannot exceed `cap()` (denominated in shares, not assets)

## Access Control
8. Only ALPHA_ROLE can call `execute(FuseAction[])` — strategy execution
9. Only ATOMIST_ROLE can: add/remove fuses, configure markets, update fees, set withdraw windows
10. Only GUARDIAN_ROLE can: pause vault (`updateTargetClosed`), cancel scheduled operations
11. Only ADMIN_ROLE can: initialize AccessManager, manage OWNER_ROLE
12. Only FUSE_MANAGER_ROLE can: add/remove fuses and balance fuses via governance
13. TECH_ roles (3,5,6) are system-only — assigned to contracts at init, not changeable at runtime
14. Role execution delays enforced — `grantRole` validates `executionDelay >= minimalExecutionDelay`
15. Redemption delay: after certain actions, account locked for `REDEMPTION_DELAY_IN_SECONDS` (max 7 days)

## Fuse System
16. Only registered fuses (via `FusesLib.isFuseSupported`) can be executed — prevents unauthorized protocol calls
17. Each market has exactly 0 or 1 balance fuse — no duplicate balance tracking
18. Balance fuse cannot be removed if position balance > dust threshold
19. Fuse operations use delegatecall — fuse code runs in vault's storage context
20. Market substrates must be granted before fuse can operate on a market

## Withdrawal System
21. Scheduled withdrawal: request → releaseFunds (by ALPHA) → withdraw within window
22. Instant withdrawal from unallocated: `vault balance - sharesToRelease >= withdrawal amount`
23. Pending requests NOT released by governance do NOT reduce unallocated balance
24. Withdraw window enforced: `requestTimestamp < releaseFundsTimestamp && block.timestamp <= endWindowTimestamp`
25. Request fee (WAD) deducted from shares at request time; withdraw fee (WAD) deducted at withdrawal time
26. Max fee rate = 100% (1e18) — enforced in updateWithdrawFee/updateRequestFee

## Rewards
27. Underlying token cannot be transferred out of RewardsClaimManager (only to PlasmaVault)
28. Vesting: rewards vest linearly over `vestingTime` seconds from `updateBalanceTimestamp`
29. Only registered reward fuses can be used in `claimRewards()`

## Price Oracle
30. All prices returned in 18-decimal WAD USD
31. Custom price feed checked first; Chainlink Feed Registry as fallback (Ethereum only)
32. Price must be > 0; revert on zero/negative
33. Zero address asset always reverts with UnsupportedAsset
34. CHAINLINK_FEED_REGISTRY = address(0) disables Chainlink fallback entirely
