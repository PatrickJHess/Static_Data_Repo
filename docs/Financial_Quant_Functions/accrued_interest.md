## Calculate accrued interest

```py
def accrued_interest(maturity, coupon, day_type='Actual/Actual', settlement=None, freq=2):
    """
      Returns the accrued interest for a bond.  Returns the accrued interest for a bond.

       Returns the accrued interest for a bond.  Returns the accrued interest for a bond.

    Args:
        maturity (datetime.date,timestamp, or datetime64): The maturity date of the bond.
        coupon (float): The annual coupon rate (e.g., 0.05 for 5%).
        day_types:
            Actual/Actual, Actual/365, Actual/360. and 30/360.
        settlement ((datetime.date,timestamp, or datetime64), optional):
         The settlement date. Defaults to today.
        freq (int, optional):
        Coupon frequency per year: 1 (annual). 2 (semi-annual)
                                    4 (quarterly), 12 (monthly)
                                    Defaults to 2.
    """

    #Validate Data
    def validate_date(datetime_object):
      # check for datetime or date
      if not isinstance(datetime_object, (datetime, date)):
          raise TypeError("Input must be a datetime or date object.")
      # convert datetime to date
      if isinstance(datetime_object, datetime):
        datetime_object = datetime_object.date()
      return datetime_object
    maturity = validate_date(maturity)

    if settlement is None:
        settlement = date.today()
    else:
        settlement = validate_date(settlement)

    if freq not in [1, 2, 4, 12]:
        print(f"⚠️ Warning: Freq {freq} invalid. Assumed Semi-Annual (2).")
        freq = 2

    try:
        coupon = float(coupon)
        if coupon < 0: raise ValueError
    except:
        raise ValueError("Coupon must be a positive number.")

    # Define strategic_date
    # Get all the bond's potential payment dates in the next year
    mat_is_last=maturity.day==calendar.monthrange(maturity.year,maturity.month)[1]
    if mat_is_last:
      lastDay=calendar.monthrange(settlement.year+1,maturity.month)[1]
      strategic_date=date(settlement.year+1,maturity.month,lastDay)
    else:
     strategic_date=date(settlement.year+1,maturity.month,maturity.day)

    #The strategic date: minimum of actual and next year's maturity date
    strategic_date=min(maturity,strategic_date)
    pay_dates = scheduled_pay_dates(strategic_date, settlement=settlement, freq=freq)

    # Should sorted but check
    pay_dates.sort()

    # The first date after the settlement date is the next coupon date
    next_coupon = None
    for d in pay_dates:
        if d >= settlement:
            next_coupon = d
            break

    # Bond has matured or annual coupon is zero
    if next_coupon is None or coupon==0:
        return 0.0

    #Calculate Previous Coupon Date
    num_months = int(12 // freq)
    prev_coupon = next_coupon - relativedelta(months=num_months)

    # Check for Month End adjustment on the calculated previous date
    is_next_month_end = next_coupon.day == calendar.monthrange(maturity.year, maturity.month)[1]

    if is_next_month_end:
        last_day_of_prev_month = calendar.monthrange(prev_coupon.year, prev_coupon.month)[1]
        prev_coupon = date(prev_coupon.year, prev_coupon.month, last_day_of_prev_month)

    # The day a coupon is paid is also the first day of the new cycle.

    accrued_value = 0.0

    if day_type == 'Actual/Actual':
        days_since_last = (settlement - prev_coupon).days
        days_between = (next_coupon - prev_coupon).days
        #  (DaysHeld / DaysInPeriod)
        accural_ratio= days_since_last/days_between
        # Actual/Actual uses the coupon paid on the date
        accrued_value = (coupon / freq) * accural_ratio


    elif day_type == '30/360':
        accural_ratio =_30_360_(settlement,prev_coupon)
        # Formula: Coupon * accrural ratio
        accrued_value = coupon * accural_ratio

    elif day_type == 'Actual/360':
        days_since_last = (settlement - prev_coupon).days
        accural_ratio= days_since_last/360
      # Formula: Coupon * accrural ratio        
        accrued_value = coupon * accural_ratio

    elif day_type == 'Actual/365':
        accural_ratio = convert_isda(settlement,prev_coupon)
      # Formula: Coupon * accrural ratio        
        accrued_value = coupon * accural_ratio

    else:
        # Fallback
        print(f"⚠️ Warning: Unknown day_type {day_type}. Using Actual/Actual.")
        days_since_last = (settlement - prev_coupon).days
        days_between = (next_coupon - prev_coupon).days
        #  (DaysHeld / DaysInPeriod)
        accural_ratio= days_since_last/days_between
        # Actual/Actual uses the coupon paid on the date
        accrued_value = (coupon / freq) * (days_since_last / days_between)

    return accrued_value

```
