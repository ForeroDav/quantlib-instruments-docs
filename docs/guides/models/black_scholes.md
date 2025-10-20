# Black-Scholes Model

Analytical closed-form solution for pricing European-style options under the lognormal asset price assumption and constant volatility.

## Core Assumptions

- Underlying follows a geometric Brownian motion with drift and constant volatility.
- Markets are frictionless with continuous trading and no arbitrage.
- Risk-free rate and dividend yield remain constant over the option life.

## Inputs

- Spot price, strike price, time to maturity.
- Risk-free rate, dividend yield, volatility.

## Outputs

- Option fair value, delta, gamma, vega, theta, rho.

## Related Examples

- [Equity European Option](../../instruments/equity/european_option.md)
- [Quick Start Pricing Walkthrough](../quickstart.md)
