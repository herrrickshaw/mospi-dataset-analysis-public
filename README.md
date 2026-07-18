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

## Effective buying power & the China currency-manipulation comparison

[`charts/real_exchange_rate_and_currency_policy.html`](charts/real_exchange_rate_and_currency_policy.html) — open in a browser — a follow-on to the currency segment, separating the rupee's nominal fall into "just India's higher inflation catching up" versus a genuine real (purchasing-power-adjusted) depreciation, using India's implied GDP deflator (from the GDP chart's own NAS data) against US CPI-U.

**Headline**: nominal depreciation since Jan 2015 is **+54.9%**, but the real (PPP-adjusted) rate still moved **+37.6%** — meaning a rupee's **effective buying power for dollar-priced goods has fallen ~27%** even after fully crediting India's own inflation. The last 12 months are almost entirely real, not inflationary: India's implied inflation this year (~0.9%) was *lower* than the US's, so PPP alone would have predicted the rupee strengthening — instead it fell 11.9% nominally (14.0% in real terms).

That's then read against the framework China was scrutinized under: the US Treasury's three-test "currency manipulator" criteria (trade surplus, current account surplus, and — the one that matters here — persistent net *purchases* of foreign currency to keep a currency artificially weak). China was formally designated a manipulator in Aug 2019, un-designated in Jan 2020. India's reserves data (from the segment above) shows the RBI doing the mechanical opposite of what tripped that third test — *selling* reserves to defend the rupee, not buying foreign currency to suppress it — the opposite motive from the one "manipulation" describes, even though the real depreciation is genuine. Cites Brookings' ["China's Currency Policy, Explained"](https://www.brookings.edu/articles/chinas-currency-policy-explained/) for the China policy context.

Underlying data: [`data/rupee_real_exchange_rate_2015_to_2026.json`](data/rupee_real_exchange_rate_2015_to_2026.json).

## Trade balance — is India an export-surplus country?

[`charts/trade_balance_hsn_analysis.html`](charts/trade_balance_hsn_analysis.html) — open in a browser — HSN (2-digit HS chapter) merchandise export and import data from the Commerce Ministry's own **TRADESTAT** database (`tradestat.commerce.gov.in`, not MoSPI/RBI), summed across all 98 tracked chapters to reconstruct India's national trade balance for FY2018-19 through FY2025-26.

**Verdict: no — in goods alone, or goods plus services.** India has run a merchandise trade deficit in **every one of the 8 fiscal years** on record here, widening from **-$184.0bn (FY2018-19) to -$334.3bn (FY2025-26)**. Exports covered 64.2% of the import bill in FY2018-19; they cover just **56.9%** now. **Mineral fuels (HS27, -$147.5bn)** and **pearls/gems/jewellery (HS71, -$81.1bn)** are the two largest deficit-driving chapters; **pharmaceuticals, vehicles, cereals and apparel** are the strongest surplus chapters. This closes the loop on the currency and reserves segments above: a persistent, widening goods deficit is exactly the kind of structural pressure that would push the rupee down over time, independent of any single shock — the backdrop the RBI has been spending reserves to lean against.

**How it was pulled**: TRADESTAT is a Laravel/Livewire app whose report URLs return `405` on a cold GET — visiting the search page first (to pick up a session cookie + CSRF token) then POSTing the form fields works, and each query returns two fiscal years at once, so 8 requests (4 per trade direction) covered the full 8-year window.

**Added: the services-trade offset.** Checked DPIIT and the Commerce Ministry for services data per request — **DPIIT doesn't publish services-trade statistics** (its main product is a goods import-monitoring system plus FDI data); the Commerce Ministry's own DGCI&S does track services exports via annual "Service Exports Reporting Form" PDF/Excel reports, but not as a queryable database. Used **RBI's Balance of Payments "Non-factor Services" invisibles series** (via the MoSPI connector's RBI dataset) instead, converted from ₹ Crore using fiscal-year-average exchange rates built from this repo's own currency segment. Result: India's services surplus **more than doubled**, from **$82.1bn (FY2018-19) to $188.9bn (FY2024-25)** — enough to cut the FY2024-25 overall gap from **-$283.5bn to just -$94.6bn**, and to nearly close it in the COVID year (-$14.1bn). Still a deficit every year — just roughly half the size the goods numbers alone suggest.

Underlying data: [`data/tradestat_hsn_export_import_2018-19_to_2025-26.json`](data/tradestat_hsn_export_import_2018-19_to_2025-26.json) (goods, all 98 HS chapters) and [`data/services_trade_and_overall_balance_2015-16_to_2024-25.json`](data/services_trade_and_overall_balance_2015-16_to_2024-25.json) (services + combined balance).

**Cross-validated (2026-07-18)** against three more official sources, per request: the Commerce Ministry's **Trade Intelligence and Analytics Portal** (`trade-analytics.commerce.gov.in`), **DGFT**'s trade-statistics hub, and a **PIB** press release. Imports match almost to the dollar ($776,013.65M on TIA Portal vs. $776,013.6M here); the largest export chapter (HS27) matches exactly ($55,871.2M both); PIB's official FY2025-26 exports ($441.78bn) and deficit ($333.19bn) match within a rounding error. One flagged discrepancy: TIA Portal's own export total reads ~2% ($9.3bn) higher than both this bulletin and PIB — likely a data-vintage difference on a still-provisional fiscal year, not a methodology error. DGFT's "Import Export Data Bank" turned out to just be a link back to TRADESTAT itself, not an independent source. DPIIT's 295-page 2025-26 Annual Report doesn't publish trade-balance data (as already found), but its Industrial Entrepreneur Memorandum investment-intention filings independently corroborate the historical-trends chart below: proposed investment in Electricals Equipment and Fertilizers — this repo's two fastest-growing high-value import chapters — is also scaling up fastest in DPIIT's own, completely separate licensing data.

Underlying validation data: [`data/trade_data_cross_validation_2026-07-18.json`](data/trade_data_cross_validation_2026-07-18.json).

## HSN-wise historical trends — which imports are growing fastest?

[`charts/hsn_historical_trends.html`](charts/hsn_historical_trends.html) — open in a browser — the same TRADESTAT chapter-level data as the trade-balance bulletin above, re-cut to track each HS-2-digit chapter's own 8-year trajectory (FY2018-19 → FY2025-26) rather than just its latest-year snapshot. Ranks the top-12 import and top-12 export chapters by FY2025-26 value (independently-scaled sparklines per chapter), then re-ranks the same 12 import chapters by **CAGR** to separate "large" from "fast-growing."

**Headline: the biggest imports aren't the fastest-moving ones.** Mineral fuels (HS27) is by far the largest single import ($203.4bn) but has grown only **+21.2%** (2.8% CAGR) over eight years — largely a price story. The real acceleration is in **Fertilisers (HS31, +118.9%, 11.8% CAGR)** — plausibly global fertiliser price shocks since 2022 stacked on rising volumes — followed by **Electrical machinery (HS85, +101.5%, 10.5% CAGR)** and **Animal/vegetable fats & oils (HS15, +97.9%, 10.2% CAGR)**. Top 12 import chapters account for **81.5%** of all FY2025-26 imports. On the export side, **Electrical machinery (HS85) has quadrupled** (+324.1%, 22.9% CAGR, $12.7bn → $54.0bn) — by far the fastest mover on either side of the ledger — while **Pearls, gems & jewellery (HS71)** is the only top-12 chapter shrinking on either side (**-29.9%**).

Read against the trade-balance bulletin: the chapters driving the *deficit* (mineral fuels, gems/jewellery) are not the chapters driving its *momentum* — fertilisers, electronics components, and edible oils are where new import dependency is actually building.

Underlying data: [`data/hsn_historical_trends_2018-19_to_2025-26.json`](data/hsn_historical_trends_2018-19_to_2025-26.json).

## Imported fertiliser & fuel prices → diesel → CPI/WPI

[`charts/fertiliser_fuel_price_transmission.html`](charts/fertiliser_fuel_price_transmission.html) — open in a browser — correlates the MoSPI connector's own WPI item-level series for diesel (HSD) and fertilisers (Urea, DAP) against WPI and CPI headline/food inflation, 2012–2026, to test the import-price pass-through described in five published studies (cited in full inside the bulletin: James Hamilton's IJCB framing paper, Bhattacharya & Gupta's NIPFP food-inflation SVAR, Mishra & Roy's India Policy Forum paper, a JETIR 2025 survey, and an MBA thesis on India's import composition).

**Headline: diesel passes through to WPI almost mechanically (r=0.92); fertiliser mostly doesn't, because subsidy absorbs it.** Diesel (WPI HSD) is up **+79.0%** since April 2012, moving in near lockstep with WPI headline. **Urea is up only +13.8%** over the same window despite the 2021–23 global fertiliser shock — the subsidy regime holds it down almost regardless of the border price — while **DAP (smaller, less consistent subsidy) is up +38.4%**, roughly midway between the two. Against CPI, the picture is more textured: same-year correlations between these input costs and CPI headline/food/oils-and-fats are weak or inconsistent (r between -0.52 and +0.29) at annual resolution — not because the transmission described in the literature isn't real, but because (per Bhattacharya & Gupta's SVAR) it operates on a 1–4 month lag and fades within the year, which annual year-on-year correlation washes out. The one series that *does* show a dramatic, real swing is **CPI oils & fats**, whose monthly print went from **-18.17% (Jun 2023) to +21.24% (Aug 2025)** — matching Mishra & Roy's finding that edible oils are the one food category with near-1:1 global pass-through, and this repo's own trade data showing edible-oil imports (HS15) up 97.9% over the same window.

Underlying data: [`data/fertiliser_fuel_price_transmission_2012_to_2026.json`](data/fertiliser_fuel_price_transmission_2012_to_2026.json).

## Are import-dependent sectors getting government attention?

[`charts/import_dependency_policy_gap_analysis.html`](charts/import_dependency_policy_gap_analysis.html) — open in a browser — cross-references this repo's own 12-chapter import ranking (value and growth, FY2018-19 → FY2025-26) against a compiled six-month roundup of PIB-reported government schemes (Budget 2026-27, Cabinet Decisions Feb–Jun 2026, PLI/ISM 2.0 scorecards), to check whether the sectors India depends on most for imports are actually getting matching policy attention.

**Headline: the fastest-growing high-value import (fertilisers, +118.9%) had no production-capacity scheme in the six-month window — until three days before this analysis.** The one chapter with strong, matched coverage is **electrical machinery (HS85)**: Semiconductor Mission 2.0 (₹76,000 crore) and the bulk of 836 approved PLI applications sit directly on it. Everything else is partial at best: **mineral fuels (HS27, 26.2% of the entire import bill)** gets only diffuse, adjacent levers (coal gasification, CCUS, battery/solar PLI) with nothing sized to its scale; **plastics (HS39)**, **inorganic chemicals (HS28)**, and **aircraft & spacecraft (HS88)** have no matching scheme at all in the window. **Fertilisers (HS31)** is the most interesting case: only a demand-side subsidy appeared in the six-month recap, but the Cabinet approved the **National Investment Policy for Urea-2026** on 15 July 2026 — three days before this bulletin — targeting roughly the exact size of India's current urea import gap. Supplementary verification also found a real, Cabinet-approved edible-oil self-sufficiency mission (NMEO-Oilseeds, ₹10,103 crore) that likewise never surfaced in the "achievements" recap despite edible oils being the repo's third-fastest-growing import and the sector with the sharpest CPI swing documented in the bulletin above — a visibility gap, not a scheme gap.

Underlying data: [`data/import_dependency_policy_gap_analysis_2026-07-18.json`](data/import_dependency_policy_gap_analysis_2026-07-18.json).

## Where India's forex actually goes — country by country, with a decade of policy history

[`charts/country_trade_deficit_and_policy_history.html`](charts/country_trade_deficit_and_policy_history.html) — open in a browser — ranks India's FY2025-26 bilateral trade balance by partner country (Commerce Ministry TIA Portal), then traces the policy history behind the five biggest deficit relationships back several years — not just the six-month window used above — to check whether sustained attention has actually narrowed each gap. Was also prompted by a look at the Commerce Ministry's **NIRYAT portal** (niryat.gov.in) for visualization ideas; direct access was blocked at the TLS level (the same DGFT-family bot protection this repo has run into before), so its World Map/Commodity/Table-view design and traffic-light target-tracking pattern are described here from public documentation rather than a first-hand look.

**Headline: China alone is a -$112.2bn bilateral deficit — more than double the next-largest — and six years of Atmanirbhar Bharat/PLI hasn't closed it.** China still supplies an estimated 62% of India's electronic components despite PLI (launched March 2020) and its 2023 expansion to 14 sectors. **Russia is the fastest-moving entry on the list**: with no formal import-substitution scheme behind it, Russian oil went from ~2.5% of India's crude imports before 2022 to 35.8–35.9% in each of the last two fiscal years — a market-driven shift (discounted pricing, settled in UAE dirhams to route around sanctions exposure) that made Russia India's #2 deficit country from a near-zero base in four years. **UAE (-$26.5bn) and Switzerland (-$23.1bn)** are dominated by gold/rough-diamond bullion flows tied to the gems-and-jewellery re-export trade — a deliberately open door, not an import-substitution gap. **Indonesia and Malaysia (palm oil)** show the sharpest contrast between effort and outcome: the National Mission on Edible Oils - Oil Palm has run since August 2021 (₹11,040 crore), yet India remains roughly 57% import-dependent on edible oil, costing an estimated $20.56bn/year in forex.

**Validated (2026-07-18)** against TRADESTAT's own country×commodity data for the four countries anchoring this argument (China, Russia, UAE, USA) — the two independent Commerce Ministry sources agree to within rounding on every country. The chapter-level detail sharpens the picture: China's imports are **58% electrical machinery + mechanical machinery (HS85/HS84)** — industrial inputs feeding India's own manufacturing and export lines, not consumer goods, which is exactly why a currency-settlement change wouldn't touch this deficit. Russia's imports are **86% mineral fuels (HS27)** alone. UAE's imports turn out to be an almost even split between **gems/pearls (HS71) and mineral fuels (HS27)** — not "almost entirely gold" as first read, correcting an earlier pass through this data. And India's own exports to the USA are led by the **same HS85 electrical-machinery chapter** that dominates imports from China (+61.5% growth) — consistent with a components-in, finished-goods-out assembly pattern, not a substitutable consumer import.

Underlying data: [`data/country_trade_deficit_and_policy_history_2026-07-18.json`](data/country_trade_deficit_and_policy_history_2026-07-18.json) (country balances, policy timeline) and [`data/country_commodity_breakdown_2026-07-18.json`](data/country_commodity_breakdown_2026-07-18.json) (TRADESTAT country×commodity validation).

## Priority trade partners by import-growth sector, and real PLI coverage

[`charts/sector_country_priority_and_pli_coverage.html`](charts/sector_country_priority_and_pli_coverage.html) — open in a browser — maps all 12 of this repo's fastest-growing/highest-value import chapters to their actual top supplier countries (TRADESTAT "commodity-wise, all countries" report), builds a cross-sector country-priority ranking, and checks exactly which Production Linked Incentive sub-schemes — including White Goods and Specialty Steel — reach these chapters in practice.

**Headline: China supplies 8 of the 12 growth chapters ($102.4bn combined, #1/#2 rank in 5 of them) — more than double any other country — while PLI schemes cover the money-value of only 2 of the 12 outright.** UAE ($56.6bn, 7 sectors) and Russia ($52.6bn, only 3 sectors but #1 in both mineral fuels and fertilisers) round out the top tier; the USA is the most diversified major partner (9 of 12 sectors, never dominant in any one) and the one country in overall bilateral surplus. On the policy side: **PLI White Goods** (₹11,198 crore committed across all 85 selected companies, targeting AC compressors, copper tubes and aluminium foils) turns out to touch HS84 (machinery) rather than HS85, since compressors are classified there — a precise, easy-to-miss detail. **PLI Specialty Steel** is the most substantial single lever found (₹44,106 crore in rounds 1.0/1.1, plus a further ₹11,887 crore in round 1.2, MoUs signed 9 February 2026, targeting 42 million tonnes of production by FY2026-27 explicitly to cut imports) and directly targets HS72. Seven of the twelve growth chapters — mineral fuels, gems &amp; jewellery, plastics, edible oils, fertilisers, inorganic chemicals, and aircraft — have no PLI sub-scheme at all.

Underlying data: [`data/sector_country_priority_and_pli_coverage_2026-07-18.json`](data/sector_country_priority_and_pli_coverage_2026-07-18.json).

## Priority export destinations by growth sector, and the export side of PLI

[`charts/export_destination_priority_and_pli_coverage.html`](charts/export_destination_priority_and_pli_coverage.html) — open in a browser — the export-side mirror: maps the 12 fastest-growing/highest-value HSN **export** chapters to their top destination countries, ranks countries by cross-sector exposure, and checks PLI Auto, PLI Bulk Drugs, and PLI Textiles coverage.

**Headline: the USA is the top-8 destination in 11 of 12 growth export chapters — a bigger concentration than China's grip on imports.** The USA takes **$63.9bn combined, #1/#2 rank in 9 of those 11 chapters** — more concentrated by both breadth and depth than China's import-side dominance (8 of 12 chapters, but more evenly spread across rank positions). UAE is a clear #2 destination (10 sectors, $26.9bn). On the PLI side, **Auto & Auto Components (₹25,938cr)** and **Bulk Drugs (₹6,940cr, explicitly targeting the ~70% China API dependency that also shows up in the companion import bulletin)** are both real, substantial, and correctly targeted. The one genuine mismatch: **PLI Textiles (₹10,683cr)** exists, but was deliberately built to cover man-made-fibre apparel and technical textiles rather than the **traditional cotton-based apparel (HS61/HS62)** that actually accounts for the export volume in this data — the scheme isn't absent, it's aimed at a different, smaller part of the textile export base.

Underlying data: [`data/export_destination_priority_and_pli_coverage_2026-07-18.json`](data/export_destination_priority_and_pli_coverage_2026-07-18.json).

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
