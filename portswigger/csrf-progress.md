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




## Lab 5: CSRF Where Token Is Tied to Non-Session Cookie

Topic: Cross-Site Request Forgery | Difficulty: Practitioner

## Vulnerability
This application's CSRF defense is a step more sophisticated than Lab 4 —
the token IS tied to something, but that something is a **separate cookie**
(e.g. `csrfKey`), completely independent of the actual session cookie
used for authentication. Since these two cookies aren't linked to each
other, an attacker who can get their own `csrfKey` cookie value (and its
matching token) planted into the *victim's* browser can make a token
generated under the attacker's control appear valid for the victim's
session.

## Why This Is Different from Lab 4
- **Lab 4:** the token wasn't tied to any session at all — any valid
  token worked everywhere
- **This lab:** the token IS properly tied to a cookie value — but that
  cookie itself is separate from the session cookie, meaning if an
  attacker can force their own known cookie value onto the victim's
  browser, the token check will still pass, since it only verifies
  consistency between the token and this other cookie — not the actual
  logged-in session

## Steps Taken

### Step 1: Identify the separate CSRF cookie
Compared requests across both provided accounts and observed a distinct
cookie (`csrfKey`) alongside the standard session cookie — confirming the
CSRF token was validated against this separate cookie's value, not the
session itself.

### Step 2: Find a way to plant a chosen cookie value on the victim
Identified a header injection vulnerability in the site's search
functionality — the `search` parameter was reflected in a way that
allowed injecting raw HTTP response headers, including a new `Set-Cookie`
header, via CRLF (carriage return/line feed) injection:

search=test%0d%0aSet-Cookie:%20csrfKey=hpx4F8LbD6J5eRWJafPDvL8sQmG7B1Ob%3b%20SameSite=None

`%0d%0a` represents an encoded CRLF sequence, which — if not properly
sanitized — terminates the current HTTP header/response line and allows
injecting an entirely new header, in this case forcing the victim's
browser to set a `csrfKey` cookie value of the attacker's choosing.

### Step 3: Obtain a matching, valid CSRF token for that same cookie value
Using an account under the attacker's own control, generated a request
where the `csrfKey` cookie matched the value being injected, and captured
the corresponding valid CSRF token the server issued for that pairing.

### Step 4: Build the combined exploit
```html
<html>
  <body>
    <form action="https://[lab-id].web-security-academy.net/my-account/change-email" method="POST">
      <input type="hidden" name="email" value="hello1@hello.com" />
      <input type="hidden" name="csrf" value="[matching token for the injected csrfKey]" />
      <input type="submit" value="Submit request" />
    </form>
    <img src="https://[lab-id].web-security-academy.net/?search=test%0d%0aSet-Cookie:%20csrfKey=[chosen-value]%3b%20SameSite=None"
         onerror="document.forms[0].submit()">
  </body>
</html>
```

### Step 5: Deliver to victim
- The `<img>` tag's broken `src` (pointing at the header-injection URL)
  fails to load as a real image, triggering its `onerror` handler
- Before that error fires, though, the browser processes the injected
  `Set-Cookie` header from the response, silently setting the victim's
  `csrfKey` cookie to the attacker-chosen value
- Once `onerror` fires, the form (containing the matching, pre-obtained
  CSRF token) submits automatically
- Since the victim's browser now holds the exact `csrfKey` cookie value
  the submitted token was originally generated for, the server's
  token-to-cookie consistency check passes — even though the actual
  session cookie (authentication) still belongs to the victim

## Result
Successfully chained a header injection (CRLF) vulnerability with a CSRF
token that was tied only to a separate, non-session cookie, forcing a
matching cookie value onto the victim's browser and reusing a
pre-obtained valid token to change their email address, solving the lab.

## What I Learned
This lab was a strong example of vulnerability chaining, similar in spirit
to XSS Lab 24 (chaining XSS with CSRF token theft) but using a
completely different combination: a header/response-splitting
vulnerability combined with a CSRF defense that was almost correct, but
tied to the wrong anchor point. It reinforced that a CSRF token must be
bound specifically to the user's **authenticated session** — binding it
to any other cookie, even one that looks session-like, creates an
exploitable gap the moment an attacker finds any way (like a header
injection bug elsewhere on the site) to control that cookie's value in
the victim's browser. This also introduced CRLF/header injection as a
distinct vulnerability class in its own right — encoding `\r\n` sequences
into an input that gets reflected into HTTP headers can allow an attacker
to inject entirely new headers, including `Set-Cookie`, giving them
partial control over the victim's browser state without ever needing
direct script execution (XSS).



## Lab 6: CSRF Where Token Is Duplicated in Cookie (Double-Submit Defense Bypass)

Topic: Cross-Site Request Forgery | Difficulty: Practitioner

## Vulnerability
The application uses the "double submit cookie" pattern — a common CSRF
prevention technique where the server checks that a token submitted in
the request body/parameter matches a token also present in a cookie,
without needing any server-side storage of issued tokens at all. This
technique is inherently insecure whenever an attacker can independently
set both values themselves, since the "check" only ever verifies the two
submitted values match each other — not that either one was legitimately
issued by the server for that session.

## What Is "Double Submit Cookie" and Why It's Risky Here
The idea behind double-submit is: an attacker (without XSS) normally
can't read or set cookies for a domain they don't control, so if the
cookie value matches the submitted form/parameter value, it's assumed the
request must have originated from a legitimate page that could read that
cookie. This assumption completely breaks down if an attacker has *any*
alternative way to set a cookie on the victim's browser for that domain —
such as a header injection vulnerability, as seen again in this lab.

## Steps Taken

### Step 1: Confirm the double-submit pattern
Observed that the CSRF token submitted in the form body was also expected
to match a `csrf` cookie value — with no indication that the server
tracks or validates the token against anything beyond this simple
cookie-vs-parameter comparison.

### Step 2: Reuse the same header injection technique from Lab 5
The same CRLF/header injection point (via the `search` parameter) allowed
setting an arbitrary `csrf` cookie value on the victim's browser:

search=test%0d%0aSet-Cookie:%20csrf=fake%3b%20SameSite=None


### Step 3: Build the exploit using a self-chosen, arbitrary token value
Since the attacker can now set the cookie to *any* value using the header
injection, there's no need to steal or reuse a real, server-issued token
at all — a completely made-up value (`fake`) works just as well, as long
as the exact same value is used in both the injected cookie and the
submitted form parameter:
```html
<html>
  <body>
    <form action="https://[lab-id].web-security-academy.net/my-account/change-email" method="POST">
      <input type="hidden" name="email" value="hello@hello.com" />
      <input type="hidden" name="csrf" value="fake" />
      <input type="submit" value="Submit request" />
    </form>
    <img src="https://[lab-id].web-security-academy.net/?search=test%0d%0aSet-Cookie:%20csrf=fake%3b%20SameSite=None"
         onerror="document.forms[0].submit();"/>
  </body>
</html>
```

### Step 4: Deliver to victim
The broken `<img>` tag's `onerror` handler fires after the header
injection sets the victim's `csrf` cookie to `fake`; the form then
auto-submits with the matching `csrf=fake` parameter, satisfying the
double-submit check entirely with attacker-invented values.

## How the Exploit Works
- The server's validation logic is simply: *"does the submitted `csrf`
  parameter equal the `csrf` cookie value?"* — with no reference to any
  actual, securely-generated, per-session token at all
- Since both the cookie (via header injection) and the form parameter are
  entirely attacker-controlled in this scenario, the attacker can set both
  to any matching value they like — including something as trivial as the
  literal string `"fake"` — and the check will still pass
- This demonstrates that double-submit cookie defenses are only secure
  when there is truly no way for an attacker to set cookies on the
  victim's browser for the target domain; the moment any such mechanism
  exists (header injection here, but it could equally be another
  vulnerability like XSS), the entire defense collapses

## Result
Successfully bypassed a double-submit cookie CSRF defense by using a
header injection vulnerability to plant an arbitrary, self-chosen cookie
value, then submitting a form with a matching arbitrary token value,
solving the lab.

## What I Learned
This lab was a direct continuation of the header-injection technique from
Lab 5, but demonstrated an even simpler underlying weakness: the
double-submit cookie pattern doesn't actually require stealing or
predicting any real, meaningful data at all — since the check is just a
self-referential comparison, an attacker with cookie-setting ability can
trivially satisfy it using entirely made-up values. This reinforced why
double-submit cookie CSRF protection is generally considered a weaker
defense pattern compared to server-side, session-bound tokens (as would
be correctly implemented, in contrast to Labs 1–5's various flawed
attempts) — it trades server-side storage requirements for a security
assumption (that attackers can't set cookies on the target domain) that
frequently doesn't hold in practice, especially on applications with any
other cookie-setting vulnerability like this one.




## Lab 7: SameSite Lax Bypass via Method Override

Topic: Cross-Site Request Forgery (SameSite Bypass) | Difficulty: Practitioner

## Vulnerability
This application has no CSRF token at all — but it relies on modern
browsers' default `SameSite=Lax` cookie behavior as an implicit defense.
Under `Lax` restrictions, cookies are NOT sent on most cross-site
requests, but there's a specific exception: cookies ARE still sent on
cross-site **GET** requests that involve a top-level navigation (i.e.
the browser actually navigating to a new URL, like clicking a link or
being redirected — not a background AJAX/fetch call). Combined with a
separate flaw (accepting a method override parameter), this exception
became fully exploitable.

## What Is SameSite=Lax and Why It Normally Helps
Since 2020, browsers apply `SameSite=Lax` as the *default* cookie
behavior when a site doesn't explicitly set a `SameSite` attribute. This
was introduced specifically as a broad, built-in mitigation against CSRF:
under `Lax`, cross-site POST requests (the traditional CSRF vector used
in Labs 1–6) no longer include the victim's cookies at all, which would
normally make classic form-based CSRF attacks like Lab 1 ineffective by
default on modern browsers.

## Steps Taken

### Step 1: Confirm no CSRF token exists
Studied the `POST /my-account/change-email` request via Burp Suite's
Proxy history and confirmed no unpredictable CSRF token was present at all.

### Step 2: Identify that SameSite=Lax is the only real protection
Checked the `POST /login` response headers and confirmed the session
cookie was set without any explicit `SameSite` attribute — meaning the
browser defaults to `Lax`, which should normally block a cross-site POST
request (like the email-change action) from including the session cookie.

### Step 3: Test converting the request to GET
Sent the change-email request to Burp Repeater and used "Change request
method" to convert it to GET. The server rejected this, since the
endpoint was coded to only accept POST.

### Step 4: Discover a method override parameter
Tried adding a `_method` parameter to the GET request's query string:

GET /my-account/change-email?email=foo%40web-security-academy.net&_method=POST

The server accepted this — revealing that the backend framework silently
supports overriding the actual HTTP method via this parameter, treating a
GET request carrying `_method=POST` as if it were a genuine POST request.

### Step 5: Combine both findings into a working exploit
Since the request could now effectively perform the sensitive POST
action while technically being sent as a GET request, and GET requests
DO include cookies under `SameSite=Lax` during a top-level navigation,
built the following exploit:
```html
<script>
    document.location = "https://[lab-id].web-security-academy.net/my-account/change-email?email=hello@hello.com&_method=POST";
</script>
```

### Step 6: Deliver to victim
Stored and delivered the exploit — `document.location = ...` forces a
genuine top-level navigation (exactly the condition under which `Lax`
still permits the cookie to be sent), successfully triggering the
email-change action using the victim's authenticated session.

## How the Exploit Works
- `document.location = "..."` (unlike a background `fetch()` or an
  AJAX-submitted form) causes the *entire browser tab* to navigate to
  the new URL — this is what qualifies as a top-level navigation under
  `SameSite=Lax`'s exception rule
- Because the request is a real GET request, the `Lax` policy allows the
  victim's session cookie to be attached to it, even though it's
  cross-site (initiated from the attacker's exploit page)
- The `_method=POST` query parameter exploits a separate, unrelated
  convenience feature (common in backend frameworks that support HTML
  forms, which can only natively send GET/POST) that lets a client
  specify the "real" intended method — without realizing this creates a
  way to perform POST-equivalent actions through a GET request, which has
  very different CSRF exposure characteristics
- Combining these two individually low-risk-seeming details (SameSite's
  GET exception + a method override parameter) creates a fully working
  CSRF exploit against an application with no explicit CSRF token at all

## Result
Successfully bypassed SameSite=Lax cookie protection — normally an
effective, browser-level CSRF mitigation — by combining a top-level GET
navigation with a method-override parameter that caused the server to
treat it as the sensitive POST action, changing the victim's email
address, solving the lab.

## What I Learned
This lab was a significant real-world lesson: SameSite cookie attributes
are a strong, modern, broadly effective CSRF mitigation, but they are not
a complete substitute for proper server-side CSRF tokens, precisely
because of edge cases like this one. The core issue wasn't a flaw in
SameSite itself — it behaved exactly as designed — but rather those two
separate, individually reasonable-looking features (allowing top-level
navigations to carry cookies under Lax, and supporting method-override
parameters for framework convenience) combined to recreate the exact
attack surface SameSite was meant to close. This reinforces a recurring
theme across the whole CSRF topic: defense-in-depth matters, since relying
on any single mitigation (whether a token implementation or a browser-
level cookie policy) can be undermined by an unrelated feature elsewhere
in the same application that reintroduces the original risk.
