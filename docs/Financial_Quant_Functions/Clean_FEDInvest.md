## Standarize FEDInvest Data

```py
def clean_FEDInvest(df):


    import pandas as pd
    # Filters for Standard Securities
    keep_rows=df['SECURITY TYPE'].str.contains('bill|note|bond',case=False)
    security_df=df[keep_rows].copy()
 
    # Removes Clutter
    drop_columns=['CUSIP','CALL DATE']
    security_df.drop(columns=drop_columns,inplace=True)


    # Creates a Time-Series Index
    security_df.set_index('MATURITY DATE',inplace=True)
    security_df.index=pd.to_datetime(security_df.index)
    security_df.sort_index(inplace=True)


    # Standardizes Financial Terms
    change_column_names={'RATE':'Coupon',
                         'BUY':'Price Ask',
                         'SELL':'Price Bid'}
    security_df.rename(columns=change_column_names,inplace=True)
    security_df['Coupon']=security_df['Coupon']*100


    # Formats Numeric Data
    numeric_cols = ['Coupon', 'Price Ask', 'Price Bid', 'YIELD']
    for col in numeric_cols:
        if col in security_df.columns:
            security_df[col] = security_df[col].astype(str).str.replace('%', '', regex=False).astype(float)


    return security_df
```

