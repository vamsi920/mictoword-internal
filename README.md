Below is the complete rewritten set using Jevy consistently. I also tightened the prompts specifically for your situation: small/cheap Qwen model, SQLite first, low token usage, unlimited/reasonable repeated LLM calls, deterministic logic wherever possible, self-repair, fuzzy terminology, regex/pattern understanding, strong grounding, and zero fabricated analytics.

Use these one phase at a time. Do not give all phases to Claude/Copilot at once.

Phase 0 — Audit the Existing Jevy Architecture

You are working on an existing AI analytics assistant called Jevy.

Jevy is part of a Node.js analytics platform used to analyze infrastructure/server-related company data.

LONG-TERM GOAL

Jevy should eventually be capable of:

* answering natural-language questions about company data
* querying SQLite
* calling MCP tools
* calling internal APIs
* performing multi-step analysis
* combining multiple data sources
* creating charts and structured visualizations
* maintaining useful conversational context
* recovering from failed SQL/tool calls
* producing answers backed by actual data

CURRENT PROBLEM

Jevy is unreliable.

Observed behavior includes:

* incorrect answers
* dummy/template answers
* hallucinated numbers
* incorrect SQL
* failure to understand questions
* incorrect tool selection
* failure to recover from errors
* answers that are not grounded in database results

IMPORTANT CONSTRAINT

The production LLM is currently a small, inexpensive Qwen model.

Do not solve reliability problems simply by assuming we will use a larger model.

The architecture must make a small model successful.

PHASE OBJECTIVE

Do NOT perform a large refactor yet.

First inspect the existing codebase and document exactly how Jevy currently works.

Trace:

User question
→ frontend
→ backend endpoint
→ prompt/context construction
→ LLM call
→ tool/SQL selection
→ execution
→ result handling
→ additional LLM calls
→ final response

Investigate:

1. Where user messages enter the backend.
2. How system prompts are constructed.
3. What conversation history is supplied.
4. Which Qwen model is used.
5. Model parameters.
6. How SQLite access currently works.
7. How SQL is generated.
8. Whether schema information is supplied.
9. How tools are exposed.
10. How MCP is currently integrated.
11. How APIs are integrated.
12. How tool results return to Jevy.
13. Whether Jevy receives another reasoning opportunity after tool execution.
14. Whether SQL errors are returned to Jevy.
15. Existing retry logic.
16. Existing validation logic.
17. Existing fallback logic.
18. Source of dummy/template responses.
19. Whether Jevy can produce analytical answers without executing a data operation.
20. Token-heavy or redundant context.
21. Places where deterministic code could replace LLM reasoning.

Specifically identify likely causes of:

* hallucinations
* incorrect numbers
* incorrect SQL
* bad joins
* wrong columns
* wrong tool selection
* excessive context
* excessive token consumption
* small-model confusion
* failure loops

OUTPUT

Create a concise technical report containing:

CURRENT ARCHITECTURE

CURRENT REQUEST FLOW

FILES INVOLVED

PROBLEMS FOUND

ROOT CAUSES

QUICK WINS

STRUCTURAL PROBLEMS

RECOMMENDED PHASED ARCHITECTURE

Do not modify unrelated functionality.

Do not modify the working chatbot UI or chat history unless necessary.

Do not begin Phase 1 automatically.

Stop after presenting the audit.

Phase 1 — Database Intelligence Layer

Implement Phase 1 of Jevy: Database Intelligence Layer.

Do not implement MCP, APIs, charts, or multi-source reasoning in this phase.

OBJECTIVE

Jevy must develop an accurate machine-readable understanding of the SQLite database.

The LLM must NOT need the complete database schema in every prompt.

Build a deterministic database intelligence system around SQLite.

STEP 1 — DATABASE INTROSPECTION

Automatically inspect SQLite and discover:

* tables
* views
* columns
* SQLite datatypes
* primary keys
* foreign keys
* indexes
* nullable fields
* unique constraints
* table relationships
* approximate/useful row counts
* useful categorical values
* representative values
* useful numeric/date ranges

Use SQLite metadata facilities including where appropriate:

sqlite_master
PRAGMA table_info
PRAGMA foreign_key_list
PRAGMA index_list

Do not hardcode information that SQLite can provide.

STEP 2 — SEMANTIC CATALOG

Create a persistent machine-readable semantic catalog.

Conceptually:

{
“tables”: {
“servers”: {
“description”: “…”,
“columns”: {},
“relationships”: [],
“categorical_values”: {}
}
}
}

Separate:

A. automatically discovered metadata

B. manually maintained business definitions

Automatically generated information must NEVER overwrite manually curated business definitions.

STEP 3 — BUSINESS GLOSSARY

Create an extensible business glossary.

Example concepts:

prod → production
production → PROD

RHEL → Red Hat Enterprise Linux
Red Hat → Red Hat Enterprise Linux

crit → CRITICAL
critical vuln → CRITICAL vulnerability

The glossary must support:

* synonyms
* abbreviations
* business terminology
* technical terminology
* common user terminology

STEP 4 — DATABASE VALUE DISCOVERY

For useful categorical columns, discover actual values.

Example:

os_name:

Red Hat Enterprise Linux 7
Red Hat Enterprise Linux 8
Red Hat Enterprise Linux 9
Windows Server 2019
Ubuntu Linux

Do NOT collect huge numbers of values from high-cardinality columns such as IDs.

Use configurable limits.

STEP 5 — RELATIONSHIP GRAPH

Build a machine-readable relationship graph.

Example:

applications
→ application_id
→ servers
→ server_id
→ vulnerabilities

Jevy should later be able to determine valid join paths without asking the LLM to rediscover database relationships.

STEP 6 — SCHEMA RETRIEVAL

Create a retrieval service.

Input example:

[“Red Hat”, “production”, “server”]

Output should contain ONLY relevant:

* tables
* columns
* relationships
* glossary concepts
* known database values

Do NOT return the entire schema by default.

SMALL MODEL REQUIREMENT

Production currently uses a small Qwen model.

Optimize specifically for it:

* compact JSON
* minimal schema context
* short field descriptions
* deterministic preprocessing
* cached metadata
* cached glossary
* no repeated schema discovery
* no unnecessary natural-language explanation

Anything that can reliably be determined using code should NOT consume an LLM call.

TESTS

Add tests for:

* schema discovery
* column discovery
* foreign keys
* relationships
* categorical values
* glossary retrieval
* relevant schema retrieval
* catalog refresh after schema changes

DEMONSTRATION

At completion demonstrate what context would be retrieved for:

“How many Red Hat servers are there?”

Show:

user concepts
→ glossary concepts
→ relevant tables
→ relevant columns
→ known database values

Do not generate the final SQL yet.

Stop after Phase 1.

Phase 2 — Terminology, Entity, Fuzzy and Pattern Resolution

Implement Phase 2 of Jevy: Terminology and Database Value Resolution.

OBJECTIVE

Users will rarely use exact database terminology.

Examples:

Red Hat
redhat
RedHat
RHEL
RHEL8
RHEL 8
Red Hat Linux

The actual database may contain:

Red Hat Enterprise Linux Server 8.9

Jevy must understand these variations without depending entirely on the LLM.

BUILD A RESOLUTION PIPELINE

For important terms extracted from the question:

1. normalize case
2. normalize whitespace
3. normalize punctuation
4. tokenize
5. check business glossary
6. check aliases
7. check exact database values
8. check case-insensitive values
9. check token/substring matches
10. check controlled pattern matching
11. check fuzzy similarity
12. use LLM semantic resolution only when deterministic resolution is insufficient

Example:

User:

“How many Red Hat servers?”

Do NOT blindly generate:

os_name = ‘Red Hat’

First inspect known values.

Potential known values:

Red Hat Enterprise Linux 7
Red Hat Enterprise Linux 8
Red Hat Enterprise Linux 9

Resolve the user concept against actual database values.

RESOLUTION RESULT

Return structured information such as:

{
“concept”: “operating_system”,
“user_term”: “Red Hat”,
“column”: “servers.os_name”,
“match_type”: “ALIAS”,
“candidate_values”: [
“Red Hat Enterprise Linux 7”,
“Red Hat Enterprise Linux 8”,
“Red Hat Enterprise Linux 9”
],
“confidence”: 0.98
}

CONFIDENCE LEVELS

Support:

EXACT
ALIAS
HIGH_CONFIDENCE_FUZZY
SEMANTIC
AMBIGUOUS
UNKNOWN

Do not silently guess when genuinely ambiguous.

PATTERN / REGEX SUPPORT

Jevy must understand questions such as:

“servers starting with nyc-”

“hostnames containing db”

“applications ending with API”

“RHEL 8 servers”

“servers matching app-*”

Use safe deterministic SQL operators whenever possible:

LIKE
GLOB
controlled REGEXP if explicitly implemented

Never interpolate raw user regex directly into SQL.

Generate parameterized filters.

Example:

{
“column”: “hostname”,
“operator”: “LIKE”,
“value”: “%db%”
}

Prefer simple LIKE/GLOB operations over regex when they satisfy the request.

CACHE

Cache frequent resolutions.

Examples:

RHEL
Red Hat
PROD
production
critical

TESTS

Create tests for:

Red Hat
redhat
RedHat
RHEL
RHEL8
RHEL 8
Windows
Win Server
prod
production
crit
critical
hostname contains db
hostname starts with nyc-
application ends with API

DEMONSTRATE

Show the full resolution process for:

“How many Red Hat servers are there?”

Stop after Phase 2.

Phase 3 — SQL-Only Jevy

Implement Phase 3: Make Jevy an extremely reliable SQLite analytics agent.

IMPORTANT

Temporarily remove MCP, external APIs, and chart generation from Jevy’s reasoning path.

Jevy’s only analytical data capability during this phase is SQLite.

OBJECTIVE

Make SQLite analytics highly reliable before introducing additional capabilities.

TARGET PIPELINE

User Question
→ Question Classification
→ Intent Extraction
→ Entity/Value Resolution
→ Relevant Schema Retrieval
→ Query Planning
→ SQL Generation
→ SQL Validation
→ SQL Execution
→ Result Validation
→ Answer

Do NOT ask the LLM to solve everything using one giant prompt.

QUESTION CLASSIFICATION

First distinguish:

CONVERSATIONAL

ANALYTICAL

DATABASE ANALYTICAL

Example:

“Hello Jevy”

requires no database.

“How many production servers are there?”

requires database evidence.

INTENT REPRESENTATION

Create compact structured intent.

Example:

{
“operation”: “count”,
“entity”: “servers”,
“metrics”: [],
“dimensions”: [],
“filters”: [
{
“concept”: “operating_system”,
“value”: “Red Hat”
}
]
}

SCHEMA RETRIEVAL

Retrieve only relevant schema.

Never automatically send the complete database schema.

QUERY PLANNING

Before SQL generation, create a compact query plan.

Example:

{
“base_table”: “servers”,
“joins”: [],
“filters”: [“servers.os_name matches Red Hat variants”],
“aggregation”: “COUNT”
}

SQL GENERATION

Give Qwen only:

* original question
* normalized intent
* resolved values
* relevant schema
* valid relationships
* relevant successful SQL examples if available

Require structured output:

{
“sql”: “…”,
“parameters”: [],
“expected_result_shape”: “…”,
“assumptions”: []
}

Do not request verbose chain-of-thought.

SQL SAFETY

Only permit read operations.

Reject:

INSERT
UPDATE
DELETE
DROP
ALTER
CREATE
ATTACH
dangerous PRAGMA operations
multiple SQL statements

Use parameterized queries.

Before execution validate:

* tables exist
* columns exist
* joins are legal
* SQL is read-only

ANSWER GENERATION

Final analytical answers MUST be derived from executed SQL results.

No successful execution means Jevy cannot state internal company-data numbers as fact.

SMALL QWEN OPTIMIZATION

Optimize aggressively:

* short prompts
* structured JSON
* relevant schema only
* relevant history only
* deterministic preprocessing
* deterministic validation
* cache static instructions where possible
* reuse known successful query patterns

Stop after Phase 3.

Phase 4 — Self-Healing SQL Loop

Implement Phase 4: Jevy Self-Healing SQL.

OBJECTIVE

Incorrect SQL should not immediately cause the request to fail.

Implement:

PLAN
→ GENERATE
→ VALIDATE
→ EXECUTE
→ OBSERVE
→ REPAIR
→ EXECUTE
→ VERIFY
→ ANSWER

Configure maximum attempts.

Start with:

MAX_ATTEMPTS = 4

ERROR CLASSIFICATION

Before asking the LLM to repair anything, classify errors deterministically.

Support categories including:

UNKNOWN_COLUMN
UNKNOWN_TABLE
AMBIGUOUS_COLUMN
INVALID_JOIN
SYNTAX_ERROR
TYPE_MISMATCH
INVALID_VALUE
EMPTY_RESULT
TIMEOUT
OTHER

REPAIR CONTEXT

Do NOT resend the entire original prompt during repair.

Provide only:

* normalized intent
* failed SQL
* parameters
* SQLite error
* relevant schema
* resolved database values
* useful previous attempt information

Example:

Attempt 1:

SELECT COUNT(*)
FROM servers
WHERE os = ?

Error:

no such column: os

Relevant schema:

servers.os_name

Repair:

SELECT COUNT(*)
FROM servers
WHERE LOWER(os_name) LIKE LOWER(?)

Parameter:

%red hat%

Execute again.

EMPTY RESULTS

Zero rows do not automatically mean failure.

Determine whether:

1. zero is legitimate
2. terminology resolution may be wrong
3. value does not exist
4. filter may be incorrect
5. join may be incorrect

Do not silently weaken user filters.

LOOP PROTECTION

Never execute identical failed SQL repeatedly.

Track query fingerprints.

Stop after MAX_ATTEMPTS.

OBSERVABILITY

Record:

attempt number
query
parameters
validation result
execution status
error category
repair action
final status

Do NOT expose private chain-of-thought.

Operational traces are sufficient.

Example:

Question
→ Red Hat resolved
→ servers selected
→ SQL attempt 1
→ UNKNOWN_COLUMN
→ repaired column
→ SQL attempt 2
→ success
→ result verified

Add automated tests deliberately causing SQL failures and verify that Jevy repairs them.

Stop after Phase 4.

Phase 5 — Evidence, Verification and Zero Hallucination

Implement Phase 5: Jevy Evidence-Grounded Analytics.

CORE INVARIANT

Jevy MUST NOT make factual claims about internal company data unless those claims are supported by successful data operations.

Create an Evidence object.

Example:

{
“question”: “…”,
“sources”: [“sqlite”],
“queries”: [],
“parameters”: [],
“results”: [],
“result_metadata”: {},
“verified”: true
}

ANALYTICAL ANSWERS

For questions such as:

“How many production servers are there?”

Jevy must obtain evidence.

If SQL returns:

147

Jevy may answer:

“There are 147 production servers.”

If execution fails, Jevy must NOT invent a value.

NO EVIDENCE

NO INTERNAL-DATA FACTUAL CLAIM

Remove all dummy/template analytical values from production paths.

VERIFICATION

Before answer generation validate:

* query succeeded
* result exists
* result shape matches expectation
* aggregation makes sense
* required filters were applied
* required joins were applied

Where possible perform deterministic verification rather than another LLM call.

PROVENANCE

Maintain:

Question
→ Intent
→ Resolution
→ Query
→ Parameters
→ Result
→ Verification
→ Answer

NUMERIC GROUNDING

Numbers in analytical answers must originate from:

* query results
* tool results
* deterministic calculations performed on verified results

Never allow the language model to invent numerical values.

TESTING

Create adversarial tests attempting to make Jevy:

* answer without querying
* invent counts
* use values from previous questions
* use dummy values
* answer after failed SQL

These tests must fail if unsupported analytical claims reach the user.

Stop after Phase 5.

Phase 6 — Jevy Query Memory / Analytics Cookbook

Implement Phase 6: Jevy Query Memory.

OBJECTIVE

Jevy should learn reusable analytical patterns from successful queries.

This is especially important because production currently uses a small Qwen model.

Do not regenerate SQL from scratch when Jevy already knows a verified pattern.

After successful verified execution store a compact reusable representation.

Example:

{
“intent_signature”: “count servers filtered by operating_system”,
“tables”: [“servers”],
“columns”: [“os_name”],
“join_path”: [],
“sql_template”: “…”,
“success_count”: 17,
“failure_count”: 0
}

For complex analytics store:

* normalized intent
* entities
* metrics
* dimensions
* filters
* tables
* columns
* join path
* SQL structure
* success count
* failure count
* last validated timestamp

QUERY PROCESS

Before generating new SQL:

1. normalize question
2. determine intent
3. search successful query patterns
4. calculate pattern similarity
5. reuse a pattern if confidence is sufficiently high
6. substitute current filter values safely
7. validate SQL
8. execute
9. fall back to generation if reuse fails

IMPORTANT

Never accidentally reuse old literal filter values.

Example:

Stored:

critical vulnerabilities by application

New:

high vulnerabilities by application

Reuse:

tables
joins
aggregation
grouping

Replace:

severity CRITICAL
with
severity HIGH

Only VERIFIED successful patterns should improve query memory.

Failed queries may be stored separately for debugging but must not become trusted patterns.

MEASURE

Track:

memory hit rate
SQL generation avoided
LLM calls avoided
token savings
latency savings
success rate

Stop after Phase 6.

Phase 7 — Golden Question Evaluation System

Implement Phase 7: Automated Jevy Evaluation.

OBJECTIVE

We need measurable evidence that Jevy is improving rather than relying on manual testing.

Create a Golden Question evaluation dataset.

Include:

simple counts
filters
multiple filters
grouping
sorting
top-N
bottom-N
dates
ranges
aggregations
single joins
multi-table joins
business terminology
aliases
fuzzy terminology
patterns
regex-like requests
comparisons
zero-result questions
ambiguous questions
invalid questions

Examples:

“How many servers do we have?”

“How many Red Hat servers?”

“How many RHEL 8 production servers?”

“How many PROD servers?”

“Top 10 applications by server count.”

“Critical vulnerabilities by application.”

“Servers whose hostname contains db.”

“Servers beginning with nyc-.”

“Compare PROD and UAT server counts.”

Do NOT validate only exact SQL strings.

Different SQL can produce the same correct result.

Validate:

* correct result
* correct tables
* correct filters
* correct joins
* successful execution
* grounded answer
* retry behavior
* unsupported claims

METRICS

Report:

Answer Accuracy

SQL Execution Success Rate

First-Attempt SQL Success Rate

Repair Success Rate

Grounding Success Rate

Hallucination Rate

Average SQL Attempts

Average LLM Calls

Average Input Tokens

Average Output Tokens

Average Latency

Query Memory Hit Rate

Create one command that executes the complete Jevy evaluation suite.

Generate regression reports.

Every future change to Jevy’s reasoning architecture must run this suite.

Do not proceed to MCP until SQLite performance reaches an agreed reliability threshold.

Stop after Phase 7.

Phase 8 — MCP/API Capability Registry and Router

Implement Phase 8: Jevy MCP/API Capability Routing.

IMPORTANT

SQLite analytics is now considered a stable capability.

Do not destabilize the existing SQLite pipeline.

OBJECTIVE

Introduce external APIs and MCP tools without overwhelming the small Qwen model.

CAPABILITY REGISTRY

Create a machine-readable registry.

Each capability should define something similar to:

{
“name”: “get_live_server_status”,
“description”: “Returns current operational status for servers.”,
“domains”: [“server”, “live_status”],
“freshness”: “real-time”,
“inputs”: {},
“output_schema”: {},
“cost”: “low”
}

Include SQLite as a capability:

{
“name”: “sqlite_analytics”,
“domains”: [“server_inventory”, “historical_data”, “…”]
}

TWO-STAGE ROUTING

Do NOT provide every available tool to Qwen.

Implement:

Question
→ Domain Detection
→ Capability Shortlisting
→ Planning

Example:

“Which production Red Hat servers are currently offline?”

Required domains:

server inventory
operating system
environment
live server status

Capabilities:

SQLite
get_live_server_status

Only these capabilities should enter planning context.

PLANNER OUTPUT

Use compact structured plans:

{
“steps”: [
{
“id”: 1,
“capability”: “sqlite_analytics”,
“purpose”: “Find production Red Hat servers”
},
{
“id”: 2,
“capability”: “get_live_server_status”,
“depends_on”: [1],
“purpose”: “Determine which matching servers are offline”
}
]
}

Support dependent calls.

Tool failures should enter an observe/repair loop.

Never fabricate tool output.

Record tool provenance.

Optimize descriptions aggressively for the small model.

Stop after Phase 8.

Phase 9 — Multi-Step Agentic Execution

Implement Phase 9: Jevy Multi-Step Agentic Execution.

OBJECTIVE

Jevy must solve questions requiring multiple data operations without requiring one giant reasoning call.

Implement a bounded execution loop:

UNDERSTAND
→ PLAN
→ EXECUTE NEXT STEP
→ OBSERVE
→ UPDATE STATE
→ DETERMINE IF MORE INFORMATION IS NEEDED
→ EXECUTE
→ VERIFY
→ ANSWER

Create explicit execution state.

Example:

{
“question”: “…”,
“intent”: {},
“resolved_entities”: {},
“plan”: [],
“completed_steps”: [],
“pending_steps”: [],
“evidence”: [],
“errors”: [],
“status”: “RUNNING”
}

Each iteration should receive only the context needed for the next decision.

Do NOT repeatedly send:

* complete conversation history
* complete schema
* every tool
* every previous result

Use references/summaries where possible.

Jevy may:

* query SQLite
* inspect results
* call MCP
* inspect results
* perform another SQL query
* repair failed steps
* combine verified results
* determine when sufficient evidence exists

BOUNDED AUTONOMY

Configure:

maximum reasoning iterations
maximum SQL attempts
maximum tool attempts
maximum total execution time

The loop must terminate safely.

SMALL MODEL STRATEGY

Prefer multiple small, focused Qwen calls over one enormous complicated prompt when doing so improves reliability.

Each call should have one clear responsibility.

Examples:

resolve
plan
generate
repair
summarize

Do not request verbose reasoning.

Use structured outputs.

Stop after Phase 9.

Phase 10 — Charts and Visualization

Implement Phase 10: Jevy Visualization.

CORE RULE

The LLM must NEVER invent chart data.

Visualization happens only AFTER verified structured data exists.

Pipeline:

Question
→ Analytics
→ Verified Dataset
→ Visualization Decision
→ Visualization Specification
→ Frontend Renderer

Example:

“Show critical vulnerabilities by application.”

First obtain verified data:

[
{“application”: “Payments”, “count”: 37},
{“application”: “Trading”, “count”: 24}
]

Then determine visualization.

Return a structured visualization specification such as:

{
“type”: “bar”,
“title”: “Critical Vulnerabilities by Application”,
“x”: “application”,
“y”: “count”,
“data_source”: “verified_result_123”
}

Support appropriate visualization types:

bar
line
area
pie/donut only where appropriate
scatter
table
metric/KPI
stacked bar where appropriate

Prefer deterministic visualization rules.

Examples:

time series → line

category comparison → bar

single metric → KPI

large detailed dataset → table

The LLM should only choose visualization when deterministic rules are insufficient.

Charts must reference verified result data.

Never regenerate chart numbers using the LLM.

Stop after Phase 10.

Phase 11 — Context and Conversation Memory

Implement Phase 11: Jevy Context and Conversation Memory.

OBJECTIVE

Jevy should understand follow-up questions without sending the complete conversation to Qwen.

Example:

User:
“Show Red Hat servers in production.”

Follow-up:
“Only ones with critical vulnerabilities.”

Follow-up:
“Now group them by application.”

Jevy must understand that these questions belong to the same analytical context.

Maintain structured conversation state.

Example:

{
“current_entity”: “servers”,
“filters”: {
“os”: “Red Hat”,
“environment”: “PROD”,
“severity”: “CRITICAL”
},
“dimensions”: [],
“previous_result_reference”: “…”,
“previous_query_reference”: “…”
}

Separate:

CONVERSATION MEMORY

from

DATABASE KNOWLEDGE

from

QUERY MEMORY

Do not mix these systems.

CONVERSATION MEMORY:
what this user is currently discussing

DATABASE KNOWLEDGE:
what tables/fields/business concepts mean

QUERY MEMORY:
previously successful reusable analytical patterns

Use structured state instead of repeatedly supplying complete chat transcripts.

Include only relevant recent natural-language messages when required.

Implement context expiration and topic-reset detection.

Test multi-turn questions extensively.

Stop after Phase 11.

Phase 12 — Small-Model Optimization

This phase matters a lot for your Qwen setup.

Implement Phase 12: Optimize Jevy specifically for a small inexpensive Qwen model.

IMPORTANT

Do NOT sacrifice accuracy merely to reduce tokens.

First measure the current system.

Collect:

average LLM calls/question
average input tokens/question
average output tokens/question
average latency
SQL first-attempt success
repair rate
answer accuracy
hallucination rate

Then optimize.

APPLY THESE PRINCIPLES

1. Deterministic code before LLM.

Use code for:

normalization
schema lookup
alias matching
value lookup
SQL validation
result validation
simple visualization selection

2. Retrieve rather than dump.

Never send the entire:

database schema
tool registry
conversation
query memory

Retrieve only relevant pieces.

3. Structured output.

Prefer compact JSON contracts.

4. Specialized calls.

Prefer small focused calls:

CLASSIFY

RESOLVE

PLAN

GENERATE

REPAIR

ANSWER

Do not make one prompt perform every responsibility.

5. Skip unnecessary LLM calls.

Example:

If query memory contains a high-confidence verified template, reuse it directly.

6. Cache stable information.

Cache:

schema
relationships
glossary
capability descriptions
common resolutions
successful query templates

7. Dynamic model escalation.

Design Jevy so models can eventually be tiered.

Example:

small Qwen
→ normal requests

small Qwen + repair
→ moderate requests

larger model
→ only unusually complex/ambiguous requests

Do not require the larger model today, but make the architecture support future escalation.

8. Context budgets.

Establish token budgets for each agent operation.

If retrieved context becomes too large, rank and reduce it before calling the model.

9. Measure optimization.

Compare before vs after:

accuracy
tokens
latency
LLM calls
SQL success
repair success

Optimization is successful only if cost decreases without unacceptable accuracy regression.

Stop after Phase 12.

Phase 13 — Production Hardening and Jevy Observability

Implement Phase 13: Jevy Production Hardening and Observability.

OBJECTIVE

Every Jevy analytical request should be diagnosable.

Create a trace ID for every request.

Capture structured operational events:

QUESTION_RECEIVED
INTENT_RESOLVED
SCHEMA_RETRIEVED
VALUE_RESOLVED
QUERY_PATTERN_REUSED
SQL_GENERATED
SQL_VALIDATED
SQL_EXECUTED
SQL_FAILED
SQL_REPAIRED
TOOL_SELECTED
TOOL_EXECUTED
TOOL_FAILED
EVIDENCE_VERIFIED
ANSWER_GENERATED
REQUEST_COMPLETED
REQUEST_FAILED

Capture metrics including:

latency
LLM calls
tokens
SQL attempts
tool calls
repair attempts
query-memory hits
errors
final status

Create useful development/debug views.

For one Jevy request, developers should be able to inspect:

Question

↓
Intent

↓
Resolved terminology

↓
Selected schema

↓
Execution plan

↓
SQL/tool calls

↓
Errors/repairs

↓
Evidence

↓
Final answer

Do not store or expose private chain-of-thought.

Only operational decisions, structured state, queries, tool activity, results metadata, and evidence should be visible.

Add protections for:

SQL injection
prompt injection through database content
malicious tool output
oversized results
timeouts
runaway loops
duplicate execution
unexpected tool responses

Add configurable limits.

Run the complete Jevy golden-question evaluation suite after hardening.

Produce a final comparison:

BEFORE JEVY ARCHITECTURE

vs

AFTER JEVY ARCHITECTURE

including:

accuracy
hallucination rate
SQL success
repair success
average LLM calls
average tokens
latency
failure rate

Stop after Phase 13.

The order I recommend

Do not jump directly into agent loops. The progression should be:

0 Audit → 1 Database Intelligence → 2 Terminology Resolution → 3 SQL Agent → 4 Self-Repair → 5 Evidence/Verification → 6 Query Memory → 7 Evaluation → 8 MCP/APIs → 9 Agentic Execution → 10 Charts → 11 Conversation Context → 12 Qwen Optimization → 13 Production Hardening.

There is one architectural principle I would keep throughout all 13 phases:

Make Jevy smart through the system around the model, not by expecting the model itself to be smart.

For example, "How many RHEL 8 prod servers?" should eventually become something close to:

RHEL 8 → deterministic terminology resolver → actual DB values → prod → glossary → PROD → schema retriever → only servers fields supplied → query-memory lookup → SQL → validation → execution → if failed, repair loop → evidence validation → answer.

That architecture can make even a relatively small Qwen model considerably more reliable because you’re reducing how many things it has to figure out simultaneously.