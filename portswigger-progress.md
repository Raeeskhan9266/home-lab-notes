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



