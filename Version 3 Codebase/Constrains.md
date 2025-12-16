# CONSTRAINTS & CRITERIA
## Focus: Get Code Running in Google Colab → Placing Trades on Alpaca

---

## CORE PRINCIPLES

### 1. Gamma Dealer Hedging (The Foundation)
The entire 0DTE ecosystem runs on dealer gamma hedging:
```
Retail buys 0DTE options
  ↓
Dealers are SHORT gamma (must delta hedge)
  ↓
Dealers BUY/SELL ES futures to stay neutral
  ↓
This creates VOLUME SPIKES (observable)
  ↓
Price moves 0.5-2% in 10-30 min (predictable duration)
  ↓
We ride the wave, exit before reversal
```

**Why this matters:**
- Traditional TA (EMA, RSI, MACD) doesn't predict dealer flows
- Dealer hedging is MECHANICAL (follows Greek math, not emotion)
- Volume spike = dealers hedging RIGHT NOW (real-time signal)
- We're not predicting, we're REACTING to forced hedging

### 2. Dynamic ATR Regression (Phase 2 Enhancement)
Linear ATR exits miss money. Regression adapts to gamma conditions:

**Phase 1 (Current - Linear):**
```
TP = Entry + (2.0 × ATR)  # Fixed multiplier
SL = Entry - (1.0 × ATR)  # Fixed multiplier
```

**Phase 2 (After 50 trades - Dynamic):**
```python
# Adapt based on observed gamma/volatility
if atr < 0.8:      # Low vol day
    tp_mult, sl_mult = 2.5, 0.8   # Tighter targets
elif atr > 1.5:    # High vol day
    tp_mult, sl_mult = 1.5, 1.2   # Wider targets
else:              # Normal
    tp_mult, sl_mult = 2.0, 1.0

TP = Entry + (tp_mult × ATR)
SL = Entry - (sl_mult × ATR)
```

**Impact:** +40-60% P&L improvement once calibrated with real trade data

---

## PART A: CODE STRUCTURE (Minimalist)

### 3. Block Size
- ✗ No blocks > 15 lines
- ✓ Each block: 5-15 lines MAX
- ✓ Each block testable ALONE
- ✓ Each block PRINTS output

### 4. The 5 Blocks
```
Block 1: Connect to Alpaca (5 lines)
Block 2: Check conditions (10 lines)
Block 3: Calculate setup (8 lines)
Block 4: Submit trade (10 lines)
Block 5: Scanner loop (15 lines)
```

### 5. Testing Order
```
1. Block 1 alone → prints balance? ✓ Next
2. Block 2 alone → prints time/volume? ✓ Next
3. Block 3 alone → prints ATR/TP/SL? ✓ Next
4. Block 4 alone → prints order ID? ✓ Next
5. Block 5 → runs loop with [SCAN #]? ✓ Done
```
✗ DON'T combine until each works alone

---

## PART B: ENTRY LOGIC (Gamma-Based)

### 6. Time Gate (When Dealers Hedge)
- ✓ 9:35-10:30 AM ET (opening gamma rehedge)
- ✓ 3:00-3:55 PM ET (0DTE expiration unwind)
- ✗ Outside windows = no trades

### 7. Volume Gate (Confirm Hedging Active)
- ✓ Current volume ≥ 2x average (20-bar lookback)
- ✗ Volume < 2x = no trade (hedging not confirmed)

### 7.5. IV Percentile Gate (Volatility Filter)
- ✓ Current IV ≥ 30th percentile (volatility must be present)
- ✗ IV < 30th percentile = no trade (not enough gamma)
- Fetch from: Alpaca OptionChain API (native)

### 8. Direction (Price Action, Not Time)
```python
price_change = current_price - bars['close'].iloc[-5]

if volume_spike:
    if price_change > 0:
        direction = "call"   # Dealers buying → ride up
    else:
        direction = "put"    # Dealers selling → ride down
else:
    direction = None         # No spike = no trade
```

---

## PART C: EXIT LOGIC (ATR-Based)

### 9. Phase 1 Exits (Linear - Use Now)
```
Take Profit: Entry + (2.0 × ATR)
Stop Loss: Entry - (1.0 × ATR)
Risk:Reward = 1:2 (need 33% win rate to profit)
```

### 10. Phase 2 Exits (Regression - After 50 Trades)
```
Analyze trade data → find optimal multipliers per ATR range
Low ATR days: Tighter TP (2.5x), tighter SL (0.8x)
High ATR days: Wider TP (1.5x), wider SL (1.2x)
```

### 11. Time Exits
- Max hold: 20 minutes (hedging wave duration)
- Hard stop: 3:55 PM (close before EOD)

### 12. Order Type
- ✓ OCO bracket (entry + TP + SL in one order)
- Alpaca handles exits automatically

---

## PART D: POSITION SIZING

### 13. Risk Per Trade (Base 1%, IV-Scaled)
```
Base risk: 1% of account
Size = (Account × 0.01) / (ATR × 100)

IV Scaling Multiplier:
  If IV < 30%:  contracts × 0.8  (low vol, smaller size)
  If IV 30-50%: contracts × 1.0  (normal vol, baseline)
  If IV > 50%:  contracts × 1.5  (high vol, larger size)

Final contracts = base_size × iv_multiplier
Min: 1 contract
```

---

## PART E: COLAB EXECUTION (Critical - Prevents Known Issues)

### 14. Problem: "Green checkmark, nothing executes"
**Root cause:** No visibility into decisions
**Solution:** Print at EVERY decision point
```python
print(f"Time: {now} [{'IN' if in_window else 'OUT'}]")
print(f"Volume: {vol} / Avg: {avg} = {vol/avg:.1f}x [{'✓' if spike else '✗'}]")
print(f"Direction: {direction}")
```

### 15. Problem: "Colab timed out"
**Root cause:** Session dies after 30 min inactivity
**Solution:** 
```python
# Option A: Split into 2 runs (55 min each)
for i in range(55):  # AM run: 9:35-10:30
    # scan
    time.sleep(60)

# Option B: Heartbeat every 10 scans
if i % 10 == 0:
    print(f"[SCAN {i}] ✓ Still running...")
```

### 16. Problem: "Volume never spikes"
**Root cause:** 2x threshold too high for testing
**Solution:**
```python
DEBUG = True
THRESHOLD = 1.5 if DEBUG else 2.0  # Lower for testing
```

### 17. Loop Structure (Prevents Timeout)
```python
for i in range(55):  # 55 min max
    try:
        # scan logic
        print(f"[SCAN {i+1}] {datetime.now().strftime('%H:%M:%S')}")
    except Exception as e:
        print(f"✗ Error: {e}")
        continue  # Don't crash, keep going
    time.sleep(60)
```

✗ NO: `while True` (causes timeout)
✓ YES: `for i in range(55)`

---

## PART F: ERROR HANDLING (Prevents Crashes)

### 18. Every Block Pattern
```python
try:
    # logic
    print("✓ [what it did]")
except Exception as e:
    print(f"✗ [what failed]: {e}")
```

### 19. Handle Gracefully
- Market closed → print, skip scan
- No data → print, skip scan
- API error → print, continue to next scan
- ✗ NEVER crash the whole scanner

---

## PART G: DEBUGGING (Visibility)

### 20. Print Everything
```
[SCAN 1 - 9:37:23 AM]
Time: 9:37 AM [IN WINDOW] ✓
Volume: 1.2M / Avg: 0.6M = 2.0x [THRESHOLD MET] ✓
Price change (5 min): +$0.15 [UP] → Direction: CALL
ATR: $1.25 | TP: $352.50 | SL: $348.75
✓ SIGNAL - Submitting order...
```

### 21. Debug Mode
```python
DEBUG = True  # False in production

if DEBUG:
    print(f"vol={vol}, avg={avg}, ratio={vol/avg:.2f}")
    print(f"threshold={'MET' if vol > avg * THRESHOLD else 'NOT MET'}")
```

---

## PART H: PRE-RUN CHECKLIST

Before running in Colab:
```
☐ API_KEY set (not "PASTE_HERE")
☐ API_SECRET set
☐ paper=True
☐ Block 1 prints balance
☐ Block 2 prints time/volume
☐ Block 3 prints ATR/TP/SL
☐ Market open right now?
☐ In trading window (9:35-10:30 or 3:00-3:55)?
```

---

## PART I: IMPLEMENTATION ROADMAP

### Week 1: Deploy
- Run Blocks 1-5 in Colab (paper)
- Target: 5+ trades/day
- Goal: System runs without crashing

### Week 2-3: Collect Data
- Log every trade (entry, exit, P&L, ATR, volume)
- Target: 50+ trades
- Store in CSV

### Week 4: Analyze
- Calculate win rate (need ≥ 35%)
- Find patterns in ATR vs profit
- Implement regression if beneficial

## PART K: BLOCK OUTPUT CONTRACTS

### Block 3 Output (Check Conditions)
```python
{
  'in_window': bool,           # Time in 9:35-10:30 or 3:00-3:55 AM ET?
  'vol_spike': bool,           # Volume ≥ 2x average?
  'direction': str,            # 'call', 'put', or None
  'bars': DataFrame,           # Last 20 bars (OHLCV)
  'avg_volume': float,         # 20-bar average volume
  'current_volume': float,     # Latest bar volume
  'iv_percentile': float       # 0-1 scale (0.35+ triggers trade)
}
```

### Block 4 Output (Calculate Setup)
```python
{
  'entry': float,              # Current price (entry point)
  'tp': float,                 # Take profit target
  'sl': float,                 # Stop loss target
  'atr': float,                # Average true range (14-bar)
  'contracts': int,            # Position size (1+ contracts)
  'iv_multiplier': float       # 0.8, 1.0, or 1.5 (from IV percentile)
}
```

### Block 5 Output (Submit Order)
```python
{
  'status': str,               # 'paper', 'live', or 'error'
  'order_id': str,             # Order ID or 'PAPER_SIM_###'
  'timestamp': datetime,       # When order submitted
  'contracts': int             # Contracts in order
}
```

### Block 6 Output (Scanner Loop)
```
Prints every 60 seconds:
[SCAN 1] 09:37:23 - Time: IN | Volume: 2.1x | Direction: CALL | ATR: $1.25 | TP: $352.50 | SL: $348.75 | ✓ SUBMITTED
[SCAN 2] 09:38:23 - Time: IN | Volume: 1.8x | Direction: NONE | Status: SKIPPED
[SCAN 3] 09:39:23 - ...
```

---

## PART J: FINAL GOAL

✓ Runs in Colab without timeout
✓ Prints [SCAN #] every minute (visible progress)
✓ Detects volume spikes (gamma hedging signal)
✓ Submits OCO brackets (mechanical exits)
✓ Stops gracefully after 55 minutes
✓ You can SEE what it's doing at all times
✓ Phase 2: Regression adapts to market conditions
✓ Runs in Colab without timeout
✓ Prints [SCAN #] every minute (visible progress)
✓ Detects volume spikes (gamma hedging signal)
✓ Submits OCO brackets (mechanical exits)
✓ Stops gracefully after 55 minutes
✓ You can SEE what it's doing at all times
✓ Phase 2: Regression adapts to market conditions
