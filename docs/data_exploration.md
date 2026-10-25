# Data Exploration: ENTSO-E Day-Ahead Prices

## 1. The source

**What is the ENTSO-E Transparency Platform?**
It is a public platform that provides data about electricity generation, consumption, transmission and markets all across Europe.

**What is a day-ahead price, and why is it set a day in advance?**

The day-ahead price is the market price of electricity fixed a day in advance: tomorrow's prices are fixed today. It is set a day in advance so that power plants can plan their day accordingly. Operating a big power plant requires a lot of energy and resources, so if they only knew the prices at the last minute, running the plant could be a loss for them. Not just for power plants: day-ahead prices also help consumers and transmission system operators prepare better.

**When are the prices published each day, in CET and in IST, for summer and winter?**

The prices are published each day at around 12:57 Amsterdam time (CET/CEST), which is 16:27 IST in summer and 17:27 IST in winter.

## 2. The request

```
https://web-api.tp.entsoe.eu/api?securityToken=$ENTSOE_TOKEN&documentType=A44&in_Domain=10YNL----------L&out_Domain=10YNL----------L&periodStart=202610120000&periodEnd=202610130000
```

**Which parameters did you use, and what does each one mean?**

- `documentType`: A44 → day-ahead prices
- `in_Domain`, `out_Domain`: `10YNL----------L` → the Netherlands bidding zone. This code is called an EIC code.
- `periodStart`, `periodEnd`: `202610120000` → `202610130000`. This period covers 1 day in UTC, and the same period overlaps two Dutch market days: 12 Oct and 13 Oct (local midnight to midnight).

**Which time format and time zone does the request use?**

The time format is `yyyyMMddHHmm` and the time zone is UTC.
- periodStart: `202610120000` → 12 Oct 2026, 00:00 UTC
- periodEnd: `202610130000` → 13 Oct 2026, 00:00 UTC

In Amsterdam time:
- periodStart: `202610120000` → 12 Oct 2026, 02:00 CEST (Central European Summer Time, UTC+2)
- periodEnd: `202610130000` → 13 Oct 2026, 02:00 CEST (Central European Summer Time, UTC+2)

Although we asked for one full day of data (12 Oct in UTC), the API returned 2 market days of data. This is the behaviour of the ENTSO-E platform: it returns the complete market day for every market day the request touches. For Dutch prices, the market day starts at midnight in Amsterdam.

| Amsterdam market day | UTC start | UTC end |
| --- | --- | --- |
| 12 Oct | 11 Oct, 22:00Z | 12 Oct, 22:00Z |
| 13 Oct | 12 Oct, 22:00Z | 13 Oct, 22:00Z |

My requested interval overlaps both Dutch market days, so ENTSO-E returned the complete data for both days:
- `202610120000` → 12 Oct 2026, 02:00 CEST (falls between 11 Oct 22:00Z and 12 Oct 22:00Z, so I got that full day)
- `202610130000` → 13 Oct 2026, 02:00 CEST (falls between 12 Oct 22:00Z and 13 Oct 22:00Z, so I got that full day)

**What's the rate limit, and what happens if you exceed it?**

The documented rate limit is 400 requests per minute per IP address and security token. If you exceed it, your IP address and/or token is temporarily blocked for 10 minutes.

## 3. The response structure

**What are the three zones of the XML, and what does each contain?**

1. **The envelope (top lines):** information about the document itself, like its ID, type (A44 = day-ahead prices) and when it was created. Mostly not needed for analysis.
2. **The TimeSeries block:** what the data is about: which area (the Netherlands), which currency (EUR) and which unit (MWH).
3. **The Period block inside it:** the actual data. A time window (start, end), the gap between values (resolution), and then many Point entries, each with a position and a price.amount.

**Which fields will you use, and which will you ignore?**

I will use `revisionNumber`, `type`, `createdDateTime`, `period.timeInterval` (start, end), `in_Domain.mRID`, `out_Domain.mRID`, `currency_Unit.name`, `price_Measure_Unit.name`, `curveType`, and inside each Period: `timeInterval`, `resolution`, and each Point's `position` and `price.amount`. The others can be ignored.

**What is the XML namespace, and why does it matter when you parse the file in code?**

The XML namespace is `xmlns="urn:iec62325.351:tc57wg16:451-3:publicationdocument:7:3"`. When I parse this in Python, every tag lookup has to include the namespace, or my code will find nothing.

## 4. Grain

**What does one Point represent?**

A Point represents the actual price for one time slot. Its slot start is calculated as: slot start = Period start + (position − 1) × resolution.

**What will one row of your final price table represent?**

One row of my final table represents the price of electricity for one time slot (15 minutes or 1 hour), along with the timestamp and a key for the bidding zone the price belongs to. **One row per eic_code per timestamp.**

## 5. Time

**How do you calculate a slot's start time? Write the formula and one worked example.**

slot start = Period start + (position − 1) × resolution

```xml
<timeInterval>
  <start>2026-10-11T22:00Z</start>
  <end>2026-10-12T22:00Z</end>
</timeInterval>
<resolution>PT15M</resolution>
<Point>
  <position>1</position>
  <price.amount>179.99</price.amount>
</Point>
<Point>
  <position>2</position>
  <price.amount>179.22</price.amount>
</Point>
<Point>
  <position>3</position>
  <price.amount>177.76</price.amount>
</Point>
```

- Slot start = 22:00 + (1 − 1) × 15 min = 22:00
- Slot start = 22:00 + (2 − 1) × 15 min = 22:15
- Slot start = 22:00 + (3 − 1) × 15 min = 22:30

**Why did the API return two days when you asked for one?**

Because my requested interval overlaps two Dutch market days, ENTSO-E returns the complete data covering both days:
- `202610120000` → 12 Oct 2026, 02:00 CEST (falls between 11 Oct 22:00Z and 12 Oct 22:00Z, so I got that full day)
- `202610130000` → 13 Oct 2026, 02:00 CEST (falls between 12 Oct 22:00Z and 13 Oct 22:00Z, so I got that full day)

**When does a Dutch market day start in UTC and in IST, in summer and in winter?**

A Dutch market day runs from midnight to midnight in Amsterdam local time. The corresponding UTC and IST times change depending on whether the Netherlands is on summer or winter time.

| Season | Amsterdam | UTC | IST |
| --- | --- | --- | --- |
| Summer | 00:00 | 22:00 (previous day) | 03:30 |
| Winter | 00:00 | 23:00 (previous day) | 04:30 |

**How many slots does a normal day, an autumn clock-change day and a spring clock-change day have, and when are the next ones?**

A normal day has 96 slots. On the clock change from summer to winter time in October (the last Sunday of October, 25 Oct 2026), the clock goes back by 1 hour, so the day has 100 slots. On the clock change from winter to summer time in March (the last Sunday of March, 28 Mar 2027), the day has 92 slots.

## 6. Data quirks

**What does curveType A03 mean, and how will your pipeline handle it?**

curveType A03 means variable-sized blocks: if position 3 and position 4 have the same value, position 4 is left out and only position 3 appears. In these cases, the pipeline must fill the missing position with the same value as the previous one.

**Why can't you assume 96 slots per day? Give both reasons.**

There are two reasons: first, curveType A03 can leave out repeated values; second, the clock change between summer and winter time makes some days 23 or 25 hours long.

**What does revisionNumber tell you, and how does it affect your pipeline?**

revisionNumber is the version of the published data. revisionNumber 1 means it is the first published version; if the data gets revised, the revisionNumber becomes 2. My pipeline should snapshot the data using the revision number: it should update my table with the latest data but also keep a record of the change.

## 7. Codes and units

**What is an EIC code, and how will you turn it into a readable country or zone name?**

An EIC code identifies an area, for example `10YNL----------L` is the Netherlands bidding zone. I will need a seed lookup table that maps each EIC code to a readable zone name and country.

**What units are prices in, and how would you convert them to €/kWh?**

Prices are in euros per MWh (megawatt-hour). To convert them to euros per kWh (kilowatt-hour), divide by 1,000.

## 8. Impact on the pipeline

**How will overlapping requests create duplicates, and how will you prevent them?**

If we ask for data for 12 Oct, the API returns two market days (12 and 13 Oct). If we then ask for data for 13 Oct, it returns 13 and 14 Oct, so 13 Oct is loaded twice, creating duplicates. One way to reduce this is to request exact market-day windows in UTC, for example 11 Oct 22:00 to 12 Oct 22:00 for 12 Oct. But the main defence is making the pipeline idempotent: loading the same data twice must leave the same result, using a unique key and deduplication.

**Why should everything be stored in UTC?**

Everything should be stored in UTC because it avoids ambiguity. For example, on 25 Oct 2026 the Amsterdam clock jumps back from 03:00 to 02:00, so 02:30 happens twice. If we saved local timestamps, we couldn't tell which 02:30 is which, but UTC stays consistent. On the spring transition, 02:30 doesn't exist at all. UTC gives every instant a unique timestamp, which lets me join Dutch, German and French electricity data accurately. I use Europe/Amsterdam separately to identify Dutch market-day boundaries.

**When should your pipeline fetch new data each day?**

Since the data is published at around 12:57 Amsterdam time, my pipeline should fetch data at around 11:05 UTC (16:35 IST) in summer and around 12:05 UTC (17:35 IST) in winter.

**What should the pipeline do if the data isn't published yet when it runs?**

My pipeline should retry every 20 minutes. If the data is still not there after about 5 hours, it should log the failure and send an alert, preferably an email notification to me. I already know how many slots to expect. If I only get partial data, for example 60 out of 96 slots on a normal day, I will first apply the A03 gap-filling logic. If the slot count still doesn't match, I will raise an alert to the stakeholders.

## 9. Open questions

1. Where will the data be stored?
2. Where will my dbt code be deployed? I need a server or scheduler to run my code at set intervals.
3. Which tables and views will I need?


## 10. Business Questions

1. When and why do electricity prices go negative, and how is that linked to solar and wind generation?
2. What’s the cheapest time of day to run energy-heavy things like EV charging, by bidding zone and season?
3. How has the renewable share of generation changed over time, and does a higher share lower prices?