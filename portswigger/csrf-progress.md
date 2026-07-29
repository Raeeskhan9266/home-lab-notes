# Cross-Site Request Forgery (CSRF) — PortSwigger Web Security Academy

Tracking hands-on labs completed for the CSRF topic, following the same
methodology used for SQL Injection and XSS: understanding the
vulnerability, constructing the exploit, and documenting the underlying
technique.

---

## Lab 1: CSRF Vulnerability with No Defenses

Topic: Cross-Site Request Forgery | Difficulty: Practitioner

## Vulnerability
The email change functionality accepts a POST request to update a user's
email address, but includes no protection against forged requests — no
CSRF token, no verification of the request's origin, and no re-
authentication requirement. This means any external website can craft a
form that, when submitted by a logged-in victim's browser, silently
changes their email address without their knowledge or consent.

## What is CSRF?
CSRF exploits the fact that browsers automatically attach a user's
session cookies to any request sent to a site they're logged into —
regardless of which website actually triggered that request. If an
attacker can get a victim's browser to submit a request to a vulnerable
endpoint (e.g. by hosting a malicious form on an entirely different site),
the request will be processed using the victim's authenticated session,
as if the victim had submitted it themselves.

## Steps Taken

### Step 1: Identify the vulnerable request
Logged into the application using provided credentials (`wiener:peter`),
then submitted the "Update email" form. Used Burp Suite's Proxy history
to find and inspect the resulting request:

POST /my-account/change-email

Confirmed the request included only an `email` parameter — no CSRF token
or other anti-forgery protection.

### Step 2: Build a CSRF exploit HTML page
Constructed the following HTML, designed to automatically submit a forged
version of the same request the moment the page loads:
```html
<html>
  <body>
    <form action="https://[lab-id].web-security-academy.net/my-account/change-email" method="POST">
      <input type="hidden" name="email" value="hello&#64;hello&#46;com" />
      <input type="submit" value="Submit request" />
    </form>
    <script>
      history.pushState('', '', '/');
      document.forms[0].submit();
    </script>
  </body>
</html>
```

### Step 3: Test the exploit on my own session
Stored the HTML on the lab's exploit server and clicked "View exploit" to
confirm it correctly changed the account's email address.

### Step 4: Modify and deliver to the victim
Changed the target email value to a different, attacker-chosen address,
then clicked "Deliver to victim" — the simulated victim's browser
automatically submitted the forged request using their own authenticated
session.

## How the Exploit Works
- The form's `action` targets the real, legitimate email-change endpoint
  on the vulnerable site itself — not a fake lookalike page
- `<input type="hidden" name="email" value="...">` pre-fills the exact
  parameter the real form would normally send, using HTML entity-encoded
  characters (`&#64;` for `@`, `&#46;` for `.`) to safely embed an email
  address inside an HTML attribute
- `history.pushState('', '', '/')` is a minor cleanup technique — it
  changes the visible URL in the victim's address bar without reloading
  the page, making the redirect/submission less noticeable
- `document.forms[0].submit()` automatically submits the form the instant
  the page loads, requiring zero interaction from the victim beyond
  simply visiting the malicious page
- Because the victim's browser already holds a valid, authenticated
  session cookie for the real site, this forged request is processed
  exactly as if the victim had filled out and submitted the form
  themselves — the server has no way to distinguish a genuine user action
  from this automated, attacker-triggered one

## Result
Successfully forged a request that changed the victim's account email
address without their knowledge, using an entirely external, attacker-
hosted HTML page, solving the lab.

## What I Learned
This lab introduced CSRF as a distinct vulnerability class from injection-
based attacks (SQLi, XSS) — rather than injecting malicious code into the
target application itself, CSRF exploits the browser's default behavior
of automatically attaching session credentials to any request, regardless
of the request's true origin. This is why session cookies alone are
insufficient to prove a request is legitimate — an application also needs
some mechanism (like an unpredictable CSRF token, tied to the user's
session and required on every state-changing request) to verify that the
request actually originated from its own pages, not from an external,
attacker-controlled site. This directly connects to Lab 24 from the XSS
topic, where a CSRF token was stolen via XSS rather than being absent
entirely — together, these labs show both sides of CSRF protection
failure: no token at all (this lab) versus a token that exists but can be
read and reused via a separate vulnerability.
