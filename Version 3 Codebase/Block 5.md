import datetime
from alpaca.trading.requests import MarketOrderRequest
from alpaca.trading.enums import OrderSide, TimeInForce

# PAPER/LIVE TOGGLE (set at top of notebook)
PAPER_TRADING = True

def submit_order(block4_result, symbol='QQQ'):
    """
    Block 5: Submit OCO bracket order
    Returns dict with status, order_id, timestamp, contracts
    """
    try:
        contracts = block4_result['contracts']
        entry = block4_result['entry']
        tp = block4_result['tp']
        sl = block4_result['sl']
        direction = 'call' if tp > entry else 'put' if tp < entry else None

        # Skip if no contracts (no signal)
        if contracts == 0 or direction is None:
            print("X No signal - skipping order")
            return {
                'status': 'skipped',
                'order_id': None,
                'timestamp': datetime.datetime.now(),
                'contracts': 0
            }

        # Paper mode - print order, don't submit
        if PAPER_TRADING:
            order_id = f"PAPER_{datetime.datetime.now().strftime('%H%M%S')}"
            print(f"PAPER ORDER:")
            print(f"   Symbol: {symbol}")
            print(f"   Direction: {direction.upper()}")
            print(f"   Contracts: {contracts}")
            print(f"   Entry: ${entry:.2f}")
            print(f"   TP: ${tp:.2f} | SL: ${sl:.2f}")
            print(f"   Order ID: {order_id}")

            return {
                'status': 'paper',
                'order_id': order_id,
                'timestamp': datetime.datetime.now(),
                'contracts': contracts
            }

        # Live mode - submit actual OCO bracket order
        else:
            # Build market order with OCO bracket
            order_side = OrderSide.BUY if direction == 'call' else OrderSide.SELL

            order_data = MarketOrderRequest(
                symbol=symbol,
                qty=contracts,
                side=order_side,
                time_in_force=TimeInForce.DAY,
                order_class='bracket',
                take_profit={'limit_price': tp},
                stop_loss={'stop_price': sl}
            )

            order = trading_client.submit_order(order_data)

            print(f"LIVE ORDER SUBMITTED:")
            print(f"   Order ID: {order.id}")
            print(f"   Symbol: {symbol} | {direction.upper()}")
            print(f"   Contracts: {contracts}")
            print(f"   Entry: ${entry:.2f} | TP: ${tp:.2f} | SL: ${sl:.2f}")

            return {
                'status': 'live',
                'order_id': str(order.id),
                'timestamp': datetime.datetime.now(),
                'contracts': contracts
            }

    except Exception as e:
        print(f"X Block 5 Error: {e}")
        return {
            'status': 'error',
            'order_id': None,
            'timestamp': datetime.datetime.now(),
            'contracts': 0
        }