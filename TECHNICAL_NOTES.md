# Technical Implementation Notes

## Problem Statement

Design and implement a MEV sandwich bot that can:
1. Calculate optimal front-run amounts given victim trade parameters
2. Execute profitable sandwich attacks on constant-product AMMs
3. Handle edge cases and numerical stability issues

## Mathematical Foundation

### Constant Product AMM

The AMM maintains invariant: `x * y = k`

For a swap of `dx` tokens:
```
x_new = x_old + dx * (1 - fee)
y_new = k / x_new
dy = y_old - y_new
```

### Sandwich Attack Mechanics

Given:
- Initial reserves: (x₀, y₀)
- Victim trade: dx_victim
- Victim slippage tolerance: s (in basis points)

Goal: Find maximum dx_front such that:
1. Victim still gets acceptable output (≥ minimum)
2. Final profit > 0

### Quadratic Optimization

The victim expects minimum output based on pre-attack reserves:
```
minOut = (dx_victim * 0.997 * y₀) / x₀ * (1 - s/10000)
```

After front-run to state (x₁, y₁), victim's output must satisfy:
```
k * dv / (x₁ * (x₁ + dv)) ≥ minOut
```

where `dv = dx_victim * 0.997` (fee-adjusted)

Rearranging:
```
minOut * x₁² + minOut * dv * x₁ - k * dv ≤ 0
```

This is a quadratic inequality. Maximum x₁ is:
```
x₁_max = (-b + √(b² + 4ac)) / 2a

where:
  a = minOut
  b = minOut * dv
  c = -k * dv
```

## Implementation Challenges

### Challenge 1: Integer Overflow

**Problem**: Computing `b² + 4ac` with realistic token amounts (1000 ether = 10²¹ wei) causes overflow.

Example:
```
k = 1000e18 * 1000e18 = 10⁴⁸
4 * minOut * k * dv ≈ 4 * 18e18 * 10⁴⁸ * 20e18 ≈ 1.4e⁸⁷
```

This exceeds `uint256` max (≈1.16e⁷⁷).

**Solution**: Scale all intermediate calculations by dividing by 10⁹:

```solidity
uint256 k_scaled = (x0 / 1e9) * (y0 / 1e9);
uint256 vm_scaled = victimMinOut / 1e9;
uint256 dv_scaled = dv / 1e9;

uint256 bSq_scaled = vm_scaled * vm_scaled * dv_scaled * dv_scaled;
uint256 fourAC_scaled = 4 * vm_scaled * k_scaled * dv_scaled;
```

After solving in scaled space, multiply result by 10⁹ to restore original magnitude.

**Trade-off**: Loses 9 decimal places of precision, but maintains sufficient accuracy for 18-decimal tokens.

### Challenge 2: Fee Conversion

**Problem**: Quadratic gives fee-adjusted amount `df`, but AMM requires gross input.

**Solution**: Convert using fee formula:
```solidity
dxFront = (dfMax * 10000) / 9970
```

### Challenge 3: Profit Verification

**Problem**: Mathematical solution may not yield actual profit due to rounding or edge cases.

**Solution**: Simulate entire sandwich sequence and verify `profit > 0` before returning.

## Gas Optimization Techniques

1. **Early Returns**: Check zero/invalid inputs first to save gas
2. **Scope Blocks**: Use `{}` to free temporary variables from stack
3. **Minimal Storage**: All calculations in memory, only reading immutable state
4. **Inline Helpers**: `_applyFee()` and `_sqrt()` are internal for reduced call overhead

## Testing Strategy

### Unit Tests
- `testComputeFrontRunDeterministicAmount`: Validates exact output against known value
- `testComputeFrontRunZeroWhenSlippageTooTight`: Checks slippage enforcement
- `testVictimSlippageBinding`: Verifies tight slippage prevents attack

### Integration Tests
- `testFrontAndBackRunProfitableAndPaysOwner`: Full sandwich flow
- `testFrontRunRevertsWhenInsufficient`: Balance validation

### Edge Case Tests
- `testFrontRunZeroNoRevert`: Zero-amount handling
- `testBackRunZeroNoRevert`: Zero-balance handling

## Performance Analysis

Test results on M1 Mac with Foundry:

| Test | Gas Used |
|------|----------|
| computeFrontRunAmount | ~53,943 |
| frontRun + backRun (full sandwich) | ~187,564 |
| frontRun (zero amount) | ~24,256 |

## Lessons Learned

1. **Numerical Stability**: Always consider overflow when multiplying large numbers
2. **Precision vs Range**: Scaling trades precision for computational feasibility
3. **Sanity Checks**: Mathematical optimality doesn't guarantee real-world profitability
4. **Testing Edge Cases**: Zero values, overflows, and boundary conditions are critical

## Future Optimizations

### Potential Improvements

1. **Assembly Optimization**: Critical math operations could use Yul for gas savings
2. **Binary Search**: Alternative to quadratic formula for certain parameter ranges
3. **Lookup Tables**: Pre-compute common scaling factors
4. **Multi-AMM Support**: Extend to Uniswap v3 concentrated liquidity

### Production Considerations

- **Mempool Monitoring**: Detect victim transactions in real-time
- **Gas Price Optimization**: Bid strategically for transaction ordering
- **Flashbots Integration**: Private transaction submission to avoid detection
- **Multi-block Strategies**: Coordinate across blocks for larger profits

## References

- Uniswap v2 Core: https://github.com/Uniswap/v2-core
- Flash Boys 2.0: Frontrunning, Transaction Reordering, and Consensus Instability in Decentralized Exchanges
- MEV Research: https://ethereum.org/en/developers/docs/mev/
- Foundry Book: https://book.getfoundry.sh/
