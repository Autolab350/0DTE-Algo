import time as sleep_time
import datetime

def scanner_loop(symbol='QQQ', volume_threshold=2.0, total_minutes=55):
    """
    Block 6: Main scanner loop
    Runs for 55 minutes, scans every 60 seconds
    Prevents Colab timeout with print heartbeat
    """
    print(f"SCANNER STARTED: {datetime.datetime.now().strftime('%Y-%m-%d %H:%M:%S')}")
    print(f"   Symbol: {symbol}")
    print(f"   Volume Threshold: {volume_threshold}x")
    print(f"   Duration: {total_minutes} minutes")
    print(f"   Paper Mode: {PAPER_TRADING}")
    print("=" * 60)

    for i in range(total_minutes):
        scan_num = i + 1
        timestamp = datetime.datetime.now().strftime('%H:%M:%S')

        try:
            print(f"\n[SCAN {scan_num}] {timestamp}")

            # Block 3: Check conditions
            block3_result = check_conditions(symbol, volume_threshold)

            # Block 4: Calculate setup
            block4_result = calculate_setup(block3_result)

            # Block 5: Submit order (if signal exists)
            block5_result = submit_order(block4_result, symbol)

            # Summary print
            if block5_result['status'] == 'paper':
                print(f"   ORDER PLACED: {block5_result['contracts']} contracts")
            elif block5_result['status'] == 'live':
                print(f"   LIVE ORDER: {block5_result['order_id']}")
            else:
                print(f"   No signal")

        except Exception as e:
            print(f"   X Scan {scan_num} Error: {e}")
            # Continue to next scan (don't crash entire loop)

        # Sleep 60 seconds before next scan (skip on last iteration)
        if i < total_minutes - 1:
            sleep_time.sleep(60)

    print("\n" + "=" * 60)
    print(f"SCANNER COMPLETED: {datetime.datetime.now().strftime('%Y-%m-%d %H:%M:%S')}")
    print(f"   Total Scans: {total_minutes}")

