## Single bond calculate ytm


```py
def calc_ytm(guesses, cash_flows, mat_years, prices):
def calc_ytm(guesses, cash_flows, mat_years, prices):
    """
    Calculates the Yield to Maturity (YTM) using the Newton-Raphson method.
    """
    # FIX: Use np.isscalar to check if the user passed a single number instead of an array
    if np.isscalar(guesses):
        # If a single guess is provided, broadcast it to match the length of the prices array
        guesses = [guesses] * len(prices) 

    def ytm_objective(y, cash_flows, times, market_price):
        """Objective function: difference between modeled PV and actual price."""
        # Continuous compounding: PV = CF * e^(-y * t)
        pv = sum([cf * np.exp(-y * t) for cf, t in zip(cash_flows, times)])
        return pv - market_price

    def ytm_derivative(y, cash_flows, times, market_price):
        """Derivative of the objective function with respect to the yield (y)."""
        # Power rule applied to e^(-y * t) brings down the (-t)
        return sum([-t * cf * np.exp(-y * t) for cf, t in zip(cash_flows, times)])
    
    # Run the Newton-Raphson optimization
    # FIX: Updated x0 to 'guesses' and args to use 'mat_years'
    ytm = optimize.newton(
        func=ytm_objective,
        x0=guesses,
        fprime=ytm_derivative,
        args=(cash_flows, mat_years, prices)
    )
    
    return ytm
```
