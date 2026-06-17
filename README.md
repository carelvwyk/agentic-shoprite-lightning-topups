# agentic-shoprite-lightning-topups

A Claude Code skill that lets an AI agent top up a Shoprite Money Market Account (MMA) with rands, paid via Bitcoin Lightning Network.

## What it does

The skill drives a two-step API flow against the MoneyBadger CryptoQR API:

1. **Creates a top-up order** for a given SA mobile number and rand amount.
2. **Fetches a BOLT11 Lightning invoice** for that order and presents it to the user, ready to pay from any Lightning wallet.

Funds reflect on the Shoprite MMA within a few minutes of payment confirmation.

## Inputs required

| Field | Description | Example |
|---|---|---|
| `phone_number` | SA mobile number linked to the Shoprite MMA | `+2771XXXXXXX` |
| `amount` | Rand amount to load (minimum R5) | `50` |

## Installation

Drop `SKILL.md` into your Code project skills directory. Your agent will auto-trigger on phrases like:

- "top up Shoprite"
- "load my Shoprite MMA"
- "fund Shoprite account with Lightning"
- any combination of a rand amount + SA phone number + Shoprite context

## API

Uses the [MoneyBadger](https://www.moneybadger.co.za) API.