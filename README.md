# Statistical Arbitrage Pair Trading System

## Overview

This project implements a quantitative statistical arbitrage (pair trading) strategy that identifies and trades mean-reverting relationships between stocks. The system builds a diversified portfolio of pair strategies, applying strict statistical filters, dynamic risk control, and transaction cost modeling. It represents a complete pipeline from research to portfolio construction, similar to workflows used in quantitative hedge funds.

---

## Key Features

- Sector-aware pair selection with controlled cross-sector allowance
- Cointegration testing (Engle-Granger)
- Mean-reversion modeling using z-score
- Dynamic hedge ratio via Kalman filter
- Train/test split with no lookahead bias
- Trend filter to avoid non-stationary regimes
- Adaptive trading thresholds
- Volatility targeting
- Transaction cost modeling (commission + slippage)
- Portfolio construction with risk-adjusted weights
- Turnover monitoring

---

## Strategy Workflow

### 1. Data Collection

- Historical price data is downloaded using `yfinance`
- Prices are adjusted for dividends and splits (`auto_adjust=True`)
- Missing data is cleaned prior to processing

### 2. Pair Selection

All stock combinations are evaluated and filtered by:

- Same-sector pairs (primary), with limited cross-sector inclusion (~30%)
- Cointegration test (p-value < 0.1)
- Mean-reversion speed (half-life < 120 days)
- Correlation stability between train and test periods
- Preliminary Sharpe ratio from training backtest

Top-performing pairs are ranked and selected (Top N).

### 3. Signal Generation

For each pair:

- Estimate dynamic hedge ratio using a Kalman filter
- Compute spread between assets
- Calculate z-score: `z = (spread - rolling_mean) / rolling_std`
- Generate signals:
  - Long spread when undervalued
  - Short spread when overvalued

### 4. Strategy Execution

- Signal direction is determined using training data only
- Trading occurs exclusively on out-of-sample test data
- Entry/exit thresholds are data-driven (quantile-based)

---

## Risk Management

The system applies multiple layers of protection:

- **Signal control** — extreme returns are clipped before and after scaling
- **Volatility targeting** — position size is adjusted to maintain stable risk: `strat *= target_vol / rolling_vol`
- **Trend filter** — trading is suppressed when the spread exhibits a strong directional trend
- **Trade filtering** — strategies with too few trades or consistently negative returns are excluded

---

## Transaction Costs

Realistic execution costs are modelled explicitly:

```python
turnover = pos.diff().abs()
costs = turnover * 2 * (commission + slippage)
strat = strat - costs
```

- Turnover is measured and tracked per strategy
- High-turnover strategies are penalised
- Performance degrades as costs increase, reflecting realistic execution conditions

---

## Portfolio Construction

All individual strategies are combined into a single portfolio. Time indices are aligned across strategies, and weights are assigned based on risk-adjusted performance:

```python
weights = sharpe / volatility
```

Additional smoothing is applied to prevent concentration:

```python
weights = 0.7 * weights + 0.3 / N
```

---

## Performance Metrics

Evaluated across multiple cost scenarios:

- Total return (%)
- Sharpe ratio
- Number of active strategies
- Portfolio equity curve

### Example Results

| Scenario  | Return % | Sharpe | Pairs Used |
|-----------|----------|--------|------------|
| No cost   | 11.83    | 1.22   | 4          |
| Low cost  | 9.98     | 1.01   | 3          |

Key observations:

- Performance decreases with higher transaction costs
- Some strategies become unprofitable under realistic execution
- Portfolio size shrinks as weaker strategies are filtered out

---

## Turnover Analysis

Turnover is monitored to evaluate trading intensity:

**Average turnover: ~0.10 – 0.18**

| Turnover   | Interpretation     |
|------------|--------------------|
| < 0.01     | Very low activity  |
| 0.01–0.05  | Light trading      |
| 0.1+       | Active strategy    |

---

## Limitations

- Static pair selection with no rolling rebalancing
- No market impact modelling
- Real-world slippage may be underestimated
- High Sharpe ratios may reflect optimistic backtest bias
- Strategy performance is sensitive to the chosen stock universe

---

## Future Improvements

- Rolling pair re-selection for dynamic portfolio management
- Regime detection (mean-reversion vs. trend environments)
- Cost-aware signal generation
- Execution delay and latency modelling
- Cross-validation and Monte Carlo testing

---

## Strategy Intuition

The system exploits temporary divergences between correlated assets. When two related stocks diverge, the model trades on their expected convergence.

Example:

- Stock A rises faster than Stock B
- Short A, Long B
- Profit when the spread normalises

---

## Summary

This project represents a complete quantitative trading pipeline covering:

- Alpha generation
- Feature engineering
- Signal construction
- Risk management
- Portfolio optimisation

The implementation reflects a realistic approach to systematic strategy development, from raw data through to backtested portfolio performance.
