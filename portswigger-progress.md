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

TrackingId=xyz' AND (SELECT SUBSTRING(password,1,1) FROM users WHERE username='administrator')='§a§
To test the character at each position, you'll need to send suitable payloads in the payload position that you've defined. You can assume that the password contains only lowercase alphanumeric characters. In the Payloads side panel, check that Simple list is selected, and under Payload configuration add the payloads in the range a - z and 0 - 9. You can select these easily using the Add from list drop-down.
To be able to tell when the correct character was submitted, you'll need to grep each response for the expression Welcome back. To do this, click on the  Settings tab to open the Settings side panel. In the Grep - Match section, clear existing entries in the list, then add the value Welcome back.
Launch the attack by clicking the  Start attack button.
Review the attack results to find the value of the character at the first position. You should see a column in the results called Welcome back. One of the rows should have a tick in this column. The payload showing for that row is the value of the character at the first position.
Now, you simply need to re-run the attack for each of the other character positions in the password, to determine their value. To do this, go back to the Intruder tab, and change the specified offset from 1 to 2. You should then see the following as the cookie value:

TrackingId=xyz' AND (SELECT SUBSTRING(password,2,1) FROM users WHERE username='administrator')='a
Launch the modified attack, review the results, and note the character at the second offset.
Continue this process testing offset 3, 4, and so on, until you have the whole password.
In the browser, click My account to open the login page. Use the password to log in as the administrator user.
Note
For more advanced users, the solution described here could be made more elegant in various ways. For example, instead of iterating over every character, you could perform a binary search of the character space. Or you could create a single Intruder attack with two payload positions and the cluster bomb attack type, and work through all permutations of offsets and character values.
