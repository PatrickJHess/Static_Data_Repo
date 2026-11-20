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

