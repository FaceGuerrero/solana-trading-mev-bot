# Solana Mev Bot

The solana mev bot is designed to perform backrun arbs on the solana blockchain, specifically targeting SOL and USDC trades. It utilizes the Jito mempool and bundles to backrun trades, focusing on circular arbitrage strategies. The bot supports multiple platforms including Raydium, Raydium CLMM, Orca Whirlpools, and Orca AMM pools.

## Overview

Backrunning in the context of decentralized finance (DeFi) is a strategy that takes advantage of the public nature of blockchain transactions. When a large trade is made on a decentralized exchange (DEX), it can cause a temporary imbalance in the price of the traded assets. A backrun is a type of arbitrage where a trader, or in this case a bot, sees this incoming trade and quickly places their own right after it, aiming to profit from the price imbalance.

The Jito Backrun Arb Bot implements this strategy in three main steps:

1. **Identifying trades to backrun**: The bot monitors the mempool for large incoming trades that could cause a significant price imbalance.

2. **Finding a profitable backrun arbitrage route**: The bot calculates potential profits from various arbitrage routes that could correct the price imbalance.

3. **Executing the arbitrage transaction**: The bot places its own trade immediately after the large trade is executed, then completes the arbitrage route to return the market closer to its original balance.


## Detailed Explanation

### Identifying Trades to Backrun

The first step in the backrun strategy is to identify trades that can be backrun. This involves monitoring the mempool, which is a stream of pending transactions. For example, if a trade involving the sale of 250M BONK for 100 USDC on the Raydium exchange is detected, this trade can potentially be backrun.

To determine the direction and size of the trade, the bot simulates the transaction and observes the changes in the account balances. If the USDC vault for the BONK-USDC pair on Raydium decreases by $100, it indicates that someone sold BONK for 100 USDC. This means that the backrun will be at most 100 USDC to bring the markets back in balance.

During this process, the bot monitors the memory pool to identify all transactions involving relevant decentralized exchanges (DEXs). Many transactions utilize lookup tables, which we must first parse to determine whether a transaction involves any associated treasuries.

### Finding Profitable Backrun Arbitrage

The next step is to find a profitable backrun arbitrage opportunity. This involves considering all possible 2 and 3 hop routes. A hop is a pair, and in this context, it refers to a trade from one asset to another.

For example, if the original trade was a sale of BONK for USD on Raydium, the possible routes for backrun arbitrage could be:

- Buy BONK for USD on Raydium -> Sell BONK for USDC on another exchange (2 hop)
- Buy BONK for USD on Raydium -> Sell BONK for SOL on Raydium -> Sell SOL for USDC on another exchange (3 hop)

The bot calculates the potential profit for each route in increments of the original trade size divided by a predefined number of steps. The route with the highest potential profit is selected for the actual backrun.

For accurate calculations, the bot needs recent pool data. On startup, the bot subscribes to Geyser for all pool account changes. To perform the actual math, the bot uses Amm objects from the Jupiter SDK. These "calculator" objects are initialized and updated with the pool data from Geyser and can be used to calculate a quote. Each worker thread has its own set of these Amm objects, one for each pool.

### Executing the Arbitrage Transaction

The final step is to execute the arbitrage transaction. To do this without providing capital, the bot uses flashloans from Solend, a decentralized lending platform.

The basic structure of the arbitrage transaction is:

- Borrow SOL or USDC from Solend using a flashloan
- Execute the arbitrage route using the Jupiter program
- Repay the flashloan
- Tip the validator

The Jupiter program is used because it supports multi-hop swaps, which are necessary for executing the arbitrage route.

However, one challenge with executing the transaction is the transaction size. Some hops require a lot of accounts, which can make the transaction too large. To address this, the bot uses lookup tables to reduce the transaction size.

However, the Jito bundle imposes one limitation: transactions within the bundle cannot utilize lookup tables that have been modified within the same bundle. To address this issue, the bot caches all lookup tables encountered during transactions (from the memory pool) and then selects up to three that can most effectively minimize transaction size. This solution performs exceptionally well, particularly after the bot has been running for some time.

Once the transaction is executed, the bot queries the RPC for the backrun transaction after a delay of 30 seconds. The result and other data are then recorded in a CSV file.

## How to run

### Pre-requisites

- [Node.js](https://nodejs.org/en/download) has been installed
- Wallet private keys and some SOL
- Fast RPC connection. For first-time use, you can use the default link to check for errors.
- After confirming that the local environment functions properly under good network conditions, it is recommended to migrate to a dedicated server.

### Run directly

1. Download repo: git clone https://github.com/FaceGuerrero/solana-trading-mev-bot
2. Copy `.env.example` to `.env` and fill in the values.
3. `BIRDEYE_API_KEY` can use the default value or be replaced with your own; this has no impact.
4. Simply enter your wallet private key (ensure the environment is secure), and after it functions properly, replace the fast RPC link. Avoid modifying other parameters whenever possible. (Community testing has confirmed this is currently the optimal configuration)
5. Run the following commands:

```bash
npm install
npm run mev
```

### Donation

If this project has been helpful to you, please click a star for it. If possible, consider donating some SOL to me—it would be a huge help for future updates:
`7GtsKuSsM3hP3MGZoryvYqUGnsw7W4Lw3Pbui45mU65V`
