# RiskLab policy specification

## Status and purpose

This document defines baseline policy `v1` for synthetic RiskLab applications. The rules are invented for a programming exercise. In particular, the minimum age of 18 is a project choice rather than a statement about applicable law.

Once a saved decision refers to `v1`, the definition below is immutable. Any behavioral change must use a new policy version.

## Inputs used by the policy

Policy `v1` receives already validated and normalized applicant features:

- `age`;
- `monthly_income`;
- `existing_monthly_payments`;
- `proposed_monthly_payment`;
- `missed_payments_last_year`;
- `total_monthly_payments`;
- `payment_to_income_ratio`, or `null` when income is zero.

Amounts and thresholds use exact decimal arithmetic. No monetary value passes through binary floating-point arithmetic.

`total_monthly_payments` is:

```text
existing_monthly_payments + proposed_monthly_payment
```

For positive income, the displayed ratio is derived from:

```text
total_monthly_payments / monthly_income
```

The decision rules compare the exact payment amounts with income multiplied by the applicable decimal threshold. They do not make decisions from a rounded display ratio.

## Rules

Evaluate every applicable rule in this stable order:

| Order | Rule | Matching condition | Finding | Reason code |
|---:|---|---|---|---|
| 1 | Minimum age | `age < 18` | `DECLINE` | `BELOW_MINIMUM_AGE` |
| 2 | Income | `monthly_income == 0` | `DECLINE` | `NO_INCOME` |
| 3 | Payment history | `missed_payments_last_year >= 2` | `DECLINE` | `TOO_MANY_MISSED_PAYMENTS` |
| 4 | High payment burden | `total_monthly_payments > monthly_income * Decimal("0.50")` | `DECLINE` | `PAYMENT_BURDEN_TOO_HIGH` |
| 5 | Intermediate payment burden | `total_monthly_payments > monthly_income * Decimal("0.40")` and `total_monthly_payments <= monthly_income * Decimal("0.50")` | `REVIEW` | `PAYMENT_BURDEN_REQUIRES_REVIEW` |

The high and intermediate burden conditions are mutually exclusive. When monthly income is zero, skip both burden rules, set `payment_to_income_ratio` to `null`, and never divide by zero.

## Combining rule findings

All applicable rules are evaluated. Collect every matching reason code in the stable rule order above. This includes a review reason when another rule also produces a decline finding.

The final decision uses this explicit precedence:

```text
DECLINE > REVIEW > APPROVE
```

Therefore:

- one or more `DECLINE` findings produce `DECLINE`;
- otherwise, one or more `REVIEW` findings produce `REVIEW`;
- no findings produce `APPROVE` with an empty reason-code list.

## Boundary behavior

When no other rule matches:

| Payment burden | Result |
|---|---|
| Exactly 40% of income | `APPROVE` |
| Greater than 40% and less than or equal to 50% | `REVIEW` |
| Exactly 50% of income | `REVIEW` |
| Greater than 50% of income | `DECLINE` |

With income `1000.00`, this means `400.00` is approved, `400.01` and `500.00` are reviewed, and `500.01` is declined, provided no age, income, or payment-history rule also matches.
