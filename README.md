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

### Post-signup fields

The current pledge format covers the moment of signup. In practice, several pieces of state arrive shortly after signup and are commonly exchanged with downstream CRMs as part of the same pledge record:

- The signed pledge document (PDF, alongside the existing audio signature URL)
- Welcome email / SMS delivery status
- Donor feedback (rating + comment) from a follow-up survey
- A photo of the recruiter

Adding these as optional top-level fields keeps the pledge a single self-contained record rather than forcing consumers to stitch state from multiple sources.

#### `pledgeDocumentUrl`

URL to the signed pledge PDF (the signed paper form, signed digital agreement, etc.). Distinct from `signatureSignedUrl` which references the consent audio recording. Same expiry semantics as other signed URLs — emit `pledgeDocumentUrlExpiresAt` for file-based delivery.

#### `welcomeCommunication`

Delivery status of the welcome email and/or SMS sent shortly after signup. Helpful for downstream CRMs to detect undeliverable contact info and flag the donor for follow-up.

```json
"welcomeCommunication": {
  "email": { "status": "delivered", "updatedAt": "2017-07-22T08:14:11Z" },
  "sms":   { "status": "failed",    "updatedAt": "2017-07-22T08:14:11Z" }
}
```

`status` SHOULD be one of `pending`, `delivered`, `bounced`, `failed`, `deferred`, `opened`. Either `email` or `sms` MAY be omitted if the producer doesn't operate that channel.

#### `feedback`

Post-signup donor feedback from a follow-up survey or NPS-style prompt.

```json
"feedback": {
  "rating": 5,
  "comment": "Friendly fundraiser, clear explanation",
  "receivedAt": "2017-07-23T10:02:00Z"
}
```

`rating` is a 1–5 integer (NPS-friendly). All fields optional individually.

#### `recruiterUser.photoURL`

Optional photo of the fundraiser who took the pledge. Useful for identification on receipts, fraud-prevention follow-ups, and donor support. Same expiry semantics as other signed URLs if pre-signed.
