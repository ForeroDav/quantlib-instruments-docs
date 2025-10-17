# Credit Default Swap (CDS)

<span class="category-badge badge-credit">CREDIT</span>

Derivative contract providing protection against credit default events.

## Overview

**Class:** `CreditDefaultSwap`
**Category:** Credit Derivatives
**Type:** Credit Protection

A Credit Default Swap is a bilateral financial derivative contract where the buyer makes periodic premium payments to the seller in exchange for protection against a credit event (default, bankruptcy, restructuring) of a reference entity. If a credit event occurs, the seller compensates the buyer for the loss.

## 💡 Key Features

- **Credit Protection**: Transfer credit risk without selling the underlying asset
- **Premium Payments**: Quarterly or semi-annual premium payments (spread)
- **Credit Events**: Default, bankruptcy, failure to pay, restructuring
- **Settlement**: Cash or physical settlement upon credit event
- **ISDA Standard**: Industry-standard documentation and pricing

## 📐 Python Class Implementation

```python
from dataclasses import dataclass
from datetime import datetime
from typing import List, Optional
from enum import Enum
import numpy as np

class CreditEvent(Enum):
    """Types of credit events"""
    BANKRUPTCY = "bankruptcy"
    FAILURE_TO_PAY = "failure_to_pay"
    RESTRUCTURING = "restructuring"
    DEFAULT = "default"

class SettlementType(Enum):
    """CDS settlement methods"""
    CASH = "cash"
    PHYSICAL = "physical"

@dataclass
class PremiumPayment:
    """Single premium payment"""
    payment_date: datetime
    premium_amount: float
    accrued_days: int
    survival_probability: float = 1.0

@dataclass
class CDSCashFlows:
    """All cash flows for a CDS"""
    premium_leg: List[PremiumPayment]
    protection_leg_pv: float
    upfront_fee: float
    total_pv: float

class CreditDefaultSwap:
    """
    Complete implementation of a Credit Default Swap

    Attributes:
        notional: Notional amount of protection
        spread: CDS spread in basis points (e.g., 100 for 100 bps = 1%)
        recovery_rate: Expected recovery rate on default (typically 0.40 for senior unsecured)
        start_date: Protection start date
        maturity_date: Protection end date
        payment_frequency: Premium payments per year (typically 4 for quarterly)
        upfront_fee: Initial upfront payment (can be negative)
        position: 'buyer' (protection buyer) or 'seller' (protection seller)
    """

    def __init__(
        self,
        notional: float,
        spread: float,  # In basis points
        recovery_rate: float,
        start_date: datetime,
        maturity_date: datetime,
        payment_frequency: int = 4,  # Quarterly
        upfront_fee: float = 0.0,
        position: str = "buyer"
    ):
        self.notional = notional
        self.spread = spread / 10000.0  # Convert bps to decimal
        self.recovery_rate = recovery_rate
        self.start_date = start_date
        self.maturity_date = maturity_date
        self.payment_frequency = payment_frequency
        self.upfront_fee = upfront_fee
        self.position = position

        # Validate
        if position not in ['buyer', 'seller']:
            raise ValueError("Position must be 'buyer' or 'seller'")
        if maturity_date <= start_date:
            raise ValueError("Maturity must be after start date")
        if recovery_rate < 0 or recovery_rate > 1:
            raise ValueError("Recovery rate must be between 0 and 1")
        if notional <= 0:
            raise ValueError("Notional must be positive")

    def generate_payment_schedule(self) -> List[datetime]:
        """
        Generate premium payment dates (typically quarterly)

        Returns:
            List of payment dates
        """
        schedule = []
        current = self.start_date
        months_per_payment = 12 // self.payment_frequency

        while current < self.maturity_date:
            month = current.month + months_per_payment
            year = current.year + (month - 1) // 12
            month = ((month - 1) % 12) + 1

            # Standard CDS dates: 20th of month
            try:
                current = datetime(year, month, 20)
            except ValueError:
                current = datetime(year, month, 28)

            if current <= self.maturity_date:
                schedule.append(current)

        # Ensure maturity is included
        if not schedule or schedule[-1] != self.maturity_date:
            schedule.append(self.maturity_date)

        return schedule

    def calculate_hazard_rate(
        self,
        survival_probability: float,
        time_years: float
    ) -> float:
        """
        Calculate constant hazard rate from survival probability

        Args:
            survival_probability: Probability of no default
            time_years: Time period in years

        Returns:
            Hazard rate (intensity)
        """
        if survival_probability <= 0 or survival_probability > 1:
            raise ValueError("Survival probability must be between 0 and 1")

        return -np.log(survival_probability) / time_years

    def calculate_survival_probability(
        self,
        hazard_rate: float,
        time_years: float
    ) -> float:
        """
        Calculate survival probability from hazard rate

        Args:
            hazard_rate: Constant intensity
            time_years: Time period in years

        Returns:
            Survival probability
        """
        return np.exp(-hazard_rate * time_years)

    def bootstrap_hazard_rate_from_spread(
        self,
        discount_rate: float = 0.05
    ) -> float:
        """
        Bootstrap hazard rate from CDS spread (simplified)

        Args:
            discount_rate: Risk-free discount rate

        Returns:
            Bootstrapped hazard rate
        """
        # Simplified approximation: spread ≈ hazard_rate * (1 - recovery_rate)
        hazard_rate = self.spread / (1 - self.recovery_rate)
        return hazard_rate

    def calculate_premium_leg_pv(
        self,
        hazard_rate: float,
        discount_rate: float,
        valuation_date: datetime = None
    ) -> tuple:
        """
        Calculate present value of premium leg

        Args:
            hazard_rate: Credit intensity
            discount_rate: Risk-free rate for discounting
            valuation_date: Valuation date

        Returns:
            Tuple of (premium_leg_pv, premium_payments list)
        """
        if valuation_date is None:
            valuation_date = self.start_date

        schedule = self.generate_payment_schedule()
        premium_payments = []
        pv = 0.0
        prev_date = self.start_date

        for payment_date in schedule:
            if payment_date > valuation_date:
                # Calculate period
                period_days = (payment_date - prev_date).days
                time_to_payment = (payment_date - valuation_date).days / 365.0

                # Premium amount
                premium = self.notional * self.spread * (period_days / 360.0)

                # Survival probability to payment date
                survival_prob = self.calculate_survival_probability(
                    hazard_rate, time_to_payment
                )

                # Discount factor
                discount_factor = np.exp(-discount_rate * time_to_payment)

                # PV of this payment
                pv += premium * survival_prob * discount_factor

                premium_payments.append(PremiumPayment(
                    payment_date=payment_date,
                    premium_amount=premium,
                    accrued_days=period_days,
                    survival_probability=survival_prob
                ))

            prev_date = payment_date

        return pv, premium_payments

    def calculate_protection_leg_pv(
        self,
        hazard_rate: float,
        discount_rate: float,
        valuation_date: datetime = None,
        num_steps: int = 365
    ) -> float:
        """
        Calculate present value of protection leg using numerical integration

        Args:
            hazard_rate: Credit intensity
            discount_rate: Risk-free rate
            valuation_date: Valuation date
            num_steps: Number of integration steps

        Returns:
            Protection leg present value
        """
        if valuation_date is None:
            valuation_date = self.start_date

        T = (self.maturity_date - valuation_date).days / 365.0
        if T <= 0:
            return 0.0

        dt = T / num_steps
        protection_pv = 0.0

        # Loss given default
        lgd = 1 - self.recovery_rate

        # Integrate over time: LGD * hazard_rate * survival(t) * discount(t) dt
        for i in range(num_steps):
            t = (i + 0.5) * dt  # Midpoint

            survival_prob = self.calculate_survival_probability(hazard_rate, t)
            discount_factor = np.exp(-discount_rate * t)

            # Default probability density = hazard_rate * survival_prob
            default_density = hazard_rate * survival_prob

            protection_pv += lgd * default_density * discount_factor * dt

        return self.notional * protection_pv

    def calculate_fair_spread(
        self,
        hazard_rate: float,
        discount_rate: float,
        valuation_date: datetime = None
    ) -> float:
        """
        Calculate fair CDS spread (market spread where PV = 0)

        Args:
            hazard_rate: Credit intensity
            discount_rate: Risk-free rate
            valuation_date: Valuation date

        Returns:
            Fair spread in basis points
        """
        if valuation_date is None:
            valuation_date = self.start_date

        # Calculate protection leg PV
        protection_pv = self.calculate_protection_leg_pv(
            hazard_rate, discount_rate, valuation_date
        )

        # Calculate risky annuity (PV01 of premium leg)
        schedule = self.generate_payment_schedule()
        risky_annuity = 0.0
        prev_date = self.start_date

        for payment_date in schedule:
            if payment_date > valuation_date:
                period_days = (payment_date - prev_date).days
                time_to_payment = (payment_date - valuation_date).days / 365.0

                survival_prob = self.calculate_survival_probability(
                    hazard_rate, time_to_payment
                )
                discount_factor = np.exp(-discount_rate * time_to_payment)

                risky_annuity += (period_days / 360.0) * survival_prob * discount_factor

            prev_date = payment_date

        # Fair spread = protection_pv / (notional * risky_annuity)
        fair_spread = protection_pv / (self.notional * risky_annuity)

        return fair_spread * 10000  # Convert to basis points

    def calculate_npv(
        self,
        market_spread: float,  # Current market spread in bps
        discount_rate: float,
        valuation_date: datetime = None
    ) -> float:
        """
        Calculate mark-to-market value of CDS

        Args:
            market_spread: Current market CDS spread (bps)
            discount_rate: Risk-free rate
            valuation_date: Valuation date

        Returns:
            NPV of CDS (positive = gain for protection buyer)
        """
        if valuation_date is None:
            valuation_date = self.start_date

        # Bootstrap hazard rate from market spread
        market_spread_decimal = market_spread / 10000.0
        market_hazard_rate = market_spread_decimal / (1 - self.recovery_rate)

        # Calculate protection leg PV
        protection_pv = self.calculate_protection_leg_pv(
            market_hazard_rate, discount_rate, valuation_date
        )

        # Calculate premium leg PV at contract spread
        premium_pv, _ = self.calculate_premium_leg_pv(
            market_hazard_rate, discount_rate, valuation_date
        )

        # NPV from protection buyer perspective
        npv = protection_pv - premium_pv - self.upfront_fee

        # Flip sign if protection seller
        if self.position == "seller":
            npv = -npv

        return npv

    def calculate_dv01(
        self,
        market_spread: float,
        discount_rate: float,
        valuation_date: datetime = None,
        bump_size: float = 1.0  # 1 basis point
    ) -> float:
        """
        Calculate DV01 (dollar value of 1 basis point)

        Args:
            market_spread: Current market spread (bps)
            discount_rate: Risk-free rate
            valuation_date: Valuation date
            bump_size: Spread bump in bps

        Returns:
            Change in value for 1bp spread widening
        """
        base_npv = self.calculate_npv(market_spread, discount_rate, valuation_date)
        bumped_npv = self.calculate_npv(
            market_spread + bump_size, discount_rate, valuation_date
        )

        return (bumped_npv - base_npv) / bump_size

    def calculate_credit_spread_01(
        self,
        hazard_rate: float,
        discount_rate: float,
        valuation_date: datetime = None
    ) -> float:
        """
        Calculate CS01 (sensitivity to 1bp change in credit spread)

        Returns:
            CS01 value
        """
        fair_spread = self.calculate_fair_spread(
            hazard_rate, discount_rate, valuation_date
        )

        # Bump hazard rate
        bumped_hazard = (fair_spread + 1) / 10000.0 / (1 - self.recovery_rate)

        bumped_spread = self.calculate_fair_spread(
            bumped_hazard, discount_rate, valuation_date
        )

        return abs(bumped_spread - fair_spread)

    def get_summary(self) -> dict:
        """Get CDS summary information"""
        tenor_years = (self.maturity_date - self.start_date).days / 365.0

        return {
            'instrument_type': 'Credit Default Swap',
            'notional': f"${self.notional:,.0f}",
            'spread': f"{self.spread * 10000:.2f} bps",
            'recovery_rate': f"{self.recovery_rate * 100:.1f}%",
            'start_date': self.start_date.strftime('%Y-%m-%d'),
            'maturity': self.maturity_date.strftime('%Y-%m-%d'),
            'tenor': f"{tenor_years:.2f} years",
            'payment_frequency': f"{self.payment_frequency}x per year",
            'position': self.position.capitalize(),
            'upfront_fee': f"${self.upfront_fee:,.2f}"
        }
```

## 📊 Usage Examples

### Example 1: Create a 5-Year CDS

```python
from datetime import datetime

# Create CDS on corporate reference entity
# Notional: $10M, Spread: 250 bps, Recovery: 40%
cds = CreditDefaultSwap(
    notional=10_000_000,
    spread=250,  # 250 basis points = 2.5%
    recovery_rate=0.40,
    start_date=datetime(2025, 1, 20),
    maturity_date=datetime(2030, 1, 20),
    payment_frequency=4,  # Quarterly
    upfront_fee=0.0,
    position="buyer"  # Buying protection
)

# Get summary
print("CDS Summary:")
for key, value in cds.get_summary().items():
    print(f"  {key}: {value}")
```

**Output:**
```
CDS Summary:
  instrument_type: Credit Default Swap
  notional: $10,000,000
  spread: 250.00 bps
  recovery_rate: 40.0%
  start_date: 2025-01-20
  maturity: 2030-01-20
  tenor: 5.00 years
  payment_frequency: 4x per year
  position: Buyer
  upfront_fee: $0.00
```

### Example 2: Calculate Fair Spread

```python
# Bootstrap hazard rate and calculate fair spread
hazard_rate = cds.bootstrap_hazard_rate_from_spread(discount_rate=0.05)
print(f"\nBootstrapped Hazard Rate: {hazard_rate*100:.4f}% per year")

# Calculate fair spread
fair_spread = cds.calculate_fair_spread(
    hazard_rate=hazard_rate,
    discount_rate=0.05,
    valuation_date=datetime(2025, 1, 20)
)

print(f"Fair CDS Spread: {fair_spread:.2f} bps")
```

### Example 3: Calculate CDS Valuation (Mark-to-Market)

```python
# Entered CDS at 250 bps, now market spread is 300 bps
# (credit quality has deteriorated)
valuation_date = datetime(2025, 6, 20)

npv = cds.calculate_npv(
    market_spread=300,  # Current market spread
    discount_rate=0.05,
    valuation_date=valuation_date
)

print(f"\nCDS Mark-to-Market:")
print(f"  Contract Spread: 250 bps")
print(f"  Market Spread: 300 bps")
print(f"  NPV (Protection Buyer): ${npv:,.2f}")

if npv > 0:
    print(f"  Status: In-the-money (spreads widened, protection more valuable)")
else:
    print(f"  Status: Out-of-the-money")
```

### Example 4: Risk Metrics (DV01)

```python
# Calculate DV01 - sensitivity to 1bp spread change
dv01 = cds.calculate_dv01(
    market_spread=250,
    discount_rate=0.05,
    valuation_date=datetime(2025, 1, 20)
)

print(f"\nRisk Metrics:")
print(f"  DV01: ${dv01:,.2f} per 1bp")
print(f"  10bp Spread Widening Impact: ${dv01 * 10:,.2f}")
print(f"  50bp Spread Widening Impact: ${dv01 * 50:,.2f}")

# Calculate for different notionals
notionals = [1_000_000, 5_000_000, 10_000_000, 50_000_000]
print(f"\n{'Notional':<15} {'DV01':<15} {'10bp Impact'}")
print("-" * 50)
for notional in notionals:
    temp_cds = CreditDefaultSwap(
        notional, 250, 0.40,
        datetime(2025, 1, 20), datetime(2030, 1, 20)
    )
    dv01_temp = temp_cds.calculate_dv01(250, 0.05, datetime(2025, 1, 20))
    print(f"${notional:>13,} ${dv01_temp:>13,.2f} ${dv01_temp*10:>13,.2f}")
```

### Example 5: Premium Leg Cash Flows

```python
# Calculate premium payments
hazard_rate = 0.04  # 4% annual hazard rate
premium_pv, payments = cds.calculate_premium_leg_pv(
    hazard_rate=hazard_rate,
    discount_rate=0.05,
    valuation_date=datetime(2025, 1, 20)
)

print("\nPremium Payment Schedule:")
print("-" * 80)
print(f"{'Date':<12} {'Premium':<15} {'Survival Prob':<15} {'PV Contribution'}")
print("-" * 80)

for payment in payments[:8]:  # Show first 8 payments
    time_years = (payment.payment_date - datetime(2025, 1, 20)).days / 365.0
    discount = np.exp(-0.05 * time_years)
    pv_contrib = payment.premium_amount * payment.survival_probability * discount

    print(f"{payment.payment_date.strftime('%Y-%m-%d'):<12} "
          f"${payment.premium_amount:>13,.2f} "
          f"{payment.survival_probability:>13.6f} "
          f"${pv_contrib:>13,.2f}")

print(f"\nTotal Premium Leg PV: ${premium_pv:,.2f}")
```

## 🎯 Use Cases

### 1. Credit Hedging

```python
# Bank hedges $50M corporate loan exposure
loan_notional = 50_000_000
loan_spread = 300  # bps over risk-free

# Buy CDS protection
hedge_cds = CreditDefaultSwap(
    notional=loan_notional,
    spread=250,  # Pay 250 bps for protection
    recovery_rate=0.40,
    start_date=datetime(2025, 1, 20),
    maturity_date=datetime(2030, 1, 20),
    position="buyer"
)

print("Credit Hedge Position:")
print(f"  Loan Exposure: ${loan_notional:,.0f}")
print(f"  Loan Spread Earned: {loan_spread} bps")
print(f"  CDS Premium Paid: 250 bps")
print(f"  Net Spread: {loan_spread - 250} bps")
print(f"  Annual Net Income: ${loan_notional * (loan_spread - 250) / 10000:,.2f}")
```

### 2. Credit Arbitrage

```python
# Long/short credit arbitrage
# Long protection on company A (250 bps)
# Short protection on company B (300 bps)

cds_long = CreditDefaultSwap(10_000_000, 250, 0.40,
                             datetime(2025, 1, 20), datetime(2030, 1, 20),
                             position="buyer")

cds_short = CreditDefaultSwap(10_000_000, 300, 0.40,
                              datetime(2025, 1, 20), datetime(2030, 1, 20),
                              position="seller")

# Calculate combined P&L if both spreads widen by 50 bps
npv_long = cds_long.calculate_npv(300, 0.05, datetime(2025, 1, 20))
npv_short = cds_short.calculate_npv(350, 0.05, datetime(2025, 1, 20))

print("Credit Arbitrage Position:")
print(f"  Long CDS A: ${npv_long:,.2f}")
print(f"  Short CDS B: ${npv_short:,.2f}")
print(f"  Combined P&L: ${npv_long + npv_short:,.2f}")
```

### 3. Credit Curve Trading

```python
# CDS curve: 1Y, 3Y, 5Y, 10Y
maturities = [1, 3, 5, 10]
spreads = [100, 180, 250, 350]  # bps

print("CDS Curve:")
print(f"{'Tenor':<10} {'Spread':<12} {'Hazard Rate':<15} {'Default Prob'}")
print("-" * 60)

for tenor, spread in zip(maturities, spreads):
    maturity = datetime(2025 + tenor, 1, 20)
    cds_temp = CreditDefaultSwap(
        10_000_000, spread, 0.40,
        datetime(2025, 1, 20), maturity
    )
    hazard = cds_temp.bootstrap_hazard_rate_from_spread(0.05)
    default_prob = 1 - cds_temp.calculate_survival_probability(hazard, tenor)

    print(f"{tenor}Y{'':<8} {spread:>6} bps {hazard*100:>11.4f}% {default_prob*100:>11.2f}%")
```

## 📈 Advanced Pricing with QuantLib

```python
import QuantLib as ql

def price_cds_with_quantlib(
    notional: float,
    spread: float,  # in bps
    recovery_rate: float,
    start_date: datetime,
    maturity_date: datetime,
    hazard_rate: float,
    discount_rate: float
) -> dict:
    """
    Price CDS using QuantLib

    Returns:
        Dictionary with NPV, fair spread, DV01
    """
    # Convert dates
    ql_start = ql.Date(start_date.day, start_date.month, start_date.year)
    ql_maturity = ql.Date(maturity_date.day, maturity_date.month, maturity_date.year)

    # Set evaluation date
    ql.Settings.instance().evaluationDate = ql_start

    # Create schedule
    schedule = ql.Schedule(
        ql_start,
        ql_maturity,
        ql.Period(ql.Quarterly),
        ql.NullCalendar(),
        ql.Following,
        ql.Unadjusted,
        ql.DateGeneration.Forward,
        False
    )

    # Create CDS
    cds = ql.CreditDefaultSwap(
        ql.Protection.Buyer,
        notional,
        spread / 10000.0,  # Convert bps to decimal
        schedule,
        ql.Following,
        ql.Actual360()
    )

    # Create default probability term structure
    hazard_curve = ql.FlatHazardRate(
        ql_start,
        hazard_rate,
        ql.Actual365Fixed()
    )

    # Create discount curve
    discount_curve = ql.FlatForward(
        ql_start,
        discount_rate,
        ql.Actual365Fixed()
    )

    # Set pricing engine
    engine = ql.MidPointCdsEngine(
        ql.DefaultProbabilityTermStructureHandle(hazard_curve),
        recovery_rate,
        ql.YieldTermStructureHandle(discount_curve)
    )

    cds.setPricingEngine(engine)

    return {
        'npv': cds.NPV(),
        'fair_spread': cds.fairSpread() * 10000,  # Convert to bps
        'default_probability': 1 - hazard_curve.survivalProbability(ql_maturity)
    }
```

## 🔗 Related Instruments

- [CDS Index](cds_index.md) - Portfolio of CDS contracts
- [CDX/iTraxx Options](cdx_options.md) - Options on CDS indices
- [Total Return Swap](../equity/equity_swap.md) - Alternative credit exposure

## 📚 References

- ISDA Documentation - [CDS Standard Model](https://www.isda.org/)
- O'Kane, D. (2008). *Modelling Single-name and Multi-name Credit Derivatives*
- [QuantLib Credit Risk Documentation](https://www.quantlib.org/)

## ⚠️ Important Notes

!!! warning "Counterparty Risk"
    CDS contracts have significant counterparty risk - protection seller may default

!!! info "Wrong-Way Risk"
    CDS value and counterparty credit quality may be correlated

!!! tip "ISDA Standard"
    Use ISDA standard model for pricing consistency with market

!!! danger "Jump-to-Default Risk"
    Sudden credit events can cause large losses for protection sellers

---

**Last Updated:** 2025-01-17 | **Version:** 1.0 | **Status:** Production Ready
