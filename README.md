# Loyalty Radar: Airline Member Retention and Engagement Analytics

![status](https://img.shields.io/badge/status-in--progress-yellow) ![Power BI](https://img.shields.io/badge/Power%20BI-DAX%20%7C%20Power%20Query-F2C811)

A Power BI analysis of an airline loyalty program that shows who is cancelling, how actively members fly, how points are earned and redeemed, and where retention effort should go first.

> **Status:** Data preparation, data model, DAX measures, analysis (7 questions) and written findings are complete. The three-page dashboard (Executive Summary, Retention, Engagement and Points) is drafted in the `.pbix`; final visual polish and screenshots are pending.

## Business Problem

It is early 2019 and the Head of Loyalty at a Canadian airline is planning next year's retention and engagement spend. Leadership cannot yet see who is cancelling, who is actively flying, and how points are earned and used, so retention spend cannot be targeted. This project answers that with evidence and a prioritised recommendation.

| Stakeholder | What they need |
|---|---|
| Head of Loyalty Program | One-page health check, key risks, prioritised actions |
| CRM and Marketing Manager | Which segments to target and what sets them apart |

## Dataset

| Table | Rows | Description |
|---|---|---|
| Customer Loyalty History | 16,737 | One row per member: province, gender, education, loyalty card (Star, Nova, Aurora), CLV, enrollment type, enrollment and cancellation dates |
| Customer Flight Activity | 392,936 | Monthly flights, distance and points per member, 2017 and 2018 only |
| Calendar | 2,557 | Daily calendar, 2012 to 2018 |

The raw CSV files are not included in this repository. See [`data/README.md`](data/README.md) for the expected files. The data has no revenue, cost, route, promotion spend or age fields, so the analysis makes no profitability or causal claims, and CLV is the only value proxy.

## Approach

1. **Profile and clean** the data in Power Query: resolve duplicates, handle missing and negative salary, check activity outside the membership window, build real dates, and remove unused columns.
2. **Model** the data as a star schema with measures held in a dedicated table.
3. **Define KPIs and DAX measures** with written definitions and explicit time bases.
4. **Answer seven business questions**, each with a finding and a caveat.
5. **Build a three-page dashboard** where every visual maps to one of the seven questions.

### Data decisions

| Issue | What I found | What I did | Why |
|---|---|---|---|
| Duplicate activity rows | 7,718 rows in repeated member-months, of which 3,828 were identical with zero activity | Removed identical copies. For differing pairs, kept the row with the most flights (then highest distance), using `Table.Buffer` to preserve the sort | Summing duplicates would inflate totals. Result: 392,936 to 389,065 rows |
| Missing and negative salary | 4,238 blanks (25.3%), all in the College group, plus 20 negatives | Kept all members, did not impute, excluded income from the analysis and removed Salary | Deleting rows would remove a whole education group, and averages would invent data |
| Activity after cancellation | 33,988 rows, all zero | Removed them | Totals unchanged. Result: 355,077 rows |
| Activity before enrollment | 52,874 rows, of which 3,199 (498 members) have flights, 8,207 in total (about 1.6%) | Kept them | Real activity should not be deleted when the enrollment date may be the error |
| Cancellation fields | 88% blank, always blank together | Built a Status column and real dates | One consistent Active versus Cancelled rule |
| Date grain | Monthly data, daily Calendar | Used the first day of the month for all dates | Matches the grain of the data |
| Sort order | Text sorts alphabetically | Added numeric sort columns for Loyalty Card and Education | Business order |
| Unused columns | Country (one value), City, Postal Code, Marital Status, Salary, year and month pairs | Removed | Not needed for the seven questions |

Full notes, including the column-removal log, are in [`docs/data-decisions.md`](docs/data-decisions.md).

### Data model

A star schema with one fact table and two dimensions:

- **Activity** (fact): one row per member per month.
- **Members** (dimension): one row per member, one-to-many to Activity with single-direction filtering, so slicing by tier or province filters flights and points but activity cannot alter the member list.
- **Calendar** (dimension): marked as the date table and connected to Activity by month start. The two Calendar relationships to Members (enrollment date and cancellation date) are inactive and activated in measures with `USERELATIONSHIP`.
- **CLV** is stored once per member in Members, so Average CLV is averaged over members and never summed across monthly activity rows.

<!-- Add the model screenshot once saved, then uncomment:
![Data model](screenshots/data-model.png)
-->

### Definitions

**Active versus Cancelled:** a member is Cancelled if both Cancellation Year and Month are filled in, and Active if they are blank, on the assumption that a blank means the member was still in the program at the data cut-off of 31 December 2018.

| KPI | Definition | Time basis |
|---|---|---|
| New Enrollments | Members whose enrollment date falls in the selected period | Enrollment month |
| Cancelled Members | Members whose cancellation date falls in the selected period | Cancellation month |
| Active Members | Members with no cancellation date, still in the program at the 31 December 2018 cut-off | As of cut-off, not filtered by period |
| Cancellation Rate | Cancelled Members in the period divided by members at risk (enrolled by the end of the period and not cancelled before it started) | Selected period |
| Total Flights | Flights booked in the selected period, summed from monthly activity | Activity month |
| Points Accumulated | Points earned in the selected period | Activity month |
| Points Redeemed | Points used in the selected period | Activity month |
| Redemption Rate | Points Redeemed divided by Points Accumulated in the same period | Selected period |
| Average CLV | Average customer lifetime value per member in the selected segment, averaged over members and never summed across activity rows | Not time-based |

## Key Findings

1. **Membership is growing, but cancellations are rising.** In 2018, 3,010 members enrolled and 645 cancelled (net +2,365), against 506 cancellations in 2017. Enrollments exceeded cancellations in every month shown, yet December 2018 cancellations still equalled about 30% of that month's enrollments (72 against 238).
2. **Aurora and Nova members cancel at higher rates than Star members.** 2018 cancellation rates were 4.52% (Aurora), 4.47% (Nova) and 3.88% (Star), against 4.21% overall. The gap is about 0.6 points and runs in the same direction in 2017. Education and province extremes rest on 26 or fewer cancellations and flip between years, so they are not treated as reliable.
3. **Aurora holds the most value at risk per member.** Aurora is 20.4% of members at risk but 28.6% of the 5.06M in CLV lost to 2018 cancellations, because each cancelling Aurora member averages about 10,256 in CLV against 6,672 for Star.
4. **Engagement surged in 2018, but points are barely redeemed.** Flights rose 28% (222,547 to 284,613) and points earned rose 38% (333M to 461M). Members redeemed only 1.54% of the points they earned across 2017 and 2018, and the monthly rate fell to about 1.1% to 1.2% in May through August 2018 as earning surged.

Cancelled and active members look very similar on profile and CLV (average CLV 8,132 against 7,969), and flights per member are almost identical across tiers (30.2 to 30.4). Tier and education therefore do not explain who leaves or who flies more.

## Recommendations

1. **Prioritise Aurora and Nova members for retention outreach.** They cancel at the highest rates, and Aurora members carry the highest CLV per cancelled member.
2. **Run a points-redemption campaign.** With only 1.54% of earned points redeemed and the rate dipping through summer 2018, prompting members to use points is a low-cost engagement lever to test.
3. **Treat Ontario as a scale priority.** It holds about a third of lost CLV at close to average risk (4.35%), so a broad Ontario effort covers the most value.

## Limitations

The data cannot say why members cancel, and it has no cost, revenue, route or promotion-spend fields, so no causal claims are made. The 2018 enrollment peak coincides with the "2018 Promotion" enrollment type, but without promotion dates or costs no link is claimed. CLV is lifetime value already earned, not future revenue, so "value at risk" is a proxy. Flight growth may partly reflect a larger membership rather than more flying per member. Blank cancellation fields are assumed to mean the member was still active at the cut-off.

## Dashboard

| Page | Purpose |
|---|---|
| Executive Summary | Health check for the Head of Loyalty: KPI cards and enrollments versus cancellations by year |
| Retention | Enrollments versus cancellations over time, cancellation rate by loyalty card and education, active versus cancelled comparison, CLV lost by segment, with Province, Loyalty Card and Year slicers |
| Engagement and Points | Monthly flights 2017 versus 2018, points accumulated over time, flights per member and redemption rate by loyalty card |

The `.pbix` also contains seven analysis pages (Que 1 to Que 7) used to answer each business question and a Validation Test page used to check measures.

<!-- Add screenshots once saved, then uncomment:
![Executive Summary](screenshots/executive-summary.png)
![Retention](screenshots/retention.png)
![Engagement and Points](screenshots/engagement-and-points.png)
-->

## Tech Stack

- **Power BI Desktop** (.pbix)
- **Power Query (M)** for cleaning and shaping
- **DAX** for measures (time intelligence, status logic, safe ratios, share of total)
- Star-schema data modeling

## Repository Structure

| Path | Contents |
|---|---|
| `Airline_Loyalty_Project.pbix` | The Power BI report: model, DAX measures, analysis pages and dashboard pages |
| `docs/data-decisions.md` | Data-cleaning decisions, model notes, KPI definitions and findings for each question |
| `docs/Project_Brief.docx` | The project brief and checklist this analysis follows |
| `data/README.md` | Expected source files (the raw CSVs are not committed) |
| `screenshots/` | Dashboard and model screenshots |

## Setup / Usage

1. Install **Power BI Desktop** (free, Windows only).
2. Open `Airline_Loyalty_Project.pbix`.
3. To refresh from source, place the CSV files listed in `data/README.md` in a local folder and update the data source paths in Power Query (Transform data, then Data source settings).
4. Start on the **Executive Summary** page and use the slicers on the Retention page to filter by province, loyalty card and year.

## Configuration

No API keys or environment variables are required. The only configuration is the local path to the source CSV files.

## Contributing

This is primarily a personal portfolio project. Suggestions are welcome through issues or pull requests.

## License

All rights reserved. This project is not currently licensed for reuse or redistribution.
