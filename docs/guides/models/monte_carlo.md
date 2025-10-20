# Monte Carlo Simulation

Stochastic simulation framework for pricing path-dependent derivatives by averaging discounted payoffs across numerous scenarios.

## Workflow

1. Generate correlated random paths for underlying factors.
2. Revalue instrument cash flows along each path.
3. Discount path payoffs to present value and average the results.
4. Estimate standard error to quantify statistical confidence.

## Enhancements

- Variance reduction: antithetic variates, control variates, stratified sampling.
- Quasi-random sequences for faster convergence.
- Least Squares Monte Carlo (LSM) for American-style exercise.

## Practical Tips

- Increase path count until price error is below tolerance.
- Seed random generators for reproducibility.
- Parallelize simulations to reduce wall-clock time.

## Related Examples

- [Equity American Option (LSM)](../../instruments/equity/american_option.md#3-least-squares-monte-carlo-lsm)
- [Equity Swap](../../instruments/equity/equity_swap.md)
