## Scheduled pay dates of bonds (not actual)

```py
def scheduled_pay_dates(last_date,settlement=None,freq=2):
  '''
    Generates a chronological list of coupon payment dates from settlement to last_date.
    The function calculates dates backward from the last_date date based on the
    specified frequency. It handles standard bond market "end-of-month" logic:
    if the last_date date is the last day of a month, all preceding coupon payments
    are snapped to the last day of their respective months.
    Args:
        last_date (datetime.date,timestamp, or datetime64): The last date checked.
            Accepts a date object.
        settlement (datetime.date,timestamp, or datetime64): The settlement date (start of analysis).
            Coupons falling before this date are excluded. Defaults to date.today().
        freq (int, optional): The number of coupon payments per year.
            Accepted values:
            * 1: Annual, 2: Semi-Annual (Default), 4: Quarterly, 12: Monthly

    Returns:
        list[datetime.date]: A list of coupon dates sorted chronologically
        (earliest to latest), ending with the last_date date..
  '''



# convert settlement and last_date to datetime.date
  def make_date(date_value):
    if not isinstance(date_value,(datetime,date)):
      date_value=pd.Timestamp(date_value).date()
      return date_value
    # convert timestamps and datetimes to date
    else:
      try:
          return date_value.date()
      except:        
         return date_value


  #Validate the data- last_date, coupon, settlement, freq
  #last_date
  last_date=make_date(last_date)

  #settlement
  if settlement is None:
      settlement = date.today()
  else:
      settlement=make_date(settlement)
  #freq
  if int(freq) not in [1,2,4,12]:
      display(md(f"### ⚠️  your assigned freq {freq} it must be (1, 2, 4, or 12)\
     \n     semi-annual assumed (2)."))
      freq=int(2)

  # check last_date greater than settlement
  if last_date<=settlement:
    raise ValueError("last_date must be greater than the settlement date")

  # calculate the number of months between each coupon payment.
  num_months=int(12/freq)
 
  # timestamps so hat we can use pandas offset
  last_day=pd.Timestamp(last_date)
  settlement=pd.Timestamp(settlement)
  # raw_dates is a pandas datetime index that starts and goes 60 times
  # need to check for month_end and respect it (MonthsEnd)
  if last_day.is_month_end:
    raw_dates = pd.DatetimeIndex([last_day - pd.offsets.MonthEnd(num_months * i) for i in range(360)])
  else:
    raw_dates = pd.DatetimeIndex([last_day - pd.DateOffset(months=num_months * i) for i in range(360)])

  # only include dates greater than settlement date
  valid_dates=raw_dates[raw_dates>settlement]

  # convert timestamps back to date and make the order chronological with splice
  valid_dates=pd.to_datetime(valid_dates).date

  return valid_dates[::-1]

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
