# Equity Swap

<span class="category-badge badge-equity">EQUITY</span>

Exchange of cash flows where at least one leg is based on equity returns.

## Overview

**Class:** `EquitySwap`
**Category:** Equity
**Type:** Total Return Swap

An equity swap is a bilateral financial derivative contract where two parties agree to exchange sets of future cash flows. Typically, one leg is based on the total return (price appreciation plus dividends) of an equity or equity index, while the other leg is a fixed or floating rate.

## 💡 Key Features

- **Total Return Exchange**: Combines price appreciation and dividend payments
- **Leverage**: Gain equity exposure without owning the underlying
- **Customizable**: Flexible notional amounts and payment schedules
- **Hedging**: Effective tool for managing equity risk

## 📐 Python Class Implementation

```python
from dataclasses import dataclass
from datetime import datetime
from enum import Enum
from typing import List, Optional
import QuantLib as ql

class SwapLegType(Enum):
    """Type of swap leg"""
    EQUITY_RETURN = "equity_return"
    FIXED_RATE = "fixed_rate"
    FLOATING_RATE = "floating_rate"

@dataclass
class EquityLeg:
    """Equity leg of the swap"""
    underlying_ticker: str
    initial_price: float
    notional: float
    dividend_yield: float = 0.0

    def calculate_return(self, final_price: float,
                        period_days: int) -> float:
        """
        Calculate total return including price appreciation and dividends

        Args:
            final_price: Final price of underlying
            period_days: Number of days in period

        Returns:
            Total return amount
        """
        # Price return
        price_return = (final_price - self.initial_price) / self.initial_price

        # Dividend return (assuming continuous compounding)
        dividend_return = self.dividend_yield * (period_days / 365.0)

        # Total return
        total_return = price_return + dividend_return

        return self.notional * total_return

@dataclass
class FixedLeg:
    """Fixed rate leg of the swap"""
    notional: float
    fixed_rate: float
    day_count: str = "ACT/360"

    def calculate_payment(self, period_days: int) -> float:
        """
        Calculate fixed payment

        Args:
            period_days: Number of days in period

        Returns:
            Fixed payment amount
        """
        if self.day_count == "ACT/360":
            year_fraction = period_days / 360.0
        elif self.day_count == "ACT/365":
            year_fraction = period_days / 365.0
        else:  # 30/360
            year_fraction = period_days / 360.0

        return self.notional * self.fixed_rate * year_fraction

class EquitySwap:
    """
    Complete implementation of an Equity Total Return Swap

    Attributes:
        equity_leg: Equity return leg
        fixed_leg: Fixed rate leg
        start_date: Swap start date
        maturity_date: Swap maturity date
        payment_frequency: Payments per year
        position: 'payer' or 'receiver' of equity returns
    """

    def __init__(
        self,
        equity_leg: EquityLeg,
        fixed_leg: FixedLeg,
        start_date: datetime,
        maturity_date: datetime,
        payment_frequency: int = 4,  # Quarterly
        position: str = "payer"
    ):
        self.equity_leg = equity_leg
        self.fixed_leg = fixed_leg
        self.start_date = start_date
        self.maturity_date = maturity_date
        self.payment_frequency = payment_frequency
        self.position = position  # 'payer' pays equity, receives fixed

        # Validate
        if position not in ['payer', 'receiver']:
            raise ValueError("Position must be 'payer' or 'receiver'")

        if maturity_date <= start_date:
            raise ValueError("Maturity must be after start date")

    def generate_payment_schedule(self) -> List[datetime]:
        """
        Generate payment dates based on frequency

        Returns:
            List of payment dates
        """
        schedule = []
        current = self.start_date
        months_per_payment = 12 // self.payment_frequency

        while current < self.maturity_date:
            # Add months
            month = current.month + months_per_payment
            year = current.year + (month - 1) // 12
            month = ((month - 1) % 12) + 1

            current = datetime(year, month, current.day)
            if current <= self.maturity_date:
                schedule.append(current)

        # Ensure maturity is included
        if schedule[-1] != self.maturity_date:
            schedule.append(self.maturity_date)

        return schedule

    def calculate_cash_flows(
        self,
        equity_prices: List[float],
        valuation_date: datetime = None
    ) -> dict:
        """
        Calculate all cash flows for the swap

        Args:
            equity_prices: List of equity prices at each payment date
            valuation_date: Date for NPV calculation

        Returns:
            Dictionary with cash flow details
        """
        schedule = self.generate_payment_schedule()

        if len(equity_prices) != len(schedule):
            raise ValueError(
                f"Need {len(schedule)} prices for {len(schedule)} payments"
            )

        cash_flows = []
        prev_price = self.equity_leg.initial_price

        for i, (date, price) in enumerate(zip(schedule, equity_prices)):
            # Calculate period days
            if i == 0:
                period_days = (date - self.start_date).days
            else:
                period_days = (date - schedule[i-1]).days

            # Equity leg payment
            equity_payment = self.equity_leg.calculate_return(
                price, period_days
            )

            # Fixed leg payment
            fixed_payment = self.fixed_leg.calculate_payment(period_days)

            # Net cash flow (from payer perspective)
            if self.position == "payer":
                net_cf = fixed_payment - equity_payment
            else:
                net_cf = equity_payment - fixed_payment

            cash_flows.append({
                'date': date,
                'equity_price': price,
                'equity_payment': equity_payment,
                'fixed_payment': fixed_payment,
                'net_cash_flow': net_cf,
                'period_days': period_days
            })

            prev_price = price

        return {
            'cash_flows': cash_flows,
            'total_equity_payments': sum(cf['equity_payment']
                                        for cf in cash_flows),
            'total_fixed_payments': sum(cf['fixed_payment']
                                       for cf in cash_flows),
            'total_net_cash_flow': sum(cf['net_cash_flow']
                                      for cf in cash_flows)
        }

    def calculate_npv(
        self,
        equity_prices: List[float],
        discount_rate: float,
        valuation_date: datetime = None
    ) -> float:
        """
        Calculate Net Present Value of the swap

        Args:
            equity_prices: Projected equity prices
            discount_rate: Discount rate for NPV
            valuation_date: Valuation date (default: start_date)

        Returns:
            NPV of the swap
        """
        if valuation_date is None:
            valuation_date = self.start_date

        result = self.calculate_cash_flows(equity_prices, valuation_date)
        npv = 0.0

        for cf in result['cash_flows']:
            days_to_payment = (cf['date'] - valuation_date).days
            discount_factor = 1 / (1 + discount_rate) ** (days_to_payment / 365.0)
            npv += cf['net_cash_flow'] * discount_factor

        return npv

    def get_summary(self) -> dict:
        """Get swap summary information"""
        return {
            'instrument_type': 'Equity Swap',
            'underlying': self.equity_leg.underlying_ticker,
            'notional': self.equity_leg.notional,
            'fixed_rate': f"{self.fixed_leg.fixed_rate * 100:.2f}%",
            'start_date': self.start_date.strftime('%Y-%m-%d'),
            'maturity': self.maturity_date.strftime('%Y-%m-%d'),
            'frequency': f"{self.payment_frequency}x per year",
            'position': self.position,
            'tenor_days': (self.maturity_date - self.start_date).days
        }
```

## 📊 Usage Examples

### Example 1: Basic Equity Swap

```python
from datetime import datetime, timedelta

# Set up equity leg
equity_leg = EquityLeg(
    underlying_ticker="SPY",
    initial_price=450.00,
    notional=10_000_000,  # $10M
    dividend_yield=0.015  # 1.5% annual dividend yield
)

# Set up fixed leg
fixed_leg = FixedLeg(
    notional=10_000_000,
    fixed_rate=0.05,  # 5% annual
    day_count="ACT/360"
)

# Create swap
swap = EquitySwap(
    equity_leg=equity_leg,
    fixed_leg=fixed_leg,
    start_date=datetime(2025, 1, 15),
    maturity_date=datetime(2026, 1, 15),
    payment_frequency=4,  # Quarterly payments
    position="payer"  # Pay equity, receive fixed
)

# Get summary
print("Swap Summary:")
for key, value in swap.get_summary().items():
    print(f"  {key}: {value}")
```

**Output:**
```
Swap Summary:
  instrument_type: Equity Swap
  underlying: SPY
  notional: 10000000
  fixed_rate: 5.00%
  start_date: 2025-01-15
  maturity: 2026-01-15
  frequency: 4x per year
  position: payer
  tenor_days: 365
```

### Example 2: Calculate Cash Flows

```python
# Projected SPY prices at each quarterly payment
projected_prices = [
    460.00,  # Q1: +2.22%
    465.00,  # Q2: +1.09%
    470.00,  # Q3: +1.08%
    480.00   # Q4: +2.13%
]

# Calculate all cash flows
result = swap.calculate_cash_flows(projected_prices)

print("\nCash Flow Schedule:")
print("-" * 80)
for cf in result['cash_flows']:
    print(f"Date: {cf['date'].strftime('%Y-%m-%d')}")
    print(f"  Equity Price: ${cf['equity_price']:.2f}")
    print(f"  Equity Payment: ${cf['equity_payment']:,.2f}")
    print(f"  Fixed Payment: ${cf['fixed_payment']:,.2f}")
    print(f"  Net Cash Flow: ${cf['net_cash_flow']:,.2f}")
    print()

print(f"Total Equity Payments: ${result['total_equity_payments']:,.2f}")
print(f"Total Fixed Payments: ${result['total_fixed_payments']:,.2f}")
print(f"Total Net Cash Flow: ${result['total_net_cash_flow']:,.2f}")
```

### Example 3: NPV Calculation

```python
# Calculate NPV with 4% discount rate
npv = swap.calculate_npv(
    equity_prices=projected_prices,
    discount_rate=0.04,
    valuation_date=datetime(2025, 1, 15)
)

print(f"\nSwap NPV: ${npv:,.2f}")

# Sensitivity analysis
discount_rates = [0.03, 0.04, 0.05, 0.06]
print("\nNPV Sensitivity to Discount Rate:")
for rate in discount_rates:
    npv = swap.calculate_npv(projected_prices, rate)
    print(f"  {rate*100:.1f}%: ${npv:,.2f}")
```

## 🎯 Use Cases

### 1. Synthetic Equity Position

```python
# Investor wants S&P 500 exposure without buying shares
# Uses equity swap to gain exposure while paying LIBOR + spread

synthetic_exposure = EquitySwap(
    equity_leg=EquityLeg("SPX", 4500.0, 50_000_000, 0.0175),
    fixed_leg=FixedLeg(50_000_000, 0.0525),  # LIBOR + 25bps
    start_date=datetime.now(),
    maturity_date=datetime.now() + timedelta(days=365),
    payment_frequency=4,
    position="receiver"  # Receive equity returns
)

print("Synthetic long position created")
print(f"Exposure: ${synthetic_exposure.equity_leg.notional:,.0f}")
```

### 2. Portfolio Hedging

```python
# Fund manager hedges $100M equity portfolio
hedge_swap = EquitySwap(
    equity_leg=EquityLeg("Portfolio", 100.0, 100_000_000, 0.02),
    fixed_leg=FixedLeg(100_000_000, 0.045),
    start_date=datetime(2025, 1, 1),
    maturity_date=datetime(2025, 12, 31),
    payment_frequency=2,  # Semi-annual
    position="payer"  # Pay equity returns (short position)
)

print("Portfolio hedge established")
```

## 📈 Pricing Methods

### Monte Carlo Simulation

For path-dependent features or complex payoffs:

```python
import numpy as np

def monte_carlo_equity_swap_pricing(
    swap: EquitySwap,
    spot_price: float,
    volatility: float,
    num_simulations: int = 10000
) -> dict:
    """
    Price equity swap using Monte Carlo

    Returns:
        Dict with NPV statistics
    """
    np.random.seed(42)
    schedule = swap.generate_payment_schedule()
    dt = 1/swap.payment_frequency

    npvs = []

    for _ in range(num_simulations):
        prices = [spot_price]

        for _ in range(len(schedule)):
            # Geometric Brownian Motion
            dW = np.random.normal(0, np.sqrt(dt))
            next_price = prices[-1] * np.exp(
                (0.05 - 0.5 * volatility**2) * dt +
                volatility * dW
            )
            prices.append(next_price)

        prices = prices[1:]  # Remove initial price
        npv = swap.calculate_npv(prices, 0.04)
        npvs.append(npv)

    return {
        'mean_npv': np.mean(npvs),
        'std_npv': np.std(npvs),
        'confidence_95': (
            np.percentile(npvs, 2.5),
            np.percentile(npvs, 97.5)
        )
    }

# Example usage
mc_result = monte_carlo_equity_swap_pricing(swap, 450.0, 0.20)
print(f"\nMonte Carlo NPV: ${mc_result['mean_npv']:,.2f}")
print(f"Std Dev: ${mc_result['std_npv']:,.2f}")
print(f"95% CI: ${mc_result['confidence_95'][0]:,.2f} to "
      f"${mc_result['confidence_95'][1]:,.2f}")
```

## 📊 Risk Metrics

### Greeks Calculation

```python
def calculate_swap_delta(
    swap: EquitySwap,
    current_price: float,
    projected_prices: List[float],
    bump_size: float = 0.01  # 1% bump
) -> float:
    """
    Calculate swap delta (sensitivity to spot price)
    """
    # Base NPV
    base_npv = swap.calculate_npv(projected_prices, 0.04)

    # Bump prices
    bumped_prices = [p * (1 + bump_size) for p in projected_prices]
    bumped_npv = swap.calculate_npv(bumped_prices, 0.04)

    # Delta
    delta = (bumped_npv - base_npv) / (current_price * bump_size)

    return delta

delta = calculate_swap_delta(swap, 450.0, projected_prices)
print(f"\nSwap Delta: {delta:,.2f}")
print(f"For 1% move in underlying: ${delta * 4.50:,.2f} P&L impact")
```

## 🔗 Related Instruments

- [Equity Index Swap](equity_index_swap.md) - Swap on equity index
- [Interest Rate Swap](../rates/irs.md) - Rate swaps comparison
- [Total Return Swap](equity_swap.md) - General TRS documentation

## 📚 References

- Hull, J. C. (2018). *Options, Futures, and Other Derivatives*
- ISDA Documentation - Equity Derivatives
- [QuantLib Documentation](https://www.quantlib.org/)

## ⚠️ Important Notes

!!! warning "Mark-to-Market Risk"
    Equity swaps require daily marking-to-market and may trigger margin calls

!!! info "Tax Considerations"
    Tax treatment varies by jurisdiction - consult tax advisors

!!! tip "Counterparty Risk"
    Consider credit risk of the counterparty when entering swaps

---

**Last Updated:** 2025-01-17 | **Version:** 1.0 | **Status:** Production Ready
