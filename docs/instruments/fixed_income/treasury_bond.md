# Treasury Bond

<span class="category-badge badge-fixed-income">FIXED INCOME</span>

Government-issued debt security with fixed coupon payments and principal repayment at maturity.

## Overview

**Class:** `TreasuryBond`
**Category:** Fixed Income
**Type:** Government Bond

A Treasury Bond is a debt security issued by a government to finance spending. It pays a fixed coupon (interest) semi-annually and returns the face value at maturity. Treasury bonds are considered risk-free investments in their home currency and serve as benchmarks for other interest rates.

## 💡 Key Features

- **Risk-Free Rate**: Government-backed, considered default risk-free
- **Fixed Coupon**: Regular semi-annual interest payments
- **Benchmark**: Used to construct yield curves for pricing other instruments
- **Liquidity**: Highly liquid secondary market
- **Maturity Range**: Typically 10-30 years

## 📐 Python Class Implementation

```python
from dataclasses import dataclass
from datetime import datetime
from typing import List, Optional
import QuantLib as ql
from enum import Enum

class DayCountConvention(Enum):
    """Day count conventions for bond calculations"""
    ACT_ACT = "ACT/ACT"
    ACT_365 = "ACT/365"
    ACT_360 = "ACT/360"
    THIRTY_360 = "30/360"

class CompoundingFrequency(Enum):
    """Compounding frequency for yield calculations"""
    ANNUAL = 1
    SEMI_ANNUAL = 2
    QUARTERLY = 4
    MONTHLY = 12
    CONTINUOUS = -1

@dataclass
class CouponPayment:
    """Represents a single coupon payment"""
    payment_date: datetime
    coupon_amount: float
    accrued_days: int
    discount_factor: float = 1.0

@dataclass
class BondCashFlows:
    """All cash flows for a treasury bond"""
    coupon_payments: List[CouponPayment]
    principal_payment: float
    maturity_date: datetime
    total_present_value: float

class TreasuryBond:
    """
    Complete implementation of a Treasury Bond

    Attributes:
        face_value: Par value of the bond (typically 1000 or 100)
        coupon_rate: Annual coupon rate (e.g., 0.05 for 5%)
        issue_date: Date bond was issued
        maturity_date: Date bond matures
        payment_frequency: Payments per year (typically 2 for semi-annual)
        day_count: Day count convention (typically ACT/ACT for US Treasury)
    """

    def __init__(
        self,
        face_value: float,
        coupon_rate: float,
        issue_date: datetime,
        maturity_date: datetime,
        payment_frequency: int = 2,  # Semi-annual
        day_count: str = "ACT/ACT"
    ):
        self.face_value = face_value
        self.coupon_rate = coupon_rate
        self.issue_date = issue_date
        self.maturity_date = maturity_date
        self.payment_frequency = payment_frequency
        self.day_count = day_count

        # Validate inputs
        if maturity_date <= issue_date:
            raise ValueError("Maturity date must be after issue date")
        if coupon_rate < 0:
            raise ValueError("Coupon rate must be non-negative")
        if face_value <= 0:
            raise ValueError("Face value must be positive")

    def generate_coupon_schedule(self) -> List[datetime]:
        """
        Generate coupon payment dates

        Returns:
            List of payment dates
        """
        schedule = []
        current = self.issue_date
        months_per_payment = 12 // self.payment_frequency

        while current < self.maturity_date:
            # Add months
            month = current.month + months_per_payment
            year = current.year + (month - 1) // 12
            month = ((month - 1) % 12) + 1

            # Handle day overflow (e.g., Jan 31 -> Feb 28)
            try:
                current = datetime(year, month, current.day)
            except ValueError:
                # Day doesn't exist in this month, use last day
                if month == 2:
                    day = 29 if year % 4 == 0 and (year % 100 != 0 or year % 400 == 0) else 28
                else:
                    day = 30
                current = datetime(year, month, day)

            if current <= self.maturity_date:
                schedule.append(current)

        # Ensure maturity date is included
        if not schedule or schedule[-1] != self.maturity_date:
            schedule.append(self.maturity_date)

        return schedule

    def calculate_coupon_payment(self, period_days: int) -> float:
        """
        Calculate single coupon payment

        Args:
            period_days: Days in the coupon period

        Returns:
            Coupon payment amount
        """
        if self.day_count == "ACT/ACT":
            year_fraction = period_days / 365.0
        elif self.day_count == "ACT/365":
            year_fraction = period_days / 365.0
        elif self.day_count == "ACT/360":
            year_fraction = period_days / 360.0
        else:  # 30/360
            year_fraction = period_days / 360.0

        return self.face_value * self.coupon_rate * year_fraction

    def calculate_cash_flows(
        self,
        valuation_date: datetime = None
    ) -> BondCashFlows:
        """
        Calculate all future cash flows

        Args:
            valuation_date: Date for valuation (default: issue_date)

        Returns:
            BondCashFlows object with all payment details
        """
        if valuation_date is None:
            valuation_date = self.issue_date

        schedule = self.generate_coupon_schedule()
        coupon_payments = []
        prev_date = self.issue_date

        for payment_date in schedule:
            if payment_date > valuation_date:
                period_days = (payment_date - prev_date).days
                coupon_amount = self.calculate_coupon_payment(period_days)

                coupon_payments.append(CouponPayment(
                    payment_date=payment_date,
                    coupon_amount=coupon_amount,
                    accrued_days=period_days
                ))

            prev_date = payment_date

        return BondCashFlows(
            coupon_payments=coupon_payments,
            principal_payment=self.face_value,
            maturity_date=self.maturity_date,
            total_present_value=0.0  # Will be calculated in price method
        )

    def calculate_price(
        self,
        yield_to_maturity: float,
        valuation_date: datetime = None,
        compounding: int = 2
    ) -> float:
        """
        Calculate bond price given yield

        Args:
            yield_to_maturity: YTM as decimal (e.g., 0.05 for 5%)
            valuation_date: Valuation date
            compounding: Compounding frequency (default: 2 for semi-annual)

        Returns:
            Bond price
        """
        if valuation_date is None:
            valuation_date = self.issue_date

        cash_flows = self.calculate_cash_flows(valuation_date)
        price = 0.0

        # Discount coupon payments
        for cf in cash_flows.coupon_payments:
            days_to_payment = (cf.payment_date - valuation_date).days
            years = days_to_payment / 365.0
            discount_factor = 1 / ((1 + yield_to_maturity / compounding) ** (years * compounding))
            price += cf.coupon_amount * discount_factor

        # Discount principal
        days_to_maturity = (self.maturity_date - valuation_date).days
        years = days_to_maturity / 365.0
        discount_factor = 1 / ((1 + yield_to_maturity / compounding) ** (years * compounding))
        price += self.face_value * discount_factor

        return price

    def calculate_yield(
        self,
        market_price: float,
        valuation_date: datetime = None,
        compounding: int = 2,
        tolerance: float = 1e-6,
        max_iterations: int = 100
    ) -> float:
        """
        Calculate yield to maturity given market price (using Newton-Raphson)

        Args:
            market_price: Current market price of the bond
            valuation_date: Valuation date
            compounding: Compounding frequency
            tolerance: Convergence tolerance
            max_iterations: Maximum iterations

        Returns:
            Yield to maturity
        """
        if valuation_date is None:
            valuation_date = self.issue_date

        # Initial guess based on current yield
        ytm = self.coupon_rate

        for _ in range(max_iterations):
            price = self.calculate_price(ytm, valuation_date, compounding)
            diff = price - market_price

            if abs(diff) < tolerance:
                return ytm

            # Calculate derivative (duration approximation)
            delta_y = 0.0001
            price_up = self.calculate_price(ytm + delta_y, valuation_date, compounding)
            derivative = (price_up - price) / delta_y

            # Newton-Raphson update
            ytm = ytm - diff / derivative

        raise ValueError("Yield calculation did not converge")

    def calculate_duration(
        self,
        yield_to_maturity: float,
        valuation_date: datetime = None,
        compounding: int = 2
    ) -> dict:
        """
        Calculate Macaulay and Modified duration

        Returns:
            Dictionary with duration metrics
        """
        if valuation_date is None:
            valuation_date = self.issue_date

        cash_flows = self.calculate_cash_flows(valuation_date)
        price = self.calculate_price(yield_to_maturity, valuation_date, compounding)

        weighted_time = 0.0

        # Weight coupon payments by time
        for cf in cash_flows.coupon_payments:
            days_to_payment = (cf.payment_date - valuation_date).days
            years = days_to_payment / 365.0
            discount_factor = 1 / ((1 + yield_to_maturity / compounding) ** (years * compounding))
            pv = cf.coupon_amount * discount_factor
            weighted_time += years * pv

        # Weight principal payment
        days_to_maturity = (self.maturity_date - valuation_date).days
        years = days_to_maturity / 365.0
        discount_factor = 1 / ((1 + yield_to_maturity / compounding) ** (years * compounding))
        pv = self.face_value * discount_factor
        weighted_time += years * pv

        macaulay_duration = weighted_time / price
        modified_duration = macaulay_duration / (1 + yield_to_maturity / compounding)

        return {
            'macaulay_duration': macaulay_duration,
            'modified_duration': modified_duration,
            'price_sensitivity': -modified_duration * price  # Dollar duration
        }

    def calculate_convexity(
        self,
        yield_to_maturity: float,
        valuation_date: datetime = None,
        compounding: int = 2
    ) -> float:
        """
        Calculate bond convexity

        Returns:
            Convexity value
        """
        if valuation_date is None:
            valuation_date = self.issue_date

        price = self.calculate_price(yield_to_maturity, valuation_date, compounding)
        cash_flows = self.calculate_cash_flows(valuation_date)

        convexity_sum = 0.0

        for cf in cash_flows.coupon_payments:
            days_to_payment = (cf.payment_date - valuation_date).days
            t = days_to_payment / 365.0
            discount_factor = 1 / ((1 + yield_to_maturity / compounding) ** (t * compounding))
            pv = cf.coupon_amount * discount_factor
            convexity_sum += pv * t * (t + 1)

        # Principal
        days_to_maturity = (self.maturity_date - valuation_date).days
        t = days_to_maturity / 365.0
        discount_factor = 1 / ((1 + yield_to_maturity / compounding) ** (t * compounding))
        pv = self.face_value * discount_factor
        convexity_sum += pv * t * (t + 1)

        return convexity_sum / (price * (1 + yield_to_maturity / compounding) ** 2)

    def get_summary(self) -> dict:
        """Get bond summary information"""
        tenor_years = (self.maturity_date - self.issue_date).days / 365.0

        return {
            'instrument_type': 'Treasury Bond',
            'face_value': f"${self.face_value:,.2f}",
            'coupon_rate': f"{self.coupon_rate * 100:.3f}%",
            'issue_date': self.issue_date.strftime('%Y-%m-%d'),
            'maturity': self.maturity_date.strftime('%Y-%m-%d'),
            'tenor': f"{tenor_years:.2f} years",
            'payment_frequency': f"{self.payment_frequency}x per year",
            'day_count': self.day_count,
            'annual_coupon': f"${self.face_value * self.coupon_rate:,.2f}"
        }
```

## 📊 Usage Examples

### Example 1: Create a 10-Year Treasury Bond

```python
from datetime import datetime

# Create a 10-year Treasury Bond
# Face value: $1,000, Coupon: 3.5%, Semi-annual payments
treasury = TreasuryBond(
    face_value=1000.0,
    coupon_rate=0.035,  # 3.5% annual coupon
    issue_date=datetime(2025, 1, 15),
    maturity_date=datetime(2035, 1, 15),
    payment_frequency=2,  # Semi-annual
    day_count="ACT/ACT"
)

# Get summary
print("Treasury Bond Summary:")
for key, value in treasury.get_summary().items():
    print(f"  {key}: {value}")
```

**Output:**
```
Treasury Bond Summary:
  instrument_type: Treasury Bond
  face_value: $1,000.00
  coupon_rate: 3.500%
  issue_date: 2025-01-15
  maturity: 2035-01-15
  tenor: 10.00 years
  payment_frequency: 2x per year
  day_count: ACT/ACT
  annual_coupon: $35.00
```

### Example 2: Calculate Bond Price

```python
# Calculate price given a yield to maturity of 4%
ytm = 0.04
valuation_date = datetime(2025, 1, 15)

price = treasury.calculate_price(
    yield_to_maturity=ytm,
    valuation_date=valuation_date,
    compounding=2
)

print(f"\nBond Price at {ytm*100:.2f}% YTM: ${price:.2f}")
print(f"Price as % of par: {price/treasury.face_value*100:.3f}%")

# Calculate at different yields
print("\nPrice-Yield Relationship:")
yields = [0.02, 0.03, 0.035, 0.04, 0.05]
for y in yields:
    p = treasury.calculate_price(y, valuation_date)
    print(f"  YTM {y*100:.1f}%: ${p:.2f} ({p/1000*100:.2f}% of par)")
```

### Example 3: Calculate Yield from Market Price

```python
# Bond trading at $950 in the market
market_price = 950.00

calculated_ytm = treasury.calculate_yield(
    market_price=market_price,
    valuation_date=valuation_date,
    compounding=2
)

print(f"\nMarket Price: ${market_price:.2f}")
print(f"Calculated YTM: {calculated_ytm*100:.4f}%")
print(f"Current Yield: {(treasury.face_value * treasury.coupon_rate / market_price)*100:.4f}%")
```

### Example 4: Duration and Convexity Analysis

```python
ytm = 0.04

# Calculate duration metrics
duration_metrics = treasury.calculate_duration(
    yield_to_maturity=ytm,
    valuation_date=valuation_date
)

# Calculate convexity
convexity = treasury.calculate_convexity(
    yield_to_maturity=ytm,
    valuation_date=valuation_date
)

print("\nRisk Metrics:")
print(f"  Macaulay Duration: {duration_metrics['macaulay_duration']:.4f} years")
print(f"  Modified Duration: {duration_metrics['modified_duration']:.4f} years")
print(f"  Dollar Duration: ${duration_metrics['price_sensitivity']:.2f}")
print(f"  Convexity: {convexity:.6f}")

# Price sensitivity analysis
print("\n1% Yield Change Impact:")
delta_y = 0.01
price_base = treasury.calculate_price(ytm, valuation_date)
price_up = treasury.calculate_price(ytm + delta_y, valuation_date)
price_down = treasury.calculate_price(ytm - delta_y, valuation_date)

print(f"  Base Price: ${price_base:.2f}")
print(f"  Price if yield +100bp: ${price_up:.2f} ({(price_up-price_base):.2f})")
print(f"  Price if yield -100bp: ${price_down:.2f} ({(price_down-price_base):.2f})")
```

### Example 5: Cash Flow Schedule

```python
# Generate full cash flow schedule
cash_flows = treasury.calculate_cash_flows(valuation_date)

print("\nCoupon Payment Schedule:")
print("-" * 70)
print(f"{'Date':<15} {'Coupon':<12} {'Days':<10} {'Cumulative'}")
print("-" * 70)

cumulative = 0
for cf in cash_flows.coupon_payments[:10]:  # Show first 10 payments
    cumulative += cf.coupon_amount
    print(f"{cf.payment_date.strftime('%Y-%m-%d'):<15} "
          f"${cf.coupon_amount:>10,.2f} "
          f"{cf.accrued_days:>8}d "
          f"${cumulative:>12,.2f}")

print(f"\nPrincipal at Maturity: ${cash_flows.principal_payment:,.2f}")
print(f"Total Coupons: ${sum(cf.coupon_amount for cf in cash_flows.coupon_payments):,.2f}")
print(f"Total Cash Flow: ${cash_flows.principal_payment + sum(cf.coupon_amount for cf in cash_flows.coupon_payments):,.2f}")
```

## 🎯 Use Cases

### 1. Yield Curve Construction

```python
# Multiple Treasury bonds for yield curve
bonds = [
    TreasuryBond(1000, 0.025, datetime(2025, 1, 15), datetime(2027, 1, 15), 2),  # 2Y
    TreasuryBond(1000, 0.030, datetime(2025, 1, 15), datetime(2030, 1, 15), 2),  # 5Y
    TreasuryBond(1000, 0.035, datetime(2025, 1, 15), datetime(2035, 1, 15), 2),  # 10Y
    TreasuryBond(1000, 0.040, datetime(2025, 1, 15), datetime(2055, 1, 15), 2),  # 30Y
]

market_prices = [985, 970, 950, 920]

print("Treasury Yield Curve:")
print(f"{'Tenor':<10} {'Price':<12} {'YTM':<10} {'Duration'}")
print("-" * 50)

for bond, price in zip(bonds, market_prices):
    tenor = (bond.maturity_date - bond.issue_date).days / 365.0
    ytm = bond.calculate_yield(price, datetime(2025, 1, 15))
    duration = bond.calculate_duration(ytm, datetime(2025, 1, 15))

    print(f"{tenor:<10.1f} ${price:<10,.2f} {ytm*100:>6.3f}% {duration['modified_duration']:>8.3f}")
```

### 2. Portfolio Duration Matching

```python
# Duration matching for liability management
liability_duration = 7.5  # years
total_portfolio = 10_000_000  # $10M

# Find bond combination to match duration
bond_5y = TreasuryBond(1000, 0.030, datetime(2025, 1, 15), datetime(2030, 1, 15), 2)
bond_10y = TreasuryBond(1000, 0.035, datetime(2025, 1, 15), datetime(2035, 1, 15), 2)

dur_5y = bond_5y.calculate_duration(0.04, datetime(2025, 1, 15))['modified_duration']
dur_10y = bond_10y.calculate_duration(0.04, datetime(2025, 1, 15))['modified_duration']

# Weight calculation: w * dur_5y + (1-w) * dur_10y = target_duration
weight_5y = (dur_10y - liability_duration) / (dur_10y - dur_5y)
weight_10y = 1 - weight_5y

print(f"Portfolio Duration Matching:")
print(f"  Target Duration: {liability_duration:.2f} years")
print(f"  5Y Bond Weight: {weight_5y*100:.2f}% (${total_portfolio * weight_5y:,.0f})")
print(f"  10Y Bond Weight: {weight_10y*100:.2f}% (${total_portfolio * weight_10y:,.0f})")
```

## 📈 Advanced Pricing with QuantLib

```python
import QuantLib as ql

def price_treasury_with_quantlib(
    face_value: float,
    coupon_rate: float,
    issue_date: datetime,
    maturity_date: datetime,
    settlement_date: datetime,
    discount_curve_handle: ql.YieldTermStructureHandle
) -> dict:
    """
    Price Treasury bond using QuantLib

    Returns:
        Dictionary with NPV, clean price, accrued interest, etc.
    """
    # Convert dates
    ql_issue = ql.Date(issue_date.day, issue_date.month, issue_date.year)
    ql_maturity = ql.Date(maturity_date.day, maturity_date.month, maturity_date.year)
    ql_settlement = ql.Date(settlement_date.day, settlement_date.month, settlement_date.year)

    # Set evaluation date
    ql.Settings.instance().evaluationDate = ql_settlement

    # Create schedule
    schedule = ql.Schedule(
        ql_issue,
        ql_maturity,
        ql.Period(ql.Semiannual),
        ql.UnitedStates(ql.UnitedStates.GovernmentBond),
        ql.Unadjusted,
        ql.Unadjusted,
        ql.DateGeneration.Backward,
        False
    )

    # Create bond
    bond = ql.FixedRateBond(
        2,  # settlement days
        face_value,
        schedule,
        [coupon_rate],
        ql.ActualActual(ql.ActualActual.Bond)
    )

    # Set pricing engine
    bond_engine = ql.DiscountingBondEngine(discount_curve_handle)
    bond.setPricingEngine(bond_engine)

    return {
        'npv': bond.NPV(),
        'clean_price': bond.cleanPrice(),
        'dirty_price': bond.dirtyPrice(),
        'accrued_interest': bond.accruedAmount(),
        'yield': bond.bondYield(ql.ActualActual(ql.ActualActual.Bond), ql.Compounded, ql.Semiannual)
    }
```

## 🔗 Related Instruments

- [Callable Bond](callable_bond.md) - Bond with embedded call option
- [Floating Rate Note](frn.md) - Variable coupon payments
- [Interest Rate Swap](../rates/irs.md) - Fixed vs floating rate exchange

## 📚 References

- U.S. Department of the Treasury - [TreasuryDirect](https://www.treasurydirect.gov/)
- Fabozzi, F. J. (2007). *Fixed Income Analysis*
- [QuantLib Fixed Income Documentation](https://www.quantlib.org/)

## ⚠️ Important Notes

!!! warning "Interest Rate Risk"
    Treasury bonds have significant interest rate risk - prices fall when yields rise

!!! info "Inflation Risk"
    Fixed coupon payments lose purchasing power in inflationary environments

!!! tip "Benchmark Role"
    Treasury yields serve as risk-free rates for discounting other instruments

---

**Last Updated:** 2025-01-17 | **Version:** 1.0 | **Status:** Production Ready
