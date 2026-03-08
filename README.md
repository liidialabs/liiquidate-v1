# Liiquidate Liquidation Engine (LLE)

An automated liquidation system for the LiiBorrow lending protocol, built on Chainlink CRE (Chainlink Automation). The system consists of two integrated components: an off-chain CRE workflow for liquidation detection and an on-chain smart contract for liquidation execution.

## Overview

Liiquidate monitors borrower positions in the LiiBorrow lending protocol and automatically liquidates undercollateralized positions when they become eligible. The system uses flash loans to execute liquidations without requiring upfront capital.

### How It Works

```
    ┌─────────────────────────────┐  ┌─────────────────────────────┐
    │    LiiBorrowV1Adapter       │  │    LiiBorrow Protocol       │
    └─────────────────────────────┘  └─────────────────────────────┘
                  |                               |
  Query positions |             Listen to events  |
                  ▼                               ▼
    ┌──────────────────────────────────────────────────────────┐    ┌───────────────────────────────┐
    │                      CRE Workflow                        │    │       Consumer Contract       │
    │   ┌─────────────────────┐     ┌──────────────────────┐   │    │   ┌────────────────────────┐  │
    │   │ (Detection + Risk)  │───▶│  Signed Report       │───│────│──▶│ Flash Loan Execution   │  |
    │   │                     │     │ (encode + sign)      │   │    │   | (Aave V3 / UniswapV4)  |  |
    │   └─────────────────────┘     └──────────────────────┘   │    |   └────────────────────────┘  |  
    └──────────────────────────────────────────────────────────┘    |               │               |
                | ▲                                                 |               ▼               |    
         Write  │ │ Read                                            |   ┌────────────────────────┐  |      
                ▼ │                                                 |   │  LiiBorrow Protocol    │  |     
        ┌─────────────────────┐                                     |   │   (Liquidation)        │  |
        │  Supabase Database  │                                     |   └────────────────────────┘  |
        │ (Position Storage)  │                                     |                │              |
        └─────────────────────┘                                     |                ▼              |
                                                                    |  ┌────────────────────────┐   |         
                                                                    |  │     Token Swaping      │   |
                                                                    |  │       (UniswapV4)      │   |
                                                                    |  └────────────────────────┘   |
                                                                    └───────────────────────────────┘                           

```

1. **Detection**: The CRE workflow queries borrower  from supabase and recalculates health factors according to a given schedule and during contract interaction based on events. 
2. **Risk Assessment**: Positions are categorized as HOT/WARM/COLD based on health factor
3. **Report Generation**: When a position becomes liquidatable (HF < 1), the workflow encodes a liquidation report
4. **Signing**: The report is signed using CRE's ECDSA signing with keccak256 hashing
5. **Execution**: The signed report is submitted to the consumer contract, which executes the liquidation on-chain using flash loans

## Why Automated Liquidation?

Lending protocols like LiiBorrow face insolvency risk when borrower positions become undercollateralized (health factor < 1). When collateral value drops relative to debt, the protocol becomes exposed to losses. This comes with a major critical challenge:

 **- Operational Overhead**: Continuously monitoring all borrower positions, calculating health factors, and executing transactions requires dedicated infrastructure with high uptime guarantees.

### How CRE Solves This

Chainlink CRE (Chainlink Automation) materially simplifies this system in four ways:

| Capability | Benefit |
|------------|---------|
| **Off-chain Computation** | CRE monitors positions and calculates health factors off-chain, avoiding expensive on-chain checks for every borrower |
| **Trustless Verification** | ECDSA-signed reports let the consumer contract verify reports originated from your authorized workflow—no centralized oracle needed |
| **Reliable Automation** | Native log/cron triggers ensure liquidation detection runs consistently without managing your own infrastructure |
| **Separated Concerns** | CRE handles detection; consumer contracts handle execution—each optimized for its specific task |

This architecture enables fully automated, capital-efficient liquidations without requiring manual intervention or a centralized bot operator.

This repository uses Git submodules to manage two related projects:

| Directory | Project | Description |
|-----------|---------|-------------|
| `consumer-contracts/` | [liiquidate-consumer-v1](https://github.com/liidialabs/liiquidate-consumer-v1) | Consumer contracts for on-chain liquidation execution |
| `cre-workflow/` | [liiquidate-workflow-v1](https://github.com/liidialabs/liiquidate-workflow-v1) | CRE workflow for off-chain automation |

### Consumer Contracts (`consumer-contracts/`)

Solidity smart contracts built with Foundry that handle the on-chain portion of liquidations:

- **`Liiquidate`**: Main consumer contract, receives reports from Chainlink CRE Automation
- **`FlashLoanRouter`**: Routes flash loans across multiple providers (Aave V3, Uniswap V4)
- **`UniversalSwapRouter`**: Multi-DEX swap router with circuit breaker
- **`AdapterRegistry`**: Maps protocol names to liquidation adapter addresses

**Tech Stack**: Solidity, Foundry (Forge), Viem

### CRE Workflow (`cre-workflow/`)

A Chainlink CRE workflow written in TypeScript that monitors positions and detects liquidatable accounts:

- **Position Monitoring**: Reads and updates borrower positions from Supabase database and events
- **Risk Assessment**: Evaluates health factors and categorizes positions
- **Liquidation Detection**: Identifies undercollateralized positions (HF < 1)
- **Report Submission**: Signs and submits liquidation reports to the consumer contract

**Tech Stack**: TypeScript, Bun, Chainlink CRE, Viem, Supabase

## Relationship Between Projects

The two projects work together in a producer-consumer pattern:

| Component | Role | Responsibility |
|-----------|------|----------------|
| CRE Workflow | Producer | Off-chain automation that detects liquidatable positions and generates signed reports |
| Consumer Contract | Consumer | On-chain contract that receives reports and executes flash loan liquidations |

### Communication Flow

1. **Data Source**: The CRE workflow reads borrower positions from a Supabase database
2. **Risk Analysis**: It queries LiiBorrow to calculate health factors
3. **Report Creation**: When a position is undercollateralized (HF < 1), it creates a liquidation report containing:
   - Borrower address
   - Collateral asset address
   - Debt asset address
   - Amount to cover
   - Protocol identifier
4. **Signing**: The workflow signs the report using ECDSA/keccak256
5. **Submission**: The signed report is submitted via `writeReport()` to the CRE
6. **Execution**: The CRE forwards the report to the consumer contract's `onReport()` function
7. **On-Chain**: The consumer contract executes the liquidation using flash loans

### Contract Integration

The workflow interacts with these contracts:

| Contract | Purpose |
|----------|---------|
| `LiiBorrowV1` | Core lending protocol - emits events |
| `LiiBorrowV1Adapter` | Risk assessment - calculates health factors and builds liquidation params |
| `Liiquidate` (Consumer) | Executes liquidations via `onReport()` |

## Architecture Diagram

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                           Liiquidate System                                     │
├─────────────────────────────────────────────────────────────────────────────────┤
│                                                                                 │
│  ┌─────────────────────────────┐      ┌──────────────────────────────────────┐  │
│  │     CRE Workflow            │      │   Consumer Contracts                 │  │
│  │  (cre-workflow/)            │      │  (consumer-contracts/)               │  │
│  │                             │      │                                      │  │
│  │  ┌──────────────────────┐   │      │  ┌────────────────────────────┐      │  │
│  │  │  Position Monitor    │   │      │  │  Liiquidate                │      │  │
│  │  │  - Supabase queries  │   │      │  |  (Consumer Contract)       │      │  │
│  │  │  - Health factor calc│   │      │  │  - Receives reports        │      │  │
│  │  └──────────────────────┘   │      │  │  - Routes to flash loans   │      │  │
│  │            │                │      │  └────────────┬───────────────┘      │  │
│  │            ▼                │      │               │                      │  │
│  │  ┌───────────────────────┐  │      │               ▼                      │  │
│  │  │  Risk Assessment      │  │      │  ┌────────────────────────────┐      │  │
│  │  │  - HF < 1.05: HOT     │  │      │  │  FlashLoanRouter           │      │  │
│  │  │  - HF 1.05-1.10:WARM  │  │      │  │  - AaveV3 Provider         │      │  │
│  │  │  - HF > 1.10: COLD    │  │      │  │  - UniswapV4 Provider      │      │  │
│  │  └───────────────────────┘  │      │  └────────────┬───────────────┘      │  │
│  │            │                │      │               │                      │  │
│  │            ▼                │      │               ▼                      │  │
│  │  ┌───────────────────────┐  │      │  ┌────────────────────────────┐      │  │
│  │  │  Report Generator     │  │      │  │   LiiBorrow Protocol       │      │  │
│  │  │  - Encode report      │  │      │  │  - Execute liquidation     │      │  │
│  │  │  - ECDSA sign         │──┼────▶│  │  - Return collateral       │      │  │
│  │  └───────────────────────┘  │      │  └────────────────────────────┘      │  │
│  │            │                │      │               │                      │  │
│  │            ▼                │      │               ▼                      │  │
│  │  ┌──────────────────────┐   │      │  ┌────────────────────────────┐      │  │
│  │  │  writeReport()       │   │      │  │    UniversalSwapRouter     │      │  │
│  │  │  (to CRE)            │   │      │  │  - Swap collateral         │      │  │
│  │  └──────────────────────┘   │      │  │  - Repay flash loan        │      │  │
│  └─────────────────────────────┘      │  └────────────────────────────┘      │  │
│                                       └──────────────────────────────────────┘  │
│                                                                                 │
│  ┌─────────────────────────────┐      ┌─────────────────────────────────────┐   │
│  │  Supabase                   │      │  EVM Networks                       │   │
│  │  (Position Storage)         │      │  (Sepolia / Mainnet / Tenderly)     │   │
│  └─────────────────────────────┘      └─────────────────────────────────────┘   │
│                                                                                 │
└─────────────────────────────────────────────────────────────────────────────────┘
```

## Setup

### Prerequisites

- **Git** with submodule support
- **Foundry** (for consumer-contracts)
- **Bun** (for cre-workflow)
- **Chainlink CRE CLI** (`cre`)
- **Supabase** project (for position data storage)

### Cloning

Clone this repository with submodules:

```bash
git clone --recurse-submodules https://github.com/liidialabs/liiquidate-v1.git
```

If you've already cloned without submodules:

```bash
git submodule update --init --recursive
```

### Setting Up Consumer Contracts

```bash
cd consumer-contracts

# Install dependencies
forge install

# Set up environment variables
cp .env.example .env
# Edit .env with your private keys and RPC URLs

# Build contracts
forge build

# Run tests
forge test
```

### Setting Up CRE Workflow

```bash
cd cre-workflow/liiquidate-workflow

# Install dependencies
bun install

# Set up environment variables
cp .env.example .env
# Edit .env with your configuration:
#   - CRE_ETH_PRIVATE_KEY: Wallet private key
#   - SUPABASE_SERVICE_ROLE_KEY: Supabase service role key
```

### Setting Up LiiBorrow

```bash
git clone https://github.com/liidialabs/liiborrow-v1.git

# Install dependencies
forge install

# Set up environment variables
cp .env.example .env
# Edit .env with your private keys and RPC URLs

# Build contracts
forge build

# Run tests
forge test
```

### Configuration

1. **Database Setup**: Create Supabase tables by running the SQL from `cre-workflow/liiquidate-workflow/README.md`
2. **Network Config**: Update `cre-workflow/project.yaml` with your RPC endpoints
3. **Contract Addresses**: Update `config.staging.json` with deployed contract addresses


## Running the System

This section describes the complete end-to-end simulation flow for testing the liquidation system. The simulation involves running on-chain transactions and using the CRE workflow to record positions and trigger liquidations.

### Simulations with Tenderly

The project supports Tenderly Virtual TestNet for simulations on the Mainnet:

_To simulate on Sepolia, use ARGS="--network sepolia" when deploying and configuring the contracts. Update the RPC to that of Sepolia in your .env and uncomment Sepolia's RPC_URL and NETWORK_ARGS in the Makefile._

#### LiiBorrow Contract

| Command | Description |
|---------|-------------|
| `make deploy-config ARGS="--network tenderly"` | Configure mocks for simulation |
| `make deploy ARGS="--network tenderly"` | Deploy DebtManager and Aave contracts |

```solidity
// After running the two deployment commands, record the addresses logged for
// 1. Mock WETH, USDC, MockAaveOracle & MockAaveV3Pool
// 2. Aave & DebtManager
// They'll be used in the consumer contract and workflow
```

#### Liiquidate Consumer Contract

| Command | Description |
|---------|-------------|
| `make deploy-script ARGS="--network tenderly"` | Deploy the core protocol contracts |
| `make deploy-configs ARGS="--network tenderly"` | Configure mock tokens, oracles and pools |
| `make deploy-adapters` | Deploy and register LiiBorrowAdapter |
| `make supply` | Supply collateral to LiiBorrow |
| `make forward-time` | Advance time to bypass cooldown periods |
| `make borrow` | Borrow assets against collateral |
| `make drop-price` | Drop asset price to trigger liquidation |
| `make man-liiquidate` | Manually execute liquidation |

```solidity
// Before deployment make sure these addresses are 0, update after "make deploy-script"
address aaveV3PoolAddress = address(0);
address uniswapV4PoolAddress = address(0);

// Each deployment command emits contract addresses that should be recorded in HelperConfig
sepoliaNetworkConfig = NetworkConfig({
        forwarderAddress: 0xA3D1AD4Ac559a6575a114998AffB2fB2Ec97a7D9, // this is the chainlink simulation Mainnet forwarder address, change to the chain you'll be simulating with
        aaveV3PoolAddress: aaveV3PoolAddress,
        uniswapV4PoolAddress: uniswapV4PoolAddress
});
sepoliaLiiBorrowConfig = LiiBorrowConfig({
        debtManagerAddress: address(0),
        aaveAddress: address(0),
        weth: address(0),
        usdc: address(0)
});
sepoliaLiiquidateConfig = LiiquidateConfig({
        adapterRegistryAddress: address(0),
        flashLoanRouterAddress: address(0),
        universalSwapRouterAddress: address(0),
        liiquidateAddress: address(0),
        uniswapV4FlashloanAddress: address(0),
        aaveV3FlashloanAddress: address(0),
        uniswapV4AdapterAddress: address(0),
        liiBorrowV1AdapterAddress: address(0)
});
sepoliaMockConfig = MockConfig({
        mockChainlinkOracle: address(0),
        mockAaveV3Oracle: address(0),
        mockAaveV3Pool: address(0)
});
```

### Simulations in CRE

```json
// Update the addresses in config to match the deployed addresses
{
  "evms": [
    {
      "schedule": "*/1 * * * *", 
      "WethUsdPriceOracle": "0x0000000000000000000000000000000000000000",
      "WbtcUsdPriceOracle": "0x0000000000000000000000000000000000000000",
      "proxyAddress": "0x0000000000000000000000000000000000000000",
      "AaveAddress": "0x0000000000000000000000000000000000000000",
      "liiBorrowAddress": "0x0000000000000000000000000000000000000000",
      "liiBorrowAdapter": "0x0000000000000000000000000000000000000000",
      "chainSelectorName": "ethereum-mainnet",
      "gasLimit": "6500000"
    }
  ]
}
```
```bash
# In cre-workflow/liiquidate-workflow/

# Simulate locally
make simulate

# Simulate with broadcast (During liquidating positions)
make simulate-bc
```

### Steps

#### Step 1: Supply Collateral

Deposit collateral (WETH) into the LiiBorrow protocol, exactly 1 WETH @ $2000 USD (2000e8):

```bash
# in consumer-contracts/
make supply
```

**Output**: Note the transaction hash for `depositCollateralERC20()` - you'll need this for the CRE workflow.

#### Step 2: Record Supply Position

Run the CRE workflow to record the user's supply position in Supabase:

```bash
# in cre-workflow/liiquidate-workflow/
make simulate
```

When prompted:
1. Select `1` for the first handler (onLogTriggerLiidiaV1 Handler)
2. Pass the transaction hash from Step 1
3. Select `0` for the index to capture the Supply event and record the user's position in Supabase

#### Step 3: Borrow Assets

Borrow tokens (USDC) against the collateral, exactly 1200 USDC is borrowed:

```bash
# in consumer-contracts/
make borrow
```

**Output**: Note the transaction hash for the borrow transaction, `borrowUsdc()`.

#### Step 4: Record Borrow Position

Run the CRE workflow to update the user's position in Supabase:

```bash
# in cre-workflow/liiquidate-workflow/
make simulate
```

When prompted:
1. Select `1` for the first handler (onLogTriggerLiidiaV1 Handler)
2. Pass the transaction hash from Step 3
3. Select `0` for the events index to capture Borrow event data and update user's position in Supabase

#### Step 5: Drop Asset Price

Drop the collateral price to trigger liquidation (price drops to exactly 1600 USD):

_The MockUniswapV4Pool has been configured to facilitate swaps only at this price_

```bash
# in consumer-contracts/
make drop-price
```

**Output**: Note the transaction hash for `UpdatedAnswer` - this is the Chainlink price update transaction.

#### Step 6: Update Position for Liquidation

Run the CRE workflow to update the user's position in Supabase with the new price:

```bash
# in cre-workflow/liiquidate-workflow/
make simulate
```

When prompted:
1. Select `3` to update the user's position based on the new price
2. The system will now updated the position as liquidatable

#### Step 7: Trigger Liquidation

Run the CRE workflow again to detect the liquidatable position and execute liquidation after price drop:

```bash
# in cre-workflow/liiquidate-workflow/
make simulate-bc
```

When prompted:
1. Select `2` to trigger the liquidation handler
2. Pass the transaction hash for `UpdatedAnswer` (from Step 5)
3. Select `0` to get the new price from the `AnswerUpdated` event

The workflow will compare the new price to the one in supabase, the price drop will trigger a supabase lookup for liquidatable position

The liquidation will use flash loans to:
1. Borrow the debt token
2. Repay the debt to the LiiBorrow protocol
3. Receive the collateral
4. Swap the collateral back to the debt token
5. Repay the flash loan

### Quick Reference

| Command | Description |
|---------|-------------|
| `make supply` | Deposit WETH as collateral into LiiBorrow |
| `make borrow` | Borrow USDC tokens against collateral |
| `make drop-price` | Drop collateral price to trigger liquidation |
| `make simulate` | Run CRE workflow simulation |
| `make simulate-bc` | Run CRE workflow simulation with broadcast |

### Handler Options in CRE Workflow

| Option | Handler | Action |
|--------|---------|--------|
| 1 | onLogTriggerLiidiaV1 Handler | Record and update user's position in Supabase based on events emitted after contract interactions |
| 2 | onLogTriggerOracles Handler | Look-up and execute liquidatable positions when price drops |
| 3 | onCronTriggerPoolHealth Handler | Update users positions in Supabase after a given time |

## APPENDICES

- [Tenderly Virtual TestNet Public Explorer](https://dashboard.tenderly.co/explorer/vnet/0e9aa9e0-7791-4cf0-8675-7bc0f55644d0/transactions) 
- [LiiBorrow Repo](https://github.com/liidialabs/liiborrow-v1.git)

## Important Notice

Liiquidate was developed as the Liquidation Engine for the LiiBorrow lending protocol. The Liiquidate project was built during the hackathon period, while the LiiBorrow lending protocol had been developed prior to the hackathon as the main protocol. This liquidation engine was primarily built to help identify and mitigate potential insolvency scenarios for the LiiBorrow protocol during its early stage before mass adoption when pro-liquidators will be interested in it.

Hence, it is not intended to compete for liquidation opportunities on mainnet, it only serves one protocol.