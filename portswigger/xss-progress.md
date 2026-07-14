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


## Lab 3: DOM XSS in document.write Sink Using Source location.search

Topic: Cross-Site Scripting (DOM-based) | Difficulty: Apprentice

## Vulnerability
This lab's search tracking functionality uses client-side JavaScript to
take data directly from the URL (`location.search`, the query string
portion of the URL) and pass it into `document.write()`, which writes
content directly into the page's HTML. Since this happens entirely in the
browser via JavaScript — with no server involvement in processing that
specific data — this is a DOM-based vulnerability rather than a
server-side one.

## What Makes This Different from Labs 1 & 2 (Reflected/Stored)
- **Reflected/Stored XSS (Labs 1–2):** the server itself inserts
  unsanitized user input into the HTML response
- **DOM-based XSS (this lab):** the server's HTML response is safe as
  delivered — the vulnerability exists entirely in client-side JavaScript,
  which reads attacker-controllable data (a "source") and writes it
  somewhere dangerous (a "sink") without sanitization

## Source and Sink Concept
- **Source** = where attacker-controllable data enters the JavaScript
  (here, `location.search` — the query string of the URL, fully
  controllable by crafting a malicious link)
- **Sink** = where that data ends up being used dangerously (here,
  `document.write()`, which writes raw content directly into the page's HTML)

## Steps Taken

### Step 1: Probe how input is reflected
Entered a random alphanumeric test string into the search box, then
right-clicked the resulting page and used "Inspect Element" to examine
where that string landed in the actual HTML.

### Step 2: Identify the injection context
Found that the test string had been placed inside an `img` tag's `src`
attribute (e.g. `<img src="[my test string]">`) — meaning a simple
`<script>` tag (like in Labs 1–2) would not work here directly, since the
injection point sits inside an existing HTML attribute, not in a place
where a new tag can just be opened freely.

### Step 3: Break out of the attribute and inject a new element
Used the following payload in the search box:
"><svg onload=alert(1)>

## How the Payload Works
- `"` closes the `img` tag's `src` attribute value
- `>` closes the `img` tag itself, ending it early
- `<svg onload=alert(1)>` injects a brand new HTML element (an SVG image
  tag) that includes an `onload` event handler — this handler
  automatically fires the moment the browser finishes loading the SVG
  element, executing `alert(1)` without needing a `<script>` tag at all

This technique is necessary specifically because the injection point was
inside an attribute value, not free-standing HTML — the attacker first has
to "escape" out of that attribute/tag context before injecting anything new.

## Result
Successfully broke out of the `img` attribute context and triggered
`alert(1)` using an SVG element with an `onload` event handler, solving
the lab.

## What I Learned
This lab introduced two important concepts beyond the basic `<script>`
injection from Labs 1–2: first, the source/sink model for understanding
DOM-based XSS (data flows from an attacker-controllable source through
JavaScript into a dangerous sink, entirely client-side); and second, that
successful XSS often requires first identifying the exact HTML context
the payload lands in (an attribute vs. free text) before choosing the
right technique to escape that context. Event handlers like `onload` are
a critical alternative to `<script>` tags — this becomes especially
important in later labs where `<script>` tags or certain characters are
likely to be filtered or blocked.


## Lab 4: DOM XSS in innerHTML Sink Using Source location.search
Completed: [aaj ki date]
Topic: Cross-Site Scripting (DOM-based) | Difficulty: Apprentice

## Vulnerability
The search blog functionality reads the search term from `location.search`
(the URL's query string) and assigns it directly to a `div` element's
`innerHTML` property. This changes the actual HTML content inside that div,
using attacker-controllable data, without any sanitization.

## Source and Sink (this lab)
- **Source:** `location.search` — same as Lab 3
- **Sink:** `innerHTML` assignment — different from Lab 3's `document.write`

## Important Difference from document.write
A key detail with `innerHTML`: it does **not** execute `<script>` tags
even if injected directly — browsers deliberately block script execution
via innerHTML assignment as a partial safety measure. This is why a plain
`<script>alert(1)</script>` payload would silently fail here, unlike in
Labs 1–2. Instead, the payload needs to rely on an HTML element whose
event handler fires automatically, since event handlers *do* still execute
even when the surrounding HTML is inserted via innerHTML.

## Steps Taken
1. Entered the following payload directly into the search box:
<img src=1 onerror=alert(1)>
```
2. Clicked "Search"
How the Payload Works

<img src=1 ...> creates an image tag with an intentionally invalid
src value (1 is not a real image path)
Since the browser cannot load a valid image from that broken source, it
fires the onerror event handler automatically
onerror=alert(1) means the moment that error event fires, alert(1)
executes

This works specifically because the payload doesn't rely on a <script>
tag at all — it uses a naturally-occurring browser event (a failed image
load) to trigger JavaScript execution, sidestepping innerHTML's
script-blocking behavior entirely.
Result
Successfully triggered alert(1) via the img tag's onerror handler,
solving the lab.
What I Learned
This lab reinforced why event-handler-based payloads (like onerror,
onload from Lab 3) are often more reliable than <script> tags in
DOM-based XSS — many DOM sinks (like innerHTML) specifically block script
execution as a built-in browser protection, but this protection doesn't
extend to event handlers on other HTML elements. This means understanding
which sink is being used matters directly for choosing a working
payload: document.write generally allows script tags to execute, while
innerHTML does not, requiring an event-handler-based workaround instead.
