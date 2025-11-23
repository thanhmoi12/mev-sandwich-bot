# MEV Sandwich Bot Simulator

A Solidity implementation of a Maximal Extractable Value (MEV) sandwich attack bot for CS190 Blockchain course at UCSB. This project demonstrates understanding of automated market makers, transaction ordering, and MEV strategies in decentralized finance.

## Project Overview

This project implements a minimal MEV sandwich bot that interacts with a constant-product AMM (Automated Market Maker) on a local Foundry chain. The bot executes profitable three-transaction sequences by front-running and back-running victim trades.

### Attack Flow

1. **Front-run**: Bot swaps X tokens for Y tokens before victim's transaction
2. **Victim trade**: Victim executes their intended X→Y swap at a worse price
3. **Back-run**: Bot swaps Y tokens back to X tokens, capturing profit from price impact

## Technical Implementation

### Core Components

#### 1. Optimal Front-Run Calculation (`computeFrontRunAmount`)
- Solves quadratic optimization problem to find maximum profitable front-run amount
- Respects victim's slippage tolerance constraints
- Handles integer overflow through strategic scaling (divide by 1e9)
- Returns zero for unprofitable scenarios

**Key Challenge**: Computing discriminant for large token amounts (1000+ ether) without overflow
- **Solution**: Scaled arithmetic by dividing intermediate calculations by 1e9 to stay within uint256 bounds

#### 2. Front-Run Execution (`frontRun`)
- Approves AMM to spend bot's X token balance
- Executes swap via `swapXForY`
- Gracefully handles zero-amount edge cases

#### 3. Back-Run Execution (`backRun`)
- Swaps entire Y token balance back to X
- Transfers profit directly to owner
- Prevents reverts on zero balance

### Architecture Decisions

**Constant Product Formula**: Uses Uniswap v2 style x * y = k invariant
- Accounts for 0.3% (30 basis points) trading fee on input side
- Fee-adjusted calculations: `amountInAfterFee = amountIn * 9970 / 10000`

**Profit Optimization**: Quadratic constraint solving
```
Constraint: k * dv / (x1 * (x1 + dv)) ≥ victimMinOut
Quadratic: a*x1² + b*x1 + c ≤ 0
where:
  a = victimMinOut
  b = victimMinOut * dv
  c = -k * dv
```

## Project Structure

```
mev-sandwich-bot/
├── contracts/
│   ├── SandwichBot.sol      # Main implementation
│   ├── SimpleAMM.sol         # Constant-product AMM
│   ├── TestTokens.sol        # ERC20 test tokens
│   └── IERC20.sol           # Minimal ERC20 interface
├── test/
│   └── SandwichBot.t.sol    # Foundry test suite
├── foundry.toml              # Foundry configuration
└── README.md
```

## Setup & Installation

### Prerequisites
- [Foundry](https://book.getfoundry.sh/getting-started/installation)
- Solidity ^0.8.20

### Installation

```bash
# Clone repository
git clone https://github.com/KushagraKanaujia/mev-sandwich-bot.git
cd mev-sandwich-bot

# Install dependencies
forge install foundry-rs/forge-std

# Compile contracts
forge build
```

## Testing

Run the complete test suite:

```bash
forge test -vv
```

### Test Coverage

All 7 tests pass:
- ✅ `testComputeFrontRunDeterministicAmount` - Verifies optimal front-run calculation
- ✅ `testComputeFrontRunZeroWhenSlippageTooTight` - Validates slippage constraints
- ✅ `testFrontAndBackRunProfitableAndPaysOwner` - End-to-end profitability check
- ✅ `testFrontRunZeroNoRevert` - Edge case handling
- ✅ `testBackRunZeroNoRevert` - Zero balance handling
- ✅ `testFrontRunRevertsWhenInsufficient` - Insufficient balance protection
- ✅ `testVictimSlippageBinding` - Slippage tolerance verification

## Technical Learnings

### Challenges & Solutions

1. **Integer Overflow in Quadratic Formula**
   - Problem: Computing `4 * a * k * dv` overflowed uint256 for realistic token amounts
   - Solution: Scaled intermediate calculations by 1e9 to reduce magnitude while preserving precision

2. **Victim Slippage Constraint**
   - Problem: Calculating maximum front-run while respecting victim's minimum output
   - Solution: Derived quadratic inequality from constant product formula and victim's slippage tolerance

3. **Profit Distribution**
   - Problem: Initial implementation kept profits in contract
   - Solution: Modified `backRun` to send X tokens directly to owner address

## Performance Metrics

- **Gas Optimization**: Functions use minimal storage reads
- **Mathematical Precision**: Maintains accuracy through scaled arithmetic
- **Edge Case Handling**: Zero-amount and insufficient balance scenarios covered

## Future Enhancements

Potential improvements for production deployment:

- [ ] Gas optimization through assembly for critical paths
- [ ] Multi-block MEV strategies (e.g., cross-AMM arbitrage)
- [ ] Flashbots integration for private transaction submission
- [ ] Dynamic slippage detection from mempool analysis
- [ ] Support for multi-hop sandwich attacks

## Academic Context

**Course**: CS190 - Blockchain Technology
**Institution**: UC Santa Barbara
**Assignment**: Homework 4 - MEV Sandwich Attack Simulator
**Concepts Covered**:
- Automated Market Maker (AMM) mechanics
- Constant product formula (x * y = k)
- Transaction ordering and MEV
- Smart contract optimization
- Solidity testing with Foundry

## Disclaimer

This project is for educational purposes only. MEV sandwich attacks can be considered harmful to DeFi ecosystem participants and may violate terms of service on public networks. This implementation should only be used on local test networks.

## License

MIT License - Academic Project

---

**Contact**: For questions about implementation details or MEV strategies, open an issue or reach out via GitHub.
