# Cross-Site Scripting (XSS) — PortSwigger Web Security Academy

Tracking hands-on labs completed for the XSS topic, following the same
methodology used for SQL Injection: understanding the vulnerability,
constructing the payload, and documenting the underlying technique.

---

## Lab 1: Reflected XSS into HTML Context with Nothing Encoded

Topic: Cross-Site Scripting | Difficulty: Apprentice

## Vulnerability
The search functionality takes user input (the search term) and reflects
it directly back into the HTML of the response page — without encoding or
sanitizing any special characters. This means any HTML or JavaScript
submitted in the search box gets rendered and executed directly by the
browser, exactly as if it were part of the page's original code.

## What is Reflected XSS?
Reflected XSS occurs when user-supplied input is immediately echoed back
in the server's response, without being stored anywhere. The malicious
script only executes when a specific crafted request/link is used — unlike
Stored XSS, where the payload is saved on the server and affects every user
who later views the page.

## Steps Taken
1. Entered the following payload into the search box:
<script>alert(1)</script>
2. Clicked "Search"

## How the Payload Works
Since the application inserts the search term directly into the HTML
response without encoding characters like `<` and `>`, the browser
interprets `<script>alert(1)</script>` as literal HTML/JavaScript rather
than plain text. This causes the browser to execute the enclosed
JavaScript (`alert(1)`), popping up an alert box — proving that arbitrary
JavaScript can be injected and executed in the context of the page.

## Result
Successfully triggered the `alert(1)` popup, confirming the reflected XSS
vulnerability and solving the lab.

## What I Learned
This lab demonstrated the most fundamental form of XSS: when input is
reflected into an HTML context with zero encoding, injecting a `<script>`
tag directly is often all that's needed. In a real attack, this same
technique could be used to steal cookies/session tokens, redirect users to
malicious sites, or perform actions on behalf of the victim — the `alert()`
call here is simply a safe, standard proof-of-concept used to demonstrate
that arbitrary script execution is possible. This connects conceptually to
the SQL injection labs: in both cases, the root cause is the same pattern —
user input being trusted and inserted directly into a context (SQL query
or HTML page) without proper sanitization or encoding.



## Lab 2: Stored XSS into HTML Context with Nothing Encoded

Topic: Cross-Site Scripting | Difficulty: Apprentice

## Vulnerability
The blog's comment functionality accepts user input (comment text) and
stores it directly, then renders it back — unencoded — every time the blog
post page is loaded. Unlike Lab 1's search feature (which only reflected
the payload back once, immediately), this comment is permanently saved on
the server and displayed to anyone who later visits that blog post.

## What Makes This "Stored" (vs. Reflected)
- **Reflected XSS (Lab 1):** payload only executes for the specific request
  containing it — an attacker typically needs to trick a victim into
  clicking a specially crafted link
- **Stored XSS (this lab):** payload is saved in the application's data
  (e.g. a database), and executes automatically for *every* user who
  simply views the affected page — no special link or trick needed, making
  it generally more dangerous since it can silently affect many victims

## Steps Taken
1. Filled in the comment form's Name, Email, and Website fields
2. Entered the following payload into the Comment box:
<script>alert(1)</script>
3. Clicked "Post comment"
4. Navigated back to the blog post page

## How the Payload Works
Same underlying mechanism as Lab 1 — the comment text is inserted into the
page's HTML without encoding special characters. Because the comment is
now permanently stored and re-rendered on every page load, the
`<script>alert(1)</script>` payload executes automatically each time the
blog post is viewed, not just once.

## Result
Successfully triggered the `alert(1)` popup upon revisiting the blog post,
confirming the stored XSS vulnerability and solving the lab.

## What I Learned
This lab reinforced the distinction between reflected and stored XSS —
while the injection technique itself was identical to Lab 1, the *impact*
is significantly higher here because the malicious script persists and
runs for every visitor automatically, without requiring the attacker to
send a crafted link to a specific victim. In a real-world scenario, a
stored XSS payload in a public comment section could silently compromise
every user who views that page — for example, stealing session cookies
from many victims simultaneously, rather than just one targeted individual.
This is why stored XSS is generally treated as more severe than reflected
XSS in vulnerability assessments.
