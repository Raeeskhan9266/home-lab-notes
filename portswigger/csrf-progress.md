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



## Lab 2: CSRF Where Token Validation Depends on Request Method

Topic: Cross-Site Request Forgery | Difficulty: Practitioner

## Vulnerability
The email change functionality does implement CSRF token validation —
but only for requests submitted via one HTTP method. Switching the same
request to a different method (while keeping the same parameters) bypasses
the token check entirely, since the server-side validation logic wasn't
applied consistently across all methods that route to the same
functionality.

## Steps Taken

### Step 1: Compare the same action across HTTP methods
Logged in using the provided credentials and submitted the "Update email"
form normally, observing (via Burp Suite) that the resulting POST request
required a valid CSRF token to succeed.

### Step 2: Test the same endpoint using GET instead of POST
Discovered that the application also accepts the exact same email-change
action as a **GET** request, with the email value passed as a URL query
parameter — and critically, this GET-based route does **not** enforce the
CSRF token check that the POST route does.

### Step 3: Build the exploit
Since a GET request requires no form submission at all — simply loading
the URL is enough to trigger it — the exploit only needed to redirect
the victim's browser to the vulnerable URL:
```html
<script>
location = 'https://[lab-id].web-security-academy.net/my-account/change-email?email=yello@yello.com';
</script>
```

### Step 4: Deliver to victim
Stored this on the exploit server and delivered it — the victim's browser
automatically navigated to the crafted URL, triggering the email change
using their authenticated session, with no CSRF token required at all.

## How the Exploit Works
- `location = '...'` in JavaScript immediately redirects the current page
  to the specified URL the moment the malicious page loads — no form,
  no button, no user interaction needed at all, since a GET request is
  triggered simply by *navigating* to a URL
- Because the vulnerable endpoint accepts the email-change action via
  GET **without** requiring the CSRF token that the POST version enforces,
  this redirect alone was suffient to change the victim's email
- This is an even simpler delivery mechanism than Lab 1's auto-submitting
  form, since GET requests don't require any form submission step at all —
  just loading the URL is the "request"

## Result
Successfully bypassed CSRF token validation by using an alternate HTTP
method (GET) that the server failed to protect, changing the victim's
email address via a simple page redirect, solving the lab.

## What I Learned
This lab exposed a common real-world CSRF defense mistake: implementing
token validation for one HTTP method (typically POST, since that's the
"expected" way to submit the form) while forgetting that the same backend
functionality might also be reachable through a different method (GET)
that bypasses the check entirely. This reinforces that CSRF protection
needs to be enforced consistently at the level of the *endpoint/action*
itself, not tied to assumptions about which specific HTTP method a
legitimate client "should" use — if the server logic accepts multiple
methods for the same state-changing action, every one of those paths needs
identical protection, or the weakest one effectively determines the real
security of the entire feature.



## Lab 3: CSRF Where Token Validation Depends on Token Being Present

Topic: Cross-Site Request Forgery | Difficulty: Practitioner

## Vulnerability
The email change functionality validates the CSRF token correctly *if* one
is submitted — but the server-side logic only performs this check when a
`csrf` parameter actually exists in the request. If the parameter is
omitted entirely, the validation step is skipped altogether, and the
request is processed as if it were valid.

## Steps Taken

### Step 1: Confirm normal token validation
Logged in and submitted the "Update email" form normally, confirming (via
Burp Suite) that the request included a `csrf` parameter, and that
tampering with its value caused the request to be rejected.

### Step 2: Test removing the token parameter entirely
Rather than submitting an invalid token, removed the `csrf` field from the
request completely and resubmitted it. The request succeeded — revealing
that the validation logic only runs a check *if* the field is present,
rather than requiring it unconditionally.

### Step 3: Build the exploit without a CSRF token field
```html
<html>
  <body>
    <form action="https://[lab-id].web-security-academy.net/my-account/change-email" method="POST">
      <input type="hidden" name="email" value="yello&#64;hello&#46;com" />
      <input type="submit" value="Submit request" />
    </form>
    <script>
      history.pushState('', '', '/');
      document.forms[0].submit();
    </script>
  </body>
</html>
```
No `csrf` input field is included at all — the form only submits the
`email` parameter.

### Step 4: Deliver to victim
Stored and delivered the exploit — the victim's browser auto-submitted
the forged request, and since no CSRF token was present, the server's
flawed validation logic let it through unchecked.

## How the Exploit Works
- The server's CSRF defense appears to follow logic like: *"if a `csrf`
  parameter is submitted, verify it matches the session; otherwise, skip
  validation"* — rather than the correct logic, which should always
  require a valid, present token for any state-changing request
- By constructing a form that never includes the `csrf` field at all
  (rather than including a blank or incorrect one), the exploit avoids
  triggering the validation branch entirely
- The rest of the exploit mechanism (auto-submitting hidden form, using
  the victim's existing authenticated session) is identical to Lab 1

## Result
Successfully bypassed CSRF protection by omitting the token parameter
entirely, exploiting a validation flow that only checks the token *if*
it's present, changing the victim's email address, solving the lab.

## What I Learned
This lab, combined with Lab 2, reinforced a broader theme: CSRF
protections are frequently implemented with subtle logical gaps rather
than being completely absent. Here, the flaw wasn't in *how* the token
was validated (the comparison logic itself may have been perfectly
correct) but in *when* that validation was triggered — conditioning the
check on the parameter's mere presence, rather than enforcing its
presence as a hard requirement. This is a valuable lesson for secure
development: any security check should default to "deny" and only allow
the request through after an explicit, unconditional pass — never
structured so that simply omitting a piece of data skips the check
altogether. This is a very realistic bug pattern, since developers often
write validation as "if token exists, check it" without considering the
"what if it's just missing" case as equally dangerous as "what if it's
wrong."




## Lab 4: CSRF Where Token Is Not Tied to User Session

Topic: Cross-Site Request Forgery | Difficulty: Practitioner

## Vulnerability
The application generates a CSRF token and validates it correctly on
every request — but the token is never linked to the specific user's
session. This means the server only checks "is this a token my system
has ever issued to *anyone*?" rather than "is this the specific token
belonging to *this* user's current session?" As a result, an attacker can
obtain a valid token using their own account, and that same token will
be accepted for a completely different victim's session.

## Steps Taken

### Step 1: Compare tokens across two different accounts
Logged in separately as both provided accounts (`wiener:peter` and
`carlos:montoya`) and submitted the "Update email" form as each. Observed
that each login generated a CSRF token, but testing showed these tokens
weren't uniquely bound to a single session — a token obtained under one
account's session could still be successfully submitted using a
completely different session/account.

### Step 2: Confirm the token is only checked for general validity
Determined that the server's validation logic simply checks whether a
submitted token matches *some* valid, previously-issued token in its
records — not specifically the token that was issued to the current
requester's own session.

### Step 3: Build the exploit using a token obtained from my own account
```html
<html>
  <body>
    <form action="https://[lab-id].web-security-academy.net/my-account/change-email" method="POST">
      <input type="hidden" name="email" value="hello&#64;hello&#46;com" />
      <input type="hidden" name="csrf" value="[a valid token obtained from my own account]" />
      <input type="submit" value="Submit request" />
    </form>
    <script>
      history.pushState('', '', '/');
      document.forms[0].submit();
    </script>
  </body>
</html>
```

### Step 4: Deliver to victim
Stored and delivered the exploit — even though the embedded CSRF token
was originally generated for *my own* account/session (not the victim's),
the server accepted it as valid when submitted using the victim's session,
successfully changing their email.

## How the Exploit Works
- Properly implemented CSRF protection should tie each token to the exact
  session it was issued for, so that even a completely valid, real,
  unexpired token from one user's session is rejected if submitted under
  a different user's session
- Here, the server's check was effectively: "does this token exist/was it
  ever validly issued?" — a check that any attacker can trivially satisfy
  by simply logging into their *own* account first, grabbing a legitimately
  issued token, and reusing that exact token in an exploit aimed at a
  victim's session
- Since the attacker doesn't need to steal or predict the victim's actual
  token at all (unlike a properly-implemented system, where this would be
  required), this defense provides essentially no real protection —
  it just adds an extra static-looking parameter that doesn't verify
  anything meaningful about *who* is making the request

## Result
Successfully reused a CSRF token generated from my own account's session
to forge a request against a victim's session, since the server never
verified that the submitted token actually belonged to the requester's
specific session, solving the lab.

## What I Learned
This lab was a critical lesson in what makes a CSRF token defense
*actually* effective versus merely present: the token itself isn't the
security mechanism — the *binding* between the token and the specific
user's session is. A CSRF token that isn't cryptographically tied to and
verified against the current session is essentially just an extra
parameter, providing a false sense of security while contributing almost
nothing to actual protection, since any attacker can generate their own
valid token via their own account and reuse it against anyone. This
reinforces that secure CSRF implementations must validate two things
together: that the token is valid, AND that it matches the specific
session making the request — checking only the first condition (as seen
in this lab) is a common and dangerous implementation mistake.




CSRF where token is tied to non-session cookie

This lab's email change functionality is vulnerable to CSRF. It uses tokens to try to prevent CSRF attacks, but they aren't fully integrated into the site's session handling system.

To solve the lab, use your exploit server to host an HTML page that uses a CSRF attack to change the viewer's email address.

You have two accounts on the application that you can use to help design your attack. The credentials are as follows:


