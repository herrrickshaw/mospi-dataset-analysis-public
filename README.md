# MoSPI Dataset Analysis

A record of accessing India's official statistics via the **MoSPI (Ministry of Statistics and Programme Implementation)** data API/connector, plus a snapshot of the latest published figures pulled on 2026-07-18.

## What MoSPI provides

The connector fronts `api.mospi.gov.in` and exposes **25 datasets** covering 500+ indicators:

| Dataset | Covers |
|---|---|
| CPI | Consumer Price Index (retail inflation), Group/Item hierarchy, base years 2010/2012/2024 |
| WPI | Wholesale Price Index, base years 1993-94/2004-05/2011-12 |
| IIP | Index of Industrial Production (manufacturing/mining/electricity), monthly & annual |
| NAS | National Accounts Statistics (GDP, GVA, consumption, savings) |
| RBI | External sector: forex reserves, exchange rates, balance of payments, external debt |
| PLFS | Periodic Labour Force Survey (LFPR, WPR, unemployment rate, wages) |
| ASI | Annual Survey of Industries (factory-level financials) |
| ASUSE, EC | Informal sector / unincorporated enterprises, Economic Census |
| AISHE, UDISE | Higher education & school education statistics |
| GENDER, NFHS | Gender statistics, National Family Health Survey |
| ENVSTATS, MNRE | Environment statistics, renewable energy capacity |
| HCES, TUS | Household consumption expenditure, Time use survey |
| NSS75E–NSS80 | Various NSS survey rounds (education, disability, housing, telecom, land & livestock, AIDIS, AYUSH) |
| CPIALRL | CPI for Agricultural/Rural Labourers |

## Access workflow

Four steps, each depending on the previous:

```
list_datasets()                          -> identify the dataset
get_indicators(dataset)                  -> list available indicators
get_metadata(dataset, indicator_code...) -> valid filter values (years, states, categories...)
get_data(dataset, filters)                -> fetch the actual data
```

Filter codes are dataset-specific and not standardized (e.g. `indicator_code=3` means different things in different datasets) — always resolve them via `get_metadata` rather than guessing.

This repo currently documents that workflow as exercised through the MoSPI MCP connector inside Claude Code; it is not (yet) a standalone script, since reproducing it outside that environment would require reverse-engineering `api.mospi.gov.in`'s undocumented auth/request format from scratch.

## Session findings (as of 2026-07-18)

| Series | Latest period returned | Value |
|---|---|---|
| CPI (base 2024, All India, Combined) | June 2026 | Index 107.00, **+4.38% YoY** (rural 4.74%, urban 3.92%; food & beverages +5.05%) |
| IIP (General, base 2011-12) | March 2026 | Index 173.2, **+4.1% YoY** |
| WPI (base 2011-12, overall) | April 2026 | Index 167 (Jan 157.6 → Feb 158.4 → Mar 160.8 → Apr 167) |
| RBI Foreign Exchange Reserves (Total, US$) | June 2025 | $698.1bn |

Raw responses: [`data/mospi_snapshot_2026-07-18.json`](data/mospi_snapshot_2026-07-18.json).

## Known quirks / caveats

- **RBI forex reserves lag hard** — requesting 2025/2026 explicitly still only returned data through June 2025. Don't assume this series is current to the same month.
- **WPI's April 2026 jump** (+~6% over one quarter) and **IIP's February 2026 dip** (index fell from 169.9 in Jan to 158.8 in Feb, then rose to 173.2 in March) both look unusually steep/non-monotonic — worth cross-checking against an official MoSPI/RBI press release before treating either as fact.
- **NAS (GDP) indicator list** hit an upstream `500` from `api.mospi.gov.in` during this session and silently fell back to a bundled local indicator list (`_source: local_definitions_fallback`) — the live GDP endpoint may be intermittently flaky.
- **IIP's `get_data` requires an explicit `type` param** (`General`/`Sectoral`/`Use-based category`) — omitting it errors with `missing_required: type`, even though `category_code` alone looks sufficient.
- Large hierarchical metadata payloads (CPI, WPI) can exceed a single response's size limit and need paging/streaming rather than one bulk pull.
