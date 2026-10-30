# Data Decisions and Findings

Working notes for the Airline Loyalty Retention and Engagement Analytics project. This document records every decision that changes a result, the model and KPI definitions, and the finding and caveat for each of the seven business questions.

## 1. Data preparation

### 1.1 Duplicate rows in Customer Flight Activity

| Step | Rows |
|---|---|
| Original | 392,936 |
| After removing fully identical rows | 391,014 |
| After resolving member-months that differ | 389,065 |

Total removed: 3,871 rows (392,936 to 389,065).

**Rule.** About 3,828 rows sat in groups of fully identical copies, all with zero activity, so the extra copies were removed. For the remaining groups that shared a member, year and month but differed in values (about 135 members), rows were sorted by Total Flights (high first), then Distance (high first), and the first row was kept. `Table.Buffer` was used so the sort order was preserved before removing duplicates on member, year and month. The result has exactly one row per member per month, verified on member 101902 and by confirming no repeated member-months remain.

**Why.** Summing the duplicates would inflate totals.

**Limitation.** The data does not show why the conflicting rows exist.

### 1.2 Missing and negative salary

4,238 members (25.3%) have no salary, all of them in the College education group (100% of that group). 20 members (0.12%) have negative salary. Deleting rows would remove every College member and filling in averages would invent data, so all members were kept and income was excluded from the main analysis.

**Limitation.** The data does not show why College salaries are missing, so any income comparison would be unreliable.

### 1.3 Active versus Cancelled, and dates

A member is **Cancelled** if both Cancellation Year and Cancellation Month are filled in, and **Active** if they are blank, on the assumption that a blank means the member was still in the program at the data cut-off of 31 December 2018.

Enrollment and cancellation are stored as separate year and month numbers. Each pair was combined into a real date using the first day of the month, because the data has no day. Cancellation Date is blank for active members. No cancellation date is earlier than its enrollment date (0 cases among cancelled members).

### 1.4 Activity before enrollment or after cancellation

- **Before enrollment:** 52,874 rows, of which 3,199 (across 498 members, about 3%) have real flights, totalling 8,207 flights or about 1.6% of all flights. Rows were kept. For these members the enrollment date does not mark the start of their activity, and the data does not say why.
- **After cancellation:** 33,988 rows, all with zero activity. Rows were removed.

Flight Activity row path: 392,936, then 389,065 after removing duplicates, then 355,077 after removing the zero-activity rows dated after cancellation. 15,821 of 16,737 members remain in the activity table, because members who cancelled before 2017 have no activity in the data window.

### 1.5 Columns removed

| Query | Column removed | Reason |
|---|---|---|
| Members | Salary | Excluded from the Must Have analysis: 25% blank (all in the College group) and 20 negative values |
| Members | Country | Only one value (Canada), so it cannot split any analysis |
| Members | City | Not used by any of the seven questions or slicers, and too many values to chart |
| Members | Postal Code | Same as City, with even more values |
| Members | Marital Status | Not needed: Province, Gender, Education and Enrollment Type already give enough segments |
| Members | Enrollment Year, Enrollment Month | Replaced by the Enrollment Date column |
| Members | Cancellation Year, Cancellation Month | Replaced by Cancellation Date and Status |
| Activity | Year, Month | Replaced by Activity Date; the Calendar supplies year and month |
| Activity | Distance | Not one of the nine KPIs |
| Activity | Dollar Cost Points Redeemed | Only needed for an optional KPI |
| Activity | Cancellation Date, Keep Flag | Temporary helper columns used to remove the after-cancellation rows |

### 1.6 Decisions table

| Issue | What I found | What I did | Why |
|---|---|---|---|
| Duplicate activity rows | 7,718 rows in repeated member-months, of which 3,828 were identical, all with zero activity | Removed identical copies. For differing pairs, kept the row with the most flights (then highest distance), using Table.Buffer to preserve the sort | Summing duplicates would inflate totals. Result: 392,936 to 389,065 rows |
| Missing and negative salary | 4,238 blanks (25.3%), all in the College group, plus 20 negatives | Kept all members, did not fill in values, excluded income from the analysis and removed Salary | Deleting rows would remove a whole education group, and averages would invent data |
| Activity after cancellation | 33,988 rows, all zero | Removed them | Totals unchanged. Result: 355,077 rows |
| Activity before enrollment | 52,874 rows, of which 3,199 (498 members) have flights, 8,207 in total (about 1.6%) | Kept them | Real activity should not be deleted when the enrollment date may be the error |
| Cancellation fields | 88% blank, always blank together | Built Status and real dates | One consistent rule |
| Date grain | Monthly data, daily Calendar | First day of the month for all dates | Matches the grain of the data |
| Sort order | Text sorts alphabetically | Added numeric sort columns for Loyalty Card and Education | Business order |
| Unused columns | Country (one value), City, Postal Code, Marital Status, Salary and the year and month pairs | Removed | Not needed for the seven questions |

## 2. Data model

**Structure.** A star schema: one fact table, Activity (one row per member per month), and two dimensions, Members (who) and Calendar (when).

**Members to Activity.** Each member appears once in Members and many times in Activity, so the relationship is one-to-many. Filters flow in one direction only, from Members to Activity, so choosing a loyalty tier or province filters flights and points, while activity can never alter the member list.

**Calendar to Activity.** Calendar[Date] is unique for every day, so it is the "one" side. Activity Date holds the first day of each month, so each month's activity connects to its month without needing daily data.

**Enrollment and cancellation dates.** Members has two dates, but only one active path is allowed between two tables. Both Calendar relationships to Members are therefore inactive, and measures switch them on with `USERELATIONSHIP`. This avoids a second route from Calendar to Activity through Members.

**CLV.** CLV is stored once per member in Members, so Average CLV is calculated over the member table and never summed across monthly activity rows, which would count each member's lifetime value once per month.

**Measures.** All measures sit in one dedicated table, and the raw numeric columns are hidden so reports use measures only.

## 3. KPI definitions

| KPI | Definition | Time basis |
|---|---|---|
| New Enrollments | Members whose enrollment date falls in the selected period | Enrollment month |
| Cancelled Members | Members whose cancellation date falls in the selected period | Cancellation month |
| Active Members | Members with no cancellation date, assumed still in the program at the data cut-off of 31 December 2018 | As of the cut-off, not filtered by period |
| Cancellation Rate | Cancelled Members in the period divided by members at risk. At risk means enrolled by the end of the period and not cancelled before the period started | Selected period |
| Total Flights | Flights booked by members in the selected period, summed from monthly activity | Activity month |
| Points Accumulated | Points earned by members in the selected period | Activity month |
| Points Redeemed | Points used by members in the selected period | Activity month |
| Redemption Rate | Points Redeemed divided by Points Accumulated in the same period. Redeemed points may have been earned earlier, so this is an in-period ratio | Selected period |
| Average CLV | Average customer lifetime value per member in the selected segment, averaged over Members and never summed across monthly activity rows | Not time-based |

**Time basis.** Total Members, Active Members and Cancelled Members (All Time) are as of the cut-off and do not change with the period filter. Flights, points, New Enrollments, Cancelled Members, Members at Risk and Cancellation Rate follow the selected period.

## 4. Findings

### Q1. How are enrollments and cancellations changing over time?

**Setting.** New Enrollments and Cancelled Members by start of month, 2012 to 2018, all tiers.

- Enrollments: mostly about 180 to 240 a month from 2012 to 2017, then a peak of 346 in early 2018. December 2018 was 238.
- Cancellations: first appear in 2013 (1 in the earliest month), about 40 to 54 a month in 2016 and 2017, a peak of 78 in 2018, and 72 in December 2018.
- Totals: 916 cancelled up to 2016, 506 in 2017 and 645 in 2018 (2,067 all time). In 2018, 3,010 members enrolled, so net growth was +2,365.
- December 2018: 72 cancellations against 238 enrollments, about 30%.

**Finding.** Enrollments exceeded cancellations in every month shown, so membership kept growing, though December 2018 cancellations still equalled about 30% of that month's enrollments.

**Caveat.** The 2018 peak coincides with the "2018 Promotion" enrollment type, but the data has no promotion dates or costs, so no cause is claimed.

### Q2. Which segments cancel more?

**Setting.** Year 2018; Cancellation Rate by Loyalty Card, Education and Province; 2017 checked for Loyalty Card.

- Loyalty Card: Aurora 4.52% (141 cancelled), Nova 4.47% (232), Star 3.88% (272). Overall 4.21% (645). In 2017: Aurora 4.2%, Star 3.8%.
- Education: Master 5.54% (26) highest, High School or Below 2.96% (21) lowest.
- Province: Prince Edward Island 6.78% (4) and Newfoundland 5.98% (14) highest, Nova Scotia 3.16% (15) lowest.
- Stability check: Saskatchewan was 2.0% in 2017 and 5.8% in 2018.

**Finding.** Aurora and Nova members cancelled at higher rates than Star members, and Aurora and Star show the same direction in 2017, but the gap is only about 0.6 points. The education and province extremes rest on 26 or fewer cancellations and flip between years, so they are not reliable.

### Q3. How do cancelled members differ from active members in profile and CLV?

**Setting.** All members, all time, Status as the comparison field, no filters.

- Members: Cancelled 2,067 (12.4%), Active 14,670 (87.6%).
- Average CLV: Cancelled 8,131.78, Active 7,968.76, a gap of 163.02 (about 2%).
- Loyalty Card, Active against Cancelled: Star 45.92% vs 43.59%, Nova 33.77% vs 34.69%, Aurora 20.31% vs 21.72%.
- Education: College 25.30% vs 25.50%, Bachelor 62.64% vs 62.22%, all other levels combined about 12.1% vs 12.3%.

**Finding.** Cancelled and active members look very similar on profile and value. The largest gap is in loyalty card (Star is 2.3 points lower among cancelled members; Aurora is 1.4 points higher). These measures do not separate who leaves, so any explanation needs information the data does not contain.

**Caveat.** This is an all-time comparison, not a single period, and the data does not show why members cancel.

### Q4. How do flights and points move month by month, 2018 versus 2017?

**Setting.** Total Flights and Points Accumulated by month, Year as legend, all members, no slicers.

- July (peak in both years): 26,212 flights in 2017, 37,815 in 2018, up about 44%.
- January (lowest): 13,034 in 2017, 13,288 in 2018.
- Full year: flights 222,547 to 284,613 (+28%); points accumulated 333,264,227 to 460,919,354 (+38.30%).
- Points per flight (calculated from the totals): about 1,497 to 1,619 (+8%).

**Finding.** Flights and points followed the same seasonal shape in both years. 2018 started level with 2017 but ran well above it from May onward.

**Caveat.** Points rose faster than flights, and the data cannot explain why. Membership also grew, so part of the increase may reflect more members, not more flying per member. This analysis does not separate the two.

### Q5. How does engagement differ by loyalty tier and one other segment?

**Setting.** Flights per Member (Total Flights divided by Total Members, all 16,737 members), 2017 and 2018 combined, no filters.

- Loyalty Card: Aurora 30.41, Nova 30.36, Star 30.21. Overall 30.30.
- Education: High School or Below 30.84 highest, Bachelor 30.17 lowest.
- Province: Yukon 33.33 highest, Prince Edward Island 27.62 lowest. Ontario, Quebec, British Columbia and Alberta all sit between 30.21 and 30.54.

**Finding.** Members flew an average of 30.3 times across 2017 and 2018, almost identically across tiers, with education adding little. The only larger gaps are in provinces with small member counts, so tier and education do not explain who flies more.

**Caveat.** The denominator counts every member on the books, including the 916 who cancelled before 2017 and members who joined during 2018 and had only part of the period. It was applied the same way to every segment, so the figure is an average per member on the books, not per active flyer.

### Q6. How do points earned compare with points redeemed, and which segments redeem more?

**Setting.** Redemption Rate, 2017 and 2018, all members. Monthly view by year, plus Loyalty Card, Education and Province.

- Overall: 12.23M redeemed of 794.18M earned, 1.54%.
- Loyalty Card: Star 1.56%, Aurora 1.53%, Nova 1.51%.
- Education: Doctor 1.64% highest, High School or Below 1.50% lowest.
- Province: Prince Edward Island 1.74% highest, Yukon 1.28% lowest. Most provinces between 1.50% and 1.60%.
- By month: 2017 stayed within about 1.5% to 1.8%. 2018 started above it (about 2.1% in January, about 2.0% in April), fell to about 1.1% to 1.2% in May through August, and recovered to about 1.6% in the autumn.

**Finding.** Members redeemed only 1.54% of the points they earned, with almost no difference by tier or education. In 2018 the rate fell from about 2.0% in April to about 1.1% to 1.2% in May through August as earning surged, so redemption did not keep pace with earning in summer 2018.

**Caveat.** The rate compares points redeemed in a month with points earned in that month, and redeemed points may have been earned earlier, so a surge in earning lowers the rate mechanically. The data cannot say why members redeemed less. The province extremes rest on small point totals.

### Q7. Where is the most value at risk, and which segments should the team prioritise?

**Setting.** Year 2018 (2017 checked). CLV lost = total CLV of members who cancelled in the year, used as a proxy for value at risk.

- Overall: 645 cancelled members, 5.06M CLV lost, 4.21% cancellation rate.
- Aurora: 20.4% of members at risk, 4.52% rate, 1.45M lost (28.6% of the total). Average cancelled CLV 10,256.
- Nova: 33.9%, 4.47%, 1.80M (35.6%). Average cancelled CLV 7,773.
- Star: 45.8%, 3.88%, 1.81M (35.8%). Average cancelled CLV 6,672.
- Ontario: 32.3% of members, 4.35% rate, 1.67M (33.0%).
- Bachelor: 62.6% of members, 4.2% rate (average), 3.36M (66.4%).

**Finding.** Prioritise Aurora and Nova members. Aurora is the smallest tier yet accounts for 28.6% of the CLV lost in 2018, because each cancelling Aurora member is worth about 10,256 against 6,672 for Star, and Aurora and Nova cancel at the highest rates. Star loses the most in total only because it is the largest tier. Ontario is a third priority only because of scale.

**Caveat.** The tier gaps are modest (about 0.6 points) and run the same direction in 2017, but they are not strong enough to call conclusive. CLV is lifetime value already earned, not future revenue, so "value at risk" is a proxy. Education and province extremes rest on 26 or fewer cancellations, and some flip between years, so they were excluded. Bachelor holds two thirds of lost value only because it is two thirds of members. The data cannot say why members cancel, and it has no cost data to size a retention budget.
