'''
Block 3: Check Conditions

Purpose: Fetch market data, check time window, volume spike, and direction
Input: Symbol (QQQ), global `data_client` from GlobalFile 2
Output: Dict with {in_window, vol_spike, direction, bars, avg_volume, current_volume, iv_percentile}
'''

# Code

# Moved inside function to prevent scoping issues: import datetime
from alpaca.data.requests import StockBarsRequest
from alpaca.data.timeframe import TimeFrame
# Removed: from alpaca.options.requests import OptionChainRequest # New import for options chain requests
import pytz

def check_conditions(symbol='QQQ', volume_threshold=2.0):
    """
    Block 3: Check entry conditions
    Returns dict with time gate, volume gate, direction, IV percentile
    """
    import datetime # Ensure fresh import to avoid name collisions
    try:
        # 1. Get current current time (ET)
        et_tz = pytz.timezone('America/New_York')
        now = datetime.datetime.now(et_tz)
        current_time = now.time()

        # Time gates: 9:35-10:30 AM or 3:00-3:55 PM ET
        morning_window = (datetime.time(9, 35) <= current_time <= datetime.time(10, 30))
        afternoon_window = (datetime.time(15, 0) <= current_time <= datetime.time(15, 55))
        in_window = morning_window or afternoon_window

        # 2. Fetch last 20 bars (1-minute)
        request = StockBarsRequest(
            symbol_or_symbols=symbol,
            timeframe=TimeFrame.Minute,
            limit=20
        )
        bars = data_client.get_stock_bars(request).df
        current_stock_price = float(bars['close'].iloc[-1])

        # 3. Volume spike detection
        avg_volume = float(bars['volume'].iloc[:-1].mean())  # Last 19 bars
        current_volume = float(bars['volume'].iloc[-1])
        vol_spike = bool(current_volume >= (avg_volume * volume_threshold))

        # 4. Direction detection (5-bar price change)
        price_change = bars['close'].iloc[-1] - bars['close'].iloc[-5]
        if vol_spike:
            direction = 'call' if price_change > 0 else 'put'
        else:
            direction = None

        # 5. IV Percentile (Fetch live IV and use it as a proxy for percentile)
        iv_percentile = 0.50 # Default to mock value in case of API error or no options
        try:
            today = datetime.date.today()

            # NEW WAY: Use trading_client to get option chains
            # trading_client.get_option_chains returns a dict of OptionChain objects, keyed by symbol.
            option_chains_dict = trading_client.get_option_chains(
                symbol=symbol,
                expiration_date=today.strftime('%Y-%m-%d') # Alpaca expects 'YYYY-MM-DD' string
            )

            option_chain = option_chains_dict.get(symbol) # Get the OptionChain object for our symbol

            if option_chain and option_chain.option_contracts: # Check if chain and contracts exist
                option_contracts = option_chain.option_contracts # List of OptionContract objects

                # Find ATM options - simplified: find strike closest to current stock price
                atm_ivs = []
                closest_strike_diff = float('inf')
                atm_strike = None

                for contract in option_contracts:
                    if abs(contract.strike_price - current_stock_price) < closest_strike_diff:
                        closest_strike_diff = abs(contract.strike_price - current_stock_price)
                        atm_strike = contract.strike_price

                if atm_strike is not None:
                    for contract in option_contracts:
                        if contract.strike_price == atm_strike and contract.implied_volatility is not None:
                            atm_ivs.append(contract.implied_volatility)

                if atm_ivs:
                    iv_percentile = sum(atm_ivs) / len(atm_ivs) # Use average IV of ATM options
                    # Note: This is raw IV, not a percentile. For a true percentile,
                    # historical IV data would be needed which is a more complex task.
                    # We're using raw IV here as a direct replacement for the mock percentile.

        except Exception as e_iv:
            print(f"Warning: Could not fetch live IV for {symbol}: {e_iv}. Using mock value.")

        # Print output
        print(f"Time: {now.strftime('%H:%M:%S')} [{'IN' if in_window else 'OUT'}]")
        print(f"Volume: {current_volume:,.0f} / Avg: {avg_volume:,.0f} = {current_volume/avg_volume:.1f}x [{'✓' if vol_spike else '✗'}]")
        print(f"Direction: {direction if direction else 'NONE'}")
        print(f"IV Percentile: {iv_percentile:.2%}") # Format as percentage

        return {
            'in_window': bool(in_window),
            'vol_spike': vol_spike,
            'direction': direction,
            'bars': bars,
            'avg_volume': avg_volume,
            'current_volume': current_volume,
            'iv_percentile': float(iv_percentile)
        }

    except Exception as e:
        print(f"✗ Block 3 Error: {e}")
        return {
            'in_window': False,
            'vol_spike': False,
            'direction': None,
            'bars': None,
            'avg_volume': 0,
            'current_volume': 0,
            'iv_percentile': 0
        }
        