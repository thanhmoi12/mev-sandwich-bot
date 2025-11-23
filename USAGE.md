# Usage Guide

This guide demonstrates how to use the MEV Sandwich Bot for educational purposes.

## Quick Start

### 1. Clone and Setup

```bash
git clone https://github.com/KushagraKanaujia/mev-sandwich-bot.git
cd mev-sandwich-bot
forge install foundry-rs/forge-std
```

### 2. Compile Contracts

```bash
forge build
```

Expected output:
```
Compiling 24 files with Solc 0.8.20
Solc 0.8.20 finished in 5.04s
Compiler run successful
```

### 3. Run Tests

```bash
forge test -vv
```

Expected output:
```
Ran 7 tests for test/SandwichBot.t.sol:SandwichBotTest
[PASS] testBackRunZeroNoRevert() (gas: 27413)
[PASS] testComputeFrontRunDeterministicAmount() (gas: 53943)
[PASS] testComputeFrontRunZeroWhenSlippageTooTight() (gas: 29940)
[PASS] testFrontAndBackRunProfitableAndPaysOwner() (gas: 187564)
[PASS] testFrontRunRevertsWhenInsufficient() (gas: 57100)
[PASS] testFrontRunZeroNoRevert() (gas: 24256)
[PASS] testVictimSlippageBinding() (gas: 30673)

Suite result: ok. 7 passed; 0 failed; 0 skipped
```

## Detailed Usage Examples

### Example 1: Calculate Optimal Front-Run Amount

```solidity
// Given victim parameters
uint256 victimAmount = 20 ether;
uint24 victimSlippage = 1000; // 10% slippage tolerance
uint112 reserveX = 1000 ether;
uint112 reserveY = 1000 ether;

// Calculate optimal front-run
uint256 frontRunAmount = bot.computeFrontRunAmount(
    victimAmount,
    victimSlippage,
    reserveX,
    reserveY
);

// Result: 44302610152265331294 wei (~44.3 ether)
```

### Example 2: Execute Complete Sandwich Attack

```solidity
// Setup (from test file)
SandwichBot bot = new SandwichBot(tokenX, tokenY);
SimpleAMM amm = new SimpleAMM(tokenX, tokenY, 1000 ether, 1000 ether);

// Fund the bot
tokenX.transfer(address(bot), 600 ether);

// 1. Calculate front-run amount
uint256 dxFront = bot.computeFrontRunAmount(20 ether, 1000, reserveX, reserveY);

// 2. Execute front-run (as owner)
bot.frontRun(amm, dxFront);

// 3. Victim executes their trade
tokenX.approve(address(amm), 20 ether);
amm.swapXForY(20 ether, victimAddress);

// 4. Execute back-run (as owner)
bot.backRun(amm);

// Result: Owner receives ~3.59 ether profit
```

### Example 3: Handling Edge Cases

#### Tight Slippage (Attack Fails)
```solidity
uint256 amount = bot.computeFrontRunAmount(
    20 ether,
    50, // 0.5% slippage - too tight!
    1000 ether,
    1000 ether
);
// Returns: 0 (attack not profitable)
```

#### Zero Amount (No-Op)
```solidity
bot.frontRun(amm, 0);
// Executes successfully, does nothing
```

#### Insufficient Balance (Reverts)
```solidity
bot.frontRun(amm, 1000 ether);
// Reverts: ERC20 insufficient balance
```

## Testing Specific Scenarios

### Test 1: Verify Deterministic Calculation

```bash
forge test --match-test testComputeFrontRunDeterministicAmount -vvv
```

This verifies that the quadratic solver produces the expected result:
- Input: 20 ether victim trade, 10% slippage
- Expected: 44,302,610,152,265,331,294 wei
- Validates mathematical correctness

### Test 2: Profitability Check

```bash
forge test --match-test testFrontAndBackRunProfitableAndPaysOwner -vvv
```

This executes a full sandwich and verifies:
- Owner balance increases
- Profit is transferred to owner, not bot
- Entire sequence completes successfully

### Test 3: Slippage Enforcement

```bash
forge test --match-test testVictimSlippageBinding -vvv
```

This confirms:
- Tight slippage prevents attacks
- Bot returns 0 for unprofitable scenarios
- Victim protection works as designed

## Advanced Usage

### Deploying to Local Test Network

```bash
# Start Anvil (Foundry's local testnet)
anvil

# In another terminal, deploy contracts
forge create SandwichBot \
    --constructor-args <TOKEN_X_ADDR> <TOKEN_Y_ADDR> \
    --rpc-url http://127.0.0.1:8545 \
    --private-key <PRIVATE_KEY>
```

### Simulating Off-Chain

Since `computeFrontRunAmount` is a pure function, you can call it off-chain:

```javascript
// Using ethers.js
const frontRunAmount = await bot.computeFrontRunAmount(
    victimAmount,
    slippageBps,
    reserveX,
    reserveY
);

// No transaction sent, instant result
console.log(`Optimal front-run: ${ethers.utils.formatEther(frontRunAmount)} X`);
```

### Gas Profiling

```bash
forge test --gas-report
```

Output:
```
| Contract     | Function              | Gas      |
|--------------|----------------------|----------|
| SandwichBot  | computeFrontRunAmount| 53,943   |
| SandwichBot  | frontRun             | ~80,000  |
| SandwichBot  | backRun              | ~70,000  |
```

## Common Issues & Solutions

### Issue 1: "forge: command not found"

**Solution**: Install Foundry
```bash
curl -L https://foundry.paradigm.xyz | bash
foundryup
```

### Issue 2: Compilation Errors

**Solution**: Ensure Solidity version matches
```bash
# Check foundry.toml
solc_version = "0.8.20"
```

### Issue 3: Test Failures

**Solution**: Clean and rebuild
```bash
forge clean
forge build
forge test
```

### Issue 4: Gas Estimation Too High

**Explanation**: This is expected for sandwich attacks
- Front-run: ~80k gas
- Back-run: ~70k gas
- Total: ~150k gas per sandwich

In production, profit must exceed gas costs.

## Performance Benchmarks

### Local Machine (M1 Mac)

| Operation | Time | Gas |
|-----------|------|-----|
| Compile contracts | ~5s | - |
| Run all tests | ~120ms | - |
| Single test | ~2ms | - |
| computeFrontRunAmount | <1ms | 53,943 |

### Mainnet Estimates (Hypothetical)

At 50 gwei gas price:
- Transaction cost: ~0.004 ETH
- Break-even: Need >0.004 ETH profit
- In example: 3.59 ETH profit >> 0.004 ETH cost ✓

## Educational Exercises

### Exercise 1: Modify Slippage
Change victim slippage and observe impact:
```solidity
// Try different values: 50, 100, 500, 1000, 2000
uint24 slippage = 500; // 5%
```

### Exercise 2: Different Pool Sizes
Test with various reserve amounts:
```solidity
// Small pool
SimpleAMM amm = new SimpleAMM(tokenX, tokenY, 100 ether, 100 ether);

// Large pool
SimpleAMM amm = new SimpleAMM(tokenX, tokenY, 10000 ether, 10000 ether);
```

### Exercise 3: Measure Profit
Calculate exact profit in different scenarios:
```solidity
uint256 balanceBefore = tokenX.balanceOf(owner);
// ... execute sandwich ...
uint256 balanceAfter = tokenX.balanceOf(owner);
uint256 profit = balanceAfter - balanceBefore;
console.log("Profit:", profit);
```

## Next Steps

1. **Read TECHNICAL_NOTES.md**: Understand the math
2. **Read ARCHITECTURE.md**: Understand the design
3. **Modify tests**: Experiment with parameters
4. **Extend functionality**: Add multi-token support
5. **Optimize gas**: Use assembly for critical paths

## Resources

- [Foundry Documentation](https://book.getfoundry.sh/)
- [Solidity by Example](https://solidity-by-example.org/)
- [Uniswap V2 Whitepaper](https://uniswap.org/whitepaper.pdf)
- [MEV Research](https://ethereum.org/en/developers/docs/mev/)

## Disclaimer

This project is strictly for educational purposes. Running MEV bots on public networks:
- May violate terms of service
- Can harm other users
- Requires significant capital and infrastructure
- Is highly competitive

Only use on local test networks or with explicit permission.
