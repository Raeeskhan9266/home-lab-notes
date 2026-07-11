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
