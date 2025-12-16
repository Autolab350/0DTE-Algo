import numpy as np

def calculate_setup(block3_result, atr_period=14):
    """
    Block 4: Calculate entry/exit targets and position size
    Returns dict with entry, tp, sl, atr, contracts, iv_multiplier
    """
    try:
        bars = block3_result['bars']
        direction = block3_result['direction']
        iv_percentile = block3_result['iv_percentile']

        # Get current price (entry)
        entry = float(bars['close'].iloc[-1])

        # 1. Calculate ATR (14-period)
        high = bars['high'].iloc[-atr_period:]
        low = bars['low'].iloc[-atr_period:]
        close = bars['close'].iloc[-atr_period:]

        tr1 = high - low
        tr2 = abs(high - close.shift(1))
        tr3 = abs(low - close.shift(1))
        true_range = np.maximum(tr1, np.maximum(tr2, tr3))
        atr = float(true_range.mean())

        # 2. Calculate TP/SL (Phase 1: Linear)
        if direction == 'call':
            tp = entry + (2.0 * atr)
            sl = entry - (1.0 * atr)
        elif direction == 'put':
            tp = entry - (2.0 * atr)
            sl = entry + (1.0 * atr)
        else:
            # No direction, return zeros
            return {
                'entry': entry,
                'tp': 0.0,
                'sl': 0.0,
                'atr': atr,
                'contracts': 0,
                'iv_multiplier': 0.0
            }

        # 3. IV Scaling Multiplier
        if iv_percentile < 0.30:
            iv_multiplier = 0.8
        elif iv_percentile <= 0.50:
            iv_multiplier = 1.0
        else:
            iv_multiplier = 1.5

        # 4. Position Size (1% risk, IV-scaled)
        account_equity = float(account.portfolio_value)
        base_contracts = int((account_equity * 0.01) / (atr * 100))
        contracts = max(1, int(base_contracts * iv_multiplier))

        # Print output
        print(f"ATR: ${atr:.2f} | TP: ${tp:.2f} | SL: ${sl:.2f}")
        print(f"Contracts: {contracts} (IV mult: {iv_multiplier}x)")

        return {
            'entry': float(entry),
            'tp': float(tp),
            'sl': float(sl),
            'atr': float(atr),
            'contracts': int(contracts),
            'iv_multiplier': float(iv_multiplier)
        }

    except Exception as e:
        print(f"✗ Block 4 Error: {e}")
        return {
            'entry': 0.0,
            'tp': 0.0,
            'sl': 0.0,
            'atr': 0.0,
            'contracts': 0,
            'iv_multiplier': 0.0
        }

'''
 direction (no trade signal)
'''