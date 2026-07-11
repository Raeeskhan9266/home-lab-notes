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
