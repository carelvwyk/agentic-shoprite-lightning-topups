---
name: agentic-shoprite-lightning-topups
description: >
  Load a Shoprite Money Market Account (MMA) with rands using a Bitcoin Lightning Wallet.
  Use this skill whenever the user wants to top up, load, fund, or recharge a Shoprite
  Money Market account, mentions a phone number and a rand amount in the context of
  Shoprite, or asks to pay via Bitcoin or Lightning for a Shoprite wallet load. Triggers on phrases
  like "top up Shoprite", "load Shoprite MMA", "fund Shoprite account", "Shoprite Lightning
  payment", or any combination of a rand amount + SA phone number + Shoprite context.
---

# Shoprite Lightning Top-Up Skill

Loads a Shoprite Money Market Account (MMA) with a ZAR amount, paid via Bitcoin
Lightning Network, using the MoneyBadger CryptoQR API.

## Required Inputs

| Field | Description | Example |
|---|---|---|
| `phone_number` | SA mobile number linked to the Shoprite MMA | `+277XXXXXXXX` |
| `amount` | Rand amount to load (string, min R5, max R5000) | `"50"` |

---

## Workflow

### Step 1 — Create the top-up order

```
POST https://api.cryptoqr.net/shopritetopups/v1/topup
Headers:
  content-type: application/json
Body:
  {
    "phone_number": "<phone_number>",
    "amount": "<amount>",        ← string, rand amount e.g. "10"
    "device_id": "<phone_number>"  ← use the phone number as device_id
  }
```

**Success response — capture these fields:**

| Field | Use |
|---|---|
| `id` | The invoice/transaction ID — used directly in Step 2 |
| `amount_cents` | Confirm the amount is correct |
| `expires_at` | Show user — they must pay before this time |
| `status` | Should be `"REQUESTED"` |

---

### Step 2 — Get the Lightning invoice

Use the `id` from Step 1 in **both** the URL path and the request body `transaction_id`.

```
POST https://api.cryptoqr.net/api/v2/invoices/<id>/payment_methods
Headers:
  content-type: application/json
  x-merchant-code: shopritetopups
Body:
  {
    "payment_method": "lightning",
    "transaction_id": "<id>",    ← same id from Step 1
    "payment_currencies": []
  }
```

**Success response — capture these fields:**

| Field | Use |
|---|---|
| `payment_request.data` | BOLT11 Lightning invoice string (`lnbc...`) to pay |
| `payment_request.amount` | Amount in sats |
| `payment_request.exchange_rate` | ZAR/BTC rate used |
| `expires_at` | Payment deadline (same as Step 1) |
| `status` | Should be `"REQUESTED"` |

---

### Step 3 — Present to user

Display clearly:
1. **Summary**: phone number and rand amount being loaded
2. **Lightning invoice** — the `lnbc...` string in a code block so it's easy to copy
3. **Amount in sats** from `payment_request.amount`. Display the sats amount or divide by 1e8 to show BTC if you prefer.
4. **Expiry time** — how long they have to pay
5. **Exchange rate** used (e.g. "1 BTC = R{exchange_rate}")
6. Ask the user to pay and confirm when done, or say "check" to poll for payment status

---

### Step 4 — Confirm payment (on user request)

When the user says they've paid or asks you to check, poll the invoice status using long-polling:

```
GET https://api.cryptoqr.net/api/v2/invoices/<id>?wait=10
Headers:
  x-merchant-code: shopritetopups
```

The `wait=10` parameter holds the connection open for up to 10 seconds until the status changes. Poll up to 6 times (≈1 minute total) before giving up.

**Invoice status meanings:**

| Status | Meaning | Action |
|---|---|---|
| `REQUESTED` | Not yet paid | Poll again |
| `AUTHORIZED` | Payment detected, pending confirmation | Poll again — confirmation is imminent |
| `CONFIRMED` | Payment confirmed | Inform user — top-up is on its way |
| `TIMED_OUT` | Invoice expired unpaid | Offer to restart from Step 1 |
| `CANCELLED` | Invoice was cancelled | Offer to restart from Step 1 |
| `ERROR` | Something went wrong | Report error and offer to retry |

On `CONFIRMED`, show:
```
✅ Payment confirmed! R{amount} is being loaded to {phone_number}.
Funds typically reflect within a few seconds.
```

---

## Error Handling

| Scenario | Action |
|---|---|
| HTTP 400 | Bad request — check phone number format and that amount is a string |
| `expires_at` already passed | Re-run from Step 1 to get a fresh order |
| Empty `payment_methods.lightning` in response | Lightning unavailable — inform user |
| 6 polls with no `CONFIRMED` | Tell user payment not detected yet; offer to keep checking or restart |

---

## Example Output to User

```
✅ Shoprite MMA Top-Up Ready

  Account:  +277XXXXXXXX
  Amount:   R5.00
  Sats:     488 sats
  Rate:     1 BTC = R1 041 637
  Expires:  05:20 UTC (~1 minute remaining — pay now!)

⚡ Lightning invoice:

lnbc4880n1p4ze6y2pp57073rn90p3ypf0mrwnzeffl5s97l06w338x788w4s9w45920k2sq...

Scan or paste into your Lightning wallet. Let me know when you've paid and
I'll confirm the payment landed.
```
