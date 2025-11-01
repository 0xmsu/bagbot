# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Bagbot is an automated trading bot for the Bittensor network that executes buy/sell strategies for subnet alpha tokens. The bot monitors subnet prices against configurable thresholds and executes stake/unstake operations through a validator hotkey.

## Development Commands

### Setup
```bash
# Create and activate virtual environment (Python 3.9-3.12)
pip3 install virtualenv
virtualenv ~/.bagbotvirtualenv/
source ~/.bagbotvirtualenv/bin/activate

# Install dependencies
pip3 install -r requirements.txt

# Create wallet
btcli w create --wallet.name bagbot
```

### Running the Bot
```bash
# Normal operation (with confirmation prompt)
python3 bagbot.py

# Skip settings confirmation
python3 bagbot.py --nocheck
```

### Testing
```bash
# Run unit tests
python3 -m unittest test_bagbot.py
```

## Architecture

### Core Components

**BittensorUtility (bagbot.py)** - Main bot class that orchestrates all trading operations:
- `setupWallet()` - Initializes Bittensor wallet with credentials from settings
- `setupSubtensor()` - Establishes connection to Finney network with retry logic
- `refresh_stats()` - Fetches current subnet prices, stake info, and wallet balance
- `do_available_trades()` - Main execution loop that constructs and executes buy/sell trades
- `constructBuy()` / `constructSell()` - Decision logic for trade construction based on thresholds

### Trading Strategy Implementation

The bot uses a **grid trading strategy** with dynamic buy/sell thresholds:

1. **Buy threshold calculation** (bagbot.py:243-254): As the bot accumulates more alpha in a subnet, the buy price decreases linearly from `buy_upper` (when holding 0 alpha) to `buy_lower` (when holding `max_alpha`). This implements dollar-cost averaging.

2. **Sell threshold calculation** (bagbot.py:256-267): Inversely, sell thresholds increase from `sell_lower` (at max holdings) to `sell_upper` (near zero holdings), taking profits as the position scales up.

3. **Slippage protection** (bagbot.py:305-312): Before each trade, `determineTokenBuyAmount()` calculates the maximum trade size that won't exceed `MAX_SLIPPAGE_PERCENT_PER_BUY`. Uses the formula: `max_amount = (token_in_pool * slippage%) / (1 - slippage%)`

4. **Trade execution** (bagbot.py:385-437): Executes stake/unstake operations through `async_subtensor` with configurable slippage tolerance. Trades run asynchronously without waiting for finalization to maximize throughput.

### Settings Configuration System

The bot uses a two-tier configuration approach (bagbot_settings.py):

- `bagbot_settings.py` contains default settings and subnet grid configurations
- Users create `bagbot_settings_overrides.py` to override specific settings
- The override file is imported at the bottom of `bagbot_settings.py` (line 38)
- **Critical**: Users should NEVER edit `bagbot_settings.py` directly, only create overrides

**Global settings** (apply to all subnets unless overridden):
- `STAKE_ON_VALIDATOR` - Hotkey of validator to stake through (bot only sees alpha staked here)
- `WALLET_PW` / `WALLET_NAME` - Wallet credentials (cannot be overridden per-subnet)
- `MAX_TAO_PER_BUY` / `MAX_TAO_PER_SELL` - Trade size limits (minimum 0.01 to avoid gas fee losses)
- `MAX_SLIPPAGE_PERCENT_PER_BUY` - Maximum acceptable slippage per trade

**Per-subnet overrides** (optional, specified within each subnet in SUBNET_SETTINGS):
- `stake_on_validator` - Use a different validator for this specific subnet
- `max_tao_per_buy` - Custom trade size for buys on this subnet
- `max_tao_per_sell` - Custom trade size for sells on this subnet
- `max_slippage_percent_per_buy` - Custom slippage tolerance for this subnet

The `get_subnet_setting()` method (bagbot.py:70-84) implements the override logic, checking for subnet-specific values before falling back to global defaults.

### Display Layer (printHelpers.py)

Uses Rich library to render a comprehensive trading dashboard:
- Current stake amounts and values per subnet
- Buy/sell thresholds with visual proximity bars
- Portfolio percentage allocation
- Real-time price indicators with 'b' (buying) / 's' (selling) flags

The `price_proximity_bar()` function (printHelpers.py:8-67) generates ASCII visualizations showing how close current price is to triggering trades.

### Error Handling & Resilience

The bot implements robust error handling for the volatile blockchain environment:

- **Connection retry logic** (bagbot.py:49-59): `my_async_subtensor()` retries up to 15 times with exponential backoff on websocket errors
- **Continuous operation** (bagbot.py:196-240): Main loop catches and logs exceptions, automatically retrying failed operations
- **Block synchronization** (bagbot.py:217): Uses `wait_for_block()` to synchronize trades with chain state
- **Logging** (bagbot.py:24-33): All operations logged to `staking.log` for post-mortem analysis

### Validation System (bagbot.py:101-114)

`validateGrid()` ensures settings correctness before bot starts:
- Verifies required fields (`sell_lower`, `buy_upper`, `max_alpha`)
- Checks logical consistency (buy_upper must be < sell_lower)
- Prevents invalid subnet IDs (no strings, no subnet 0)
- Raises `InvalidSettings` exception with descriptive messages

## Important Notes

- The bot only operates on alpha staked to the validator specified in `STAKE_ON_VALIDATOR`. Alpha staked elsewhere is invisible.
- Minimum trade sizes (0.01 TAO) prevent gas fees from exceeding trade value.
- The bot waits for block confirmations between trading cycles to ensure accurate state.
- When modifying grid settings, always ensure buy_upper < sell_lower to prevent immediate buy/sell cycles.
- Tests in `test_bagbot.py` cover grid validation, buy/sell construction, and slippage calculations.
