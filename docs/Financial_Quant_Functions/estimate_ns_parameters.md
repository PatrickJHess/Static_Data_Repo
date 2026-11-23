## estimate Nelson-Siegel model

```py
def estimate_ns_parameters(df_payoff_matrix,P_actual,mat_years,guesses,sofr_rate=None):
  """
    Estimates Nelson-Siegel parameters by minimizing the Sum of Squared Residuals (SSR) 
    between actual bond prices and model-predicted bond prices.

    Args:
        df_payoff_matrix (pd.DataFrame or np.ndarray): Matrix of bond cash flows 
            where rows are bonds and columns represent time periods.
        P_actual (pd.Series or np.ndarray): Observed market prices of the bonds.
        mat_years (array-like): Time to maturity for each cash flow period, in years.
        guesses (list or array-like): Initial guesses for the optimization solver.
            Must have length 3 if sofr_rate is provided, or length 4 if None.
        sofr_rate (float, optional): A proxy short-term rate used to restrict the 
            short end of the curve. Defaults to None.

    Returns:
        scipy.optimize.OptimizeResult: The optimization result object containing 
            the estimated parameters (in the `.x` attribute), success status, 
            and minimum SSR achieved.

    Raises:
        ValueError: If the length of the `guesses` array does not match the 
            requirements of the chosen model type (restricted vs. unrestricted).
  """
  # objective function is sum squared residuals (SSR)
  def predict_prices(interim_estimates,df_payoff_matrix,P_actual,mat_years):

    # for step one get rates
    spot_rates,mat_years=ns_spot_rates(interim_estimates,mat_years,sofr_rate=sofr_rate)

    # calculate zero prices
    zero_prices=np.exp(-spot_rates*mat_years)

    # calculate predicted prices (@ here is the same as np.matmul)
    P_predicted=df_payoff_matrix@zero_prices

    # step 2 calculate the distance with sum of squared residuals
    ssr= np.sum((P_actual.values-P_predicted)**2)
    return ssr

  # use the scipy minimize function to estimate parameters

# Validate guess lengths based on model type
    if sofr_rate is not None and len(guesses) != 3:
        raise ValueError(f"Restricted model (SOFR={sofr_rate}) requires exactly 3 guesses. You provided {len(guesses)}.")
    
    if sofr_rate is None and len(guesses) != 4:
        raise ValueError(f"Unrestricted model requires exactly 4 guesses. You provided {len(guesses)}.")

  # Nelder-Mead doesn't take derivatives and tolerant of data
  method='Nelder-Mead'

  # minimize results
  ns_results=minimize(fun=predict_prices,
                  x0=guesses,
                  args=(df_payoff_matrix,P_actual,mat_years),
                  method=method)

  # get estimated coefficients of Nelson-Siegel


  # get status of minimization
  completion_status=ns_results.message

  if sofr_rate is not None:
    b_0,b_2,tau=ns_results.x
    display(f'Completion status: {completion_status}')
    display(f'Long Rate (Beta Zero) {b_0:.4f}..Slope (sofr_rate -Beta Zero) {sofr_rate-b_0: .4f}...\
    Shape (Beta Two) {b_2: .4f}...Scaling (Tau) {tau: .4f}') 

  else:
    b_0,b_1,b_2,tau=ns_results.x
    display(f'Completion status: {completion_status}')
    display(f'Long Rate (Beta Zero) {b_0:.4f}..Slope (Beta One) {b_1: .4f}...\
    Shape (Beta Two) {b_2: .4f}...Scaling (Tau) {tau: .4f}') 
  return ns_results

```
