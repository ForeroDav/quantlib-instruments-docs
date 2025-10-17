# API Reference

Complete API documentation for QuantLib instruments.

## Core Classes

### Instrument Base Class

All instruments inherit from the base `Instrument` class:

```python
class Instrument:
    """Base class for all financial instruments"""
    def NPV(self) -> float:
        """Net Present Value of the instrument"""
        pass

    def setPricingEngine(self, engine):
        """Set the pricing engine"""
        pass
```

## Equity Instruments

- `EquitySwap`
- `EquityIndexSwap`
- `EquityAmericanOption`
- `EquityEuropeanOption`
- `EquityIndexFutures`

## Fixed Income Instruments

- `TreasuryBond`
- `CallableBond`
- `FloatingRateNote`

## Credit Derivatives

- `CreditDefaultSwap`
- `CreditDefaultSwapIndex`

## See Documentation

For detailed API documentation, see individual instrument pages in the [Instruments](../instruments/index.md) section.
