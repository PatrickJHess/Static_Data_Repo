# ***Setup The Notebook***

* **Get `financial_quant`**  
* **Import needed libraries and modules**  
* **Instantiate FredReader and MASSIVEReader**

```py
# Use Python's built-in urllib to fetch the remote setup script
import urllib.request

# Download, decode, and execute the updater script directly from GitHub
updater_url = "https://raw.githubusercontent.com/PatrickJHess/quant_repo/master/fq_updater.py"
exec(urllib.request.urlopen(updater_url).read().decode('utf-8'))

# Run the installer/updater function and alias the package as 'fq'
fq = import_financial_quant()

# Import libraries and modules
import pandas as pd
import numpy as np
from datetime import datetime, date
from IPython.display import clear_output

# Instantiate the data pipelines from the financial_quant package
fred_data = fq.FredReader()
massive_data = fq.MASSIVEReader(calls_per_minute=5)
```

# 

# ---

# 

# ***Step 1***

## ***Get Eligible Bonds***

```py
# Load the curated CME conversion factor data for the Dec 2026 (ZBZ6) contract
# static_futures_data function included in financial_quant
df_eligible = fq.static_futures_data('CME_Data_UST_10Y_DEC_2026', index_name='Maturity')

# Convert the string index into proper pandas datetime objects so we can merge it cleanly later
df_eligible.index = pd.to_datetime(df_eligible.index, format='%m/%d/%y')

# Preview the official conversion factors
display(df_eligible.head())
```

## 

## ***Get FEDInvest Data***

```py
# settlement t+1 so data date is 11/10/2026
my_date=datetime(2026, 11,9)

# fetch Treasury data for a specific date
df, stamp,settlement = fq.FEDInvest(my_date)

#display price and date and settlement
display(f'Date of prices {stamp}  Settlement date  {settlement}')

# standardize the data
clean_df=fq.clean_FEDInvest(df)
display(clean_df.head())
```

## 

## ***Check for Missing Bonds***

```py
# Elgible are all bonds so drop SECURITY TYPE
clean_df = clean_df.drop(columns='SECURITY TYPE')

# 1. Standardize names and flatten
clean_df.index.name = 'Maturity'
clean_df_flat = clean_df.reset_index()

df_eligible = df_eligible.rename(columns={'Coupon Rate': 'Coupon'})
df_eligible_flat = df_eligible.reset_index()

# 2. Standardize 'Maturity' to pure dates
clean_df_flat['Maturity'] = pd.to_datetime(clean_df_flat['Maturity']).dt.normalize()
df_eligible_flat['Maturity'] = pd.to_datetime(df_eligible_flat['Maturity']).dt.normalize()

# 3. Standardize 'Coupon' to floats and round
clean_df_flat['Coupon'] = clean_df_flat['Coupon'].astype(float).round(4)
df_eligible_flat['Coupon'] = df_eligible_flat['Coupon'].astype(float).round(4)

# ---------------------------------------------------------
# Diagnostic: Find eligible bonds NOT in FedInvest
# ---------------------------------------------------------
df_check = pd.merge(
    df_eligible_flat,
    clean_df_flat[['Maturity', 'Coupon']].drop_duplicates(),
    on=['Maturity', 'Coupon'],
    how='left',
    indicator=True
)

missing_from_fedinvest = df_check[df_check['_merge'] == 'left_only'].drop(columns=['_merge'])

if not missing_from_fedinvest.empty:
    print("⚠️ WARNING: Eligible bonds missing from FedInvest:")
    display(missing_from_fedinvest)
else:
    print("✅ All eligible bonds are in FedInvest.")
# ---------------------------------------------------------

# 4. Final Inner Merge
df_combined = pd.merge(
    clean_df_flat,
    df_eligible_flat,
    on=['Maturity', 'Coupon'],
    how='inner'
)

# 5. Set index back and sort
df_combined = df_combined.set_index('Maturity').sort_values(by=['Maturity', 'Coupon'])
```

## 

## ***Clean up the Data***

```py
# copy used to make sure the DataFrames aren't linked
num_rows = clean_df.shape[0]
df_ctd = df_combined[:num_rows].copy()

# vectorized calculation of clean prices
bid_ask_true = (df_ctd['Price Bid'] > 0) & (df_ctd['Price Ask'] > 0)

# bid/ask average assigned to true_value
true_value = (df_ctd['Price Bid'] + df_ctd['Price Ask']) / 2

# maximum of three prices assigned to false value
false_value = df_ctd[['END OF DAY', 'Price Bid', 'Price Ask']].max(axis=1)

# equivalent to a vectorized if-else assignment of clean price
df_ctd['Clean Price'] = np.where(bid_ask_true, true_value, false_value)
```

# 

# 

# ---

# 

# ***Step 2***

## 

## ***Access SOFR***

```py
# Use FredReader to the SOFR Rate
sofr_11_09=fred_data.get_series('SOFR',start_date=settlement,end_date=settlement)

# Get the single value and convert it into floating decimal rate
sofr=sofr_11_09.values[0]
sofr_rate=float(sofr[0])
sofr_rate=sofr_rate/100

```

## 

## ***\`timing\_option\`***

```py
def timing_option(settlement, maturity, coupon, clean_price, sofr_rate, freq=2):
   """
   Calculates accrued interest, dirty price, daily carry, and the optimal delivery
   timing (Immediate vs Delay) for a Treasury bond.
   """
   settlement = pd.Timestamp(settlement)
   maturity = pd.Timestamp(maturity)
   current_year = settlement.year

   # --- HELPER FUNCTION ---
   def get_proxy_maturity(mat, target_year):
       """Creates proxy maturity, utilizing Pandas' native month-end calendar logic."""
       if mat.is_month_end:
           # Safely grab the exact last day of the target month (handles leap years natively)
           last_day = pd.Timestamp(year=target_year, month=mat.month, day=1).days_in_month
           return pd.Timestamp(year=target_year, month=mat.month, day=last_day)
       else:
           # Non-EOM dates (like the 15th) are immune to the Feb 29th ValueError
           return mat.replace(year=target_year)
   # -----------------------

   # Optimize by creating a proxy maturity date just 1 year out
   proxy_maturity = get_proxy_maturity(maturity, current_year + 1)

   # 1. Find coupon dates (fq prefix for financial_quant)
   dates = fq.scheduled_pay_dates(proxy_maturity, settlement=settlement, freq=freq)
   next_date = pd.Timestamp(dates[0])

   # Dynamically determine months between payments
   months_between = int(12 / freq)

   # 2. Handle end-of-month Treasury rule for the previous coupon
   if next_date.is_month_end:
       last_date = next_date - pd.offsets.MonthEnd(months_between)
   else:
       last_date = next_date - pd.DateOffset(months=months_between)

   # 3. Calculate Actual/Actual day counts
   days_between = (next_date - last_date).days
   days_since = (settlement - last_date).days

   # 4. Financial Calculations
   coupon_payment = coupon / freq
   accrued_interest = coupon_payment * (days_since / days_between)
   daily_accrual = coupon_payment / days_between

   dirty_price = clean_price + accrued_interest
   daily_sofr_cost = dirty_price * (sofr_rate / 360)

   return accrued_interest, dirty_price, daily_accrual, daily_sofr_cost, next_date

```

## 

## ***Calculate Values***

```py
df_ctd[['Accrued_Interest', 'Dirty Price', 'Daily_Accrual', 'Daily_Cost', 'Next_Coupon_Date']] = df_ctd.apply(
    lambda row: timing_option(settlement,row.name,row['Coupon'],row['Clean Price'],sofr_rate),
    axis=1,
    result_type='expand'
)
display(df_ctd.head(10))

```

## ***Get Settlement Price***

```py
# Use MASSIVEReader
# Query the December 2026 30-Year T-Bond futures (ZBZ6) on the transaction date
start='2026-11-09'
futures_11_09=massive_data.get_futures_data('ZBZ6',start_date=start,
                                           end_date=start)

# Extract the scalar settlement price from the single-row DataFrame
settlement_price=futures_11_09['settlement_price'].iloc[0]
settlement_price

```

## ***Accrued Interest Settlement Date***

```py
# 1. Define Dates
t_settlement = pd.Timestamp(settlement)

days_first = (first_delivery - t_settlement).days
days_last = (last_delivery - t_settlement).days

# 2. Check for Interim Coupons and calculate Days Since Coupon
has_interim_first = df_ctd['Next_Coupon_Date'] <= first_delivery
has_interim_last = df_ctd['Next_Coupon_Date'] <= last_delivery

days_since_coupon_first = (first_delivery - df_ctd['Next_Coupon_Date']).dt.days
days_since_coupon_last = (last_delivery - df_ctd['Next_Coupon_Date']).dt.days

# 3. Calculate Delivery Accrued (Resetting to 0 if an interim coupon was paid)
accrued_first = np.where(
    has_interim_first,
    df_ctd['Daily_Accrual'] * days_since_coupon_first,
    df_ctd['Accrued_Interest'] + (df_ctd['Daily_Accrual'] * days_first)
)

accrued_last = np.where(
    has_interim_last,
    df_ctd['Daily_Accrual'] * days_since_coupon_last,
    df_ctd['Accrued_Interest'] + (df_ctd['Daily_Accrual'] * days_last)
)

# 4. Calculate Invoice Prices
invoice_base = settlement_price * df_ctd['Conversion_Factor']
df_ctd['Invoice_First'] = invoice_base + accrued_first
df_ctd['Invoice_Last'] = invoice_base + accrued_last

# Display the Invoice Price results

#  Drop raw price columns to declutter our working DataFrame
df_clean_ctd = df_ctd.drop(columns=['Price Bid', 'Price Ask', 'END OF DAY']).copy()
display(df_clean_ctd[['Clean Price', 'Next_Coupon_Date', 'Invoice_First', 'Invoice_Last']].head())

```

---

# ***Step 3***

## ***Forward Cash Prices***

```py
# 1. Calculate Spot Dirty Price (Clean Price + Spot Accrued Interest)
df_clean_ctd['Spot_Dirty'] = df_clean_ctd['Clean Price'] + df_clean_ctd['Accrued_Interest']

# 2. Define the cash coupon payment (Semi-annual)
coupon_pmt = df_clean_ctd['Coupon'] / 2

# 3. Calculate Forward Cash Prices
# If an interim coupon is paid, we subtract the cash received from our total forward cost
df_clean_ctd['Forward_First'] = np.where(
    has_interim_first,
    df_clean_ctd['Spot_Dirty'] + (df_clean_ctd['Daily_Cost'] * days_first) - coupon_pmt,
    df_clean_ctd['Spot_Dirty'] + (df_clean_ctd['Daily_Cost'] * days_first)
)

df_clean_ctd['Forward_Last'] = np.where(
    has_interim_last,
    df_clean_ctd['Spot_Dirty'] + (df_clean_ctd['Daily_Cost'] * days_last) - coupon_pmt,
    df_clean_ctd['Spot_Dirty'] + (df_clean_ctd['Daily_Cost'] * days_last)
)

# Display the Forward Prices
display(df_clean_ctd[['Clean Price', 'Spot_Dirty','Daily_Cost', 'Forward_First', 'Forward_Last']].head())

```

---

# ***Step 4***

## ***Net Basis for first, last, and CTD Delivery Date***

```py
# 1. Calculate Net Basis for both dates (Forward Price - Invoice Price)
df_clean_ctd['Net_Basis_First'] = df_clean_ctd['Forward_First'] - df_clean_ctd['Invoice_First']
df_clean_ctd['Net_Basis_Last'] = df_clean_ctd['Forward_Last'] - df_clean_ctd['Invoice_Last']

# 2. THE GRAND FINALE: Find the minimum Net Basis
df_clean_ctd['Net_Basis'] = np.minimum(df_clean_ctd['Net_Basis_First'], df_clean_ctd['Net_Basis_Last'])

# 3. Tag the optimal delivery decision so we can see WHICH date won
df_clean_ctd['Optimal_Delivery'] = np.where(
    df_clean_ctd['Net_Basis_First'] <= df_clean_ctd['Net_Basis_Last'],
    'Immediate (Dec 1)',
    'Delay (Dec 31)'
)

# 4. Sort lowest to highest to reveal the true CTD bond!
df_clean_ctd = df_clean_ctd.sort_values(by='Net_Basis')


# Display the final CTD and show exactly how the minimum was chosen
display(df_clean_ctd[['Clean Price','Spot_Dirty', 'Optimal_Delivery', 'Net_Basis_First', 'Net_Basis_Last', 'Net_Basis']].head(3))

```

---

# ***Step 6***

## ***Market Depth***

```py
# 1. Isolate our CTD candidates (make a clean copy of the top 3)
df_top5 = df_clean_ctd.head(3).copy()

# 2. Map out the future cash flows for these specific bonds
df_payoff_top5 = fq.create_payoff_df(df_top5, settlement)

# 3. Calculate the time to each cash flow in years (using Actual/365.25 convention)
mat_years = (pd.to_datetime(df_payoff_top5.columns) - pd.to_datetime(settlement)).days / 365.25

# 4. Calculate Metrics: YTM, Modified Duration, and Convexity
# We pass an initial YTM guess of 5% (0.05), transpose the cash flows, and pass the spot dirty prices
ytm, duration, convexity = fq.calc_bond_metrics_2d(
    0.05,
    df_payoff_top5.T.to_numpy(),
    mat_years,
    df_top5['Spot_Dirty'].to_numpy()
)

# 5. Assign the calculated metrics back to our top 5 DataFrame
df_top5[['YTM', 'Duration', 'Convexity']] = pd.DataFrame(
    {'YTM': ytm, 'Duration': duration, 'Convexity': convexity},
    index=df_top5.index
)

# Display the baseline risk metrics rounded to 4 decimal places
display(df_top5[['YTM', 'Duration', 'Convexity']].round(4))

```

