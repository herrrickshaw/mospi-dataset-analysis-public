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

## State-wise CPI vs. national WPI trends

[`charts/cpi_wpi_state_trends.html`](charts/cpi_wpi_state_trends.html) — open in a browser — charts Combined-sector CPI (base 2024) across 14 major states plus the All-India benchmark from January 2025 through June 2026, ranked by latest year-on-year inflation, alongside the national WPI trend over the same window (WPI has no state-level breakdown in MoSPI's data).

Headline: **Telangana runs hottest at +6.36% YoY**, **NCT of Delhi coolest at +2.96%**, against an All-India print of +4.38%. WPI broke a year of near-flat readings (~154–158) with a sharp run to 167 in April 2026.

Underlying data: [`data/cpi_statewise_trend_2025-01_to_2026-06.json`](data/cpi_statewise_trend_2025-01_to_2026-06.json), [`data/wpi_national_trend_2025-01_to_2026-04.json`](data/wpi_national_trend_2025-01_to_2026-04.json).

## CPI inflation heatmap (choropleth)

[`charts/cpi_india_heatmap.html`](charts/cpi_india_heatmap.html) — open in a browser — a choropleth of India shaded by June 2026 CPI year-on-year inflation for the 13 states in this dataset (all other states/UTs shown in gray as "not tracked", not as low inflation). State boundaries come from amCharts' `amcharts4-geodata` (`india2023Low`, MIT-licensed), reprojected to SVG locally — no external map tiles or API calls at render time.

## RBI foreign exchange reserves + currency trend

[`charts/rbi_forex_reserves_trend.html`](charts/rbi_forex_reserves_trend.html) — open in a browser — two segments on India's external sector. India's total forex reserves from Jan 2015 to the latest available reading. The MoSPI connector itself is confirmed still frozen at June 2025 (re-checked 2026-07-18, no change) — everything from June 2025 onward is a **real weekly series scraped directly from RBI's own site**, shown as a visually distinct dashed extension, never silently merged with the connector data.

**How the scrape works**: RBI's Weekly Statistical Supplement archive is fully enumerable — every release from 19 Sep 1998 to today sits at a sequential `rbi.org.in/scripts/WSSView.aspx?Id=N` URL, no API key or login needed (unlike `datagovindia`, which wraps data.gov.in and requires a free API key from that portal before anything — including search — will work). One quirk worth knowing: each release's *"as on"* reserve date is 7 days before its publish date, so taking the listed release date at face value mislabels every reading by a week.

That scrape changes the story from the previous version of this chart: reserves hit a **new all-time high of $728.5bn on 27 Feb 2026** (surpassing the $705.8bn Sep 2024 peak this bulletin previously called the record), then kept sliding — through a "West Asia conflict" shock and RBI rupee-defense intervention — to a **true trough of $666.9bn on 26 Jun 2026** (-8.5% peak-to-trough), before a partial recovery to **$675.2bn by 10 Jul 2026**, the latest reading anywhere, including RBI's own site as of this bulletin.

**New currency segment**: the rupee against the US dollar, same Jan 2015–Jul 2026 window, bridged past the connector's July 2025 endpoint with a scrape of RBI's separate **Reference Rate Archive** (`rbi.org.in/scripts/referenceratearchive.aspx`) — a date-range search form, not an enumerable ID archive like the reserves pages. (A dead end along the way: the WSS's own historical exchange-rate table, `PARAM1=6`, was discontinued in December 2012.) The rupee depreciated **~38% over the decade** (₹62.23 → ₹86.11, ~3.3%/year), then accelerated to **+11.9% in the last 12 months alone**, reaching ₹96.37 on 17 Jul 2026 — the same window as the reserves chart's "war shock" slide above.

Underlying data: [`data/rbi_forex_reserves_2015-01_to_2025-06.json`](data/rbi_forex_reserves_2015-01_to_2025-06.json) (reserves) and [`data/rbi_usd_inr_exchange_rate_2015-01_to_2026-07.json`](data/rbi_usd_inr_exchange_rate_2015-01_to_2026-07.json) (currency).

## GDP growth trend, with revision history

[`charts/gdp_growth_trend.html`](charts/gdp_growth_trend.html) — open in a browser — real (constant-price) GDP growth by fiscal year, FY2012-13 through FY2025-26, from the MoSPI connector's NAS dataset. Bars show the latest revised estimate for each year; a tick marks where the original First Advance Estimate landed, so the gap between tick and bar visualizes how much each year's number moved as later data came in.

Checked after last bulletin flagged the NAS `getNasIndicatorList` metadata endpoint as broken (upstream 500) — that endpoint is **still** down, but the actual `get_data` call works fine and returns real figures through FY2025-26's First Advance Estimate. Went looking at data.gov.in/apis first for a GDP source, per request; every economic dataset found there (forex reserves, GDP, Index of Eight Core Industries) turned out to be a static one-off Rajya Sabha Q&A release or a Ministry file frozen years ago despite a recent "Updated On" timestamp — none usable. The connector's own NAS data turned out to be the better, more current source already sitting unused.

Headline: **FY2025-26 opens at +7% real growth** (First Advance Estimate only — expect revision). The COVID-19 year (FY2020-21) was revised from an initial **-8%** up to **-6%**; FY2023-24 saw the largest upward revision of any year, from **+7%** to **+9%**. Across years with more than one estimate, the typical revision is about **±0.75 percentage points**, as much as **2pp** in either direction.

Underlying data: [`data/gdp_growth_rate_2012-13_to_2025-26.json`](data/gdp_growth_rate_2012-13_to_2025-26.json).

## Known quirks / caveats

- **RBI forex reserves lag hard** — requesting 2025/2026 explicitly still only returned data through June 2025. Don't assume this series is current to the same month.
- **WPI's April 2026 jump** (+~6% over one quarter) and **IIP's February 2026 dip** (index fell from 169.9 in Jan to 158.8 in Feb, then rose to 173.2 in March) both look unusually steep/non-monotonic — worth cross-checking against an official MoSPI/RBI press release before treating either as fact.
- **NAS (GDP) indicator list** hit an upstream `500` from `api.mospi.gov.in` during this session and silently fell back to a bundled local indicator list (`_source: local_definitions_fallback`) — the live GDP endpoint may be intermittently flaky.
- **IIP's `get_data` requires an explicit `type` param** (`General`/`Sectoral`/`Use-based category`) — omitting it errors with `missing_required: type`, even though `category_code` alone looks sufficient.
- Large hierarchical metadata payloads (CPI, WPI) can exceed a single response's size limit and need paging/streaming rather than one bulk pull.
