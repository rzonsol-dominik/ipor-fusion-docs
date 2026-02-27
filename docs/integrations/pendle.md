# Pendle Integration

## Quick Reference
- Fuse contracts: `PendleSwapPTFuse.sol`, `PendleRedeemPTAfterMaturityFuse.sol`
- Market ID: PENDLE=23
- External contracts: Pendle Router (IPActionSwapPTV3 / IPAllActionV3), IPMarket, IPPrincipalToken, IStandardizedYield

## Fuse Interface

### PendleSwapPTFuse
```
enter(market, minPtOut, guessPtOut{guessMin, guessMax, guessOffchain, maxIteration, eps}, input{tokenIn, netTokenIn, tokenMintSy, pendleSwap, swapData})
  -> returns (market, netPtOut, netSyFee, netSyInterm)
  - Swaps underlying token for PT (Principal Token) via Router.swapExactTokenForPt()
  - Market address must be granted substrate
  - tokenIn validated via SY.isValidTokenIn()
  - guessPtOut: approximation params for binary search of PT output amount
  - LimitOrderData set to empty (no limit orders used)

exit(market, exactPtIn, output{tokenOut, minTokenOut, tokenRedeemSy, pendleSwap, swapData})
  -> returns (market, netTokenOut, netSyFee, netSyInterm)
  - Swaps PT for underlying token via Router.swapExactPtForToken()
  - tokenOut validated via SY.isValidTokenOut()
  - Approves PT token to router, resets to 0 after
```

### PendleRedeemPTAfterMaturityFuse
```
enter(market, netPyIn, output{tokenOut, minTokenOut, tokenRedeemSy, pendleSwap, swapData})
  -> returns (market, netTokenOut)
  - Redeems PT+YT pair for underlying tokens AFTER maturity
  - Validates pt.isExpired() == true (reverts if not expired)
  - Calls Router.redeemPyToToken(address(this), yt, netPyIn, output)
  - No exit function (redemption is one-directional)
```

## Balance Tracking
No dedicated balance fuse in this directory. PT positions are typically tracked through external balance mechanisms or the general ERC20 balance of PT tokens held by the vault.

## Key Invariants
- Substrates are Pendle market addresses (not token addresses)
- SwapPT enter validates tokenIn against SY; exit validates tokenOut against SY
- Redeem fuse requires PT to be expired (pt.isExpired() must be true)
- Router approval set to type(uint256).max during operation, reset to 0 after
- LimitOrderData always empty (no limit order integration)

## Risks & Edge Cases
- guessPtOut approximation: if guessMin/guessMax are poorly set, swap may fail or be suboptimal
- Redeem fuse is enter-only (no exit); once redeemed, operation is irreversible
- swapData.extCalldata allows arbitrary external router calls -- must be carefully constructed
- PT price can deviate significantly from underlying near maturity

## Related Files
- Source: `contracts/fuses/pendle/`
- Pendle deps: `@pendle/core-v2/contracts/interfaces/` (IPAllActionV3, IPMarket, etc.)
