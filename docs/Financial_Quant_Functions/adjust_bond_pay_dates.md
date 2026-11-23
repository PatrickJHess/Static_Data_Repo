## Adjust bond dates for actual dates


```py
def adjust_bond_pay_dates(dates,calendar='SIFMAUS'):
  """
  Adjusts bond payment dates to account for holidays and weekends.
  dates can be a scalar, pandas series, or numpy array (datetime.date,timestamp, or datetime64)
  """


  # Ensure dates is a DatetimeIndex
  if not pd.api.types.is_scalar(dates):
      dates = pd.DatetimeIndex(pd.to_datetime(dates))
  else:
      dates = pd.DatetimeIndex(pd.to_datetime([dates]))
      
  sifma = mcal.get_calendar(calendar)
  sifma_holidays = sifma.holidays().holidays
  good_fridays = GoodFriday.dates('2000-01-01', '2060-12-31')

  # Convert to DatetimeIndex and use .union() (which automatically deduplicates and sorts)
  sifma_idx = pd.DatetimeIndex(sifma_holidays)
  gf_idx = pd.DatetimeIndex(good_fridays)
  master_bond_holidays = sifma_idx.union(gf_idx)

  # Create a CustomBusinessDay offset using the combined holidays
  numpy_holidays = master_bond_holidays.values.astype('datetime64[D]')

  # Use NumPy for lightning-fast, fully vectorized date math (No warnings!)
  actual_payment_dates = np.busday_offset(
      dates.values.astype('datetime64[D]'),
      offsets=0,
      roll='forward',
      holidays=numpy_holidays
  )
  
  final_dates = pd.to_datetime(actual_payment_dates).date

  return final_dates
```
