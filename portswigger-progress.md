### Lab: SQL Injection with Filter Bypass via XML Encoding (WAF Bypass + Data Extraction)


## Objective
Bypass a Web Application Firewall (WAF) protecting an XML-based SQL injection
point, then extract usernames and passwords from the database.

## Steps Taken

### 1. Identified the injection point
Observed that the stock check feature sent `productId` and `storeId` to the
server in XML format via a `POST /product/stock` request. Sent this request
to Burp Repeater for testing.

### 2. Confirmed input was being evaluated
Tested the `storeId` field with a mathematical expression instead of a plain
number:
```xml
<storeId>1+1</storeId>
```
The application returned stock for a different store — confirming the value
was being evaluated server-side, not just passed through as a literal string.

### 3. Attempted to determine column count
Tried a UNION SELECT to test the underlying query:
```xml
<storeId>1 UNION SELECT NULL</storeId>
```
This request was blocked — the application flagged it as a potential attack,
indicating a WAF (Web Application Firewall) was filtering suspicious SQL
keywords/patterns before they reached the application.

### 4. Bypassed the WAF using XML entity encoding
Since the injection point was inside XML data, encoded the payload using XML
entities (via Burp's Hackvertor extension: Encode → hex_entities), converting
the SQL keywords into a form the WAF's pattern-matching didn't recognize as
malicious, while the XML parser on the server still decoded it back to valid
SQL before execution.

Resent the encoded request and received a normal response — confirming the
WAF was successfully bypassed.

### 5. Determined the query returns a single column
Testing showed that returning more than one column caused the application to
respond with "0 units," indicating an error — meaning the original query only
selects a single column.

### 6. Extracted credentials via string concatenation
Since only one column could be returned, concatenated the username and
password fields into a single value using a separator character:
```xml
<storeId><@hex_entities>1 UNION SELECT username || '~' || password FROM users</@hex_entities></storeId>
```
This returned all usernames and passwords from the `users` table, separated
by `~`, within the single allowed column.

### 7. Logged in as administrator
Used the extracted administrator credentials to log in, gaining
administrative access and solving the lab.

## What I Learned
This lab combined three separate concepts into one real attack chain:

1. **WAFs filter patterns, not intent** — the WAF blocked the literal string
   `UNION SELECT`, but had no way to catch the same instruction once it was
   XML-entity encoded and later decoded server-side. This shows why
   defense-in-depth matters: a WAF alone is not a substitute for fixing the
   underlying SQL injection vulnerability.
2. **Column count matters for UNION-based injection** — before data can be
   exfiltrated via UNION SELECT, the attacker must match the number of
   columns in the original query, or the query fails.
3. **String concatenation bypasses single-column limitations** — when only
   one column can be returned, multiple values (like username and password)
   can still be extracted together by concatenating them with a delimiter.

This was the most advanced lab so far in my SQL injection learning, combining
WAF evasion, blind query structure analysis, and credential extraction into a
single realistic attack scenario.
