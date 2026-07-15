# Sample Test Payloads

Replace `YOUR_WEBHOOK_URL` with your n8n webhook URL before running.

---

## CRITICAL Risk Transaction

**Scenario:** Business owner's account — ₹2,50,000 transfer at 2 AM from Dubai
using an unknown device. Account normally transacts ₹8,000 from Bengaluru/Chennai.
Deviation: 31.3x average.

**Expected:** Rule Score 105, Risk Level CRITICAL, Telegram alert fired immediately.

```bash
curl -X POST YOUR_WEBHOOK_URL \
  -H "Content-Type: application/json" \
  -d '{
    "account_number": "ACC1001",
    "amount": 250000,
    "transaction_type": "transfer",
    "location": "Dubai",
    "device_id": "DEV-X9",
    "timestamp": "2026-07-15T02:15:00"
  }'
```

**Flags triggered:**
- Large amount (>₹2,00,000) → +30
- Transaction during odd hours (12AM-5AM) → +20
- Unknown location: Dubai → +15
- Unknown device: DEV-X9 → +15
- Extreme deviation: 31.3x account average → +25

---

## HIGH Risk Transaction

**Scenario:** Customer transfers ₹80,000 at 3:30 AM from Delhi.
Account normally transacts ₹15,000 from Mumbai only. Deviation: 5.3x average.

**Expected:** Rule Score 65, Risk Level HIGH, Telegram alert fired for review.

```bash
curl -X POST YOUR_WEBHOOK_URL \
  -H "Content-Type: application/json" \
  -d '{
    "account_number": "ACC1002",
    "amount": 80000,
    "transaction_type": "transfer",
    "location": "Delhi",
    "device_id": "DEV-B1",
    "timestamp": "2026-07-15T03:30:00"
  }'
```

**Flags triggered:**
- Moderate-large amount (₹50k-2L) → +15
- Transaction during odd hours (12AM-5AM) → +20
- Unknown location: Delhi → +15
- High deviation: 5.3x account average → +15

---

## MEDIUM Risk Transaction

**Scenario:** New account (25 days old) withdraws ₹12,000 from Delhi during
business hours using a known device. Account average is ₹3,000. Deviation: 4x.

**Expected:** Rule Score ~28 (LOW by rules), AI Risk Level MEDIUM — silent log only.

> Note: The rule engine scores this LOW as no single hard threshold is strongly
> breached. The Groq AI independently identifies the behavioral deviation and new
> account age as MEDIUM risk. This is intentional — it demonstrates how the two
> scoring layers complement each other, exactly as production AML systems operate.

```bash
curl -X POST YOUR_WEBHOOK_URL \
  -H "Content-Type: application/json" \
  -d '{
    "account_number": "ACC1003",
    "amount": 12000,
    "transaction_type": "withdrawal",
    "location": "Delhi",
    "device_id": "DEV-C1",
    "timestamp": "2026-07-15T10:00:00"
  }'
```

**Flags triggered:**
- New account (<30 days old) → +20
- Moderate deviation: 4x account average → +8

---

## LOW Risk Transaction

**Scenario:** Regular customer transfers ₹18,000 from Chennai at 11 AM using
their known device. Account average is ₹20,000. Deviation: 0.9x (below average).

**Expected:** Rule Score 0, Risk Level LOW — silent log only, no Telegram alert.

```bash
curl -X POST YOUR_WEBHOOK_URL \
  -H "Content-Type: application/json" \
  -d '{
    "account_number": "ACC1005",
    "amount": 18000,
    "transaction_type": "transfer",
    "location": "Chennai",
    "device_id": "DEV-A2",
    "timestamp": "2026-07-15T11:00:00"
  }'
```

**Flags triggered:** None — known location, known device, business hours,
amount within normal range.

---

## Individual JSON Files

If you prefer individual files instead of curl commands, create these in the
`sample_payloads/` folder:

**critical_transaction.json**
```json
{
  "account_number": "ACC1001",
  "amount": 250000,
  "transaction_type": "transfer",
  "location": "Dubai",
  "device_id": "DEV-X9",
  "timestamp": "2026-07-15T02:15:00"
}
```

**high_transaction.json**
```json
{
  "account_number": "ACC1002",
  "amount": 80000,
  "transaction_type": "transfer",
  "location": "Delhi",
  "device_id": "DEV-B1",
  "timestamp": "2026-07-15T03:30:00"
}
```

**medium_transaction.json**
```json
{
  "account_number": "ACC1003",
  "amount": 12000,
  "transaction_type": "withdrawal",
  "location": "Delhi",
  "device_id": "DEV-C1",
  "timestamp": "2026-07-15T10:00:00"
}
```

**low_transaction.json**
```json
{
  "account_number": "ACC1005",
  "amount": 18000,
  "transaction_type": "transfer",
  "location": "Chennai",
  "device_id": "DEV-A2",
  "timestamp": "2026-07-15T11:00:00"
}
```
