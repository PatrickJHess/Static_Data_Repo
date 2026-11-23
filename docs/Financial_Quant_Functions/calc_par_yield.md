## Calculte par yields also used for swap rates

```py
def calc_par_yield(months_to_maturity, interim_estimates, sofr_rate=None, freq=6, forward_start_months=0):
    """
    Calculates spot or forward par yields.
    
    Args:
        months_to_maturity (int/float): Tenor in months. Must be an int if >= freq.
        interim_estimates (list): Nelson-Siegel parameters.
        sofr_rate (float, optional): Short rate for restricted model.
        freq (int, optional): Payment frequency in months. Defaults to 6.
        forward_start_months (int/float, optional): Forward start period. Defaults to 0 (spot).
    """
    # 1. Guard against negative maturities
    if months_to_maturity < 0:
        raise ValueError(f"Maturity cannot be negative. Received: {months_to_maturity}")
        
    # 2. Type check: Enforce integers ONLY for swap-market maturities (>= freq)
    if months_to_maturity >= freq and not isinstance(months_to_maturity, int):
        raise TypeError(
            f"months_to_maturity must be an integer when >= freq ({freq} months). "
            f"Received: {type(months_to_maturity).__name__} ({months_to_maturity})"
        )
    
    # 3. Safeguard against exactly 0 maturity (estimates the immediate curve intercept)
    months_to_maturity = max(months_to_maturity, 0.0001)
    
    # 4. Generate schedule of relative months and accrual periods (gamma)
    if months_to_maturity < freq:
        # Short end: allows fractional months and single payments
        payment_months = [months_to_maturity]
        accrual_taus = [months_to_maturity / 12.0]
    else:
        # Short end: integer scheduling and sloppy stub handling
        full_periods = int(months_to_maturity // freq)
        payment_months = list(range(freq, (full_periods * freq) + 1, freq))
        accrual_taus = [freq / 12.0] * full_periods
        
        stub_months = months_to_maturity % freq
        if stub_months > 0:
            payment_months=[stub_months]+payment_months
            accrual_taus=[stub_months / 12.0]+accrual_taus            

    df = pd.DataFrame({
        'relative_months': payment_months,
        'tau': accrual_taus
    })
    
    # 5. Calculate Absolute Time (T) from TODAY (t=0)
    df['T_absolute'] = (df['relative_months'] + forward_start_months) / 12.0
    
    # 6. Get Spot Rates via our Nelson-Siegel function
    # if sofr_rate present check coeff
    if sofr_rate is not None and len(interim_estimates)>3:
      interim_estimates=list(interim_estimates)
      interim_estimates.pop(1)
    spot_rates, _ = ns_spot_rates(
        interim_estimates=interim_estimates, 
        mat_years=df['T_absolute'].values, 
        sofr_rate=sofr_rate
    )
    df['spot_rate'] = spot_rates
    
    # 7. Calculate Discount Factors back to today (t=0)
    df['DF'] = np.exp(-df['spot_rate'] * df['T_absolute'])
    
    # 8. Handle Forward Start Adjustment
    if forward_start_months > 0:
        T_start = forward_start_months / 12.0
        start_spot, _ = ns_spot_rates(interim_estimates, np.array([T_start]), sofr_rate)
        DF_start = np.exp(-start_spot[0] * T_start)
    else:
        DF_start = 1.0 
        
    # 9. Calculate Par Yield
    DF_N = df['DF'].iloc[-1]
    PV01 = (df['tau'] * df['DF']).sum()
    
    par_yield = (DF_start - DF_N) / PV01
    
    return par_yield
```
