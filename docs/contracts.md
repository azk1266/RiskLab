# RiskLab contracts

## Scope

This document defines the initial external contract for submitting a synthetic application and retrieving its saved decision. Unknown fields and implicit type coercion are rejected so that malformed or misspelled data is visible to callers.

## Application request

A valid request to `POST /decisions` is a JSON object with exactly these fields:

```json
{
  "application_id": "demo-001",
  "age": 28,
  "monthly_income": "2500.00",
  "existing_monthly_payments": "600.00",
  "proposed_monthly_payment": "450.00",
  "missed_payments_last_year": 0
}
```

### Field rules

| Field | Required contract |
|---|---|
| `application_id` | JSON string containing 1–100 Unicode characters. Leading or trailing whitespace and `U+0000` are rejected. |
| `age` | JSON integer from 0 through 120. Booleans, numeric strings, and fractional values are rejected. |
| `monthly_income` | Money string as defined below. |
| `existing_monthly_payments` | Money string as defined below. |
| `proposed_monthly_payment` | Money string as defined below. |
| `missed_payments_last_year` | JSON integer greater than or equal to zero. Booleans and numeric strings are rejected. |
| Any unknown field | Rejected. |

`U+0000` is rejected because the later PostgreSQL `text` and `jsonb` representations cannot store it. Internal whitespace in `application_id` is allowed; only leading and trailing whitespace are prohibited.

### Money strings

Each money field must:

- be a JSON string rather than a JSON number;
- contain one or more ASCII digits, optionally followed by a decimal point and one or two ASCII digits;
- represent a value from `0` through `9999999999.99`, inclusive;
- contain no sign, whitespace, thousands separator, exponent, `NaN`, or infinity.

The accepted syntax is equivalent to:

```text
^[0-9]+(?:\.[0-9]{1,2})?$
```

After validation, normalize the value to exactly two decimal places. Examples include `"0" -> "0.00"`, `"12.5" -> "12.50"`, and `"2500.00" -> "2500.00"`. Reject unsupported precision rather than rounding it.

Zero income is valid input. It produces a business decision of `DECLINE` with reason `NO_INCOME`; it is not a validation failure. Missing or `null` income is invalid and must never be substituted with zero.

## Successful decision data

A successful result exposes the saved baseline policy outcome and its explanation. The response contains at least:

- the application identifier;
- `APPROVE`, `REVIEW`, or `DECLINE`;
- the ordered reason-code list;
- externally visible derived features, including a string ratio or `null` for zero income;
- the baseline policy version.

For the valid example above, policy `v1` produces:

```json
{
  "application_id": "demo-001",
  "decision": "REVIEW",
  "reason_codes": [
    "PAYMENT_BURDEN_REQUIRES_REVIEW"
  ],
  "features": {
    "payment_to_income_ratio": "0.42"
  },
  "policy_version": "v1"
}
```

Phase 3 will finalize additional response metadata such as the service-generated decision identifier. It must not change the input validation or business outcomes specified here without updating this contract and its tests.

## HTTP outcomes

| Situation | Status | Meaning |
|---|---:|---|
| A new decision is saved | `201 Created` | The response contains the saved baseline decision. |
| A decision is retrieved | `200 OK` | The response contains the existing saved decision. |
| An identical normalized application is submitted again under the same `application_id` | `200 OK` | Return the original saved decision without reevaluation or another logical event. |
| An existing `application_id` is submitted with different normalized input | `409 Conflict` | The identifier cannot be reused for another application. |
| Request data violates this contract | `422 Unprocessable Content` | No decision is created. |
| A requested decision does not exist | `404 Not Found` | No saved decision matches the requested identifier. |
| PostgreSQL is unavailable | `503 Service Unavailable` | The service cannot reliably save or retrieve the decision. |

A `DECLINE` outcome is a successful business result and therefore uses the normal success status. It is not an HTTP error.

Two submissions are identical when all fields are equal after the documented normalization. For example, `"2500"` and `"2500.00"` normalize to the same income.

## Invalid examples

Each `POST /decisions` example below returns `422 Unprocessable Content` and creates no decision:

| Invalid fragment or change | Reason |
|---|---|
| Omit `monthly_income` | A required value is missing; it must not become zero. |
| `"monthly_income": null` | Null is not a money string. |
| `"monthly_income": 2500` | JSON numbers are not accepted for money. |
| `"monthly_income": "-1.00"` | Negative amounts are outside the contract. |
| `"monthly_income": "1.001"` | More than two decimal places are rejected, not rounded. |
| `"monthly_income": "1e3"` | Scientific notation is rejected. |
| `"monthly_income": " 2500.00"` | Whitespace is rejected. |
| `"age": "28"` | Numeric strings are not accepted for integer fields. |
| `"age": true` | A JSON boolean is not an integer for this contract. |
| `"age": 28.5` | Fractional ages are rejected. |
| `"missed_payments_last_year": -1` | The count cannot be negative. |
| `"application_id": " demo-001"` | Leading whitespace is rejected. |
| `"application_id": "demo\u0000-001"` | `U+0000` cannot be persisted by the planned storage representation. |
| Add `"monthly_incomme": "2500.00"` | Unknown fields are rejected so misspellings are visible. |

## Consistency with policy `v1`

Validation occurs before policy evaluation. Age zero through 17 is valid input and is declined by the minimum-age rule; ages outside 0 through 120 are invalid input. Zero income is valid and declined; missing income is invalid. The detailed decision boundaries and stable reason order are defined in [policy.md](policy.md).
