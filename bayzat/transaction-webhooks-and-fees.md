# Transaction webhooks: types, fees, and reconciliation

This page explains the transaction webhooks you will receive, how fees are reported on each, and what `transactionReferenceUuid` points to.

---

## 1. The transaction types you will receive

| `transactionType`   | When it fires                                      | `billAmount`           | What the fee fields hold                            |
| ------------------- | -------------------------------------------------- | ---------------------- | --------------------------------------------------- |
| `Auth`              | Card is authorised at the merchant                  | Includes the fee       | Empty. Fee information is shared at settlement.      |
| `Reversal`          | An authorisation is released, fully or partly       | Includes the fee       | The share of the released amount that was fee, returned. |
| `Settlement`        | Funds actually move, usually 1–2 days after `Auth`  | **Fee already deducted** | The fee actually charged for the purchase.         |
| `Refund`            | Merchant returns money                              | Includes the fee, if the fee was refunded | The fee amount returned, as reported against the refund. |
| `Debit Adjustment`  | A correction that debits the card                   | Includes the fee       | The fee on the correction, if any.                   |
| `Credit Adjustment` | A correction that credits the card                  | Includes the fee       | The fee returned with the correction, if any.        |
| `Debit Surcharge`   | An extra amount charged after settlement            | Includes the fee       | The fee on the surcharge, if any.                    |

---

## 2. The lifecycle of one purchase

One purchase is several webhooks sharing a lifecycle. They arrive in order, hours or days apart.

### A purchase that simply settles

| Order | `transactionType` | `billAmount` | Total fee | `transactionReferenceUuid` |
| ----- | ----------------- | ------------ | --------- | -------------------------- |
| 1     | `Auth`            | `545.99`     | —         | `null`                     |
| 2     | `Settlement`      | `532.67`     | `13.32`   | the `Auth`'s `transactionUuid` |

The authorisation is gross, the settlement is net, and the difference is the fee (`fees` + `markupFee` + `vatCharges`).

### A purchase partly reversed before settling

| Order | `transactionType` | `billAmount` | Total fee | `transactionReferenceUuid` |
| ----- | ----------------- | ------------ | --------- | -------------------------- |
| 1     | `Auth`            | `12.13`      | —         | `null`                     |
| 2     | `Reversal`        | `11.27`      | `0.27`    | the `Auth`'s `transactionUuid` |
| 3     | `Settlement`      | `0.83`       | `0.03`    | the `Auth`'s `transactionUuid` |

### Lifecycles that do not complete

| Situation                                | What you see                                  | Treat as                         |
| ---------------------------------------- | --------------------------------------------- | -------------------------------- |
| Settlement has not arrived yet            | `Auth` only                                   | Pending. Do not reconcile yet.   |
| Authorisation fully released              | `Auth` + `Reversal` of the same `billAmount`  | Closed. Nothing was charged.     |
| Authorisation declined                    | No money movement                             | Exclude from balances.           |

---

## 3. The fee

### The fee is three fields, not one

A transaction's fee is the sum of all three:

```
total fee  =  fees  +  markupFee  +  vatCharges
```

| Field        | Holds                    |
| ------------ | ------------------------ |
| `fees`       | Fixed processing fee      |
| `markupFee`  | FX / scheme markup        |
| `vatCharges` | VAT charged on the above  |

Always add the three together. Any of them can be zero, and often is, but none of them should be assumed zero.

### Checking it against the amounts

For a purchase that authorised and settled once, the three fees together account for the difference between the two amounts:

| `transactionType` | `billAmount` | `fees` | `markupFee` | `vatCharges` |
| ----------------- | ------------ | ------ | ----------- | ------------ |
| `Auth`            | `545.99`     | —      | —           | —            |
| `Settlement`      | `532.67`     | `0.00` | `13.32`     | `0.00`       |

`545.99 − 532.67 = 13.32`, which is `0.00 + 13.32 + 0.00`.

If the authorisation was partly reversed first, the reversal returned part of the fee and the settlement kept the rest. The two add up to what was charged at authorisation:

| `transactionType` | `billAmount` | `fees` | `markupFee` | `vatCharges` |            |
| ----------------- | ------------ | ------ | ----------- | ------------ | ---------- |
| `Auth`            | `12.13`      | —      | —           | —            |            |
| `Reversal`        | `11.27`      | `0.00` | `0.27`      | `0.00`       | returned   |
| `Settlement`      | `0.83`       | `0.00` | `0.03`      | `0.00`       | charged    |

`0.27 + 0.03 = 0.30` was the fee at authorisation. The customer paid `0.03`.

On a full reversal the whole fee comes back, nothing settles, and nothing is charged:

| `transactionType` | `billAmount` | `fees` | `markupFee` | `vatCharges` |
| ----------------- | ------------ | ------ | ----------- | ------------ |
| `Auth`            | `376.49`     | —      | —           | —            |
| `Reversal`        | `376.49`     | `0.00` | `9.18`      | `0.00`       |

### Small differences are the conversion rate

When a purchase is made in another currency, the authorisation and the settlement are converted to the billing currency on different days, at different rates. The settled amount can differ from the authorised amount by a few fils as a result.

A `25.00 USD` purchase authorised on 12 October:

| Field                 | Value          |
| --------------------- | -------------- |
| Transaction amount    | `25.00 USD`    |
| `billAmount`          | `94.11 AED`    |
| `markupFee`           | `2.295`        |
| Amount excluding fee  | `91.81 AED`    |

Settling on 14 October, two outcomes are both normal:

| Settlement amount | Difference | Why                                   |
| ----------------- | ---------- | ------------------------------------- |
| `91.81 AED`       | `0.00`     | Same rate on both days.               |
| `91.85 AED`       | `0.04`     | The conversion rate moved in between. |

A small difference like this is the exchange rate, not a fee and not an error. The fee is the three fee fields, and it does not change between authorisation and settlement.

---

## 4. What `transactionReferenceUuid` links to

Every webhook has its own `transactionUuid`. `transactionReferenceUuid` points at
the `transactionUuid` of the transaction it belongs to — usually the
authorisation that started the lifecycle.

| `transactionType`   | `transactionReferenceUuid`                    |
| ------------------- | --------------------------------------------- |
| `Auth`              | `null` — the auth *is* the start of the chain  |
| `Reversal`          | the `Auth` it reverses                         |
| `Settlement`        | the `Auth` it settles                          |
| `Refund`            | the `Debit Adjustment` or `Auth` being refunded, or `null` if it stands alone |
| `Debit Adjustment`  | the `Auth` being adjusted, or `null` if it stands alone |
| `Debit Surcharge`   | the `Auth` being surcharged, or `null` if it stands alone |
| `Credit Adjustment` | the related `Auth`, or `null` if it stands alone |

### Legs with no parent

Some transactions are not part of an authorisation's lifecycle and carry `transactionReferenceUuid` as `null`:

| Case                                                   | Type               |
| ------------------------------------------------------ | ------------------ |
| A settlement arrived with no authorisation              | `Debit Adjustment` |
| A refund with no matching purchase, in rare cases       | `Refund`           |
| A surcharge whose authorisation cannot be identified    | `Debit Surcharge`  |

These are complete transactions in their own right. Include them in the balance as single-leg lifecycles.
