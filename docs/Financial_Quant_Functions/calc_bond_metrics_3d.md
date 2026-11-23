## Calculate metrics across bonds and term structures

```py
def calc_bond_metrics_3d(guesses, cash_flows, mat_years, prices):
    """
    Calculates YTM, Duration, and Convexity for a matrix of prices (Scenarios x Bonds).
    Assumes continuous compounding.

    Returns:
        ytm (ndarray): Yield to Maturity
        duration (ndarray): Macaulay/Modified Duration
        convexity (ndarray): Convexity
    """
    cf = np.asarray(cash_flows)              # (100, 8)
    t = np.asarray(mat_years)[:, np.newaxis] # (100, 1)
    p = np.asarray(prices)                   # (6, 8)
    is_1d = False
    if p.ndim == 1:
        is_1d = True
        p = p[np.newaxis, :]  # Temporarily promote to (1, Bonds)
    # Broadcast the initial guess
    if np.isscalar(guesses):
        guesses = np.full_like(p, guesses, dtype=float)
    else:
        guesses = np.broadcast_to(guesses, p.shape)

    def ytm_objective(y, cf, t, p):
        # Insert the Dates dimension in the middle: (6, 1, 8)
        y_3d = y[:, np.newaxis, :]

        # Multiply to get (6, 100, 8), then sum across Dates (axis=1) -> (6, 8)
        pv = np.sum(cf * np.exp(-y_3d * t), axis=1)
        return pv - p

    def ytm_derivative(y, cf, t, p):
        y_3d = y[:, np.newaxis,:]
        return np.sum(-t * cf * np.exp(-y_3d * t), axis=1)

    # --- STEP 1: Solve for YTM ---
    ytm = optimize.newton(
        func=ytm_objective,
        x0=guesses,
        fprime=ytm_derivative,
        args=(cf, t, p)
    )

    # --- STEP 2: Calculate Risk Metrics ---
    # Convert the final YTM grid into 3D: (Scenarios, 8, 1)
    y_3d_final = ytm[:, np.newaxis,:]

    # Calculate the final discounted cash flows matrix: (Scenarios, 8, 100)
    discounted_cf = cf * np.exp(-y_3d_final * t)
    # Calculate Duration and Convexity
    # We divide directly by 'p' (the market prices) because the optimizer
    # just guaranteed that sum(discounted_cf) == p
    duration = np.sum(t * discounted_cf, axis=1) / p
    convexity = np.sum((t**2) * discounted_cf, axis=1) / p

    return ytm, duration, convexity

```
