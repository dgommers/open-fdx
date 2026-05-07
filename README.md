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

### Structured `recruitingLocation`

The current `recruitingLocationLabel` is a free-text string ("Utrecht"). Modern fundraising platforms capture much richer location data:

- Coordinates (lat/lng) for proximity reporting
- A stable location identifier (often a geohash) for deduplication and cross-pledge analytics
- Venue type — public street vs. private booth vs. event venue (regulatory + operational difference)
- An organization-assigned location reference (the recruiting org's own catalog ID)

Adding a structured `recruitingLocation` object lets producers carry this data through to consumers without flattening to a label string.

```json
"recruitingLocation": {
  "locationId": "u173zm",
  "coordinates": { "lat": 52.0907, "lng": 5.1214 },
  "type": "booth",
  "venueType": "public",
  "organizationLocationId": "loc-utrecht-centraal-2025",
  "label": "Utrecht Centraal"
}
```

Field semantics:

- **`locationId`** (string): producer-stable identifier. Often a geohash (which doubles as both ID and coarse location). Producers SHOULD make this stable across pledges from the same physical location.
- **`coordinates`** (`{lat, lng}`, WGS84): point coordinates of the recruitment site.
- **`type`** (string): one of `street`, `booth`, `event`, `door`, `other` — the recruitment setting.
- **`venueType`** (string): `public` (street, sidewalk) or `private` (event venue, mall, train station with operator permit). Different regulatory regimes apply.
- **`organizationLocationId`** (string): the recruiting organization's internal location reference. Lets the receiving CRM join against the org's own location catalog without consulting the producer.
- **`label`** (string): human-readable display label. Equivalent to (and supersedes) `recruitingLocationLabel`.

`recruitmentType` (existing) remains as the high-level recruitment category (`d2d`, `f2f`, `events`, …) and is orthogonal to `recruitingLocation.type`.

### Deprecation of `recruitingLocationLabel`

Producers SHOULD prefer `recruitingLocation.label` over the standalone `recruitingLocationLabel`. To keep this PR backwards-compatible:

- Producers MAY emit both during a transition period
- When both are present, consumers MUST prefer `recruitingLocation.label`
- A future minor version will mark `recruitingLocationLabel` deprecated; a future major version will remove it.
