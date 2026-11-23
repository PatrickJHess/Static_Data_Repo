## Use isda to get accrued

```py
def convert_isda(settlement, prev_coupon):


    # make sure settlement is date
    # if not pandas timestamp convert and make it a date
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

    settlement=make_date(settlement)
    prev_coupon=make_date(prev_coupon)


    if prev_coupon.year == settlement.year:
        days_in_year = 366 if calendar.isleap(settlement.year) else 365
        actual_days = (settlement - prev_coupon).days
        return actual_days / days_in_year

    # ISDA Split: Pivot on Jan 1st of the second year
    pivot_date = date(settlement.year, 1, 1)

    # All days from the coupon up to midnight Jan 1st
    days_prev = (pivot_date - prev_coupon).days
    days_prev_year = 366 if calendar.isleap(prev_coupon.year) else 365
    prev_ratio = days_prev / days_prev_year

    # All days from Jan 1st to the accrual end
    days_settlement = (settlement- pivot_date).days
    days_in_settlement_year = 366 if calendar.isleap(settlement.year) else 365
    settlement_ratio = days_settlement / days_in_settlement_year

    return prev_ratio + settlement_ratio
```
