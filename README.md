You are working inside my existing analytics application repository.

I want you to build a **small, isolated WrenAI-based chatbot POC** that proves whether Wren's semantic/context capabilities can make our existing Javi-style data chatbot much stronger.

This is NOT a migration.
This is NOT a replacement of my current application.
This is NOT a full WrenAI deployment.
This is NOT a request to run all of WrenAI.

The goal is very narrow:

**Take only the useful WrenAI pieces needed for a powerful data chatbot, connect them to our real DuckDB data and our existing internal LLM API, and expose a simple local chat UI on a separate port so I can test asking arbitrary questions about our data.**

The chatbot should be able to:

- understand what data exists
- understand tables and fields
- understand relationships
- retrieve only relevant schema/context
- generate read-only SQL
- execute SQL against our real data
- inspect results
- retry/repair when SQL fails
- answer from real evidence
- use semantic/business context where available
- handle aliases and natural-language terminology
- preserve conversational context
- avoid hallucinating numbers
- answer complex cross-table analytical questions where the data supports them

Do not build unrelated Wren features.

---

# 1. KEEP THE POC COMPLETELY ISOLATED

Create a new root-level directory such as:

`/wren-chat-poc/`

Add it to the ROOT `.gitignore` before downloading/installing anything:

`/wren-chat-poc/`

Verify with:

`git status`

The POC must not be committed to our repository.

Do not stage or commit:

- Wren source
- virtual environments
- node_modules
- databases
- caches
- logs
- generated files
- downloaded models
- secrets
- environment files

Do not modify my existing application unless a tiny read-only integration helper is absolutely required.

My existing application must continue working exactly as before.

---

# 2. USE ONLY THE OFFICIAL WRENAI REPOSITORY

Use:

`Canner/WrenAI`

Do not use forks.

Before implementation:

- inspect the current official repository
- record the exact commit/version used
- verify which portions are Apache 2.0 licensed
- use only code/components whose license allows this POC
- do not use unrelated future modules with different licenses without explicitly reporting them

Prefer the current maintained Wren core/context components.

Do NOT deploy the old full GenBI product unless it turns out to be technically unavoidable.

---

# 3. DO NOT RUN THE ENTIRE WREN APPLICATION

This is extremely important.

WrenAI contains many things we do not need.

I only want functionality that contributes directly to:

**natural language → relevant data context → SQL/tool execution → verified answer**

Inspect Wren and identify the minimum useful components.

Potentially useful:

- metadata/schema representation
- semantic models
- relationships
- context retrieval
- instructions/business definitions
- NL-to-SQL support
- SQL validation/transformation
- query memory/example retrieval
- DuckDB support
- any lightweight reasoning/context utilities

Probably NOT needed:

- full Wren UI
- unrelated administration screens
- full BI dashboard stack
- unrelated connectors
- enterprise-style features
- demo/sample applications
- telemetry if optional
- unnecessary background services
- unrelated authentication systems
- visualization systems
- anything not required for the chat POC

Do not blindly start every Wren service.

First determine the smallest architecture that works.

---

# 4. IF POSSIBLE, USE WREN AS A LIBRARY

Prefer:

our lightweight POC service
→ Wren core/library
→ our LLM
→ our DuckDB

rather than:

our app
→ giant Wren deployment
→ many services
→ many containers.

If the current Wren core can be consumed through its Python package/library, use that.

If a small standalone Python service is the cleanest approach, create one inside `wren-chat-poc`.

Our existing application is primarily Node-based, but the POC can run a separate lightweight Python service if Wren requires Python.

Do not rewrite Wren in Node.

---

# 5. INSPECT OUR EXISTING APPLICATION

Before connecting anything, inspect the current repository.

Understand:

- DuckDB location/configuration
- existing data-loading code
- schema
- major tables
- relationships
- current analytics modules
- Javi/chat implementation
- existing LLM API wrapper/client
- model configuration
- authentication headers required by our internal LLM
- existing MCP/tools if present
- AWS-backed data sources
- any existing semantic metadata or schema catalog
- Excel ingestion
- configuration/environment patterns

Do not assume anything based only on filenames.

Use actual code and read-only database exploration.

---

# 6. USE OUR EXISTING INTERNAL LLM

Do NOT call public OpenAI, Anthropic, Gemini or any external LLM.

We already have an internal company LLM API.

Inspect how the root application calls it.

Reuse that same configuration and API pattern wherever practical.

The internal API behaves similarly to an OpenAI-style endpoint.

Determine whether Wren expects:

- OpenAI client format
- chat completions
- responses API
- tool/function calling
- structured output
- streaming

Build the smallest adapter necessary.

Do NOT expose credentials.

Do not duplicate secrets into source code.

Use existing environment/configuration patterns.

If the Wren library has a clean provider abstraction, implement our internal provider there.

If not, create a thin adapter around the model invocation.

---

# 7. CONNECT TO THE REAL DUCKDB

Use the existing real DuckDB database.

READ ONLY.

Do not modify tables.

Do not create/drop/update/delete production data.

If possible, open it explicitly in read-only mode.

Inspect:

- tables
- columns
- data types
- approximate row counts
- common values
- relationships
- IDs/keys
- dates
- business terminology

Create or generate a semantic representation for Wren from the actual schema.

Do not manually invent schema definitions.

---

# 8. CONTINUOUSLY UNDERSTAND THE DATABASE

One of the main reasons I am evaluating Wren is that I do not want the chatbot to have stale knowledge of the database.

Implement a lightweight metadata-refresh mechanism.

On startup:

- inspect the current database schema
- compare it to the stored semantic/schema cache
- update added/removed/changed tables or columns

During operation, if a query references data the agent cannot resolve:

- inspect relevant metadata again
- discover values/fields when appropriate
- refresh context
- retry

Do NOT re-scan every row of every table on every message.

The goal is:

**schema knowledge stays current without making every chat slow.**

Keep:

- table metadata
- field descriptions where available
- relationships
- aliases
- business definitions
- useful example values

cached locally.

Refresh intelligently.

---

# 9. DATA UNDERSTANDING

The chatbot needs to understand our actual business language.

Examples may include:

- Server Estate
- VA
- vulnerabilities
- incidents
- applications
- environments
- operating systems
- owners
- support state
- lifecycle state
- production
- dev
- UAT
- Red Hat
- RHEL

Do not hard-code these examples unless they truly exist.

Discover terminology from:

- existing application labels
- analytics code
- DB values
- schema
- documentation/comments
- existing Javi metadata

Create aliases only where evidence supports them.

Example:

If the DB contains:

`Red Hat Enterprise Linux`

and users commonly say:

`Red Hat`
or
`RHEL`

the chatbot should resolve those safely.

---

# 10. CHAT AGENT LOOP

The POC chatbot should use an agent loop rather than a single text-to-SQL call.

Recommended behavior:

USER QUESTION

↓

Understand what the user wants

↓

Determine whether the answer requires database evidence

↓

Retrieve relevant semantic/schema context

↓

Resolve terminology/entities/values if needed

↓

Plan a query

↓

Generate read-only SQL

↓

Validate SQL

↓

Execute against DuckDB

↓

Inspect result

↓

If SQL fails or appears wrong:
- inspect the error
- inspect schema/value context
- repair
- retry

↓

Verify that the result actually supports the answer

↓

Generate concise natural-language answer

The LLM should be allowed several focused reasoning/tool steps.

Do not require one giant model prompt containing the entire database.

---

# 11. READ-ONLY SQL SAFETY

Only allow analytical/read operations.

Allow:

- SELECT
- CTEs
- joins
- grouping
- aggregation
- filters
- sorting
- window functions where appropriate

Block:

- INSERT
- UPDATE
- DELETE
- DROP
- ALTER
- CREATE that changes persistent production state
- ATTACH arbitrary external databases unless explicitly required and safe
- filesystem/network side effects

Validate queries before executing them.

Prefer a strict read-only database connection in addition to SQL validation.

---

# 12. DON'T HALLUCINATE ANSWERS

For factual internal-data questions:

If the answer depends on the database, the chatbot must obtain evidence.

Never make up:

- counts
- percentages
- server names
- incidents
- vulnerabilities
- dates
- owners
- rankings

If the data is unavailable, say so clearly.

If the question cannot be answered from current sources, explain what information is missing.

---

# 13. QUERY REPAIR

If a query fails, do not immediately tell the user it failed.

Allow bounded repair attempts.

Example:

attempt 1:
wrong column

→ inspect schema

attempt 2:
correct column but wrong value

→ inspect representative/distinct values

attempt 3:
execute corrected query

Stop after a sensible retry limit.

Expose the final answer, not the internal reasoning chain.

---

# 14. SUCCESSFUL QUERY MEMORY

If Wren has a lightweight built-in mechanism for confirmed question-to-SQL examples, use it.

When a query is successful and clearly matches the question, store a compact reusable example locally for future context retrieval.

Do not blindly memorize failed or questionable queries.

This should improve future answers such as:

"How many production servers?"

and related follow-ups.

Keep this local to the POC.

---

# 15. MULTI-TABLE QUESTIONS

Test questions that require joins.

The chatbot should discover and use valid relationships.

Do not allow the model to randomly invent joins merely because column names look similar.

Prefer:

- declared relationships
- actual key evidence
- existing analytics joins
- validated schema metadata

If relationship confidence is weak, inspect the data/schema before joining.

---

# 16. MCP / TOOL HOOK

The first POC is primarily DuckDB.

However, our real Javi eventually needs MCP/API tools.

Design the agent layer so database SQL is just one tool/capability.

Create a clean placeholder/interface for additional tools such as:

- internal API tool
- AWS lookup
- MCP tools

Do not implement everything unless an existing lightweight tool can be connected easily.

The architecture should permit:

question
→ choose SQL
or
→ choose tool
or
→ combine multiple sources

later.

---

# 17. CREATE A MINIMAL CHAT UI

Do NOT use the full Wren user interface.

Build a very small test page.

It should contain:

- chat history
- text input
- send button
- loading/working indicator
- answer rendering

Optionally add:

**Debug Details**

for development.

Debug Details may contain:

- question
- relevant schema/context selected
- tools used
- SQL generated
- query parameters
- execution duration
- row count/result summary
- errors
- retry count
- final evidence source

Do NOT expose hidden chain-of-thought.

---

# 18. RUN ON A DIFFERENT LOCAL PORT

Run the POC independently from the existing app.

Automatically inspect which ports are already in use.

Choose a safe unused local port.

For example:

existing application → unchanged

Wren Chat POC → separate port

Report the exact endpoint when finished.

Do not change my current application's port.

---

# 19. MINIMIZE DEPENDENCIES

Before installing anything, inspect requirements.

Create a dependency table:

dependency
purpose
required/optional
already available?
alternative?

If a Wren dependency exists only because of unrelated Wren functionality, do not install it if the chat POC does not require it.

The objective is to determine the **minimum viable Wren-powered chatbot stack**.

If the complete official Wren package pulls unnecessary dependencies but still installs cleanly, that may be acceptable for the first test.

However, do not start unrelated Wren services.

Avoid Docker unless the core genuinely cannot run cleanly without it.

I specifically want to avoid another large Docker-heavy application.

---

# 20. REMOVE OR IGNORE UNUSED WREN PIECES

Do NOT immediately delete upstream code.

First identify what is actually used.

For the POC, prefer:

- importing only required modules
- not starting unnecessary components
- excluding optional extras

rather than making a giant destructive fork.

After the POC works, document:

**Required Wren components**
and
**Unused Wren components**

If we later decide to build a stripped internal version, we can do that cleanly.

For this POC, functionality and isolation matter more than physically deleting every unused upstream file.

---

# 21. PERFORMANCE

The chatbot must feel reasonably fast.

Measure separately:

- context retrieval
- LLM planning
- SQL generation
- SQL execution
- retries
- final answer generation
- total latency

Do not blame DuckDB if most latency is coming from the LLM.

Avoid repeatedly sending huge schema dumps.

Use focused context retrieval.

Cache stable metadata.

Reuse DB connections where safe.

---

# 22. TEST WITH REAL QUESTIONS

Create a test suite of at least 20 real questions based on our actual schema.

Cover:

- simple counts
- filters
- distinct values
- date ranges
- grouping
- top-N
- multiple filters
- aliases
- fuzzy/business terminology
- cross-table joins
- follow-up questions
- ambiguous questions
- questions where no data exists
- attempted write operation

Examples should be created only after inspecting the actual database.

For every question capture:

question
→ context selected
→ SQL generated
→ retries
→ result
→ final answer

Independently verify SQL/results where practical.

---

# 23. IMPORTANT QUALITY TEST

A chatbot saying:

"I found the table and executed SQL"

does NOT mean it works.

For each tested question evaluate:

1. Did it understand the question correctly?
2. Did it select the correct tables?
3. Did it select the correct fields?
4. Did it resolve values correctly?
5. Were joins correct?
6. Was SQL valid?
7. Was the result correct?
8. Did the final answer faithfully represent the result?
9. Did it avoid guessing?
10. Was latency reasonable?

Produce a pass/fail summary.

---

# 24. TEST CONVERSATIONAL FOLLOW-UPS

Test context such as:

User:
"How many production servers are there?"

Then:

"How many of those are Red Hat?"

Then:

"Only show ones with critical vulnerabilities."

Then:

"What are the top applications among them?"

The chatbot should understand the conversational narrowing without losing the underlying filters.

Do not simply concatenate entire chat history into every prompt.

Use compact structured conversation state where practical.

---

# 25. FINAL RESULT I WANT

When finished I should be able to open:

`http://localhost:<POC_PORT>`

and ask questions about our actual data.

Examples:

"How many servers do we have?"

"How many production Red Hat servers are there?"

"What were the incidents in June?"

"Which applications have the most critical vulnerabilities?"

"What environment has the highest vulnerability count?"

"What changed compared with last month?"

Only answer questions supported by the actual available data.

---

# 26. FINAL REPORT

After the POC is running, give me:

## Endpoint
The local URL.

## Architecture
A simple diagram of the actual running components.

## Wren Components Used
Exactly what Wren pieces are being used.

## Wren Components Ignored
What we did not need.

## Internal LLM
How our existing LLM API was connected.

## DuckDB
How the existing database was connected.

## Metadata / Context
How schema and semantic understanding works.

## Agent Loop
How question → context → SQL → execute → repair → answer works.

## Performance
Average timing for representative questions.

## Accuracy
Results of the 20-question test.

## Dependencies
Everything newly required.

## Licensing
Which Wren code/components were used and their verified licenses.

## Limitations
What still fails.

## Recommendation

Tell me whether we should:

A. Integrate these selected Wren core pieces into Javi.

B. Borrow the architecture but implement a smaller internal equivalent.

C. Stop using Wren because the useful part is still too heavy.

Base that recommendation on the POC evidence.

---

# MOST IMPORTANT INSTRUCTION

Do not turn this into another giant platform installation.

I already have:

- my analytics application
- DuckDB
- data
- frontend
- backend
- internal LLM
- Javi

I am evaluating Wren only for the difficult intelligence layer:

**understand my data deeply → retrieve the right context → reason about the question → run the right SQL → recover from mistakes → answer accurately from real evidence.**

Build the smallest possible POC that proves whether Wren materially improves that problem.
