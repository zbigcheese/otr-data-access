---
name: otr-data
description: Analyse OTR Systems' live recruiting data (leads, drivers, background checks, companies, invoices, recruiters) through the otr-data MCP server. Use whenever the user asks about recruiter performance, lead conversion, hires, companies, revenue, or anything stored in the OTR app, or asks to "look at the data" or "pull numbers" from OTR.
---

# Working with OTR's live data

You have read-only access to the production database of OTR Systems, a driver-recruiting agency, through the `otr-data` MCP server (tools `context`, `guide`, `list_tables`, `describe_table`, `search_schema`, `query`). The server hosts a data guide written for analysts; it is the authority on what every table, status and date means and on the metric definitions staff see in the app. **Read the guide before writing SQL.** Guessing column meanings from their names produces wrong numbers here (dates without time, zero dates, three different "who gets credit" columns, recycled leads that reset their dates).

## Order of work

1. `context` once: who you are connected as, the database, the server time zone, row and time limits.
2. `guide` with `topic` = `overview`, then the domain topic (`leads`, `drivers`, `background-checks`, `companies`, `invoices`, `people`, `other`). For any performance question read `metrics` too, and look in `recipes` for a ready query before writing your own. `guide` with `search` finds the paragraphs about a word.
3. `describe_table` for every table you will query (columns, keys, declared and guessed relations, and the guide's notes for that table).
4. `query`: one statement, aggregated in SQL. Default 200 rows, up to 5000 with `max_rows`; 30 seconds per statement. Run `EXPLAIN <query>` first when joining large tables (`activities`, `lead_calls`, `lead_lead_calls`, `leads`, `documents`, `background_check_documents`). Never `SELECT *` from a large table without a WHERE and LIMIT.
5. Report numbers with their definition: period column, attribution column, statuses counted, inclusion of deleted companies' checks, and the server time of the pull.

## Facts you must not forget

- Timestamps are Chicago wall-clock time stored as naive values; `NOW()` in SQL is server time (UTC). Compare dates as the guide says; do not `CONVERT_TZ` stored values.
- Date columns such as `leads.date_added`, `background_checks.date_sent`, `date_hired` have no time of day; `'0000-00-00'` means NULL on live rows (`NULLIF(col, '0000-00-00')`).
- Attribution: lead work -> `lead_calls.recruiter_id` (calls) and `leads.recruiter` (outcomes); checks and hires -> `background_checks.recruiter_id` (who created the check; what the app and rewards use); current ownership -> `drivers.owner_id`. Say which you used.
- "Recruiter" and "admin" are not roles; admins recruit too; user 5 (`alex`) is a parking account holding most `not_interested` leads. External recruiters (role `external_recruiter`) and sales accounts are not recruiters in rankings.
- Lead recycling (source 37) resets `date_added` and deletes calls; `created_at` keeps the true arrival.
- `drivers.status = 'hired'` is hand-maintained; hires are `background_checks.status = 'hired'` with `date_hired`.
- Deletes are hard; `background_checks.deleted = 1` only marks deleted companies and no app report filters it.
- Nothing you do can write. Credentials, sessions and the Data Access tables themselves are not readable; `users.password` is refused.

## Answering styles

- **Scorecard / ranking**: one row per recruiter with volume, rates and denominators; normalise per lead received and per active day; show lead mix by source; exclude disabled accounts and user 5; flag recruiters with small denominators.
- **Trend**: group by month on the period column the app uses for that metric; compare like with like (sent by `date_sent`, hires by `date_hired`, revenue by `date_issued`).
- **Why**: break the funnel (received -> called -> reached -> driver -> check -> approved -> hired) and locate the stage where a recruiter, source or company diverges; check data quality (zero dates, recycled share, attribution drift) before concluding.
- **Recommendations**: tie them to the stage and to levers the office controls (lead mapping and daily caps in `recruiter_source_mappings`, call speed, company mix, approval bottlenecks, billing collection); quantify what closing the gap would be worth.

## Guardrails

- Everything is logged with your name under Settings > Data Access; keep queries purposeful.
- Personal data (driver names, phones, emails, SSN documents' text) is visible: aggregate when the question is about performance, show individual rows only when asked, never paste bulk personal data into files or messages that leave the conversation.
- If a tool answers 401 or "Unauthorized", run `/mcp` and authenticate again (the browser session of an authorised OTR account is the login); if it says the account is not allowed, the root user must grant it under Settings > Data Access.
