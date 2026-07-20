# RTT Service Performance (THA/RDG/PAD corridor)

## Description

Per-service actual vs scheduled departure/arrival times, cancellation flag and
train formation (vehicle count) for services calling at Thatcham (THA), Reading
(RDG) or London Paddington (PAD), filtered to services that call at more than
one of these three stations. Sourced from the Realtime Trains (RTT) Next
Generation API. Use this to analyse GWR delays, cancellations and
short-formation trains on the THA<->RDG<->PAD corridor.

## Classification

Public. Train running data with no personal or customer information.

## Owner

Dan (data platform).

## Source & references

Realtime Trains Next Generation API, `https://data.rtt.io`. Registration via
[api-portal.rtt.io](https://api-portal.rtt.io/), requires an RTT unified login
account. API spec: https://realtimetrains.github.io/api-specification/.

## License & usage restrictions

Free tier is for personal/non-commercial use under RTT's Terms of Service
(https://www.realtimetrains.co.uk/legal/). Commercial or high-volume use
requires contacting RTT support for a separate agreement.

## Location

- S3: `s3://dantelore.data.incoming/rtt/services/`
- Glue: `incoming.rtt_services`
- Format: Parquet, partitioned by `date` (the day the location query was run for)

## Field spec

| Field name | Type | Nullable | Key role | Description |
|---|---|---|---|---|
| unique_identity | string | No | | RTT's unique service identifier, `namespace:identity:departureDate` |
| departure_date | string | No | | Date the service departed its origin, `YYYY-MM-DD` |
| station | string | No | | Station this row's platform/time data was captured at (THA, RDG or PAD) |
| operator_code | string | No | | Two-letter TOC code, e.g. `GW` for Great Western Railway |
| operator_name | string | No | | Full operator name |
| mode_type | string | No | | `TRAIN` or `BUS` (rail replacement) |
| scheduled_time | string | Yes | | Scheduled departure or arrival time (ISO, no timezone) at this station |
| actual_time | string | Yes | | Actual/realtime departure or arrival time, null if not yet reported |
| lateness_minutes | int | Yes | | Lateness in minutes against the advertised timetable, negative if early |
| is_cancelled | boolean | No | | Whether this call was cancelled |
| number_of_vehicles | int | Yes | | Number of vehicles (coaches) in the formation at this station, null if not reported |
| platform | string | Yes | | Actual platform used at this station |
| stations_called | string | No | | Comma-separated subset of THA/RDG/PAD this service called at (always 2 or 3) |
| date | string | No | Partition key | Date the ETL ran the location query for |

## Source ETL / code

`rtt_etl/rtt_bulk_load.py` in this repo. `rtt_etl/rtt_client.py` handles the
refresh-token to access-token exchange.

## Freshness & update cadence

Intended to run daily, incrementally. **Important limitation:** RTT's Next
Generation API restricts history depth per account (14 days on a standard
token, check `/api/info`'s `restrictHistoryDays`) - there is no way to backfill
further back than that window. History accumulates going forward from when
this ETL first ran (2026-07-19), it is not a historical archive.

## Known issues & caveats

- History is capped at ~14 days by the API itself - do not expect a deep
  backfill here, unlike HSP. If a longer history is ever needed, RTT's older
  classic API or HSP would be the source, not this one.
- A service only appears in this table if it calls at more than one of
  THA/RDG/PAD - single-station calls are dropped by design.
- `number_of_vehicles` (formation) is only as reliable as RTT's own upstream
  "Know Your Train" data, which is patchy pre-2020 and by operator - treat
  nulls or unexpected values here as "unknown", not "ran empty."
- Account used is restricted to the `gb-nr` namespace only (per
  `namespaceRestriction`/`permittedNamespaces` in `/api/info`) - this is fine
  for GB mainline data but would need a different token for e.g. Ireland.
- No entitlements beyond the base tier were present on the token used to build
  this (`entitlements: []` in `/api/info`) - detailed mode and STP filtering
  were not tested and may not be available.

## Interested parties

None yet - new dataset.

## Status

Active.
