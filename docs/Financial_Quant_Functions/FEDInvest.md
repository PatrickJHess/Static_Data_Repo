## Read FEDInvest Data

```py
def FEDInvest(price_date, cache_dir=DEFAULT_CACHE_DIR):
    """
    Fetches historical security prices from local Parquet cache or GitHub Static Repo.

    Args:
        price_date (datetime.date): The date for which to retrieve prices.
        cache_dir (Path): Local directory to store/retrieve cached Parquet files.

    Returns:
        tuple: (pandas.DataFrame, datetime.date, datetime.date) 
               (df_filtered, price_date, settlement_date)
        tuple: (str, None, None) if the request fails or data is missing.
    """

    def make_date(date_value):
        if not isinstance(date_value, (datetime, date)):
            try:
                date_value = pd.Timestamp(date_value).date()
            except Exception as e:
                print(f"wrong type for date: {e}")
                return None
        else:
            try:
                date_value = date_value.date()
            except AttributeError:
                pass
        return date_value

    # Normalize Dates
    price_date = make_date(price_date)
    if not price_date:
        return "Invalid date format", None, None

    price_date = adjust_bond_pay_dates(price_date)[0]
    
    if price_date > date.today():
        return "price_date is in the future", None, None

    settlement_date = price_date + relativedelta(days=1)
    settlement_date = adjust_bond_pay_dates(settlement_date)

    # 3. Standardize Parquet Filename 
    filename = f"fedinvest_{price_date.strftime('%m_%d_%Y')}.parquet"
    
    # Ensure local cache folder exists
    cache_path = Path(cache_dir)
    cache_path.mkdir(parents=True, exist_ok=True)
    file_path = cache_path / filename

    df = None

    # 4. Tier 1: Check Local/Drive Cache First
    if file_path.exists():
        df = pd.read_parquet(file_path)
        
    # 5. Tier 2: Fetch from GitHub Static Repo if not in cache
    else:
        github_url = f"{GITHUB_RAW_BASE_URL}/{filename}"
        try:
            # Fetch raw binary data from GitHub
            response = requests.get(github_url, timeout=10)
            response.raise_for_status()
            
            # Read bytes directly into Pandas
            df = pd.read_parquet(BytesIO(response.content))
            
            # Save a copy to Google Drive/Local cache to bypass GitHub next time
            df.to_parquet(file_path, index=False)
            
        except requests.exceptions.HTTPError as e:
            if response.status_code == 404:
                return f"Data for {price_date} not found in GitHub Repo: {filename}", None, None
            return f"HTTP Error fetching from GitHub: {e}", None, None
        except Exception as e:
            return f"Error loading Parquet data: {e}", None, None

    # 6. Filter Data (Drop rows maturing on or before settlement date)
    if 'MATURITY DATE' in df.columns:
        df['MATURITY DATE'] = pd.to_datetime(df['MATURITY DATE'])
        df_filtered = df[df['MATURITY DATE'] > pd.to_datetime(settlement_date[0])]
    else:
        return "Schema Error: 'MATURITY DATE' column missing in Parquet file", None, None

    return df_filtered, price_date, settlement_date[0]
```
