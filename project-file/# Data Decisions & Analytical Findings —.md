# Data Decisions & Analytical Findings — Airline Loyalty Program

**Project:** Keeping Members Flying (Airline Loyalty Program Analysis)  
**Data Coverage:** Membership 2012–2018 | Flight Activity 2017–2018 (Cut-off: 31 December 2018)  
**Tools:** Power Query (M), DAX, Power BI Desktop  

---

## 1. Data Cleaning & Preparation Decisions

### 1.1 Duplicate Rows in Flight Activity
* **Original row count:** 392,936 rows (`Customer Flight Activity`)
* **Identical duplicates removed:** 3,828 extra copies from groups of completely identical rows (all carrying zero flight/points activity), bringing the count to 391,014 rows.
* **Conflicting member-month duplicates resolved:** Within the remaining repeated `(Loyalty Number, Year, Month)` groups (affecting ~135 members), rows differed in activity values. I sorted descending by **Total Flights** and then **Distance**, applied `Table.Buffer` in Power Query to preserve the sort order, and kept the first row per member-month.
* **Final deduplicated count:** **389,065 rows** (3,871 total rows removed: `392,936 − 389,065 = 3,871`), leaving exactly one row per member per month. Verified on member `101902` and confirmed zero remaining duplicate member-months.
* **Why & Limitation:** Summing duplicate rows would double-count flights and points. The source data does not indicate why conflicting duplicate records were generated.

### 1.2 Missing and Negative Salaries
* **Finding:** 4,238 members (25.3%) have a blank `Salary`—representing **100% of the College education group**. Additionally, 20 members (0.12%) have negative salary values.
* **Decision:** Retained all 16,737 members, did not impute synthetic averages, excluded income from the core analysis, and removed the `Salary` column from the model.
* **Why & Limitation:** Deleting rows with missing salaries would wipe out the entire College segment, while imputing averages would fabricate data. Because the cause of the missing College salaries is unknown, any income-based comparison would be biased.

### 1.3 Member Status & Date Grain
* **Status Rule:** Members with both `Cancellation Year` and `Cancellation Month` populated are classified as **Cancelled** (2,067 members, 12.4%); members with blanks are classified as **Active** (14,670 members, 87.6%), assuming they remained enrolled as of the **31 December 2018** cut-off. Both fields are always populated or blank together.
* **Date Construction:** Because source dates are stored as separate year and month integers without day numbers, all enrollment, cancellation, and activity dates were converted to the **first day of each month** (`YYYY-MM-01`). `Cancellation Date` remains `null` for Active members. Validation confirmed **0 cases** where `Cancellation Date` preceded `Enrollment Date`.

### 1.4 Activity Before Enrollment or After Cancellation
* **Activity After Cancellation (Removed):** Found **33,988 rows** dated after a member's cancellation month, all containing zero flights and zero points. Removing them reduced `Activity` from **389,065** to **355,077 rows** without altering any KPI totals. (15,821 of 16,737 members appear in `Activity`, as the 916 members who cancelled prior to 2017 have no records in the 2017–2018 activity window.)
* **Activity Before Enrollment (Kept):** Found **52,874 rows** dated prior to a member's recorded enrollment month, of which **3,199 rows** (across 498 members, ~3%) contain real flight activity totaling **8,207 flights** (~1.6% of all flights). These rows were retained so valid historical flights are not lost due to potential enrollment-date recording lags.

---

## 2. Columns Removed in Power Query

| Table | Column(s) Removed | Reason |
| :--- | :--- | :--- |
| **Members** | `Salary` | Excluded from core analysis due to 25.3% blanks (all in College) and 20 negative values. |
| **Members** | `Country` | Single-value column (`Canada`); provides no analytical segmentation. |
| **Members** | `City`, `Postal Code` | High-cardinality fields not required for reporting questions or slicers. |
| **Members** | `Marital Status` | Redundant alongside `Loyalty Card`, `Province`, `Gender`, `Education`, and `Enrollment Type`. |
| **Members** | `Enrollment Year`, `Enrollment Month` | Replaced by unified `Enrollment Date` (`YYYY-MM-01`). |
| **Members** | `Cancellation Year`, `Cancellation Month` | Replaced by `Cancellation Date` and `Status`. |
| **Activity** | `Year`, `Month` | Replaced by `Activity Date`; time intelligence is supplied by the `Calendar` dimension. |
| **Activity** | `Distance` | Used only as a tie-breaker during deduplication; not part of the core KPIs. |
| **Activity** | `Dollar Cost Points Redeemed` | Excluded from core scope (only needed for optional cost-per-point metrics). |
| **Activity** | `Cancellation Date`, `Keep Flag` | Temporary merge/filter helper columns used to strip post-cancellation zero-activity rows. |

---

## 3. Summary of Data Decisions

| Issue | What Was Found | Action Taken | Rationale & Impact |
| :--- | :--- | :--- | :--- |
| **Duplicate activity rows** | 7,718 rows in repeated member-months (3,828 identical zero-activity copies + conflicting pairs) | Removed exact duplicates; kept highest flights (tie-break: distance) via `Table.Buffer` | Prevents inflated totals (`392,936 → 389,065` rows) |
| **Missing & negative salary** | 4,238 blanks (25.3%, all in College) + 20 negative values | Kept all members; excluded income analysis and dropped `Salary` | Preserves the College segment without inventing data |
| **Activity after cancellation** | 33,988 rows dated after cancellation, all with zero activity | Filtered out post-cancellation zero rows | Keeps fact table lean (`389,065 → 355,077` rows) with zero KPI change |
| **Activity before enrollment** | 52,874 pre-enrollment rows; 3,199 rows (498 members) have 8,207 real flights | Retained all pre-enrollment rows | Avoids deleting genuine flight activity caused by enrollment-date lags |
| **Cancellation fields** | 87.6% blank, always blank/populated in pairs | Derived `Status` (`Active`/`Cancelled`) and `Cancellation Date` | Establishes one consistent churn rule |
| **Date grain mismatch** | Source data is monthly; `Calendar` is daily | Mapped all dates to the 1st of the month (`YYYY-MM-01`) | Aligns fact and dimension tables cleanly at month grain |
| **Text sort order** | `Loyalty Card` and `Education` sort alphabetically by default | Added numeric sort-by columns (`Tier Sort`, `Education Sort`) | Enforces logical business order (`Star → Nova → Aurora`; education levels) |
| **Unused columns** | Single-value, high-cardinality, and superseded helper columns | Removed in Power Query | Reduces model size and simplifies the field list |

---

## 4. Data Model & Architecture

* **Star Schema:** One fact table (`Activity`, grain: 1 row per member per month) connected to two dimension tables: `Members` (16,737 unique members) and `Calendar` (daily dates).
* **Members → Activity (`1:*`, Single Direction):** Filters flow strictly from `Members` to `Activity` so demographic/tier slicers filter flight metrics without activity rows altering member counts.
* **Calendar → Activity (`1:*`, Active):** Links `Calendar[Date]` to `Activity[Activity Date]` (1st of each month) for time-series flight and points analysis.
* **Calendar → Members (`1:*`, Inactive):** Because `Members` holds two date fields (`Enrollment Date` and `Cancellation Date`), both relationships to `Calendar` are kept inactive to prevent ambiguous filter paths, and are activated inside DAX measures via `USERELATIONSHIP`.
* **CLV Granularity:** Customer Lifetime Value (`CLV`) is a member-level attribute stored in `Members`, so `Average CLV` and `CLV Lost` are evaluated over `Members`—never summed across monthly `Activity` rows.
* **Measure Organization:** All DAX measures are centralized in a dedicated `_Measures` table, with raw numeric columns hidden to enforce consistent metric usage.

---

## 5. KPI Definitions

| KPI | Definition | Time Basis |
| :--- | :--- | :--- |
| **New Enrollments** | Count of members whose `Enrollment Date` falls within the selected period (via `USERELATIONSHIP`). | Enrollment month |
| **Cancelled Members** | Count of members whose `Cancellation Date` falls within the selected period (via `USERELATIONSHIP`). | Cancellation month |
| **Active Members** | Count of members with a blank `Cancellation Date`, assumed active as of 31 December 2018. | Point-in-time snapshot (as of cut-off) |
| **Cancellation Rate** | `Cancelled Members ÷ Members at Risk`, where *Members at Risk* are members enrolled on or before the period end and not cancelled before the period began. | Selected period |
| **Total Flights** | Sum of `Total Flights` booked in `Activity` during the selected period. | Activity month |
| **Points Accumulated** | Sum of `Points Accumulated` earned in `Activity` during the selected period. | Activity month |
| **Points Redeemed** | Sum of `Points Redeemed` used in `Activity` during the selected period. | Activity month |
| **Redemption Rate** | `Points Redeemed ÷ Points Accumulated` within the same period (an in-period ratio; redeemed points may have been earned in earlier periods). | Selected period |
| **Flights per Member** | `Total Flights ÷ Total Members` (includes all members in the selected segment, including non-flyers). | Selected period / All-time |
| **Average CLV** | Average Customer Lifetime Value per member in the selected segment (evaluated on `Members`). | Member-level (non-periodic) |
| **CLV Lost** | Total `CLV` of members whose `Cancellation Date` falls in the selected period (proxy for value lost to churn). | Cancellation month |

---

## 6. Key Analytical Findings & Caveats

### Q1: How are enrollments and cancellations changing over time?
* **Setting:** `New Enrollments` and `Cancelled Members` by month, 2012–2018, all tiers.
* **Data Points:**
  * **Enrollments:** Averaged ~180–240/month from 2012 to 2017 before peaking at **346** in early 2018 (3,010 total in 2018; 238 in Dec 2018).
  * **Cancellations:** Started with 1 case in 2013, rose to ~40–54/month in 2016–2017, and peaked at **78** in 2018 (72 in Dec 2018).
  * **Totals:** 916 cancelled through 2016, 506 in 2017, and 645 in 2018 (**2,067 all-time**). Net membership growth in 2018 was **+2,365** (`3,010 − 645`).
* **Finding:** Enrollments outpaced cancellations in every single month from 2012 to 2018, driving steady net program growth (+2,365 in 2018). However, cancellations trended steadily upward over time, reaching ~30% of monthly enrollments by December 2018 (72 cancellations vs. 238 enrollments).
* **Caveat:** The 2018 enrollment spike aligns with the `"2018 Promotion"` enrollment type, though the dataset lacks campaign dates or spend to prove causality.

### Q2: Which segments cancel more?
* **Setting:** Year 2018 `Cancellation Rate` by `Loyalty Card`, `Education`, and `Province` (with 2017 checked for stability).
* **Data Points:**
  * **Loyalty Card (2018):** Aurora **4.52%** (141 cancelled), Nova **4.47%** (232), Star **3.88%** (272) vs. **4.21%** overall (645). In 2017: Aurora 4.2%, Star 3.8%.
  * **Education (2018):** Master **5.54%** (26) highest; High School or Below **2.96%** (21) lowest.
  * **Province (2018):** Prince Edward Island **6.78%** (4) and Newfoundland **5.98%** (14) highest; Nova Scotia **3.16%** (15) lowest. Saskatchewan jumped from 2.0% (2017) to 5.8% (2018).
* **Finding:** Aurora (4.52%) and Nova (4.47%) members cancelled at slightly higher rates than Star members (3.88%) in both 2017 and 2018, though the ~0.6 percentage-point spread is modest. High rates in specific education and provincial subgroups rest on small sample sizes (≤26 cancellations) and fluctuate year-over-year, making them unreliable predictors.

### Q3: How do cancelled members differ from active members in profile and CLV?
* **Setting:** All 16,737 members (all-time), comparing `Cancelled` (2,067; 12.4%) vs. `Active` (14,670; 87.6%) with no filters.
* **Data Points:**
  * **Average CLV:** Cancelled **8,131.78** vs. Active **7,968.76** (+163.02 or ~2.0% higher).
  * **Loyalty Card Share (Active vs. Cancelled):** Star 45.92% vs. 43.59% (−2.3 pts); Nova 33.77% vs. 34.69% (+0.9 pts); Aurora 20.31% vs. 21.72% (+1.4 pts).
  * **Education Share (Active vs. Cancelled):** Bachelor 62.64% vs. 62.22%; College 25.30% vs. 25.50%; all other levels ~12.1% vs. ~12.3%.
* **Finding:** Cancelled and active members have nearly identical demographic profiles and lifetime values. Star is slightly underrepresented among cancellations (−2.3 percentage points) while Aurora is slightly overrepresented (+1.4 points), but static profile attributes alone do not explain churn.
* **Caveat:** This is an all-time comparison, and the dataset does not capture cancellation reasons or customer satisfaction metrics.

### Q4: How do flights and points move month by month, 2018 versus 2017?
* **Setting:** Monthly `Total Flights` and `Points Accumulated`, 2017 vs. 2018, all members.
* **Data Points:**
  * **Seasonal Range:** Lowest in January (13,034 flights in 2017; 13,288 in 2018); highest in July (**26,212** in 2017 vs. **37,815** in 2018, **+44%**).
  * **Full-Year Totals:** Flights rose from **222,547** to **284,613** (**+28.0%**); Points Accumulated rose from **333.26M** to **460.92M** (**+38.3%**).
  * **Implied Points per Flight:** Increased ~8% from ~1,497 (2017) to ~1,619 (2018).
* **Finding:** Both years followed an identical seasonal curve—troughing in January–February, surging to a July summer peak, dipping in autumn, and rebounding in December. After tracking closely through Q1, 2018 pulled sharply ahead from May onward (+44% in July), lifting full-year flights by 28% and points earned by 38.3%.
* **Caveat:** Part of the 2018 volume surge reflects a larger active member base rather than purely higher individual flight frequency, and the dataset does not explain the ~8% rise in points earned per flight.

### Q5: How does engagement differ by loyalty tier and other segments?
* **Setting:** `Flights per Member` (`Total Flights ÷ Total Members`, across all 16,737 members), 2017–2018 combined, unfiltered.
* **Data Points:**
  * **Loyalty Card:** Aurora **30.41**, Nova **30.36**, Star **30.21** (Overall: **30.30**).
  * **Education:** High School or Below **30.84** highest vs. Bachelor **30.17** lowest.
  * **Province:** Yukon **33.33** highest vs. Prince Edward Island **27.62** lowest; major provinces (ON, QC, BC, AB) all cluster tightly between **30.21** and **30.54**.
* **Finding:** Engagement is virtually uniform across loyalty tiers (a spread of just 0.20 flights per member between Aurora and Star) and education levels. Larger provincial variances occur only in low-population territories/provinces (Yukon, PEI), indicating that tier status does not differentiate flying frequency.
* **Caveat:** The denominator includes all 16,737 members on record—including the 916 pre-2017 cancellations and partial-year 2018 enrollees—making this a book-wide average rather than an active-flyer average.

### Q6: How do points earned compare with points redeemed, and which segments redeem more?
* **Setting:** `Redemption Rate` (`Points Redeemed ÷ Points Accumulated`), 2017–2018, all members.
* **Data Points:**
  * **Overall:** **12.23M** points redeemed out of **794.18M** earned (**1.54%**).
  * **By Segment:** Loyalty Card (Star **1.56%**, Aurora **1.53%**, Nova **1.51%**); Education (Doctor **1.64%** to High School or Below **1.50%**); Province (PEI **1.74%** to Yukon **1.28%**, with most provinces between **1.50% and 1.60%**).
  * **Monthly Trend:** Held steady at ~1.5%–1.8% throughout 2017. In 2018, started at ~2.1% (Jan) and ~2.0% (Apr), dipped to **1.1%–1.2%** during May–August, and recovered to ~1.6% in autumn.
* **Finding:** Overall redemption is very low at **1.54%** and varies minimally across tiers or education levels. The sharp dip to 1.1%–1.2% in summer 2018 occurred because point accumulation surged during peak summer travel while redemptions did not scale proportionally.
* **Caveat:** Because monthly redemptions are divided by points earned in the same month (even though redeemed points may have been banked earlier), rapid summer earning mechanically depresses the monthly ratio.

### Q7: Where is the most value at risk, and which segments should the team prioritise?
* **Setting:** Year 2018 `CLV Lost` (total CLV of members who cancelled in 2018) and `Cancellation Rate` across `Loyalty Card`, `Education`, and `Province`.
* **Data Points:**
  * **Overall (2018):** **645** cancelled members | **5.06M** CLV lost | **4.21%** cancellation rate.
  * **Aurora:** 20.4% of members at risk | **4.52%** rate | **1.45M** CLV lost (**28.6%** of total) | Avg cancelled CLV: **10,256**.
  * **Nova:** 33.9% of members at risk | **4.47%** rate | **1.80M** CLV lost (**35.6%** of total) | Avg cancelled CLV: **7,773**.
  * **Star:** 45.8% of members at risk | **3.88%** rate | **1.81M** CLV lost (**35.8%** of total) | Avg cancelled CLV: **6,672**.
  * **Scale Segments:** Ontario accounts for 32.3% of members at risk and **1.67M** CLV lost (33.0% of total, 4.35% rate); Bachelor accounts for 62.6% of members and **3.36M** CLV lost (66.4% of total, 4.20% rate).
* **Finding:** Retention efforts should prioritize **Aurora** first and **Nova** second. Aurora represents just 20.4% of at-risk members yet drives **28.6% of all CLV lost** because each departing Aurora member carries an average CLV of **10,256** (54% higher than Star's 6,672) alongside the highest tier cancellation rate (4.52%). Star generates the largest raw dollar loss (1.81M) solely due to its larger member base.
* **Caveat:** Tier cancellation rate gaps (~0.6 percentage points) are directional rather than dramatic, and historical CLV serves as a proxy for future value at risk. Small-sample education/province outliers were excluded due to year-over-year volatility, and no retention-cost data is available to model campaign ROI.