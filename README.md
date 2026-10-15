# Resource Allocation and Budget Analysis in Three TAWOS Projects

How does the seniority mix of a team relate to what it delivers, what it costs and how often work comes back? This project analyzes ~20,800 real Jira issues from three open-source projects (Mule, Titanium Mobile Platform, Appcelerator Studio). It uses earned value management (EVM), burn-up, effort concentration, rework, lateness and bus-factor measures, and presents the results as a five-dashboard Tableau story.

**Stack:** Python (pandas, NumPy, SciPy, Simpful), MySQL 8.0 in Docker, SQLAlchemy, Jupyter, Tableau.
**Dashboards:** [Tableau Public story](https://public.tableau.com/views/FullProjectAllocationBudget_Dashboards_04/HoursUtil?:language=en-US&publish=yes&:sid=&:redirect=auth&:display_count=n&:origin=viz_share_link)

---

## 1. General information

**Project name:** Resource Allocation and Budget Analysis in Three TAWOS Projects
**Period covered:** 2004 to 2020 (Mule from 2004, Appcelerator Studio and Titanium from 2011)

### Where the datasets come from

| Dataset | Source | What it is |
|---|---|---|
| TAWOS | [github.com/SOLAR-group/TAWOS](https://github.com/SOLAR-group/TAWOS) | Public collection of real Jira projects. Loaded from `TAWOS.sql` (about 4.3 GB) into MySQL |
| Enterprise HRA Allocation Dataset | [kaggle.com/datasets/ziya07/enterprise-human-resource-allocation-dataset](https://www.kaggle.com/datasets/ziya07/enterprise-human-resource-allocation-dataset) | 5,000 synthetic employee records with 40 columns (experience, productivity, training, attendance, workload) |

Check each source's license before reusing the data.

### Read this first: what is real, what is modeled, what is synthetic

| Real (TAWOS) | Modeled from real activity | Synthetic |
|---|---|---|
| Issues, types, priorities, story points | **Seniority tiers**: computed from each contributor's tenure, issue volume, story points and share of high-priority work. They are relative ("top fifth of this dataset's core team"), not years of experience | Employee HR profiles (HRA), matched to the real contributors |
| Creation, sprint and resolution dates | Planned hours (story points × a rolling hours-per-point baseline) | Hourly rates (a band per tier) |
| Change log (status, assignee and story-point changes), comments, components | Capped effort hours (720 h per issue at most) | **Project budgets** (built to give three different situations) |

What this means for any result:

- **Budget outcomes are by design, not findings.** Titanium is over budget and Mule under because the budgets were built that way. What carries information is the path: when each project crossed its budget and how fast.
- **Hours are elapsed ticket time**, capped at 720 h per issue, so costs are upper-bound estimates, not real spend.
- **The Senior tier is partly built from issue volume**, so the most prolific contributors are Senior by definition.
- **There is no real schedule in the data.** SPI measures pace against an even-pace line, and "late" means resolved after the sprint ended.
- **Coverage is partial for some questions:** lateness uses 24% of issues, planned vs actual 28%, and Appcelerator's sprints hold only 38% of its pointed issues.
- Everything is association, not cause.

---

## 2. Goal

Show, with real Jira data and a clearly labeled synthetic layer, how a project manager can read **cost, pace, rework and people concentration** across three projects, and where the data runs out.

### Questions this project answers

1. Which projects performed best, and on which measures?
2. Does a higher Senior share raise output linearly, or does composition matter more?
3. How is ownership of core modules distributed across the team?
4. Which project had the most delivery delays?
5. What is the Senior-to-Junior ratio in each project?
6. When did each project cross its budget, and how did its pace change?
7. How much of the completed work was added after the sprint started?
8. How concentrated is the effort in a few people?
9. Is there a Senior review bottleneck?
10. Why does work finish late?

### Who this is for

- **PMO and portfolio managers:** compare budget, cost efficiency and pace across projects, and see why a final CPI or SPI should be read as a path, not a verdict.
- **Engineering and delivery managers:** see where work depends on one or two people, how scope grows inside sprints and whether review is a bottleneck.
- **Resource managers and HR business partners:** see how effort is spread across seniority tiers and what a tier-based staffing analysis can and cannot show.
- **Hiring managers reviewing this portfolio:** a worked example of data cleaning, modeling with synthetic data, validation and honest limits.

---

## 3. Work plan

The steps follow the notebooks.

1. **Load the data** (`00_Docker_Setup`): run MySQL 8.0 in Docker, import the TAWOS dump, export the tables of the three projects to CSV.
2. **Explore, standardize and clean** (`00_Docker_Setup`, Part 2): explore every table (shape, types, nulls, value counts), standardize dates and labels, decide how each null is treated (see 5.1), join issues with sprints, projects, comments and components, and rebuild story-point history from the change log.
3. **Build seniority tiers** (`01_hraFusion`): split contributors into Core, One-off and Other, score each one and place them in five tiers.
4. **Score the synthetic HR profiles** (`01_hraFusion`): choose columns, build four group scores, and turn them into five tiers with fuzzy logic.
5. **Match and price** (`01_hraFusion`): pair each contributor with an HRA employee of the same tier, set hourly rates, recover owners of unassigned issues and label every issue's owner category.
6. **Cost, planned hours, EVM and budgets** (`02_ratesEVMandBudgets`): cap extreme effort, calculate cost, build the per-sprint EVM, simulate three budgets and recompute the EVM in USD.
7. **Final exploration and analysis** (`FullProjectAllocationBudget`): re-explore the final datasets (types, dates, nulls, reconciliation against the source files), then analyze overview financials and EVM, burn-up, effort by type and tier, effort concentration, seniority mix vs throughput, review bottleneck, bug density and rework, planned vs actual, lateness, mentorship and bus factor.
8. **Dashboards** (Tableau): one overview and four detail dashboards. Every Python calculation feeds a chart.

---

## 4. Key questions about the data

| Question | Short answer |
|---|---|
| Where do the issues come from? | TAWOS: real Jira projects Mule, Titanium Mobile Platform and Appcelerator Studio, loaded from a public SQL dump |
| Why TAWOS? | It has what a resource analysis needs: assignees, dates, sprints, story points and a change log with 462,429 entries (reassignments, reopens, estimate changes) |
| What does TAWOS lack? | Job roles, experience, pay and budgets. There is no planned schedule either |
| Where do the employee profiles come from? | The HRA dataset: 5,000 synthetic employees. 902 hold a tech role, and 547 (Software Engineer, QA Engineer, System Administrator) are the pool for matching |
| Why use synthetic data? | Real HR records, pay and budgets are sensitive and not public. Synthetic stand-ins let the analysis show how the solution works without exposing anyone |
| How were real and synthetic data connected? | Through the seniority tier. Each of the 182 scored contributors received one HRA employee of the same tier, which gave the profile and the hourly rate |
| Which TAWOS tables are used? | Issue (20,854), Sprint (691), Change_Log, Comment (44,219), Component (191) and Issue_Component (20,467), Project, User |
| Which are loaded but not used here? | Version, Fix_Version, Affected_Version, Issue_Link and Repository |
| Which datasets does the project produce and use? | Five final model datasets, plus intermediate files and Tableau exports (table below) |

### Datasets used in the project

| Stage | File | Size | Role |
|---|---|---|---|
| Raw (TAWOS) | `Issue`, `Sprint`, `Change_Log`, `Comment`, `Component`, `Issue_Component`, `Project`, `User` | 20,854 · 691 · 462,429 · 44,219 · 191 · 20,467 · 3 · 2,524 rows | Exported from MySQL by `00_Docker_Setup`. Source of every real fact |
| Raw (synthetic) | `enterprise_hra_allocation_dataset.csv` | 5,000 × 40 | Employee profiles, read by `01_hraFusion` |
| Intermediate | `main_part2.csv` | 20,854 × 32 | Issue-level table after exploration, cleaning and joins |
| Intermediate | `main_part3.csv` | 20,854 × 52 | Same table with seniority tiers, HR profile, rates and owner categories |
| Intermediate | `team_tier_matching.csv`, `hra_tech_scored.csv`, `team_profiles.csv` | 182 · 902 · 182 rows | Tier matching, scored HR pool and contributor profiles (`team_profiles.csv` is renamed `team_profiles_v1.csv`) |
| **Final model datasets** | `projects_dataset.csv` | 20,854 × 67 | One row per issue, with tier, hours, cost, planned hours. Main Tableau source |
| | `evm_sprint.csv` | 549 × 46 | One row per sprint: scope, EV, AC, BAC, PV, SPI, CPI, EAC, VAC |
| | `project_budgets.csv` | 3 × 12 | Simulated budgets, variance and status |
| | `cost_monthly.csv` | 407 × 4 | Monthly and cumulative capped cost per project |
| | `team_profiles_v1.csv` | 182 × 70 | One row per scored contributor, with tier, rate and HR profile |
| Analysis exports (Tableau) | `A_throughput_sprint.csv` | 564 rows | One row per sprint: heads, Senior share, output per head |
| | `A_review_stints.csv` | 1,024 rows | One row per visit of an issue to a review status |
| | `B_reopen_counts.csv` | 2,627 rows | Reopen events per issue |
| Tableau export (02) | `evm_monthly.csv` | 19 columns | Monthly version of the USD EVM (budget, cost, PV, EV, CPI, EAC, VAC) |
| | `C_bus_factor_year.csv`, `C_mentorship_span.csv` | one row per project and year | Bus factor and learners per mentor |

---

## 5. What has been done and Connections

### 5.1 Data and cleaning
- MySQL 8.0 runs in Docker Desktop. The dump is imported into a database `tawos_cleaned`, and the tables of the three projects are exported to CSV.
- The base table is **one row per issue**. Sprint, project, comment count and component count are joined on.
- TAWOS's own flag for story points changed after estimation caught only about 17% of the real revisions (1,004 issues against 5,787 in the change log). Story-point history is therefore rebuilt from the change log.
- 57.5% of issues have no story points. They are mostly reactive bug work and carry 51.8% of the effort, so they stay in cost but not in earned value.
- 1,867 issues (9%, almost all in Mule) have no assignee. For 263 the previous owner is recovered from the change log. The rest are kept as their own category, split into "worked, owner unknown" (133) and "not worked or still open" (1,471).
- 6,805 issues (32.6%) were truly reassigned. Reassignment is defined as an assignee change where both the old and the new owner are real people.

**Exploration, standardization and nulls.** Every table was explored first (shape, types, null counts, value counts, correlations). Then it was standardized: dates parsed to datetime, text statuses normalized (for example "In  Review" with two spaces equals "In Review"), issue types grouped into four comparable groups, scores rescaled to 0 to 1, and one tier vocabulary used everywhere. Checks after every merge: unique issue IDs, expected row counts, no duplicated or leftover `_x`/`_y` columns, and totals reconciled against the source files. Nulls were never dropped silently. Each one got an explicit decision:

| Null | Count | Decision |
|---|---|---|
| Story points and estimation date | 8,873 filled, 57.5% empty (they coincide exactly) | Treated as unplanned work, mostly bugs. No planned hours or earned value, but hours stay in cost |
| Resolution date | 1,636 open issues | Left out of the monthly cost series and the year charts (an "Open" bucket where shown). Their cost stays in project totals |
| Sprint and sprint dates | 14,673 issues without a sprint, 277 sprinted issues without dates | Excluded from sprint-based measures (EVM, lateness). Coverage is reported next to each result |
| Assignee | 1,867 issues (98% in Mule) | 263 recovered from the change log. The rest kept as their own category and priced at the core-team median rate. The ID is set to -1 only when no owner can be recovered |
| Seniority tier | 1,959 issues without a tiered owner | Labeled "No tier (assumed rate)". A tier is never imputed |
| Contributor average story points and high-priority share | People with no solo-owned issue | Filled with the median of the pool |
| Empty or unreliable columns | `Pull_Request_URL` (all empty), `Description_Code` (89% empty), `URL`, `Jira_ID`, TAWOS's "changed after estimation" flags | Dropped. Long text columns are also dropped before Tableau |
| Early sprints | Fewer than 100 completed points before them | Kept in the table as "No baseline yet", excluded from CPI and SPI. SPI is blank in the first 5% of the timeline and EAC/VAC until 10% of points are delivered |

### 5.2 Seniority tiers from real activity
- **Core** = 10 or more issues and 30 or more days of activity (97 people). **One-off** = exactly one issue (85). **Other** = anyone else (89, no tier). In total 271 assignees.
- Four scores, each rescaled to 0 to 1: tenure, issue volume, average story points and share of high-priority work (Critical, Blocker, High, Major), the last two only from issues the person owned alone.
- Seniority score = simple average of the four, because there is no ground truth for the weights.
- Five tiers (Junior, Junior-Mid, Mid, Mid-Senior, Senior) cut at the 20th, 40th, 60th and 80th percentile of the **core team's** scores and applied to everyone. One-offs land mostly in the lower tiers and are marked as low confidence.

### 5.3 Synthetic HR profiles and fuzzy logic
- Only columns about a person's track record were used (performance, experience, growth, reliability). Columns about the current situation (workload, availability) were left out.
- A correlation matrix removed duplicated information (for example KPI score and performance rating correlate at 1.00, attendance and absenteeism risk at −0.81). A skewed column was scaled by percentile rank instead of min-max.
- Four group scores feed a **fuzzy system** (Simpful, Mamdani inference): 81 rules, sets anchored on each group's own percentiles, and one gate (low experience caps the result at Mid). The score is split into the same five tiers.
- Against a plain weighted average, 75% of employees land in the same tier. 221 of 902 change tier, 216 by one step.

### 5.4 Matching, rates and cost
- **Matching:** each contributor receives one unused HRA employee of the same tier, drawn at random (seed 42), after checking that every tier has enough supply. The pairing is arbitrary inside a tier, so conclusions hold at tier level only.
- **Rates:** a band per tier (Junior 15–20, Junior-Mid 20–26, Mid 26–33, Mid-Senior 33–42, Senior 42–55 USD per hour). The position inside the band follows the seniority score. Issues without a tiered owner use the core-team median rate and are labeled as assumed (about 2% of cost).
- **Cap:** effort is capped at 720 h per issue. In 509 tickets the recorded effort behaves like elapsed time (a median of 79% of the ticket's whole life), and they hold 51% to 80% of resolved cost. Raw values are kept, flagged and reported next to the capped ones.

### 5.5 Earned value
- Per sprint, with scope rebuilt as of each sprint's start and end date from the change log.
- **EV** = points completed. **Baseline** = capped hours per completed point over all earlier sprints (at least 100 completed points). **AC** = capped hours of finished issues. **CPI** = earned hours ÷ actual hours.
- **PV is time-based:** a straight line from the first priced sprint to the sprint where the final earned points were reached. SPI = % done ÷ % of time elapsed. A first version used each sprint's starting scope as the plan and was dropped, because scope growth alone pushed SPI to 1.3 to 1.5.
- Issues closed without delivery (Won't Fix, Duplicate, Cannot Reproduce and similar) leave the scope on their closing date, earn nothing, and their hours stay in cost.
- **USD version:** the same EVM with the simulated budget as BAC, counted from the first priced sprint. Because the first priced sprint comes after some of the cost was already incurred (23% of Appcelerator's cost, 37% of Mule's and 33% of Titanium's), the BAC is scaled to the measured window, that is, to the cost from the first priced sprint onward.

### 5.6 Simulated budgets
Three situations were built from each project's own capped cost: Mule at 120% (comfortable), Titanium at 85% (tight) and Appcelerator a fixed annual amount rounded to $100K. The variances follow from those choices.

### 5.7 Analyses and statistics
- **Seniority mix vs output:** points per person against Senior share in sprints with 3 or more people (238 sprints), straight line vs quadratic (AIC), 1,000 bootstrap resamples for the intervals.
- **Review bottleneck proxy:** hours spent in review-like statuses, by owner tier, and months with more reviews per Senior against wait time (Spearman).
- **Rework:** reopen events per 100 resolved issues, split into bugs and other work.
- **Lateness:** days between resolution and sprint end, a year-adjusted comparison by tier, and three "not the cause" tests (bug share, sprint size, sprint length).
- **Concentration:** effort Pareto, bus factor by project and year, and by component.

### 5.8 Tableau
Five dashboards connected to `projects_dataset` and the CSV exports. For bus factor and mentorship, Python is the source of truth and Tableau only displays its tables, because table calculations gave different numbers depending on layout.

### Techniques at a glance

| Area | Techniques |
|---|---|
| Data engineering | Docker, MySQL, SQLAlchemy, pandas joins, reconciliation checks against source files |
| Modeling | Min-max and percentile-rank scaling, percentile cut points, median imputation, correlation screening |
| Synthetic layer | Fuzzy logic (Simpful), one-to-one random matching with a fixed seed, rate bands |
| Project controls | Per-sprint EVM, CPI, SPI, burn-up, scope reconstruction from the change log, effort capping |
| Statistics | Quadratic vs linear fit (AIC), bootstrap intervals, Spearman correlation, year-adjusted comparison |
| Visualization | Tableau dashboards, LOD calculations, relationships between tables |

---

## 6. Results

### Q1. Which projects performed best?
No project wins on every measure.

| Measure | Appcelerator | Mule | Titanium |
|---|---|---|---|
| Cost efficiency (CPI, hours) at the end | 0.43 | **0.97** | 0.61 |
| Delivered vs sprint scope | 97% | 102% | 109% |
| Issues resolved after sprint end | **20%** | 34% | 27% |
| Reopen events per 100 resolved issues | about 36 | about 12 | **about 7** |
| People needed for 80% of the hours | 5 | **23** | 5 |

Mule held its cost per point close to its own baseline and has the broadest contributor base, but the most late work. Titanium has the least rework. Appcelerator spent far more hours per point than its history predicted and has the most rework.

### Q2. Does a higher Senior share raise output linearly?
No. Across 238 sprints, a straight line explains almost nothing (R² 0.018). The combined curve is a U-shape (R² 0.124), driven by Mule (78% of those sprints). Per project:

| Project | Sprints | Change in points per head, 0% to 100% Senior (90% interval) | Reading |
|---|---|---|---|
| Appcelerator | 40 | +32 (+16 to +53) | Higher with more Seniors |
| Mule | 186 | −8 (−14 to −1) | Slightly lower with more Seniors |
| Titanium | 12 | −2 (−11 to +5) | No clear trend |

Composition and project context matter more than a raw Senior count.

### Q3. How is ownership of core modules distributed?
Components are used as modules. One person resolved at least half of the issues in **16 components** with 30 or more resolved issues:

| Project | Flagged components | Most concentrated |
|---|---|---|
| Appcelerator | 4 of 33 | Studio Error reporting, 88% |
| Mule | 10 of 61 | Core: Non-Blocking, 77% |
| Titanium | 2 of 5 | 360, 62% |

### Q4. Which project had the most delivery delays?
Mule: 34% of delivered sprint issues were resolved after their sprint ended, against 27% in Titanium and 20% in Appcelerator. In all three the typical issue closes 2 to 5 days before sprint end. This uses 5,077 issues (24% of the total).

### Q5. What is the Senior-to-Junior ratio?
- **Senior share** (Seniors ÷ everyone resolving issues in the sprint), averaged over sprints with 3 or more people: **74%** Appcelerator, **46%** Mule, **41%** Titanium.
- **Learners (Junior, Junior-Mid, Mid) per mentor (Mid-Senior, Senior):** 0 to 0.4 in every project-year with at least 5 sprints. Mentors are always present (a median of 3 to 6 per sprint) and learners are scarce. This counts who appears in the same sprint. There is no evidence that any Junior was actually mentored by a Senior, because the data has no project context (who paired with whom, who reviewed whose work), so the ratio shows capacity for mentoring, not mentoring itself.

### Q6. When did each project cross its budget, and how did its pace change?

| Project | Scenario | Budget | Cost | Variance | Budget used |
|---|---|---|---|---|---|
| Appcelerator | Fixed annual | $7.72M | $8.06M | −4.4% | 104.4% |
| Mule | Comfortable | $26.46M | $22.05M | +16.7% | 83.3% |
| Titanium | Tight | $9.29M | $10.93M | −17.6% | 117.6% |

- Titanium crossed its budget in **August 2018**. Mule never did. Appcelerator's monthly series ends at 99.8% of its budget; it reaches 104.4% only when the cost of issues still open (about $355K, no resolution date) is added to the totals.
- SPI starts at 3.4 (Appcelerator) and 2.8 (Titanium) and converges to 1.0. Mule is the only project that starts behind (0.7 and 0.6), recovers and peaks at 1.4.
- **An SPI above 1 does not mean ahead of schedule.** The data has no planned dates, so the plan is an even-pace line from the first measured sprint to the last. Scope also kept growing after sprints started (+35% to +49%), so the target moved while the work was running. A first version used each sprint's starting scope as the plan, and the points added mid-sprint pushed SPI above 1 at the end of all three projects (1.28, 1.50 and 1.33). That version was dropped because it rewarded scope growth, not speed.
- The final SPI is always 1.0 by construction, so only the shape of delivery over time says anything: Appcelerator and Titanium delivered most of their points early, and Mule delivered most of its points in the middle years.

### Q7. How much of the completed work was added after the sprint started?
38% to 40% of completed points were not in scope when their sprint started. Sprint scope at the end was **+45%** (Appcelerator), **+49%** (Mule) and **+35%** (Titanium) larger than at the start. Most of it is new issues (62%, 68%, 47%), mostly bugs in Appcelerator, and tasks, bugs and enhancement requests in Mule. Technical debt does not explain it: debt-related work is planned more often than it is added.

### Q8. How concentrated is the effort?
Five people covered 80% of the hours in Appcelerator and in Titanium. Mule needed 23.

| Bus factor (busiest person's share of resolved issues) | Reading |
|---|---|
| Appcelerator: 18% in 2011, 90% in 2016, 77% in 2018 | Became fragile |
| Titanium: 51% to 81% every year since 2012 | Fragile throughout |
| Mule: 10% to 29% in 2006 to 2019 | Broad base, 8 people for half the work in 2018 |

### Q9. Is there a Senior review bottleneck?
No sign of one. 982 issues entered a review status. Median review time is 25.6 h for Junior-owned issues and 25.1 h for Senior-owned ones. Months with more reviews per Senior are not slower (rank correlation −0.03). This holds only within what the proxy can see, because the data does not say who reviewed.

### Q10. Why does work finish late?
Lateness rises over time for every tier. In Mule it goes from 19% (2014) to 59% (2018).

| Tested explanation | Result |
|---|---|
| More bugs in the mix | No support (−0.29) |
| Bigger sprints | No relationship (−0.04) |
| Shorter sprints | **Explains Titanium**: sprints went from 11 to 13 days to 6 days in 2016 and late work jumped from 6–8% to 46–51%. Does not explain Mule |
| Seniority tier | Weak pattern after adjusting for year, small groups |

Mule's rise stays unexplained by this data.

### Supporting numbers
- Total capped cost is about **$41.0M**: Senior 56%, Mid-Senior 24%, Mid 9%, Junior and Junior-Mid 4% each, issues without a tier 2%.
- Bugs took about a third of all effort hours (Appcelerator 49%, Titanium 35%, Mule 31%).
- Planned vs actual hours by tier (a size-based expectation, not a written plan). In Mule Juniors logged 3.0x the expected hours and Seniors 1.1x. In Titanium it goes from 1.6x (Mid) to 1.1x (Senior). Appcelerator shows no seniority pattern.
- Rework differs far more by project than by tier. 25.5% of Appcelerator's resolved issues were reopened, against 9.8% in Mule and 5.7% in Titanium.
- Bug density (bugs ÷ all issues): 58% Appcelerator, 46% Mule, 44% Titanium. It rose in Appcelerator and fell in the other two.

---

## 7. Conclusions

The three projects reached different outcomes by different paths, and in all three the project and its process explained more than seniority did. Mule held its cost per point steady with work spread across many people. Titanium had the least rework but depended on a few people and crossed its budget in 2018. Appcelerator's cost per point grew about fourfold while a small senior team handled a bug-heavy, high-rework backlog. Seniority showed up mainly in how closely hours matched expectations, not in rework or review time. Planning covered about half of the effort and a large share of the work arrived after sprints started, so reactive capacity, thin mentoring capacity and dependence on a few people were the real constraints. Process changes, such as shorter sprints, can also look like changes in performance.

This analysis matters because it allows us to know better the behavior of a team facing a project, and it supports better planning and execution decisions about capacity constraints and prioritization.

### Next Iteration, existent data
- **Share of each tier's hours spent on bug fixes, reopened and reassigned work.** It tests conclusion 3: do Seniors spend less of their time on rework?
- **Sprint composition against estimation accuracy and reopen rate.** It tests the Mule lead in conclusion 4.
- **Individual Senior velocity against the team's Senior-to-Junior ratio.** It tests whether the Appcelerator association (+32 points per head) holds for individuals or comes from a few prolific people.

### Questions for further research

These need data that this dataset does not have.

| Question | What is needed |
|---|---|
| What is the ideal Junior/Mid/Senior balance? | Rosters per sprint and teams with different compositions on similar work, so the comparison is not only a historical log |
| Which tier shows the steepest increase in delivery independence? | Records over time of how often a person's work needed help, review changes or reassignment, plus real start dates |
| What was the milestone variance? | Planned milestone or release dates to compare with actual releases |
| Which modules are mission-critical? | A definition of criticality (customer impact, incidents) per module |
| Which team structure has the lowest defect leakage? | Defects linked to the release where they were found and the release that introduced them |

The next step is to look for public datasets that include these fields and rerun the analysis with them.

---

## Repository structure

```
00_Docker_Setup.ipynb              Load TAWOS into MySQL (Docker), export, explore, clean and join the tables
01_hraFusion.ipynb                 Seniority tiers, HRA scores with fuzzy logic, matching, rates, owner categories
02_ratesEVMandBudgets.ipynb        Effort cap, planned hours, per-sprint EVM, simulated budgets, EVM in USD
FullProjectAllocationBudget.ipynb  Exploration and analysis behind every Tableau chart
tawos_data/                        Working folder of notebooks 00 to 02
    Issue.csv, Sprint.csv, Change_Log.csv, Comment.csv, ...   TAWOS tables (written by 00)
    main_part2.csv                 Issue-level table after cleaning (end of 00)
    main_part3.csv                 Same table with tiers, rates and owner categories (end of 01)
    team_tier_matching.csv, hra_tech_scored.csv, team_profiles_v1.csv   Matching and HR profiles (01)
    projects_dataset.csv, evm_sprint.csv, project_budgets.csv, cost_monthly.csv   Final datasets (02)
    evm_monthly.csv                Monthly USD EVM export for Tableau (02)
    FullProjectAllocationBudgetDatasets/   Working folder of the analysis notebook
        the five final datasets, main_part3.csv, Change_Log.csv, Comment.csv, Issue_Component.csv, Component.csv (copied from tawos_data)
        A_throughput_sprint.csv, A_review_stints.csv, B_reopen_counts.csv, C_bus_factor_year.csv, C_mentorship_span.csv (written by the analysis)
```

Not stored in the repository: `TAWOS.sql` (about 4.3 GB), the Kaggle HRA file and the raw table exports. Download the first two and regenerate the exports by running the notebooks.

## How to reproduce

1. **Get the data.** Download `TAWOS.sql` from the TAWOS repository and `enterprise_hra_allocation_dataset.csv` from Kaggle. Put both in the working folder.
2. **Start MySQL.** Install Docker Desktop, set the password in the `MYSQL_ROOT_PASSWORD` environment variable, then follow the PowerShell steps in `00_Docker_Setup.ipynb` to start the container, import the dump (a few minutes) and verify it.
3. **Run `00_Docker_Setup.ipynb`.** It exports the tables to `tawos_data/`, explores and standardizes them, and writes `main_part2.csv`.
4. **Run `01_hraFusion.ipynb`.** It builds the tiers, the fuzzy HR scores, the matching and the rates, and writes `main_part3.csv`, `team_tier_matching.csv`, `hra_tech_scored.csv` and `team_profiles.csv`. Rename `team_profiles.csv` to `team_profiles_v1.csv` (the notebook has a cell for it).
5. **Run `02_ratesEVMandBudgets.ipynb`.** It writes `evm_sprint.csv`, `evm_monthly.csv`, `project_budgets.csv`, `cost_monthly.csv` and the final `projects_dataset.csv`.
6. **Run `FullProjectAllocationBudget.ipynb`.** First copy the five final datasets, `main_part3.csv`, `Change_Log.csv`, `Comment.csv`, `Issue_Component.csv` and `Component.csv` into its working folder. It writes the A, B and C exports.
7. **Open Tableau.** Use `projects_dataset.csv` as the main source and relate the other files on project name or issue ID.

Notes:
- Run each notebook from top to bottom. The first cells set a local working folder (`C:\Users\PC\Documents\TAWOS`), so change that path to yours.
- Developed on Python 3.14 with pandas 3. It needs pandas, NumPy, SciPy, Simpful, SQLAlchemy, PyMySQL and Jupyter.
- The random matching uses a fixed seed (42), so the contributor-to-employee pairing is reproducible.
