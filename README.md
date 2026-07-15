# Fraud Detection System — Bank Transactions

A real-time automated fraud detection pipeline that monitors bank transactions,
scores risk using a hybrid rule-based and behavioral deviation engine, and triggers
tiered alerts via Telegram. Built with n8n, Groq AI, and Google Sheets.

---

## Features

- Real-time transaction monitoring via webhook ingestion
- Rule-based scoring engine (amount thresholds, odd-hour detection, location and device anomaly, new account flags)
- Behavioral deviation analysis — compares each transaction against the account's own historical average rather than fixed thresholds
- Groq AI (LLaMA 3.1) layer for contextual fraud explanation and recommended action
- Three-tier alert system: CRITICAL and HIGH trigger Telegram alerts, MEDIUM and LOW log silently
- Full 14-field audit trail in Google Sheets with color-coded risk levels

---

## Tech Stack

- **n8n** (self-hosted) — workflow automation engine
- **Groq API** (llama-3.1-8b-instant) — AI fraud analysis and recommendation
- **Google Sheets** — account history lookup and transaction audit log
- **Telegram Bot** — real-time tiered fraud alerts

---

## Architecture

```
Webhook (transaction data)
    ↓
Google Sheets — Account History Lookup
    ↓
Code Node — Account Match + Data Merge
    ↓
Code Node — Rule Engine + Behavioral Deviation Scoring
    ↓
Groq AI — Contextual Fraud Analysis + Recommendation
    ↓
Code Node — Response Parser
    ↓
Google Sheets — Transaction Audit Log
    ↓
IF CRITICAL → Telegram 🚨 Immediate Alert
IF HIGH     → Telegram ⚠️ Review Alert
MEDIUM/LOW  → Silent Log Only
```

---

## Scoring Logic

### Rule-Based Engine

| Signal | Condition | Points |
|--------|-----------|--------|
| Large amount | > ₹2,00,000 | +30 |
| Moderate amount | ₹50,000 – ₹2,00,000 | +15 |
| Odd hour | 12AM – 5AM | +20 |
| New account | < 30 days old | +20 |
| Unknown location | Not in account's known locations | +15 |
| Unknown device | Not in account's known devices | +15 |
| Extreme deviation | 10x+ account average | +25 |
| High deviation | 5x – 10x account average | +15 |
| Moderate deviation | 3x – 5x account average | +8 |

### Risk Classification

| Score | Risk Level | Action |
|-------|------------|--------|
| 70+ | CRITICAL | Immediate Telegram alert + freeze recommendation |
| 50 – 69 | HIGH | Telegram alert + flag for manual review |
| 30 – 49 | MEDIUM | Silent log only |
| < 30 | LOW | Silent log only |

### Why Two Scoring Layers?

The rule engine handles hard thresholds — fast, transparent, and auditable
(required for regulatory compliance). The Groq AI layer catches subtle patterns
the rules miss, such as a moderate amount that is still significantly higher than
an account's personal average. Together they mirror how production AML systems
like NICE Actimize and SAS AML operate: deterministic rules + machine learning scoring.

---

## Google Sheets Setup

### Step 1 — Create the workbook

Create a Google Sheet named:
```
Fraud Detection - Bank Transactions
```

### Step 2 — Account History tab

Create a tab called `Account History` with these headers and seed data:

| account_number | avg_transaction_amount | known_locations | known_devices | account_age_days |
|----------------|----------------------|-----------------|---------------|------------------|
| ACC1001 | 8000 | Bengaluru, Chennai | DEV-A1, DEV-A2 | 450 |
| ACC1002 | 15000 | Mumbai | DEV-B1 | 200 |
| ACC1003 | 3000 | Delhi | DEV-C1 | 25 |
| ACC1004 | 25000 | Kerala | DEV-C2 | 180 |
| ACC1005 | 20000 | Chennai | DEV-A2 | 250 |

### Step 3 — Transaction Log tab

Create a tab called `Transaction Log` with these headers:

```
timestamp | account_number | amount | transaction_type | location | device_id |
deviation_multiple | rule_score | risk_level | ai_score | ai_risk_level |
flags | explanation | recommendation
```

Apply conditional formatting to `risk_level` and `ai_risk_level` columns:
- **CRITICAL** → Red background (#FF0000), white text
- **HIGH** → Orange background (#FF6600), white text
- **MEDIUM** → Yellow background (#FFD700), black text
- **LOW** → Green background (#00AA00), white text

---

## n8n Setup

### Prerequisites

- n8n self-hosted instance (or n8n cloud)
- Groq API key (free at [console.groq.com](https://console.groq.com))
- Google Service Account with Sheets API enabled
- Telegram Bot token and Chat ID

### Environment Variables

```
N8N_HOST=your-domain.com
WEBHOOK_URL=https://your-domain.com
N8N_PROTOCOL=https
```

### Workflow Nodes

1. **Webhook** — POST, receives transaction JSON
2. **Google Sheets (Get Rows)** — reads all Account History rows
3. **Code Node 1** — matches account number, merges transaction + history data
4. **Code Node 2** — rule engine + behavioral deviation scoring
5. **HTTP Request** — Groq AI scoring (POST to api.groq.com)
6. **Code Node 3** — parses Groq JSON response
7. **Google Sheets (Append Row)** — logs to Transaction Log
8. **IF Node 1** — routes CRITICAL to Telegram
9. **IF Node 2** — routes HIGH to Telegram
10. **Telegram** — sends formatted alert (CRITICAL path)
11. **Telegram** — sends formatted alert (HIGH path)

---

## Testing

Replace `YOUR_WEBHOOK_URL` with your n8n webhook URL before running.

### CRITICAL Risk

Large amount, unknown location, unknown device, odd hour, extreme deviation (31.3x average)

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

Expected: Rule Score 105, Risk Level CRITICAL, Telegram alert fired

---

### HIGH Risk

Odd hour, unknown location, high deviation (5.3x average)

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

Expected: Rule Score 65, Risk Level HIGH, Telegram alert fired

---

### MEDIUM Risk

Moderate deviation, known location and device, business hours

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

Expected: Rule Score ~28 (LOW by rules), AI Risk Level MEDIUM — silent log only

> Note: The rule engine scores this LOW as no hard thresholds are breached.
> The AI independently identifies the 4x behavioral deviation as MEDIUM risk.
> This is intentional — it demonstrates how the two scoring layers complement
> each other, mirroring real AML systems where rules and AI provide separate signals.

---

### LOW Risk

Amount close to average, known location, known device, business hours

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

Expected: Rule Score 0, Risk Level LOW — silent log only

---

## Sample Alert Output

### CRITICAL Telegram Alert

```
🚨 FRAUD ALERT — CRITICAL
━━━━━━━━━━━━━━━━━━━━

👤 Account: ACC1001
💰 Amount: ₹250000
🔄 Type: transfer
📍 Location: Dubai
📱 Device: DEV-X9
🕐 Time: 2026-06-29T02:15:00

📊 Rule Score: 105/100
🤖 AI Score: 98/100
⚠️ Risk Level: HIGH
📈 Deviation: 31.3x average

🚩 Flags:
Large amount (>₹2,00,000); Transaction during odd hours (12AM-5AM);
Unknown location: Dubai; Unknown device: DEV-X9;
Extreme deviation: 31.3x account's average

💡 Explanation:
This transaction exhibits multiple high-risk indicators including large amount,
odd-hour activity, unknown location and device, and extreme deviation from average.

✅ Recommendation:
Block the transaction and freeze the account pending investigation.
```

### HIGH Risk Telegram Alert

```
⚠️ FRAUD ALERT — HIGH RISK
━━━━━━━━━━━━━━━━━━━━

👤 Account: ACC1002
💰 Amount: ₹80000
🔄 Type: transfer
📍 Location: Delhi
📱 Device: DEV-B1
🕐 Time: 2026-07-15T03:30:00

📊 Rule Score: 65/100
🤖 AI Score: 85/100
📈 Deviation: 5.3x average

🚩 Flags:
Moderate-large amount (₹50k-2L); Transaction during odd hours (12AM-5AM);
Unknown location: Delhi; High deviation: 5.3x account's average

💡 Explanation:
This transaction exceeds the account's average amount by a significant margin
and occurred during off-hours from an unknown location.

✅ Recommendation:
Flag the transaction for manual review and consider freezing the account.
```

---

## License

MIT
