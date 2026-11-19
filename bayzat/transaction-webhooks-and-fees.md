# Transaction webhooks: types, fees, and reconciliation

This page explains the transaction webhooks you will receive, what each type means, and how fees are reported.

---

## 1. The transaction types you will receive

| `transactionType`   | What it means                                                                                                    | Net impact    | `transactionReferenceUuid`                  |
| ------------------- | ---------------------------------------------------------------------------------------------------------------- | ------------- | ------------------------------------------- |
| `Auth`              | Purchase authorised at the merchant. Pending until it settles.                                                   | Debit         | `null`                                      |
| `Reversal`          | An authorisation released, fully or partly                                                                       | Credit        | the `Auth`                                  |
| `Settlement`        | Confirms the final amount of the `Auth`, usually 1–2 days after it. Carries the fee.                             | **No change** | the `Auth`                                  |
| `Refund`            | Merchant returns money                                                                                           | Credit        | the `Auth` or `Debit Adjustment`, or `null` |
| `Debit Adjustment`  | A debit correction. Also used for a settlement that has no authorisation (section 2).                            | Debit         | the `Auth`, or `null`                       |
| `Credit Adjustment` | A credit correction                                                                                              | Credit        | the `Auth`, or `null`                       |
| `Debit Surcharge`   | A debit posted by the processor's operations team, most often because the purchase settled higher than authorised | Debit         | the `Auth`, or `null`                       |
| `Credit Surcharge`  | A credit posted by the processor's operations team, for example because the purchase settled lower than authorised | Credit        | the `Auth`, or `null`                       |

Every webhook has its own `transactionUuid`. `transactionReferenceUuid` holds the `transactionUuid` of the transaction it belongs to.

### What `billAmount` and the fee fields hold

| `transactionType`   | `billAmount`                               | What the fee fields hold                                  |
| ------------------- | ------------------------------------------ | --------------------------------------------------------- |
| `Auth`              | Includes the fee                           | Empty. Fee information is shared at settlement.           |
| `Reversal`          | Includes the fee                           | The share of the released amount that was fee, returned.  |
| `Settlement`        | **Fee already deducted**                   | The fee actually charged for the purchase.                |
| `Refund`            | Includes the fee, if the fee was refunded  | The fee amount returned, as reported against the refund.  |
| `Debit Adjustment`  | Includes the fee                           | The fee on the correction, if any.                        |
| `Credit Adjustment` | Includes the fee                           | The fee returned with the correction, if any.             |
| `Debit Surcharge`   | Includes the fee                           | The fee on the surcharge, if any.                         |
| `Credit Surcharge`  | Includes the fee                           | The fee returned with the surcharge, if any.              |

The fee is always `fees + markupFee + vatCharges` (section 4).

Refunds, adjustments and surcharges are already settled when you receive them. No `Settlement` will follow them; only an `Auth` is followed by a `Settlement`.

**Adjustments and surcharges can arrive after a purchase has settled.** A purchase you have already marked as settled can still receive a `Debit Adjustment`, `Credit Adjustment`, `Debit Surcharge` or `Credit Surcharge` that references its `Auth`, sometimes days or weeks later. No further `Settlement` follows it.

---

## 2. Settlements with no authorisation

Sometimes the card network sends a settlement with no matching authorisation (a "No Auth" settlement). It is posted as a `Debit Adjustment` with `transactionReferenceUuid` set to `null`, and is final. If the merchant later refunds it, the `Refund` references this `Debit Adjustment`.

---

## 3. Examples

### A purchase that settles much higher, corrected by surcharges after settlement

A `0.10 SGD` purchase was authorised, but the merchant settled it at `3.10 SGD`. The four webhooks after the `Auth` all reference it.

| Date   | `transactionType`  | Transaction amount | `billAmount` (AED) | Total fee | Net impact | Running total |
| ------ | ------------------ | ------------------ | ------------------ | --------- | ---------- | ------------- |
| 08 Oct | `Auth`             | `0.10 SGD`         | `0.30`             | —         | `−0.30`    | `−0.30`       |
| 15 Oct | `Settlement`       | `3.10 SGD`         | `8.90`             | `0.01`    | no change  | `−0.30`       |
| 21 Oct | `Debit Surcharge`  | `86.10 AED`        | `86.10`            | —         | `−86.10`   | `−86.40`      |
| 27 Oct | `Credit Surcharge` | `86.10 AED`        | `86.10`            | —         | `+86.10`   | `−0.30`       |
| 27 Oct | `Debit Surcharge`  | `8.61 AED`         | `8.61`             | —         | `−8.61`    | `−8.91`       |

- The settled cost is `8.90 + 0.01 = 8.91`. Only `0.30` was taken at `Auth`, so `8.61` was still owed.
- The processor's operations team first posted a `Debit Surcharge` of `86.10` in error, reversed it with a `Credit Surcharge` of `86.10`, then posted the correct `Debit Surcharge` of `8.61`.
- All three surcharges arrived after the `Settlement`, up to 12 days later. `0.30 + 8.61 = 8.91`, the settled cost.

A `Credit Surcharge` works the same way when a purchase settles lower than authorised: it returns the difference.

### A purchase that settles

| Order | `transactionType` | `billAmount` | `fees` | `markupFee` | `vatCharges` | Net impact |
| ----- | ----------------- | ------------ | ------ | ----------- | ------------ | ---------- |
| 1     | `Auth`            | `545.99`     | —      | —           | —            | `−545.99`  |
| 2     | `Settlement`      | `532.67`     | `0.00` | `13.32`     | `0.00`       | no change  |

The authorisation is gross, the settlement is net, and the difference is the fee: `545.99 − 532.67 = 13.32`, which is `0.00 + 13.32 + 0.00`.

### A purchase partly reversed before settling

| Order | `transactionType` | `billAmount` | `fees` | `markupFee` | `vatCharges` | Net impact |
| ----- | ----------------- | ------------ | ------ | ----------- | ------------ | ---------- |
| 1     | `Auth`            | `12.13`      | —      | —           | —            | `−12.13`   |
| 2     | `Reversal`        | `11.27`      | `0.00` | `0.27`      | `0.00`       | `+11.27`   |
| 3     | `Settlement`      | `0.83`       | `0.00` | `0.03`      | `0.00`       | no change  |

The reversal returned part of the fee and the settlement kept the rest. `0.27 + 0.03 = 0.30` was the fee at authorisation; the customer paid `0.03`. Net amount charged `12.13 − 11.27 = 0.86`, which equals `0.83 + 0.03`.

### A purchase fully reversed

| Order | `transactionType` | `billAmount` | `fees` | `markupFee` | `vatCharges` | Net impact |
| ----- | ----------------- | ------------ | ------ | ----------- | ------------ | ---------- |
| 1     | `Auth`            | `376.49`     | —      | —           | —            | `−376.49`  |
| 2     | `Reversal`        | `376.49`     | `0.00` | `9.18`      | `0.00`       | `+376.49`  |

The whole fee comes back, nothing settles, and nothing is charged.

### A purchase refunded after settling

_Illustrative amounts._

| Order | `transactionType` | `billAmount`         | Net impact |
| ----- | ----------------- | -------------------- | ---------- |
| 1     | `Auth`            | `100.00`             | `−100.00`  |
| 2     | `Settlement`      | `98.00` (fee `2.00`) | no change  |
| 3     | `Refund`          | `100.00`             | `+100.00`  |

### A settlement with no authorisation

_Illustrative amounts._

| Order           | `transactionType`  | `billAmount` | `transactionReferenceUuid` | Net impact |
| --------------- | ------------------ | ------------ | -------------------------- | ---------- |
| 1               | `Debit Adjustment` | `58.40`      | `null`                     | `−58.40`   |
| 2 (if refunded) | `Refund`           | `58.40`      | the `Debit Adjustment`     | `+58.40`   |

### Purchases that do not complete

| Situation                       | What you see                                  | Treat as                      |
| ------------------------------- | --------------------------------------------- | ----------------------------- |
| Settlement has not arrived yet  | `Auth` only                                   | Pending. Do not reconcile yet. |
| Authorisation fully released    | `Auth` + `Reversal` of the same `billAmount`  | Closed. Nothing was charged.  |
| Authorisation declined          | `Auth` with `transactionStatus` `Decline`     | Nothing was charged.          |

---

## 4. The fee

### The fee is three fields, not one

A transaction's fee is the sum of all three:

```
total fee  =  fees  +  markupFee  +  vatCharges
```

| Field        | Holds                    |
| ------------ | ------------------------ |
| `fees`       | Fixed processing fee     |
| `markupFee`  | FX / scheme markup       |
| `vatCharges` | VAT charged on the above |

Always add the three together. Any of them can be zero, and often is, but none of them should be assumed zero.

### Small differences are the conversion rate

When a purchase is made in another currency, the authorisation and the settlement are converted to the billing currency on different days, at different rates. The settled amount can differ from the authorised amount by a few fils as a result. When the processor corrects it, you receive a surcharge, even for a difference as small as `0.01`. Larger differences, for example when a merchant settles for more than it authorised, are corrected the same way (see the first example in section 3).

A `25.00 USD` purchase authorised on 12 October:

| Field                | Value       |
| -------------------- | ----------- |
| Transaction amount   | `25.00 USD` |
| `billAmount`         | `94.11 AED` |
| `markupFee`          | `2.295`     |
| Amount excluding fee | `91.81 AED` |

Settling on 14 October, two outcomes are both normal:

| Settlement amount | Difference | Why                                   |
| ----------------- | ---------- | ------------------------------------- |
| `91.81 AED`       | `0.00`     | Same rate on both days.               |
| `91.85 AED`       | `0.04`     | The conversion rate moved in between. |

The fee is the three fee fields. For the same amount, it does not change between authorisation and settlement.
