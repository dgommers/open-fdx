# Open FDX

**Open Fundraising Data Exchange — a working group creating open standards for data exchange in the fundraising ecosystem.**

First and foremost, this document just outlines the underlying idea and background of this idea. It's far from ready or finished.

---

## Background

NGOs work with multiple systems — CRM, payment processor, email platform, fundraising suite — and moving data between them means CSV exports, copy-paste, or fragile point-to-point integrations. The result is manual work, inconsistent records, and vendor lock-in. Vendors waste engineering time on one-off connectors instead of their core product.

## Vision

Open FDX is a working group of fundraising vendors, CRM providers, and NGOs that publishes open and vendor-neutral standards to exchange donor/pledge data. This working group may create several standard over time; each can be adopted independently. 

The idea is that associated vendors implement the Open FDX standards in their systems and offer this to their customers. As the initiative and it's standard evolves vendors will adopt them in X period to ensure systems can keep operating. 

The format is easy to understand and adapt, but also leaves room for extension and customization. 

## First use case: New Pledge

The first standard covers the **new pledge** — the moment a donor commits to a gift and that commitment needs to pass reliably between systems. This standard can be used to export it from vendors to CRM partners as new pledges. 

- A JSON or CSV representation of a pledge at creation: amount, currency, frequency, start date, payment method, fund, and donor profile.
- Files are easy to read and indicate the Open FDX specification to allow for compatibility checks and multiple standards in the future.
- A lightweight exchange protocol with acknowledgement and error handling.
- Validation rules and a reference implementation for testing compliance.

This use case may be covered by just a file format or extend with an exchange protocol as well. For example, we could all support a set of endpoints available at 'Open FDX'-based endpoints. Example: `tapraise.com/api/open-fdx/import` or `tapraise.com/api/open-fdx/export`. We propose to just start with a format first.

A specification that is both CSV/JSON compatible might be nice so it can be adopted in different platforms, but let's keep that up for discussion.

## Inspiration and examples

Example of TapRaise current API format in [JSON](/examples/tapraise-pledge.json) and [CSV](/examples/tapraise-pledge.csv) based on their existing [API](https://developer.tapraise.com).

Here is another example from 2021 with a XLSX file specifying a CSV export with similar fields: [CSV spec 2021.xlsx](/examples/CSV%20spec%202021.xlsx)

Existing concepts such as [JSON Schema](https://json-schema.org/) could be adapted to simplify implementation and validation of standards:
```json
{  
    "$schema": "https://json-schema.org/draft/2020-12/schema",  
    "$id": "https://openfdx.org/specs/pledges",  
    "nodes": [....]
}
```

On CSV level you may specificy the schema per row with a first column called `Schema` or do something [like this](https://pypi.org/project/csv-schema/).

## Conventions

### `payment` object

Today the spec scatters payment-related data across three locations:

- **Top-level**: `initialDonationAmount`, `donationAmount`, `initialDonationInterval`, `donationInterval`, `preferredPaymentDay`
- **`person`**: `iban`
- **Implicit / missing**: payment method (SEPA / card / Stripe / PayPal), BIC, bank name, alternate account holder, processor subscription ID

Mixing donor identity (`person`) with payment instrument (IBAN, account holder) creates two problems:

1. **Conflated semantics.** A donor's IBAN is not part of their identity — it's part of their pledge's payment instruction. The same donor can have multiple pledges with different IBANs (gift to one charity from joint account, gift to another from personal account).
2. **Real-world fields are missing.** SEPA pledges need BIC + bank name. Stripe-backed pledges need a subscription ID. Today producers stuff these into `externalMetadata`.

Introduce a `payment` object that consolidates donation amount/interval, payment instrument, and processor reference:

```json
"payment": {
  "method": "sepa",
  "iban": "NL13ABNA6371362585",
  "bic": "ABNANL2A",
  "bankName": "ABN AMRO",
  "accountHolder": "Beth Cormier",
  "preferredPaymentDay": 13,
  "initialDonationAmount": 7.5,
  "donationAmount": 7.5,
  "initialDonationInterval": "monthly",
  "donationInterval": "monthly",
  "providerSubscriptionId": "sub_1NjHk2XqL..."
}
```

### Field semantics

- **`method`** (string): `sepa`, `card`, `stripe`, `paypal`, `bank_transfer`, `other`. Producers SHOULD document which methods they emit.
- **`iban`**: moved from `person.iban`. The IBAN is a payment instrument, not donor identity.
- **`bic`**, **`bankName`**: SEPA-mainstream fields, useful even outside DACH/NL.
- **`accountHolder`**: used when the payer differs from the donor (third-party gifts, joint accounts).
- **`preferredPaymentDay`**: moved from top-level (1–28 day-of-month for direct debit cycles).
- **`initialDonationAmount`** / **`donationAmount`** / **`initialDonationInterval`** / **`donationInterval`**: moved from top-level. Same semantics.
- **`providerSubscriptionId`**: opaque reference to the producer's payment processor (Stripe subscription ID, PayPal billing agreement ID, SEPA mandate reference, etc.). Lets receiving CRMs reconcile against the processor without producer-specific keys.

### Backwards-compatible migration

This is the most invasive change in the proposal stack — it moves existing top-level and `person` fields. To avoid a hard break:

- Producers MAY emit both the new `payment` object AND the legacy flat fields during a transition period
- When both are present, consumers MUST prefer values inside `payment`
- A future minor version will mark the legacy flat fields deprecated
- A future MAJOR version (declared via `fdxVersion: "2.0.0"` per the versioning convention) will remove the legacy fields

Producers may begin emitting `payment` immediately without breaking any current consumer.
