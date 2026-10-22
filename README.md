# Toronto Bike Share station snapshots

One Parquet file with the state of every Bike Share Toronto station at every snapshot we have, refreshed daily
(about 04:30 Toronto time). `SUMMARY.txt` gives the row count, time range and date of the last refresh.

```python
import pandas as pd
url = "https://github.com/ThatRandomUser23/toronto-bikeshare-data/releases/download/data-latest/toronto_bikeshare_snapshots.parquet"
df = pd.read_parquet(url)                     # whole file (several GB in memory)
df = pd.read_parquet(url, columns=["ts", "station_id", "classic", "ebike", "docks"],
                     filters=[("ts", ">=", pd.Timestamp("2026-10-01", tz="UTC"))])   # just what you need
```

Downloading it once and reading the local copy is faster than reading the URL each time.

## Where the rows come from

| Period | `source` | Cadence | Origin |
|---|---|---|---|
| Apr 2024 to the first live poll (2026-09-29 05:17 UTC) | `history` | about 12 min | public archive [MaxHalford/bike-sharing-history](https://github.com/MaxHalford/bike-sharing-history) (`city/toronto` branch) |
| from 2026-09-29 05:17 UTC | `live` | about 1 min | our own pollers of the official GBFS feed (two Macs and Google Apps Script, merged, duplicates removed) |
| same, a few rows | `live_citybikes` | | [CityBikes](https://api.citybik.es/v2/networks/bixi-toronto) mirror, used only where no official poll exists within 3 min |

Each row is one station at one snapshot. Rows are sorted by `ts`, then `station_id`. History has gaps where the
archive has none (e.g. most of May to July 2024); check `ts` spacing before trusting a period.

## Columns

| Column | Meaning |
|---|---|
| `ts` | snapshot time, UTC (poll time for live rows, archive commit time for history rows) |
| `station_id` | GBFS station id; join with `stations.csv` |
| `source` | `history`, `live` or `live_citybikes` (see above) |
| `name`, `lat`, `lon` | station name and location, latest known value |
| `capacity` | number of docks at the station |
| `bikes` | rentable bikes (`num_bikes_available`) |
| `classic` | rentable classic (pedal) bikes |
| `ebike` | rentable e-bikes |
| `other_vt` | rentable vehicles of an unknown type (normally 0) |
| `disabled` | bikes docked but not rentable |
| `docks` | free docks where a bike can be returned |
| `docks_disabled` | docks out of service |
| `installed`, `renting`, `returning` | station is installed / allows rentals / allows returns |
| `charging` | station charges e-bikes |
| `has_types` | the feed gave per-type counts in this row (if false, `classic`/`ebike` are empty) |
| `status` | GBFS station status, e.g. `IN_SERVICE`, `END_OF_LIFE` |
| `last_reported` | when the station itself last reported, unix seconds |
| `vt_*` | rentable bikes per GBFS vehicle type id. `ICONIC` is classic; `EFIT`, `EFIT_G5`, `ASTRO` and `BOOST` are e-bikes. `CITYBIKES_*` appear only in `live_citybikes` rows. Empty where a type did not exist in that period. |

A rider can take a classic bike if `renting` and `classic >= 1`, and return a bike if `returning` and `docks >= 1`.

`stations.csv`: one row per station: `name`, `lat`, `lon`, `capacity`, `physical_configuration`,
`is_charging_station`, and `in_live_feed` (false for stations that have since been removed).

## Licence and credit

Station data: Bike Share Toronto GBFS feed, published under the
[Open Government Licence – Toronto](https://open.toronto.ca/open-data-license/). History rows come from the public
archive by Max Halford. Built for the MIE1517 (University of Toronto) course project.
