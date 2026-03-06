# Liiquidate

An automated liquidation system for the LiiBorrow lending protocol, built on Chainlink CRE (Chainlink Automation). The system consists of two integrated components: an off-chain CRE workflow for liquidation detection and an on-chain smart contract for liquidation execution.

## Overview

Liiquidate monitors borrower positions in the LiiBorrow lending protocol and automatically liquidates undercollateralized positions when they become eligible. The system uses flash loans to execute liquidations without requiring upfront capital.

### How It Works

```
┌─────────────────────┐     ┌──────────────────────┐     ┌────────────────────────┐
│   CRE Workflow      │───▶│  Signed Report       │────▶│ Consumer Contract      │
│ (Detection + Risk)  │     │ (encode + sign)      │     │ (On-chain Execution)   │
└─────────────────────┘     └──────────────────────┘     └────────────────────────┘
        │                                                         │
        │                                                         ▼
        │                                              ┌────────────────────────┐
        │                                              │  Flash Loan Execution  │
        │                                              │  (Aave V3 / UniswapV4) │
        │                                              └────────────────────────┘
        │                                                         │
        ▼                                                         ▼
┌─────────────────────┐                                ┌────────────────────────┐
│  Supabase Database  │                                │  LiiBorrow Protocol    │
│ (Position Storage)  │                                │  (Liquidation)         │
└─────────────────────┘                                └────────────────────────┘
                                                                  │
                                                                  ▼
                                                       ┌────────────────────────┐
                                                       │  Token Swaping         │
                                                       │  (UniswapV4)           │
                                                       └────────────────────────┘
```

1. **Detection**: The CRE workflow queries borrower positions and calculates health factors
2. **Risk Assessment**: Positions are categorized as HOT/WARM/COLD based on health factor
3. **Report Generation**: When a position becomes liquidatable (HF < 1), the workflow encodes a liquidation report
4. **Signing**: The report is signed using CRE's ECDSA signing with keccak256 hashing
5. **Execution**: The signed report is submitted to the consumer contract, which executes the liquidation on-chain using flash loans

## Project Structure

This repository uses Git submodules to manage two related projects:

| Directory | Project | Description |
|-----------|---------|-------------|
| `consumer-contracts/` | [liiquidate-consumer-v1](https://github.com/liidialabs/liiquidate-consumer-v1) | Solidity smart contracts for on-chain liquidation execution |
| `cre-workflow/` | [liiquidate-workflow-v1](https://github.com/liidialabs/liiquidate-workflow-v1) | TypeScript CRE workflow for off-chain automation |

### Consumer Contracts (`consumer-contracts/`)

Solidity smart contracts built with Foundry that handle the on-chain portion of liquidations:

- **`Liiquidate`**: Main consumer contract, receives reports from Chainlink CRE Automation
- **`FlashLoanRouter`**: Routes flash loans across multiple providers (Aave V3, Uniswap V4)
- **`UniversalSwapRouter`**: Multi-DEX swap router with circuit breaker
- **`AdapterRegistry`**: Maps protocol names to liquidation adapter addresses

**Tech Stack**: Solidity, Foundry (Forge), Viem

### CRE Workflow (`cre-workflow/`)

A Chainlink CRE workflow written in TypeScript that monitors positions and detects liquidatable accounts:

- **Position Monitoring**: Reads borrower positions from Supabase database
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
2. **Risk Analysis**: It queries on-chain contracts (LiiBorrowV1, LiquidatorAdapter) to calculate health factors
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
| `LiiBorrowV1` | Core lending protocol - queries collateral positions |
| `LiquidatorAdapter` | Risk assessment - calculates health factors and liquidation params |
| `Liiquidate` (Consumer) | Executes liquidations via `onReport()` |

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
git clone --recurse-submodules https://github.com/liidialabs/liiquidate.git
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

### Configuration

1. **Database Setup**: Create Supabase tables by running the SQL from `cre-workflow/liiquidate-workflow/README.md`
2. **Network Config**: Update `cre-workflow/project.yaml` with your RPC endpoints
3. **Contract Addresses**: Update `cre-workflow/liiquidate-workflow/config.production.json` and `config.staging.json` with deployed contract addresses

## Running the System

### Local Development with Tenderly

The project supports Tenderly Virtual TestNet for local testing:

```bash
# In consumer-contracts/
make sim-deploy-script
make sim-deploy-configs
make sim-deploy-adapters
make sim-supply
make forward-time  # If needed for cooldown periods
make sim-borrow
make sim-drop-price
make sim-man-liiquidate
```

### CRE Workflow

```bash
# In cre-workflow/liiquidate-workflow/

# Simulate locally
make simulate

# Simulate with broadcast
make simulate-bc
```

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
│  │  │  - Supabase queries  │───┼────▶│  │  (Consumer Contract)       │      │  │
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
│  │  │  Report Generator     │  │      │  │  UniversalSwapRouter       │      │  │
│  │  │  - Encode report      │  │      │  │  - Swap collateral         │      │  │
│  │  │  - ECDSA sign         │──┼────▶│  │  - Repay flash loan        │      │  │
│  │  └───────────────────────┘  │      │  └────────────────────────────┘      │  │
│  │            │                │      │               │                      │  │
│  │            ▼                │      │               ▼                      │  │
│  │  ┌──────────────────────┐   │      │  ┌────────────────────────────┐      │  │
│  │  │  writeReport()       │   │      │  │  LiiBorrow Protocol        │      │  │
│  │  │  (to CRE)            │   │      │  │  - Execute liquidation     │      │  │
│  │  └──────────────────────┘   │      │  │  - Return collateral       │      │  │
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

## License

MIT
