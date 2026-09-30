# Analytics Engine Performance Optimization Playbook

## Goal

The goal is not simply to make queries fast.

The goal is to make the entire analytics experience feel immediate and native even when the underlying dataset grows to millions or tens of millions of records.

The user should feel like they are interacting with Excel or Qlik Sense:

- filters react quickly
- tables scroll smoothly
- searches feel immediate
- charts update quickly
- the page never freezes
- changing one filter does not reload the entire application
- the amount of data in storage should not directly determine how much data reaches the browser

The most important principle is:

**The UI should work with a small window into the data, while DuckDB works with the complete dataset.**

---

# 1. Never Load the Entire Dataset Into the Browser

This is probably the most important optimization.

Imagine there are 5 million server/vulnerability records.

The wrong approach is:

5 million records
→ backend
→ JSON
→ browser memory
→ frontend filtering
→ frontend sorting
→ table

Even if DuckDB returns those rows quickly, everything after DuckDB becomes expensive.

The browser now has to:

- receive a huge HTTP response
- parse a huge JSON document
- allocate memory
- keep millions of JavaScript objects
- filter them
- sort them
- render them

The application will eventually become slow regardless of how fast DuckDB is.

Instead:

5 million records stay in DuckDB.

The browser asks:

"Give me the first 100 records matching my current view."

DuckDB returns 100.

The user scrolls.

The application requests another small window.

The important point is that:

5 million rows should not feel dramatically different from 50 million rows to the browser.

The backend handles scale.

### Example

Suppose your vulnerabilities table contains:

8,400,000 rows.

The user opens the Vulnerabilities page.

Do NOT send 8.4 million rows.

Instead initially request:

- total vulnerability count
- critical count
- high count
- summary chart data
- first 100 table rows

The page may receive perhaps a few kilobytes instead of hundreds of megabytes.

### Target behavior

Opening the page should never mean:

"download the dataset."

It should mean:

"load the current view."

---

# 2. Virtualize the Table

Even if you only have 10,000 rows in the browser, rendering 10,000 HTML table rows is unnecessary.

A user may physically see perhaps 30–60 rows on the screen.

A good data grid renders roughly what the user can see plus a small buffer.

For example:

Dataset result:

50,000 matching records.

Visible screen:

rows 450–490.

Browser may actually render:

rows 420–520.

Everything else exists logically but isn't represented by thousands of DOM elements.

This is called virtualization.

### Why this matters

Without virtualization:

50,000 rows might mean:

50,000 DOM nodes × many columns.

Scrolling becomes heavy.

The browser spends time on:

- layout
- painting
- memory management
- DOM updates

With virtualization, the browser may only render around 100 rows regardless of dataset size.

### Desired experience

The scrollbar can represent millions of records.

But the browser only renders what the user currently sees.

This is how you get the feeling:

"I am scrolling through a giant spreadsheet."

without actually rendering the giant spreadsheet.

---

# 3. Filtering Must Happen in DuckDB

Suppose the user has:

2 million server records.

They select:

Environment = PROD

Then:

Severity = Critical

Then:

OS = Red Hat

The wrong model is:

Load 2 million records
→ JavaScript filters PROD
→ JavaScript filters Critical
→ JavaScript filters Red Hat.

The correct model is:

User selects filters
→ send filter definition to backend
→ DuckDB filters dataset
→ return only matching view.

DuckDB is designed for analytical filtering and aggregation.

Let it do that job.

### Example

Original dataset:

5,000,000 rows.

After:

PROD

1,800,000.

After:

Critical

62,000.

After:

Red Hat

8,700.

The browser still does not need 8,700 rows immediately.

It might receive:

first 100 rows
+
count = 8,700.

If the user changes Critical → High, DuckDB evaluates the new filter and gives the new result.

---

# 4. Treat Every UI State as a Query

This is a useful way to think about the whole engine.

The page itself represents a query state.

For example:

Environment:
PROD

Severity:
Critical

OS:
Red Hat

Search:
payments

Sort:
incident_count descending

Page:
first 100

Instead of storing a giant frontend dataset, the frontend stores this small state.

That state produces the result.

So your system becomes:

USER INTERACTION

↓

QUERY STATE

↓

DUCKDB

↓

RESULT WINDOW

↓

UI

This is similar in spirit to analytics products where selections determine what the analytical engine needs to calculate.

Qlik visualizations are built around dimensions and measures, with its engine producing the data required for the current visualization rather than simply handing the frontend the complete underlying dataset.

---

# 5. Charts Must Use Aggregated Data

Charts almost never need raw records.

Suppose:

2.5 million vulnerabilities.

A severity chart doesn't need 2.5 million objects.

It needs something like:

Critical → 14,821

High → 42,320

Medium → 81,994

Low → 103,221

That's four rows.

The browser receives four rows.

DuckDB performs the aggregation.

### Another example

User asks:

"Vulnerabilities by month for the last two years."

Raw data:

1,400,000 vulnerabilities.

Chart data:

24 monthly values.

Send the 24 points.

Not 1.4 million records.

### General rule

A visualization should receive the smallest possible dataset capable of rendering that visualization.

This is one of the biggest performance improvements you can make.

---

# 6. Precompute Expensive Dashboard Metrics

Some dashboard metrics may be requested constantly.

For example:

Total servers

Production servers

Critical vulnerabilities

Unsupported OS count

Applications affected

Open incidents

If those calculations repeatedly scan millions of records every time someone opens the page, you're wasting computation.

For metrics that are expensive and repeatedly requested, consider preparing summary data.

Think of:

Raw data

↓

Prepared analytics summaries

↓

Dashboard

Then detailed drill-down still queries the raw data when necessary.

### Scenario

Dashboard home page gets opened 500 times per day.

Every load calculates:

critical vulnerabilities across 20 million records.

Instead, maintain a summary representing the latest calculation.

Home dashboard reads the summary.

When the user drills into:

"Show these 18,492 critical vulnerabilities"

then query the detailed dataset.

Qlik's performance guidance similarly recommends pre-calculating measures when appropriate rather than repeatedly performing expensive calculations at visualization time.

---

# 7. Cache Repeated Results

Analytics users repeatedly perform similar actions.

For example:

Everyone opens:

PROD dashboard.

Everyone looks at:

Critical vulnerabilities.

People repeatedly select:

Windows

Red Hat

Production

US region.

You don't necessarily need DuckDB to recompute exactly the same result every time.

Cache suitable results.

### Example

User A:

PROD + Critical

Result:

18,422.

Thirty seconds later User B requests:

PROD + Critical.

If underlying data hasn't changed, reuse the result.

### Important distinction

Not everything should be cached equally.

Good candidates:

- dashboard summaries
- dropdown values
- common aggregations
- common filter combinations
- chart datasets
- metadata

Less useful:

- highly unique searches
- constantly changing real-time information

### Cache invalidation

The cache must understand when data changes.

If your ingestion process loads new records, relevant cached analytical results should expire.

The goal is:

same question + same data
→ avoid unnecessary repeated work.

---

# 8. Make Search Feel Instant

Search can easily ruin an otherwise fast analytics page.

Suppose the user types:

R

Re

Red

Red H

Red Ha

Red Hat

You don't want six expensive searches happening simultaneously.

Instead, wait briefly until the user pauses typing.

Then execute the search.

This is usually called debouncing.

### Scenario

User starts typing:

"payments-server"

Instead of searching on every keystroke:

wait a short moment after typing stops

→ run one search

→ return results.

### Also cancel old searches

Suppose:

Search 1 = Red

Search 2 = Red Hat

Search 1 takes longer.

Without cancellation:

Red Hat results appear.

Then the old Red request finishes.

Suddenly the UI replaces the correct results with old results.

That's terrible UX.

Older requests should become irrelevant when a newer user action occurs.

---

# 9. Cancel Old Queries

This applies beyond search.

Suppose a user clicks:

PROD

then immediately:

Critical

then immediately:

Red Hat.

You may generate three analytical requests.

If the user has already moved on, there is little value in letting outdated work control the UI.

The system should know:

Request A is stale.

Request B is stale.

Request C represents the current UI state.

Only C matters.

This creates the feeling that the application follows the user rather than lagging behind them.

---

# 10. Avoid Full Page Reloads

A dashboard should not behave like an old web page.

Changing:

Severity

should not reload:

header
navigation
user profile
every chart
every table
every unrelated metric

Only the components affected by that selection should update.

### Example

User changes:

Environment:
ALL → PROD.

Affected:

server count
OS distribution
vulnerability chart
server table

Possibly unaffected:

static documentation
navigation
application metadata.

Update only what matters.

This also creates a much smoother visual experience.

---

# 11. Load Important Things First

Not every piece of the dashboard needs to arrive simultaneously.

Imagine a page contains:

5 KPI cards

6 charts

one huge table.

The user cares about the page becoming useful quickly.

A good load sequence might be:

First:

KPI summaries.

Then:

major charts.

Then:

table rows.

Then:

secondary analytics.

The screen starts feeling useful quickly even if all computation hasn't finished simultaneously.

### Bad experience

Blank page for four seconds.

Then everything appears.

### Better experience

Within a short moment:

Server count
Critical vulnerabilities
Applications affected

appear.

Then charts populate.

Then the detailed grid becomes available.

Perceived speed matters almost as much as raw speed.

---

# 12. Separate Overview Data From Detail Data

This is extremely useful for your analytics engine.

You really have two kinds of usage.

### Overview

"How is the estate doing?"

Needs:

totals
trends
distributions
KPIs.

### Detail

"Show me every Red Hat production server with critical vulnerabilities."

Needs:

individual records.

These shouldn't necessarily use the exact same retrieval strategy.

Overview should favor:

aggregated/prepared data.

Detail should favor:

filtered DuckDB queries.

Trying to make one giant dataset satisfy every UI component often makes applications slower.

---

# 13. Use a Good Analytical Data Model

Qlik explicitly recommends efficient data models and warns that unnecessary model complexity can hurt performance. Its guidance discusses appropriate granularity, removing unnecessary fields, simplifying relationships, and avoiding problematic associations.

For your engine, think:

What does the analytics actually need?

Don't carry unnecessary information everywhere.

For example, perhaps your core analytics revolves around:

Servers

Applications

Vulnerabilities

Incidents

Owners

Dates

Environments.

You should have clean relationships between them.

The cleaner the data relationships are, the easier it is for both:

DuckDB

and

Javi

to understand them.

### Example

Instead of having:

server name appearing differently in seven tables

try to have a stable server identifier connecting them.

That reduces both query complexity and AI confusion.

---

# 14. Don't Keep More Detail Than Necessary in Every Analytical Layer

Suppose timestamps are:

2026-09-29 23:18:43.482918

But most dashboard charts only care about:

day

week

month.

You don't always want every analytical calculation working at microsecond precision.

Keep raw detail where needed.

But create useful analytical representations.

For example:

incident_day

incident_month

incident_year.

Qlik similarly recommends using appropriate data granularity rather than maintaining unnecessary precision for every analytical workload.

---

# 15. Take Advantage of DuckDB's Columnar Nature

DuckDB works especially well for analytics because it doesn't need to treat every query like a traditional row-by-row transactional database operation.

If your table has 70 columns but a chart needs:

severity

count

you should conceptually be doing work only around what is needed.

DuckDB and Parquet can take advantage of projection pushdown so unnecessary columns are not read for relevant analytical queries. DuckDB can also push filters down and skip irrelevant portions of Parquet files using metadata.

This is another reason to avoid requests that blindly retrieve:

everything.

---

# 16. Think Carefully About Parquet

If you have huge historical datasets, Parquet can be extremely useful.

For example:

2024 server history

2025 server history

2026 server history.

DuckDB can directly query Parquet efficiently and use filtering/projection optimizations. For workloads with lots of repeated joins and queries, DuckDB's own guidance says loading data into DuckDB may outperform querying Parquet directly; so the right choice depends on whether the data is archival or frequently queried.

One possible model is:

Current/high-use analytical data
→ DuckDB tables

Large historical/archive data
→ Parquet

DuckDB can query both.

Don't move everything to Parquet just because it's large.

Use it where it improves storage and scan behavior.

---

# 17. Organize Large Historical Data Intelligently

Suppose you have five years of incident data.

Most queries may be:

this month

last month

this quarter

this year.

You don't want the engine repeatedly scanning irrelevant historical data.

Organizing data around commonly filtered dimensions such as time can help reduce unnecessary reading.

For Parquet workloads, DuckDB can use row-group metadata and partitioning to skip data that cannot satisfy a filter.

### Example

Query:

June 2026 incidents.

The engine should avoid doing meaningful work against:

2019

2020

2021

etc.

The architecture should make irrelevant history cheap to ignore.

---

# 18. Don't Add Indexes Everywhere

Coming from traditional databases, it's tempting to think:

"More indexes = faster."

Not necessarily with DuckDB.

DuckDB already maintains zonemap-style metadata and can skip irrelevant data ranges. Explicit ART indexes are mainly useful for certain highly selective equality/IN lookups and don't generally speed aggregation, sorting, or joins. They also consume resources and have tradeoffs.

So don't tell your colleague:

"index every filter field."

Instead:

measure actual slow interactions

then optimize those.

---

# 19. Data Ordering Can Matter

A surprisingly useful DuckDB optimization is keeping frequently filtered data somewhat ordered.

For example:

timestamps.

If records are reasonably ordered by time, DuckDB's zonemaps can more effectively skip irrelevant blocks.

DuckDB documents that ordered columns can improve compression and allow more effective block skipping for selective filters.

### Example

You constantly query:

last 7 days.

If the date column is naturally organized chronologically, the database may avoid reading large older portions.

Again:

don't reorganize everything blindly.

But it's something worth testing for your heavy filters.

---

# 20. Make Repeated Tiny Queries Cheap

Analytics UI actions often produce small repeated queries.

Examples:

list environments

list severity values

count filtered rows

get first 100 rows.

DuckDB supports prepared statements, which can avoid repeating some query planning work for frequently executed queries with different parameters; its documentation notes they're especially useful for repeatedly run small queries.

Conceptually:

same analytical operation

different filter value.

Example:

Environment = PROD

Environment = UAT

Environment = DEV.

The application shouldn't unnecessarily rediscover everything each time.

---

# 21. Don't Block the Entire Interface With One Slow Component

Imagine the vulnerabilities chart requires a more expensive calculation.

That should not prevent the user from:

scrolling the server table

opening another filter

reading available KPIs.

Each analytical component should have its own loading state.

### Bad

Whole page:

LOADING...

### Better

Server count: ready.

Environment chart: ready.

Vulnerability trend: calculating...

Table: ready.

The user can continue working.

This contributes massively to the "native" feeling you're looking for.

---

# 22. Use Optimistic UI Where Safe

Some interactions can appear instantaneous before the analytical result finishes.

Example:

User clicks:

PROD.

The filter chip can immediately show:

PROD ✓

Then the affected charts update when the query completes.

Don't make the filter itself wait for the database.

This makes the interface feel responsive.

---

# 23. Keep Filter State Consistent Across the Dashboard

Qlik's interaction model is powerful partly because selections influence the analytical view consistently.

For your system:

If user selects:

Environment = PROD

every relevant component should understand that same filter.

Then:

OS = Red Hat

becomes:

PROD AND Red Hat.

Then:

Severity = Critical

becomes:

PROD AND Red Hat AND Critical.

You should have one clear analytical selection state.

Otherwise each chart ends up maintaining its own filter logic and the system becomes slow, inconsistent, and difficult to maintain.

---

# 24. Make Drill-Down Natural

Don't show maximum detail immediately.

Example:

Chart:

Critical vulnerabilities by application.

User clicks:

Payments.

Now show:

Payments vulnerabilities.

Then clicks:

CVE category.

Then:

affected servers.

This keeps initial calculations small while still allowing deep exploration.

The philosophy is:

summary first

detail on demand.

---

# 25. Handle Large Exports Differently From Interactive Tables

If someone wants:

"Export all 2 million rows"

that's different from:

"let me explore these rows."

Don't make the interactive grid load 2 million rows simply because exporting exists.

Interactive experience:

small pages/windows.

Export:

backend creates the large result separately.

That keeps Excel-like exploration responsive.

---

# 26. Watch Result Size, Not Just Query Time

Suppose DuckDB executes a query in:

200 ms.

Sounds excellent.

But then returns:

750 MB JSON.

The user still experiences a slow application.

Measure:

database time

serialization time

network transfer

browser parsing time

render time.

The slow part may not be DuckDB at all.

This is extremely important when diagnosing performance.

---

# 27. Add Performance Budgets

Don't optimize based on:

"It feels kind of slow."

Define goals.

For example:

Filter interaction:
target <300–500 ms for common operations.

Search:
initial response within a short perceptual threshold.

Grid scrolling:
no visible frame drops.

Dashboard initial useful content:
very fast.

Heavy charts:
allowed slightly longer but asynchronous.

The exact thresholds should be based on your infrastructure and users.

Once you have targets, your colleague can find which operations violate them.

---

# 28. Instrument Every Interaction

For every important interaction, capture:

user action

backend request

DuckDB execution time

rows scanned if available

rows returned

response payload size

frontend processing time

render time.

Then you can distinguish:

DuckDB problem

from

API problem

from

frontend problem.

Without this, you'll spend weeks guessing.

---

# 29. Profile Slow DuckDB Queries Instead of Guessing

If a particular query is slow, inspect why.

DuckDB recommends studying query plans and looking for issues such as ineffective filter pushdown, poor join order, or unexpectedly huge intermediate results.

The important philosophy:

Don't globally optimize DuckDB.

Find:

"this particular interaction takes 4 seconds"

and investigate that query.

---

# 30. Control Memory and Concurrency

DuckDB can work on data larger than memory and spill some analytical operations to disk, but machine memory, threads, storage speed, and query type still matter. DuckDB recommends considering available memory per thread and notes that SSD/NVMe storage helps larger-than-memory workloads.

For your engine this means:

don't assume:

more threads = always faster.

If many users are running analytics simultaneously, uncontrolled parallelism can actually make everything worse.

The production environment should be tested with realistic concurrent use.

---

# 31. Avoid One Giant Dashboard

A Qlik-like dashboard with:

35 charts

10 giant tables

20 filters

all calculating immediately

will eventually become heavy.

Prioritize.

Maybe the first view contains:

4 KPIs

3 important charts

one table.

More detailed analysis can live in tabs or drill-down views.

Qlik itself warns that app/sheet complexity—large tables, many calculations, complex expressions, and numerous objects—can hurt performance.

---

# 32. Make Filters Cheap to Populate

Dropdowns themselves can become expensive.

Imagine:

Hostname dropdown.

3 million unique hostnames.

Don't send 3 million dropdown options.

Instead:

searchable typeahead.

User enters:

pay-

then query relevant hostnames.

For low-cardinality values:

Environment:
PROD
DEV
UAT

load them directly.

So distinguish:

LOW CARDINALITY FILTER

from

HIGH CARDINALITY SEARCH.

---

# 33. Treat Different Columns Differently

Not every column should behave the same way.

Examples:

Environment:
small dropdown.

Severity:
small dropdown.

Date:
range selector.

Hostname:
search/typeahead.

Incident count:
numeric range.

Application:
searchable selection.

This improves both usability and performance.

---

# 34. Preload Small Metadata

Some information is tiny and frequently needed:

environment values

severity values

available years

regions

table definitions.

Load/cache those early.

Then when the user opens a filter, it feels instant.

Don't query tiny stable metadata every single click.

---

# 35. Preserve the User's Current Result

When the user changes a filter, don't necessarily blank the table immediately.

Keep the previous result visible while the updated result arrives, but clearly indicate refresh.

This avoids visual flashing:

table

→ blank

→ loading

→ table.

Instead:

old table briefly remains

→ subtle updating state

→ new table.

This small UX detail contributes a lot to perceived smoothness.

---

# 36. Build an Analytics Query Layer

Long-term, I wouldn't want every chart independently constructing arbitrary database behavior.

The page should think in analytical concepts such as:

metric

dimension

filters

sort

range

limit.

Example:

Metric:
vulnerability_count

Dimension:
application

Filters:
environment = PROD
severity = CRITICAL

That analytical request gets translated into DuckDB work.

This makes:

charts

tables

Javi

filters

all use the same data logic.

And this is particularly valuable for you because Javi can eventually consume the same analytics layer instead of inventing totally independent database logic.

---

# 37. Javi and the Dashboard Should Share the Same Truth

This is very important for your project.

If the dashboard says:

Critical vulnerabilities = 18,421

and Javi answers:

18,397

users lose trust immediately.

Ideally:

Dashboard analytics

and

Javi analytics

use the same:

semantic definitions

filters

metrics

data sources.

Then asking Javi:

"How many critical vulnerabilities are there?"

should use the same underlying definition that produces the dashboard card.

This will also reduce the problems you've been having with Javi.

---

# 38. Build a Semantic Layer Over DuckDB

As the system grows, define business concepts once.

Example:

Production Server

means:

environment = PROD.

Critical Vulnerability

means:

severity = CRITICAL.

Active Incident

means whatever your business rule actually defines.

Then:

charts

tables

Javi

exports

all use the same definition.

This is one of the most important things for consistency as your analytics engine grows.

---

# 39. Design for Progressive Scale

You don't need to prematurely build for billions of rows.

Instead test tiers.

100K rows

1M rows

10M rows

50M rows

100M rows.

At each tier measure:

page load

common filter

search

sort

grouping

top-N

chart aggregation

large join.

Then you'll know where the architecture actually starts degrading.

---

# 40. Final Target Architecture

Conceptually your analytics engine should behave like this:

DATA SOURCES

↓

INGESTION / NORMALIZATION

↓

DUCKDB + OPTIONAL PARQUET HISTORY

↓

SEMANTIC / ANALYTICS LAYER

↓

QUERY + CACHE LAYER

↓

small result sets

↓

UI

The UI then contains:

virtualized tables

aggregated charts

shared filters

incremental loading

cancellable requests

independent loading states.

And Javi sits alongside the UI:

Javi

↓

same semantic/analytics layer

↓

DuckDB/tools

↓

verified result

↓

answer.

The frontend should never become the analytics engine itself.

DuckDB should do the heavy analytical work.

The browser should primarily:

display

interact

request

render.

---

# Recommended Optimization Order

Don't try all of this simultaneously.

I would optimize in this order:

1. Make sure raw large datasets are never sent to the browser.
2. Add/verify table virtualization.
3. Move filtering/sorting/searching fully into DuckDB.
4. Make charts use aggregated results only.
5. Add request cancellation and search debouncing.
6. Add shared dashboard filter state.
7. Add caching for common analytical results.
8. Precompute expensive frequently-used metrics.
9. Separate summary queries from detail queries.
10. Add proper performance instrumentation.
11. Profile slow DuckDB queries.
12. Optimize large historical storage/Parquet where useful.
13. Add semantic/business metric definitions.
14. Make Javi consume that same analytical layer.
15. Load-test the complete experience at progressively larger dataset sizes.

The final question your colleague should ask for every feature is:

**"If this table becomes 100 million rows tomorrow, how much additional work does the browser have to do?"**

The best answer is:

**almost none.**
