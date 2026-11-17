> **Note for future sessions (added 2026-11-17):** The setup line in the first paragraph below (save this brief as CLAUDE.md, start Phase 0 in plan mode) was carried out on 2026-11-17. To see where the project stands, run `git log --oneline` (each phase ends with a commit) and continue from the first phase that has no commit.

You are helping me build my IE 310 (industrial engineering) semester project. First, save this entire brief as CLAUDE.md in this folder so you remember it in future sessions. Then start Phase 0 in plan mode.

## Project
Question: why do New York City capital projects (schools, bridges, parks, water systems) finish late, what factors are associated with the delays, and where should limited oversight go to prevent the most delay? Deliverables:
1. a correlation and regression analysis built with my own method
2. a PuLP optimization model like we used in class
3. an interactive 3D geographic dashboard
4. a one-page status report written the way a technical program manager reports to leadership
This is also a portfolio project for technical program manager jobs, so the README and dashboard must look professional. I have a six sigma green belt, so organize the work with DMAIC (define, measure, analyze, improve, control).

## My setup (Windows)
- Windows, PowerShell, Python 3.13, Docker Desktop, Git.
- Project folder: C:\Users\mtalh\Projects\nyc-capital-delays. Make it its own git repo (I'll push it to my own GitHub).
- Create a venv here: python -m venv venv, then .\venv\Scripts\Activate.ps1. Campus wifi is slow, so use pip install --timeout 120 --retries 10.
- My class already runs a MySQL container on port 3306 from C:\Users\mtalh\Projects\ie310\mySQL. Do not touch it. Create a separate docker-compose.yml in this folder with MySQL 8.4 on port 3307.
- PuLP: my class uses PuLP 4.2, where variables are created with prob.add_variable("x", 0, 3) and prob.add_variable("y", cat="Binary") instead of pulp.LpVariable(...), and old solver names like PULP_CBC_CMD don't exist. Check the installed version and use the API it actually supports. Test that the model solves.
- Keep code readable for a student: plain pandas, short functions, comments that explain why. I have to explain every step to my professor.

## Data sources
Verify every one before writing analysis code: download it, print the real column names, row counts, and min/max dates. Never assume a column exists.

Core (NYC Open Data, joined on FMS ID):
- Capital Projects Dashboard - Citywide Budget and Schedule: https://data.cityofnewyork.us/d/fb86-vt7u
- Capital Projects Dashboard - Citywide Schedule History and Variance: https://data.cityofnewyork.us/d/95tx-snak
- Capital Projects Dashboard - Budget Spend History and Variance: https://data.cityofnewyork.us/d/qj5n-h5qp (confirm this ID)
- Capital Projects Database (CPDB) Projects (Points), for locations: https://data.cityofnewyork.us/d/h2ic-zdws (only about 40% of projects are mapped)
- Legacy Capital Project Detail Data - Dollars (has DELAY_DESC and BORO; frozen in early 2024): https://data.cityofnewyork.us/d/wa2y-rh4b
- Legacy Capital Project Detail Data - Milestones: https://data.cityofnewyork.us/d/s7yh-frbm
- NYC borough boundaries GeoJSON (find the official NYC Open Data source)

External:
- NOAA GHCN-Daily, Central Park station USW00094728 (snowfall, precipitation, max/min temperature)
- FRED PPI Construction Materials, series WPUSI012011 (monthly)
- FRED NYC Mining, Logging, and Construction employment, series SMU36935611500000001 (monthly)
- New York Fed Global Supply Chain Pressure Index (monthly)
- NYC DOT Street Construction Permits (2022 to present): https://data.cityofnewyork.us/d/tqtj-sjs8 (utility street work, with locations)
- NYC DOB NOW: Build - Approved Permits: https://data.cityofnewyork.us/d/rbx6-tga4 (date range not documented, so check it)
- Optional stretch: NYT Article Search API, monthly counts of keywords like "construction strike", "supply chain", "Con Edison". I'll put my key in a .env file. Never hard-code or commit keys.

Known data problems to handle and report:
- The NYC Comptroller (Dec 2025) found the dashboard covers about 47% of FMS IDs, and only about 52% of delayed projects list a delay reason.
- FMS ID formats may differ between datasets, and IDs are not always one-to-one with projects. Normalize them and report the match rate for every join.
- Schedules only appear once a project reaches the design phase, so measured delays are a lower bound.
Store raw files in data/raw and processed files in data/processed (both git-ignored).

## Phases
Stop after each phase, summarize the results in plain English with real numbers, and wait for my OK.

Phase 0 - Research and plan:
- Verify all sources above. Write docs/data_sources.md: link, what it is, columns used, date range you measured, caveats.
- Write docs/research.md summarizing published causes of NYC capital project delays, with links. Start with the NYC Comptroller's "Flying Blind on Billions" (Dec 2025), the NY State Comptroller's reports on NYC capital projects, and the 2024 Fordham / NYC DDC Town+Gown study "Analyzing Root Causes of Construction Delay".
- Propose a detailed plan and a timeline of about 4 weeks.

Phase 1 - Load (Define):
- docker-compose MySQL, load the tables, SQL joins on FMS ID, and one analysis table with one row per project.
- Export it to data/processed/projects.parquet so the dashboard runs without MySQL.

Phase 2 - Measure:
- slip % = (latest forecast or actual finish - first reported finish) / (first reported finish - planned start)
- Also: number of slips, size of the first slip, and an open/finished flag.
- Measure delay as a % of planned length, not raw days. Never silently drop unfinished projects; report them separately with how far they've already slipped.

Phase 3 - Analyze (my own method):
- For each project, compute its exposure during its own active window: share of bad-weather days, % change in material prices, average supply chain pressure, change in construction employment, number of DOT street permits within 250 m during the window (a spatial and time join; use MySQL ST_Distance_Sphere if practical), and nearby DOB permits.
- Add project traits: log budget, budget growth %, agency, type, borough, city vs non-city funding.
- Spearman correlations of each factor vs slip %.
- A multiple linear regression I build myself: solve the normal equations (X^T X) b = X^T y with np.linalg.solve, like we did in class. Use standardized features and reporting-year dummies to control for citywide trends. Cross-check against np.linalg.lstsq or statsmodels.
- Validate: 80/20 train/test split, MAE vs a predict-the-mean baseline.
- Use the words "associated with", never "causes".
- Stretch: COVID natural experiment using the legacy data. Non-essential construction paused from 2020-03-27 to 2020-06-08, while roads, bridges, transit, utilities, and hospitals kept working. Compare slips with a difference-in-differences.

Phase 4 - Improve (PuLP):
- Choose which open, at-risk projects get extra oversight. x_p is binary.
- Maximize: sum over p of e * predicted_slip_days_p * (budget_p / planned_days_p) * x_p
- Subject to: sum of h_p * x_p <= H (staff hours); at least m projects per borough; a cap per agency.
- e, h_p, and H are assumptions: make them inputs, explain them, and run a sensitivity analysis with a chart of delay avoided vs staff hours. Report the LP relaxation's shadow price on staff hours if the API exposes it.

Phase 5 - Control (dashboard): Streamlit + pydeck app.py
- Each project is a 3D square block made with PolygonLayer(extruded=True). ColumnLayer can't vary width per object.
- Height = budget, with a toggle for budget / planned duration / slip days. Width = planned duration. Color = slip %, green to red. Tooltip with project details.
- A borough layer that's clickable with st.pydeck_chart(on_select="rerun") (every layer needs an id) and filters the whole page.
- Sidebar filters: borough, agency, type, phase, budget range slider, completion date range, slip % range.
- KPI cards, a slip-by-agency chart, a regression results chart, and an optimizer panel: sliders for staff hours and effectiveness, a Run button, and chosen projects highlighted on the map.
- Must run from the parquet export so I can deploy it free on Streamlit Community Cloud.

Phase 6 - Report:
- README: the question, screenshots, key findings with numbers at the top, how to run it.
- reports/status_report.md: one page of findings, risks, and recommendations.
- requirements.txt and a short list of limitations.

## Rules
- Plan before editing. Ask me before big decisions or anything that deletes files.
- Verify every number you report by running code. Never invent data, columns, or results.
- Cite links for every external fact in docs/.
- Commit to git at the end of each phase with a clear message. .gitignore must include data/, venv/, and .env.
- If something doesn't fit the timeline, tell me what to cut instead of doing a rushed version.
