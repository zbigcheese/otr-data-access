# OTR Data Access plugin for Claude Code

Gives authorised OTR Systems owners read-only access to the **live** OTR database from Claude Code, with a skill that explains how the recruiting data is created and used so analyses match what the app shows.

The plugin itself contains no credentials and no server code. It points Claude Code at the MCP server built into the OTR app (`https://otrsystems.com/mcp`) and signs in through your OTR account in the browser (OAuth 2.1 with PKCE). The server side lives in the OTR repository (`app/src/Otr/DataAccess`, `docs/data-access`).

## Who can use it

Only the root user and the accounts root has granted under **Settings > Data Access** in the OTR app. Everyone else gets a clear "not allowed" page when they try to connect. Every query is logged with the user's name and visible to root.

## Install (once per computer)

In Claude Code:

```
/plugin marketplace add zbigcheese/otr-data-access
/plugin install otr-data@otr-data-access
```

Then connect:

```
/mcp
```

Pick **otr-data**, choose **Authenticate**. Your browser opens the OTR site. If you are signed in to an authorised account you see a consent page; approve it and come back to Claude Code. The token lasts an hour and renews itself for 30 days; after that (or after root ends the connection) `/mcp` asks you to sign in again.

## Use

Ask in plain language. The skill makes Claude read the data guide first, then write SQL through the `query` tool. Examples:

- "Scorecard for all recruiters for Q3: leads, calls, drivers, sent, hired, with rates."
- "Which lead sources convert best this year, and how has that changed by month?"
- "Where do our drivers get rejected: outcomes by company for checks sent since January."
- "How fast does each recruiter make the first call, and who leaves leads uncalled?"
- "Invoiced vs paid per month and the companies with the largest outstanding balance."

Claude states the definitions it used (which date column, which attribution) because the data has several reasonable answers for "who gets credit"; ask for a different one when you mean it.

## Tools the server exposes

| tool | what it does |
|---|---|
| `context` | who is connected, database, server time zone, limits, guide topics |
| `guide` | the data guide: topic list, one topic, or a word search |
| `list_tables` | readable tables with row counts and purposes |
| `describe_table` | columns, indexes, relations and the guide's notes for a table |
| `search_schema` | find columns and guide paragraphs by word |
| `query` | one read-only SQL statement (SELECT / WITH / SHOW / DESCRIBE / EXPLAIN), rows as JSON |

Limits: 200 rows per query by default, 5000 at most; 30 seconds per statement. Credentials, sessions and the Data Access tables are not readable; `users.password` is refused whatever the SQL says.

## Pointing at another server

The server URL is `https://otrsystems.com/mcp` by default. To use the staging copy set `OTR_DATA_ACCESS_URL=https://otr.clicktools.app/mcp` in your environment before starting Claude Code.

## Server-side setup (for the administrator)

Done in the OTR repository and on the server; the Data Access page shows the state of each item.

1. Deploy the OTR app (migration `v450 DataAccess` creates the `data_access_*` tables; routes `/mcp`, `/oauth/*`, `/.well-known/oauth-*`).
2. Create a SELECT-only MySQL account and put it in the server's env file, then recreate the web container:
   ```sql
   CREATE USER 'otr_readonly'@'%' IDENTIFIED BY '<long random password>';
   GRANT SELECT ON `<database>`.* TO 'otr_readonly'@'%';
   ```
   ```
   DATA_ACCESS_DB_USER=otr_readonly
   DATA_ACCESS_DB_PASSWORD=<the password>
   ```
   Without it queries run as the app account behind the SQL guard and a READ ONLY transaction; the page shows a warning.
3. Grant accounts under Settings > Data Access. Root always has access.
4. Keep `docs/data-access/*.md` in the OTR repo current when the schema or workflows change: the server serves those files as the guide.

## Security model in one paragraph

Claude Code registers itself as a public OAuth client, sends the owner's browser to the OTR site, and only the signed-in OTR session of an authorised account can approve; the code is exchanged with PKCE for a short-lived token whose SHA-256 hash is stored. Tokens are bound to the account's grant: revoking the grant or ending the connection on the Data Access page invalidates them immediately. The MCP endpoint accepts only read statements (statement guard), runs them in a READ ONLY transaction under a SELECT-only database account with a server-side time limit, refuses credential tables and columns, caps result rows, and logs every call.
