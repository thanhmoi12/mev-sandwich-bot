# System Architecture

## High-Level Overview

```
┌─────────────────────────────────────────────────────────────┐
│                      Sandwich Attack Flow                    │
└─────────────────────────────────────────────────────────────┘

1. Detection Phase
   ┌──────────────┐
   │ Victim spots │ "I want to swap 20 X tokens for Y"
   │  opportunity │  with 10% slippage tolerance
   └──────┬───────┘
          │
          ▼
2. Front-Run Phase
   ┌──────────────┐         ┌─────────────┐
   │ SandwichBot  │────────▶│  SimpleAMM  │
   │              │  Swap   │  (x₀, y₀)   │
   │ Calculates   │  44.3X  │             │
   │ optimal dxF  │  for Y  │  k = x·y    │
   └──────────────┘         └──────┬──────┘
                                   │
                            State: (x₁, y₁)
                                   │
3. Victim Trade              ┌─────▼──────┐
   ┌──────────────┐          │  SimpleAMM │
   │   Victim     │─────────▶│  (x₁, y₁)  │
   │              │  Swap    │            │
   │ Executes at  │  20X     │ Price now  │
   │ worse price  │  for Y   │  worse!    │
   └──────────────┘          └──────┬─────┘
                                    │
                             State: (x₂, y₂)
                                    │
4. Back-Run Phase             ┌─────▼──────┐
   ┌──────────────┐           │  SimpleAMM │
   │ SandwichBot  │◀──────────│  (x₂, y₂)  │
   │              │  Swap all │            │
   │ Profits to   │  Y for X  │  k = x·y   │
   │   owner!     │           │            │
   └──────────────┘           └────────────┘

         💰 Profit = X_final - X_initial > 0
```

## Component Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                      SandwichBot Contract                    │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  State Variables                                             │
│  ├─ tokenX: IERC20                                          │
│  ├─ tokenY: IERC20                                          │
│  └─ owner: address                                          │
│                                                              │
│  External Functions                                          │
│  ├─ computeFrontRunAmount(...)  [pure]                      │
│  │   └─ Returns optimal front-run amount                    │
│  │                                                           │
│  ├─ frontRun(amm, amount)  [onlyOwner]                      │
│  │   └─ Executes X→Y swap                                   │
│  │                                                           │
│  └─ backRun(amm)  [onlyOwner]                               │
│      └─ Executes Y→X swap, sends profit to owner            │
│                                                              │
│  Internal Helpers                                            │
│  ├─ _applyFee(amount) → fee-adjusted amount                 │
│  ├─ _minVictimOut(...) → victim's minimum acceptable output │
│  ├─ _simulateProfit(...) → projected profit                 │
│  └─ _sqrt(x) → integer square root                          │
│                                                              │
└─────────────────────────────────────────────────────────────┘
                            │
                            │ interacts with
                            ▼
┌─────────────────────────────────────────────────────────────┐
│                      SimpleAMM Contract                      │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  State Variables                                             │
│  ├─ tokenX: IERC20                                          │
│  ├─ tokenY: IERC20                                          │
│  ├─ reserveX: uint112                                       │
│  ├─ reserveY: uint112                                       │
│  └─ FEE_BPS: 30 (0.3%)                                      │
│                                                              │
│  Functions                                                   │
│  ├─ swapXForY(amountIn, to) → amountOut                     │
│  ├─ swapYForX(amountIn, to) → amountOut                     │
│  └─ getReserves() → (reserveX, reserveY)                    │
│                                                              │
│  Invariant: k = reserveX * reserveY (constant product)       │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

## Data Flow: computeFrontRunAmount

```
Input Parameters
├─ dxVictim: uint256        (victim's intended trade size)
├─ victimSlippageBps: uint24 (victim's slippage tolerance)
├─ reserveX: uint112         (current X reserve)
└─ reserveY: uint112         (current Y reserve)

       │
       ▼
┌──────────────────────────────────────┐
│  Step 1: Validation                  │
│  • Check for zero reserves           │
│  • Check for zero victim amount      │
│  • Check slippage >= 100%            │
└──────────┬───────────────────────────┘
           │
           ▼
┌──────────────────────────────────────┐
│  Step 2: Calculate Victim's MinOut   │
│  • Apply 0.3% fee to victim amount   │
│  • Calculate ideal output            │
│  • Apply slippage tolerance          │
└──────────┬───────────────────────────┘
           │
           ▼
┌──────────────────────────────────────┐
│  Step 3: Solve Quadratic Inequality  │
│  • Scale values by 1e9               │
│  • Compute discriminant              │
│  • Calculate sqrt                    │
│  • Solve for x₁_max                  │
│  • Scale back to original magnitude  │
└──────────┬───────────────────────────┘
           │
           ▼
┌──────────────────────────────────────┐
│  Step 4: Convert to Gross Amount     │
│  • dfMax = x₁_max - x₀               │
│  • dxFront = dfMax * 10000 / 9970    │
└──────────┬───────────────────────────┘
           │
           ▼
┌──────────────────────────────────────┐
│  Step 5: Verify Profitability        │
│  • Simulate full sandwich sequence   │
│  • Check profit > 0                  │
│  • Return 0 if unprofitable          │
└──────────┬───────────────────────────┘
           │
           ▼
      dxFront (optimal front-run amount)
```

## Attack Sequence Diagram

```
Time ─────────────────────────────────────────────────────────▶

Block N                    Block N+1
  │                           │
  │  Bot detects victim tx    │
  │  in mempool               │
  │         │                 │
  │         ▼                 │
  │  ┌──────────────┐         │
  │  │ Calculate    │         │
  │  │ dxFront via  │         │
  │  │ quadratic    │         │
  │  │ solver       │         │
  │  └──────┬───────┘         │
  │         │                 │
  │         ▼                 │
  │  ┌──────────────┐         │
  ├──│ TX 1: frontRun         │
  │  │ (high gas)   │         │
  │  └──────────────┘         │
  │         │                 │
  │         ▼                 │
  │  ┌──────────────┐         │
  ├──│ TX 2: victim │         │
  │  │ (mid gas)    │         │
  │  └──────────────┘         │
  │         │                 │
  │         ▼                 │
  │  ┌──────────────┐         │
  ├──│ TX 3: backRun│         │
  │  │ (high gas)   │         │
  │  └──────────────┘         │
  │                           │
  │     All txs included      │
  │     in same block ────────┤
  │                           │
                              ▼
                      Profit realized!
```

## State Transitions

```
Initial State (Before Sandwich)
┌─────────────────────────────┐
│ AMM: (1000 X, 1000 Y)       │
│ k = 1,000,000               │
│ Price: 1 X = 1 Y            │
│                             │
│ Bot Balance: 600 X, 0 Y     │
│ Victim: 200 X, 0 Y          │
└─────────────────────────────┘
              │
              │ frontRun(44.3 X)
              ▼
State After Front-Run
┌─────────────────────────────┐
│ AMM: (1044.14 X, 957.64 Y)  │
│ k = 1,000,000               │
│ Price: 1 X = 0.917 Y ↓      │
│                             │
│ Bot: 555.7 X, 42.36 Y       │
│ Victim: 200 X, 0 Y          │
└─────────────────────────────┘
              │
              │ victim swaps 20 X
              ▼
State After Victim Trade
┌─────────────────────────────┐
│ AMM: (1063.94 X, 940.22 Y)  │
│ k = 1,000,000               │
│ Price: 1 X = 0.884 Y ↓↓     │
│                             │
│ Bot: 555.7 X, 42.36 Y       │
│ Victim: 180 X, 17.42 Y      │
└─────────────────────────────┘
              │
              │ backRun(42.36 Y)
              ▼
Final State (After Sandwich)
┌─────────────────────────────┐
│ AMM: (1016.05 X, 984.12 Y)  │
│ k = 1,000,000               │
│ Price: 1 X = 0.968 Y        │
│                             │
│ Bot: 603.59 X, 0 Y          │  ← 3.59 X profit! 💰
│ Victim: 180 X, 17.42 Y      │  ← Got ~10% less Y
└─────────────────────────────┘
```

## Key Design Decisions

### 1. Scaled Arithmetic
**Decision**: Divide by 1e9 during quadratic calculations
**Rationale**: Prevents overflow while maintaining sufficient precision
**Trade-off**: Loses 9 decimal places (acceptable for 18-decimal tokens)

### 2. Profit Verification
**Decision**: Simulate entire sequence before returning
**Rationale**: Catch edge cases where math is valid but profit is zero
**Trade-off**: Extra gas cost (~10k), but prevents failed attacks

### 3. Direct Owner Payment
**Decision**: Send profits to owner in backRun, not to contract
**Rationale**: Reduces need for withdrawal function, saves gas
**Trade-off**: Owner must be trusted (acceptable for homework context)

### 4. Pure Function for Calculation
**Decision**: Make computeFrontRunAmount() pure (no state access)
**Rationale**: Enables off-chain simulation and testing
**Trade-off**: Must pass reserves as parameters

## Security Considerations

1. **Reentrancy**: Not vulnerable (no external calls during state changes)
2. **Integer Overflow**: Prevented via scaled arithmetic
3. **Access Control**: onlyOwner modifier on execution functions
4. **Front-Running**: Bot itself is a front-runner (by design)
5. **Flash Loan**: Not implemented (would require more capital efficiency)

## Performance Characteristics

| Operation | Gas Cost | Complexity |
|-----------|----------|------------|
| computeFrontRunAmount | ~54k | O(log n) for sqrt |
| frontRun | ~80k | O(1) |
| backRun | ~70k | O(1) |
| Full sandwich | ~188k | O(1) |
