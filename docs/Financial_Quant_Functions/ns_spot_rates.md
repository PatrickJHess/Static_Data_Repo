## Calculate spot rates from Nelson-Siegel estimates

```py
def ns_spot_rates(interim_estimates, mat_years, sofr_rate=None, fixed_tau=0.731):
    """
    Calculates spot rates using a fully vectorized Nelson-Siegel model.
    Accepts either single parameter vectors or (N, columns) simulation matrices.
    Dynamically accommodates tau if included in the estimates.
    """
    # 1. Ensure mat_years is a numpy array and handle division-by-zero
    t = np.array(mat_years, dtype=float)
    t = np.where(t == 0, 1e-8, t)
    
    if isinstance(interim_estimates, pd.DataFrame):
        interim_estimates = interim_estimates.to_numpy()
        
    # 2. Check if we are passing a 2D matrix of simulations
    is_matrix = isinstance(interim_estimates, np.ndarray) and interim_estimates.ndim == 2
    
    if sofr_rate is not None:  # Restricted model
        if is_matrix:
            # Base restricted params: b0, b2 (2 columns)
            b_0 = interim_estimates[:, 0][:, np.newaxis]
            b_2 = interim_estimates[:, 1][:, np.newaxis]
            
            # If 3 columns, the 3rd is tau
            if interim_estimates.shape[1] == 3:
                tau = interim_estimates[:, 2][:, np.newaxis]
            else:
                tau = fixed_tau 
        else:
            # Fallback for 1D array
            b_0, b_2 = interim_estimates[0], interim_estimates[1]
            tau = interim_estimates[2] if len(interim_estimates) == 3 else fixed_tau

        # Vectorized math
        spot_rates = (
            b_0
            + (sofr_rate - b_0) * (1 - np.exp(-t/tau)) / (t/tau)
            + b_2 * ((1 - np.exp(-t/tau)) / (t/tau) - np.exp(-t/tau))
        )
        
    else:  # Unrestricted model
        if is_matrix:
            # Base unrestricted params: b0, b1, b2 (3 columns)
            b_0 = interim_estimates[:, 0][:, np.newaxis]
            b_1 = interim_estimates[:, 1][:, np.newaxis]
            b_2 = interim_estimates[:, 2][:, np.newaxis]
            
            # If 4 columns, the 4th is tau (Index 3)
            if interim_estimates.shape[1] == 4:
                tau = interim_estimates[:, 3][:, np.newaxis]
            else:
                tau = fixed_tau            
        else:
            # Fallback for 1D array
            b_0, b_1, b_2 = interim_estimates[0], interim_estimates[1], interim_estimates[2]
            tau = interim_estimates[3] if len(interim_estimates) == 4 else fixed_tau

        # Vectorized math
        spot_rates = (
            b_0
            + b_1 * (1 - np.exp(-t/tau)) / (t/tau)
            + b_2 * ((1 - np.exp(-t/tau)) / (t/tau) - np.exp(-t/tau))
        )

    return spot_rates, t

```
