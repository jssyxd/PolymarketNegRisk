# Polymarket Neg-Risk Convexity Arbitrage & Market Maker Engine

[![Python 3.10+](https://img.shields.io/badge/python-3.10+-blue.svg)](https://www.python.org/downloads/)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)
[![Architecture: Jane Street Hardened](https://img.shields.io/badge/Execution-Jane%20Street%20Hardened-brightgreen.svg)]()

Production-grade quantitative engine for **Negative Risk (Neg-Risk) convexity arbitrage** and **two-sided market making** across multi-outcome event markets on Polymarket CLOB.

Unlike directional speculation or slow event-driven trading, this engine operates on pure **mathematical parity invariants (MECE)**, fortified with Jane Street-grade latency-skew filtering, bottleneck-first probing, and auto-unwind state machines to eliminate legging risk.

---

## 1. Quantitative Foundations & Arbitrage Mechanics

In any Polymarket Negative Risk (Neg-Risk) multi-outcome event (such as Presidential Nominee Winner, Temperature Range Buckets, Federal Reserve Rate Cut Tiers, or Crypto Milestone Ranges), all $N$ outcomes are **Mutually Exclusive and Collectively Exhaustive (MECE)**:
$$\sum_{i=1}^N \mathbf{1}_{\{\text{Outcome}_i = \text{TRUE}\}} \equiv 1$$

Exactly ONE token resolves to **$1.00 USDC**, and all other $N - 1$ tokens resolve to **$0.00 USDC**.

```text
                     ┌─────────────────────────────────────────┐
                     │     Polymarket Neg-Risk Multi-Outcome   │
                     │          (e.g., 5 Outcome Buckets)      │
                     └────────────────────┬────────────────────┘
                                          │
            ┌─────────────────────────────┴─────────────────────────────┐
            ▼                                                           ▼
┌───────────────────────┐                                   ┌───────────────────────┐
│ Long Basket Convexity │                                   │ Two-Sided Market Maker│
│   Sum(Ask_i) < 0.95   │                                   │ Sum(Bid)<1.0<Sum(Ask) │
└───────────┬───────────┘                                   └───────────┬───────────┘
            │                                                           │
            ▼                                                           ▼
Buy 1 share of ALL Yes tokens                               Post 2-sided quotes around
Cost: < 0.95 USDC                                           normalized fair probabilities
Payout at Resolution: 1.00 USDC                             Earn 3¢-5¢ spread +
Net Guaranteed Profit: > 0.05 USDC                          Polymarket Maker Rewards
```

### 1.1 Long Basket Convexity Arbitrage (Complete Set Discount)
When market cross-book inefficiencies occur such that the sum of the best asks across all $N$ outcomes is less than $1.00$ minus fees:
$$\sum_{i=1}^N \text{Ask}_i < 1.00 - \text{HurdleRate} \quad (\text{e.g.}, \sum \text{Ask} \le 0.95)$$

- **Execution**: Simultaneously buy 1 share of all $N$ Yes tokens.
- **Cost**: $\sum \text{Ask}_i \times S < 0.95 \times S$ USDC.
- **Guaranteed Payoff**: Exactly $1.00 \times S$ USDC upon market resolution.
- **Net Profit**: Independent of the real-world outcome, realizing risk-free positive return:
  $$\text{ROI} = \frac{1.00 - \sum \text{Ask}_i}{\sum \text{Ask}_i} \ge 5.26\%$$

### 1.2 Two-Sided Market Maker (Maker Track)
Instead of paying crossing fees as an aggressive Taker:
1. Normalize orderbook midpoints into implied probabilities satisfying $\sum_{i=1}^N P_i \equiv 1.00$.
2. Quote Bids at $P_i - \frac{\text{Spread}}{2}$ and Asks at $P_i + \frac{\text{Spread}}{2}$.
3. **Structural Safety Invariant**:
   $$\sum_{i=1}^N \text{Bid}_i < 1.00 \quad \text{and} \quad \sum_{i=1}^N \text{Ask}_i > 1.00$$
   - Guarantees that even if all your bids get swept, your aggregate cost is strictly below par (< $1.00).
   - Guarantees that even if all your asks get swept, your aggregate proceeds are strictly above par (> $1.00).
4. Captures the bid-ask spread on two-way organic order flow plus Polymarket CLOB liquidity rewards.

---

## 2. Jane Street-Grade Latency & Execution Hardening

In multi-leg arbitrage ($N \ge 2$), naive concurrent execution is suicidal due to **Legging Risk** (partial fills where one or more legs fail, leaving the trader with unhedged directional exposure).

This engine embeds 5 layers of production safeguards inspired by low-latency prop trading principles:

```text
       [Market Data Scanner: Gamma API + CLOB REST/WS]
                             │
                             ▼
    ┌─────────────────────────────────────────────────┐
    │  Guard 1: Latency Skew Filter (Max Skew <= 300ms)│
    │  Rejects Phantom Arb caused by stale snapshots   │
    └────────────────────────┬────────────────────────┘
                             │ Valid
                             ▼
    ┌─────────────────────────────────────────────────┐
    │  Guard 2: Economic Hurdle Filter (Net Margin >= 5%)
    │  Covers adverse selection, fees & unwind costs   │
    └────────────────────────┬────────────────────────┘
                             │ Valid
                             ▼
    ┌─────────────────────────────────────────────────┐
    │  Guard 3: Bottleneck-First Leg Sequencing       │
    │  Sorts legs by depth ascending (thinnest first)  │
    └────────────────────────┬────────────────────────┘
                             │
             ┌───────────────┴───────────────┐
             ▼                               ▼
    [Probe Leg (Index 0)]           [Probe Leg Failed]
       fires first IOC                      │
             │                              ▼
             ├─── Filled? ─── NO ───> [PROBE ABORTED]
             │                        Zero capital spent
             ▼ YES                    Zero directional risk
    [Fire Remaining N-1 Legs]
             │
             ├─── All Filled? ─── YES ───> [COMPLETE SET LOCKED]
             │
             └─── Any Leg Failed?
                        │
                        ▼
    ┌─────────────────────────────────────────────────┐
    │  Guard 4 & 5: Auto-Unwind State Machine         │
    │  1. Instantly cancel all pending in-flight legs │
    │  2. Aggressively market-sell filled legs (FAK)  │
    │  3. Cut losses within 10ms across bid spread    │
    └─────────────────────────────────────────────────┘
```

1. **Phantom Arbitrage Filter**: Calculates maximum timestamp discrepancy across all orderbooks ($\Delta t = \max(t_i) - \min(t_i)$). If $\Delta t > 300\text{ms}$ or book age $> 2.0\text{s}$, the opportunity is immediately discarded as an asynchronous illusion.
2. **True Economic Hurdle**: Rejects razor-thin margins (< 5.0%), ensuring adequate buffer against exchange fees, gas, and execution slippage.
3. **Bottleneck-First Probe Sequencing**: Detects the thinnest liquidity leg ($\min(\text{Depth}_i)$) and designates it as the Probe Leg. If the probe is rejected, execution halts instantly with zero open exposure.
4. **Auto-Unwind State Machine**: If a partial basket fill occurs, the engine does not hold directional risk. It immediately triggers an emergency liquidation of already-filled legs into the best bid.
5. **Worst-Acceptable-Price Limits**: Every leg is bound by strict price caps to prevent orderbook sweep slippage.

---

## 3. Project Architecture

```text
PolymarketNegRisk/
├── neg_risk/
│   ├── __init__.py          # Module exports
│   ├── models.py            # Event, Bucket, OrderBook, ArbOpportunity & Execution data structures
│   ├── scanner.py           # Multi-threaded Gamma event discovery & CLOB orderbook fetcher
│   ├── arb_strategy.py      # Convexity Arbitrage evaluator with Jane Street skew & hurdle guards
│   ├── maker_strategy.py    # Two-sided quoting engine with structural safety invariants
│   ├── execution.py         # Bottleneck-first probe router & auto-unwind state machine
│   ├── paper_account.py     # High-precision paper ledger (200U baseline, complete-set settlement)
│   └── engine.py            # Unified orchestrator coordinating scanner, strategy & execution
├── tests_neg_risk.py        # Comprehensive unit tests covering arb, maker, skew & auto-unwind
├── neg_risk_runner.py       # Daemon CLI entrypoint with graceful shutdown & health snapshots
├── .env.example             # Environment template for live trading credentials
├── .gitignore               # Standard Python & runtime ignore rules
└── README.md                # Quantitative and operational manual
```

---

## 4. Quick Start

### 4.1 Prerequisites
- Python 3.10+
- Linux (Ubuntu/Debian) or macOS/Windows
- `requests` (optional, relies on standard library `urllib` for zero-dependency portability)

### 4.2 Installation
```bash
git clone https://github.com/jssyxd/PolymarketNegRisk.git
cd PolymarketNegRisk
```

### 4.3 Running Unit Tests
Validate all mathematical invariants and Jane Street execution guards:
```bash
python -m unittest tests_neg_risk.py -v
```

Expected output:
```text
test_auto_unwind_on_partial_fill ... ok
test_hurdle_rate_filter ... ok
test_long_basket_arbitrage_with_bottleneck_first ... ok
test_market_maker_generates_coherent_two_sided_quotes ... ok
test_phantom_arbitrage_rejected_due_to_latency_skew ... ok
test_probe_first_aborted_zero_exposure ... ok
test_stale_book_rejected_due_to_age ... ok

Ran 7 tests in 0.003s
OK
```

---

## 5. Operations & Daemon Modes

### 5.1 Single-Shot Scan
Scan the top 25 active multi-outcome events on Polymarket and output evaluated opportunities:
```bash
python neg_risk_runner.py --events-limit 25
```

### 5.2 Paper Trading Daemon (200 USDC Baseline)
Run the paper trading daemon with a 200.0 USDC initial bankroll, committing 20.0 USDC per complete set:
```bash
python neg_risk_runner.py \
  --mode paper \
  --initial-capital 200.0 \
  --budget 20.0 \
  --interval 20 \
  --min-profit-pct 5.0 \
  --events-limit 30 \
  --max-skew-ms 300.0
```

### 5.3 Systemd Service Setup (24/7 Background Daemon)
Create a persistent system service on your Linux server:

```ini
# /etc/systemd/system/negrisk-paper.service
[Unit]
Description=Polymarket Neg-Risk Arbitrage & Maker Paper Trading Engine (200U Baseline)
After=network.target

[Service]
Type=simple
User=root
WorkingDirectory=/path/to/PolymarketNegRisk
Environment=PYTHONPATH=/path/to/PolymarketNegRisk
ExecStart=/usr/bin/python3 neg_risk_runner.py --mode paper --initial-capital 200.0 --budget 20.0 --interval 20 --events-limit 30
Restart=always
RestartSec=5
MemoryHigh=300M
MemoryMax=450M

[Install]
WantedBy=multi-user.target
```

Enable and start the service:
```bash
sudo systemctl daemon-reload
sudo systemctl enable --now negrisk-paper.service
sudo systemctl status negrisk-paper.service
```

Inspect health snapshot:
```bash
cat data/negrisk_health.json
```

---

## 6. Live Trading Transition

To transition from Paper to Live Trading:
1. Configure `.env` with your Polymarket L1 private key and funder address.
2. Ensure Level 2 CLOB API credentials are set (or auto-derived).
3. Switch runner mode:
   ```bash
   python neg_risk_runner.py --mode live --budget 10.0 --min-profit-pct 5.0
   ```

---

## 7. Risk Disclaimer & Operational Guardrails

- **Capital Allocation**: Never risk more capital than allocated for cross-basket inventory.
- **Oracle Resolution Delay**: Polymarket Neg-Risk markets settle via UMA Optimistic Oracle. Settlement payout occurs after market finalization.
- **Gas & Network Latency**: Real-world execution latency depends on geographical distance to Polymarket CLOB endpoints (AWS `us-east-1`). Use dedicated hosting in Virginia/Ohio for optimal fill rates.

---

## 8. License

MIT License. See `LICENSE` for details.
