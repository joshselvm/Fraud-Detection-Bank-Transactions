# Fraud Detection System — Bank Transactions
An automated fraud detection pipeline built with n8n, Groq AI, and Google Sheets 
that monitors bank transactions in real time, scores risk using rule-based logic 
and behavioral deviation analysis, and triggers tiered alerts via Telegram.

## Features
- Real-time transaction monitoring via webhook ingestion
- Rule-based scoring engine (amount thresholds, odd-hour detection, 
  location/device anomaly, new account flags)
- Behavioral deviation analysis — compares each transaction against 
  the account's own historical average (not fixed thresholds)
- Groq AI (LLaMA 3.1) layer for contextual fraud explanation and 
  recommended action
- Tiered alert system: CRITICAL → immediate Telegram alert, 
  HIGH → review alert, MEDIUM/LOW → silent audit log
- Full audit trail in Google Sheets with 14 data points per transaction

## Tech Stack
- n8n (self-hosted on Ubuntu VPS) — workflow automation
- Groq API (llama-3.1-8b-instant) — AI fraud analysis
- Google Sheets — account history lookup + transaction audit log
- Telegram Bot — real-time tiered alerts

## Architecture
Webhook → Account History Lookup (Google Sheets)
        → Rule Engine + Behavioral Deviation (Code Node)
        → Groq AI Scoring + Explanation (HTTP Request)
        → Response Parser (Code Node)
        → Transaction Log (Google Sheets)
        → IF CRITICAL → Telegram Alert
        → IF HIGH → Telegram Alert
        → MEDIUM/LOW → Silent Log

## Scoring Logic
### Rule-Based (0–100+)
| Signal | Condition | Points |
|--------|-----------|--------|
| Large amount | >₹2,00,000 | +30 |
| Moderate amount | ₹50k–₹2L | +15 |
| Odd hour | 12AM–5AM | +20 |
| New account | <30 days old | +20 |
| Unknown location | Not in known locations | +15 |
| Unknown device | Not in known devices | +15 |
| Extreme deviation | 10x+ account average | +25 |
| High deviation | 5–10x account average | +15 |
| Moderate deviation | 3–5x account average | +8 |

### Risk Classification
| Score | Level |
|-------|-------|
| 70+ | CRITICAL |
| 50–69 | HIGH |
| 30–49 | MEDIUM |
| <30 | LOW |

## Sample Alert Output
CRITICAL transaction flagged:
- Account: ACC1001
- Amount: ₹2,50,000 (31.3x account average of ₹8,000)
- Location: Dubai (known: Bengaluru, Chennai)
- Device: Unknown
- Time: 2:15 AM
- Rule Score: 105/100
- AI Score: 98/100
- AI Recommendation: Block transaction and freeze account pending investigation
