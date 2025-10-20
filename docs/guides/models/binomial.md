# Binomial Tree Methods

Lattice-based approach for pricing options that captures American exercise features and path-dependent payoffs.

## When to Use

- American-style options requiring early exercise checks.
- Path-dependent contracts where tree recombination is efficient.
- Products demanding scenario-by-scenario risk analysis.

## Model Variants

- Cox-Ross-Rubinstein (CRR)
- Jarrow-Rudd
- Trigeorgis and other recombining trees

## Numerical Considerations

- Convergence improves with more time steps but increases compute cost.
- Stability depends on choosing an appropriate time step relative to volatility.
- Greeks can be approximated from finite differences on the lattice.

## Related Material

- [Equity American Option](../../instruments/equity/american_option.md)
- [Equity American Warrant](../../instruments/equity/american_warrant.md)
