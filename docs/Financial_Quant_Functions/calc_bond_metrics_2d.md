## Calc Bond Metrics (YTM, DUR, CONV)

```py
def calc_bond_metrics_2d(guesses, cash_flows, mat_years, prices):
    """
    Calculates YTM, Duration, and Convexity for a single bond OR a portfolio.
    Expects Cash Flows as (Dates x Bonds).
    """
    is_scalar_input = np.isscalar(prices)

    p = np.atleast_1d(prices)
    cf = np.asarray(cash_flows)

    # If a single bond's cash flows are passed as a flat list, stand it up into a column (Dates, 1)
    if cf.ndim == 1:
        cf = cf[:, np.newaxis]

    # Stand the dates up into a column: (Dates, 1)
    t = np.asarray(mat_years)[:, np.newaxis]

    if np.isscalar(guesses):
        guesses = np.full(len(p), guesses, dtype=float)
    else:
        guesses = np.atleast_1d(guesses)

    def ytm_objective(y, cf, t, p):
        # y is (8,). t is (100, 1). y * t automatically broadcasts to (100, 8)
        # Sum down the rows (axis=0) to collapse the dates, leaving (8,) PVs
        pv = np.sum(cf * np.exp(-y * t), axis=0)
        return pv - p

    def ytm_derivative(y, cf, t, p):
        return np.sum(-t * cf * np.exp(-y * t), axis=0)

    # --- STEP 1: Solve for YTM ---
    ytm = optimize.newton(
        func=ytm_objective,
        x0=guesses,
        fprime=ytm_derivative,
        args=(cf, t, p)
    )

    # --- STEP 2: Calculate Risk Metrics ---
    # ytm is (8,). t is (100, 1). discounted_cf becomes (100, 8)
    discounted_cf = cf * np.exp(-ytm * t)

    # Sum down the rows (axis=0) and divide by prices
    duration = np.sum(t * discounted_cf, axis=0) / p
    convexity = np.sum((t**2) * discounted_cf, axis=0) / p

    if is_scalar_input:
        return ytm[0], duration[0], convexity[0]

    return ytm, duration, convexity
```
