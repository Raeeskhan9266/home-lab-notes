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



## Lab 5: DOM XSS in jQuery Anchor href Attribute Sink Using location.search Source

Topic: Cross-Site Scripting (DOM-based) | Difficulty: Apprentice

## Vulnerability
The Submit Feedback page uses jQuery's `$` selector to find an anchor
(`<a>`) element — the "back" link — and sets its `href` attribute using a
value taken from the `returnPath` query parameter in the URL. Since this
value is inserted directly into the `href` attribute without validation,
an attacker can control where that link actually points.

## Source and Sink (this lab)
- **Source:** `location.search` (specifically the `returnPath` parameter)
- **Sink:** jQuery-set `href` attribute on an anchor element

## Steps Taken

### Step 1: Probe how input is reflected
Changed the `returnPath` query parameter to a random alphanumeric string,
then inspected the page's HTML to see where that value landed — confirmed
it was placed directly inside the anchor tag's `href` attribute.

### Step 2: Inject a javascript: URI payload
Changed `returnPath` to:
javascript:alert(document.cookie)
Pressed enter to reload the page with the new query parameter, then
clicked the "back" link.

## How the Payload Works
Unlike previous labs (which broke out of an HTML attribute using quotes
and angle brackets, or used event handlers), this technique exploits the
fact that an `href` attribute can accept a `javascript:` URI scheme
instead of a normal URL. When a link with `href="javascript:..."` is
clicked, the browser executes the code following `javascript:` as
JavaScript, instead of navigating to a page. Since the `href` value was
attacker-controlled via the `returnPath` parameter, and no validation
restricted it to normal URLs, this allowed direct JavaScript execution
simply by clicking the link.

`document.cookie` specifically returns the cookies associated with the
current page — a common real-world target for XSS attacks, since session
cookies can often be stolen this way to hijack a logged-in user's session.

## Result
Successfully triggered `alert(document.cookie)` by clicking the modified
"back" link, solving the lab.

## What I Learned
This lab introduced a new XSS delivery mechanism: the `javascript:` URI
scheme, which doesn't require injecting any new HTML tags or breaking out
of an existing attribute using quotes/angle brackets — instead, it directly
replaces the entire attribute value with executable JavaScript. This is
only exploitable because the sink is specifically an attribute that
browsers treat as "clickable/navigable" (like `href`), which accepts
`javascript:` as a valid scheme. It also introduced `document.cookie` as a
realistic attack target — in a genuine attack, this technique could be
combined with a way to exfiltrate the cookie value to an attacker-controlled
server (rather than just displaying it via alert), which is exactly how
real session hijacking via XSS often works.


## Lab 6: DOM XSS in jQuery Selector Sink Using a hashchange Event

Topic: Cross-Site Scripting (DOM-based) | Difficulty: Practitioner

## Vulnerability
The home page uses jQuery's `$()` selector function to automatically
scroll to a blog post whose title is taken from `location.hash` (the part
of a URL after the `#` symbol). This selector is re-evaluated every time
the URL hash changes — including when it's changed dynamically via
JavaScript — without sanitizing the hash value first.

## Source and Sink (this lab)
- **Source:** `location.hash`, specifically triggered via a `hashchange`
  event (fires whenever the URL's hash portion changes)
- **Sink:** jQuery `$()` selector — when passed a malicious string instead
  of a simple ID/class selector, jQuery can interpret and execute injected
  HTML

## What's New in This Lab: Exploit Delivery
Unlike Labs 1–5 (where I directly typed a payload into a search box or URL
and tested it myself), this lab required actually **delivering** a working
exploit to a separate victim user — closely mirroring how a real attack
would work, where the attacker crafts a malicious page and lures a victim
into visiting it, rather than the attacker exploiting their own browser.

## Steps Taken

### Step 1: Identify the vulnerable sink
Used browser DevTools to inspect the home page's JavaScript and confirmed
it uses jQuery's `$()` selector with a value derived from `location.hash`,
triggered on the `hashchange` event.

### Step 2: Build the exploit on the provided exploit server
Opened the lab's exploit server (a separate hosted page used to simulate
an attacker-controlled site) and added the following payload to the page body:
```html

```

### Step 3: Test the exploit against my own browser
Clicked "View exploit" to confirm the payload correctly triggered the
browser's `print()` dialog.

### Step 4: Deliver the exploit to the victim
Clicked "Deliver to victim" — the lab's simulated victim then visited the
malicious exploit page, triggering the same payload in their browser
session and solving the lab.

## How the Payload Works
- The `iframe` loads the target site's home page, appending just `#` to
  the URL initially (a valid but harmless hash)
- The `onload` event handler fires once the iframe finishes loading, and
  appends new content to the iframe's `src` — specifically adding
  `<img src=x onerror=print()>` to the URL's hash portion
- Changing the iframe's `src` in this way triggers a `hashchange` event
  on the framed page, causing the vulnerable jQuery selector to process
  this new, malicious hash value
- Since the hash now contains `<img src=x onerror=print()>`, jQuery's
  selector interprets it as HTML rather than a plain ID selector, and the
  image's broken `src` fires the `onerror` handler, calling `print()`

## Result
Successfully delivered a working exploit to a separate victim browser
session, triggering `print()` and solving the lab.

## What I Learned
This lab was a meaningful step toward realistic attack simulation —
instead of exploiting my own request, I had to construct a self-contained
malicious page (hosted separately, as an attacker's site would be) that
automatically triggers the vulnerability in a victim's browser the moment
they visit it, with no interaction required from the victim beyond loading
the page. This also introduced `location.hash` as a distinct source from
`location.search` — the hash portion of a URL is never sent to the server
at all, meaning this class of vulnerability is purely client-side from
start to finish, and traditional server-side logging/monitoring would
never even see the malicious hash value in a request log. Combining an
iframe with a dynamically modified `src` attribute is also a reusable
technique for triggering hash-based DOM XSS in an automated exploit,
rather than relying on a victim manually clicking a crafted link.



## Lab 7: Reflected XSS into Attribute with Angle Brackets HTML-Encoded

Topic: Cross-Site Scripting (Reflected) | Difficulty: Apprentice

## Vulnerability
The search blog functionality reflects the search term back into an HTML
attribute value. Unlike earlier reflected XSS labs, this application
specifically HTML-encodes angle brackets (`<` and `>`) — meaning a
straightforward `<script>` or `<img>` tag injection would fail here, since
those characters get converted to harmless entities (`&lt;`, `&gt;`)
before being rendered. However, quotation marks were **not** encoded,
leaving a different path open for exploitation.

## Steps Taken

### Step 1: Probe how input is reflected
Submitted a random alphanumeric string into the search box, then used
Burp Suite to intercept the resulting request and sent it to Burp Repeater
for easier testing. Confirmed the string was reflected inside a
double-quoted HTML attribute value.

### Step 2: Escape the attribute using an unencoded quote
Replaced the input with:
"onmouseover="alert(1)

### Step 3: Verify the exploit
Copied the resulting URL (with the payload as the search parameter) and
opened it directly in the browser. Hovering the mouse over the affected
element triggered the `alert(1)` popup.

## How the Payload Works
- The leading `"` closes the original attribute's quoted value early
- `onmouseover="alert(1)` then injects a brand new attribute — the
  `onmouseover` event handler — directly onto the same HTML element
- Since angle brackets were encoded but quotation marks were not, this
  attack never needed to create a new tag at all; it simply added a new
  *attribute* to the existing tag, sidestepping the filter entirely
- The injected `onmouseover` handler fires whenever a user's mouse passes
  over the element, executing `alert(1)` at that point

## Result
Successfully injected a new event-handler attribute and triggered
`alert(1)` on mouseover, solving the lab.

## What I Learned
This lab highlighted an important lesson about partial input filtering:
encoding only `<` and `>` blocks the creation of new HTML tags, but does
nothing to prevent an attacker from injecting new *attributes* onto an
already-existing tag, as long as quote characters remain unencoded. This
reinforces that XSS defenses need to account for every character that
carries special meaning in the relevant context (angle brackets for tags,
quotes for attribute boundaries, etc.) — encoding only some of them still
leaves a viable attack path. It also introduced event-handler-based
injection without any script tag or new element at all, which is a
technique likely to reappear in later labs targeting increasingly
restrictive filters.



## Lab 8: Stored XSS into Anchor href Attribute with Double Quotes HTML-Encoded

Topic: Cross-Site Scripting (Stored) | Difficulty: Apprentice

## Vulnerability
The comment functionality's "Website" field is stored and later used to
set the `href` attribute of an anchor tag around the comment author's
name. Double quotes are HTML-encoded here, meaning the earlier attribute-
escape technique from Lab 7 (`"onmouseover=...`) would not work — breaking
out of the attribute using a quote is blocked. However, since the value
still lands entirely inside the `href` attribute, the `javascript:` URI
scheme (from Lab 5) was usable here instead.

## Steps Taken

### Step 1: Confirm reflection point
Posted a comment with a random alphanumeric string in the "Website" field,
intercepted the request with Burp Suite, and sent it to Repeater.

### Step 2: Confirm where the value lands
Made a second request (viewing the post page) and intercepted it in a
separate Repeater tab, confirming the random string appeared inside an
anchor tag's `href` attribute.

### Step 3: Inject a javascript: URI payload
Replaced the "Website" field value with:
javascript:alert(1)
Submitted the comment, then loaded the post page and clicked the
comment author's name (the linked text).

## How the Payload Works
Since double quotes were encoded, escaping out of the attribute value
(like in Lab 7) wasn't possible — any injected quote would simply render
as literal text (`&quot;`), not break the HTML structure. Instead, this
attack didn't need to escape the attribute at all: it replaced the *entire*
`href` value with a `javascript:` URI, exactly as in Lab 5. When the
comment author's name (the anchor text) is clicked, the browser executes
the JavaScript following `javascript:` instead of navigating to a URL,
triggering `alert(1)`.

## Result
Successfully stored a comment whose author link triggers `alert(1)` when
clicked, solving the lab.

## What I Learned
This lab reinforced that the `javascript:` URI technique (first used in
Lab 5) is broadly useful specifically whenever the injection point is an
entire attribute value that accepts a URL/URI — it doesn't require
breaking out of quotes or injecting new HTML at all, which makes it
resistant to defenses that only encode quotes or angle brackets. Since
this is now stored (unlike Lab 5's reflected version), every visitor who
views the comment and clicks the author's name would trigger the payload,
making it significantly more dangerous in a real-world scenario — a single
malicious comment could compromise many users over time, not just one
targeted victim.



## Lab 9: Reflected XSS into a JavaScript String with Angle Brackets HTML-Encoded

Topic: Cross-Site Scripting (Reflected) | Difficulty: Apprentice

## Vulnerability
The search query tracking functionality reflects the search term directly
inside a JavaScript string literal within a `<script>` block on the page
(e.g. `var searchTerm = 'user-input-here';`). Angle brackets are
HTML-encoded, so injecting a new HTML tag (`<script>`, `<img>`, etc.)
would not work. However, since the reflection point is already inside
existing JavaScript code, the attack doesn't need any HTML tags at all —
it only needs to break out of the JavaScript string itself.

## Steps Taken

### Step 1: Confirm the reflection context
Submitted a random alphanumeric string into the search box, intercepted
the request with Burp Suite, and sent it to Repeater. Confirmed the value
was being reflected directly inside a JavaScript string (between single
quotes) inside a `<script>` tag — not inside regular HTML.

### Step 2: Break out of the JavaScript string
Replaced the input with:
'-alert(1)-'

### Step 3: Verify the exploit
Copied the resulting URL and opened it directly in the browser — loading
the page triggered `alert(1)` automatically.

## How the Payload Works
- The original code looked something like:
  `var searchTerm = 'INJECTION_POINT';`
- The leading `'` closes the original string literal early
- `-alert(1)-` uses the JavaScript subtraction operator (`-`) purely as a
  trick to keep the surrounding code syntactically valid while inserting
  a function call in between — it doesn't matter that subtracting a
  function's return value from a string doesn't make logical sense, since
  `alert(1)` still executes as a side effect before the (irrelevant)
  subtraction result is computed
- The trailing `-'` reopens a string literal to match the original
  trailing `'` in the code, keeping the overall JavaScript syntactically
  valid so no error is thrown and the rest of the script continues to run
  normally

The resulting code effectively becomes:
```javascript
var searchTerm = ''-alert(1)-'';
```
which JavaScript parses as: an empty string, minus the result of
`alert(1)`, minus another empty string — syntactically valid, and
`alert(1)` executes as part of evaluating that expression.

## Result
Successfully broke out of the JavaScript string context and triggered
`alert(1)`, solving the lab without needing any HTML tags or event
handlers.

## What I Learned
This lab introduced a distinct XSS context from every previous lab: HTML
tag encoding is irrelevant when the injection point is already inside a
`<script>` block, because the attack surface here is JavaScript syntax,
not HTML syntax. Breaking out of a JavaScript string requires closing the
quote character being used, then constructing a payload that keeps the
*surrounding* JavaScript syntactically valid (using operators like `-` to
"glue" the malicious code in without causing a parse error), rather than
relying on angle brackets or event handlers at all. This is an important
category to recognize during real-world testing — inspecting exactly
*where* reflected input lands (HTML body, HTML attribute, or JavaScript
string, as seen across Labs 1–9 so far) directly determines which
technique will actually work.



## Lab 10: DOM XSS in document.write Sink Using location.search Inside a select Element

Topic: Cross-Site Scripting (DOM-based) | Difficulty: Practitioner

## Vulnerability
The stock checker functionality on product pages extracts a `storeId`
parameter from `location.search` and uses `document.write()` to insert it
as a new `<option>` inside an existing `<select>` (dropdown) element. This
is the same sink (`document.write`) seen in Lab 3, but this time the
injection point sits inside a `<select>` element's content, which requires
a different escape approach.

## Steps Taken

### Step 1: Confirm the reflection point
Added a `storeId` parameter with a random alphanumeric string to the
product page URL and loaded it. Observed the random string appeared as a
new option inside the stock checker's dropdown list.

### Step 2: Inspect the exact HTML context
Right-clicked and inspected the dropdown element, confirming the
`storeId` value was being placed directly inside the `<select>` element's
option list, as raw HTML content (not inside a tag attribute).

### Step 3: Break out of the select element
Modified the URL to:
product?productId=1&storeId="></select><img src=1 onerror=alert(1)>

## How the Payload Works
- `"></select>` closes the (nonexistent, since we're already inside plain
  content) attribute context defensively, then explicitly closes the
  `<select>` element early, ending the dropdown entirely
- `<img src=1 onerror=alert(1)>` then injects a new element after the now-
  closed select — using the same broken-image/onerror technique from
  Lab 4, since this is a reliable way to trigger JavaScript execution
  without relying on `<script>` tags
- Because the payload explicitly closes the `<select>` tag before
  injecting the `<img>` element, the browser treats the `<img>` tag as a
  regular, independent HTML element outside of the dropdown — allowing it
  to load (and fail to load, triggering `onerror`) normally

## Result
Successfully broke out of the `<select>` element and triggered `alert(1)`
via the img tag's onerror handler, solving the lab.

## What I Learned
This lab extended the `document.write` sink concept from Lab 3 into a more
specific and slightly trickier scenario: the injection point wasn't just
"somewhere in the HTML body," but specifically nested inside a `<select>`
element, which meant simply injecting an `<img>` tag directly (without
first closing the select) might not behave as expected inside a dropdown
context. Explicitly closing the enclosing element (`</select>`) before
injecting new markup is a reusable pattern for breaking out of *any*
container element cleanly, not just attributes (as in earlier labs) —
reinforcing that identifying the exact enclosing HTML structure around
the injection point is just as important as identifying the sink itself.



## Lab 11: DOM XSS in AngularJS Expression with Angle Brackets and Double Quotes HTML-Encoded

Topic: Cross-Site Scripting (DOM-based, Framework-Specific) | Difficulty: Practitioner

## Vulnerability
The search functionality reflects user input inside an HTML element that
has an `ng-app` attribute — a directive that tells AngularJS (a JavaScript
framework) to actively scan and process that element's contents. Both
angle brackets and double quotes are HTML-encoded here, which blocks every
technique used in previous labs (new tags, attribute escapes). However,
because AngularJS itself evaluates any expression written inside double
curly braces (`{{ }}`) as live JavaScript, this creates an entirely
separate injection path that doesn't depend on HTML characters at all.

## Why Previous Techniques Don't Work Here
- New tags (`<script>`, `<img>`) — blocked, angle brackets encoded
- Attribute escape (`"onmouseover=...`) — blocked, double quotes encoded
- Since both primary "escape characters" used throughout Labs 1–10 are
  neutralized, a completely different mechanism was needed: AngularJS's
  own expression evaluation

## Steps Taken

### Step 1: Identify the AngularJS context
Entered a random alphanumeric string into the search box, then viewed the
page source and confirmed the reflected value sits inside an element
carrying the `ng-app` attribute — confirming AngularJS is actively
scanning and evaluating that section of the page.

### Step 2: Inject an AngularJS expression
Entered the following into the search box:
{{$on.constructor('alert(1)')()}}
Clicked "Search"

## How the Payload Works
- Anything inside `{{ }}` within an `ng-app`-scoped element is evaluated
  by AngularJS as a JavaScript-like expression, entirely separate from
  normal HTML parsing — this bypasses the angle-bracket/quote encoding
  completely, since no HTML special characters are needed at all
- `$on` is an accessible property within the AngularJS expression
  evaluation context; calling `.constructor` on it accesses the underlying
  JavaScript `Function` constructor
- `Function('alert(1)')` dynamically creates a new function whose body is
  the string `alert(1)` — effectively a way to turn an arbitrary string
  into executable code
- The trailing `()` immediately invokes that newly created function,
  executing `alert(1)`

## Result
Successfully triggered `alert(1)` by exploiting AngularJS's expression
evaluation feature, completely bypassing both angle-bracket and
double-quote encoding, solving the lab.

## What I Learned
This lab was a significant conceptual jump: it demonstrated that XSS
defenses focused purely on encoding HTML-special characters can be
completely bypassed when a client-side framework (like AngularJS)
introduces its *own* expression language that gets evaluated independently
of standard HTML/JavaScript parsing. This means understanding which
frontend frameworks/libraries are in use on a target application matters
directly for security testing — a framework's own template/expression
syntax can become an entirely separate attack surface that generic input
encoding doesn't protect against at all. This is also a good example of
using a constructor-based technique (`Function` constructor via
`.constructor`) to achieve arbitrary code execution from what looks like a
harmless string, a pattern that shows up in more advanced
sandbox-escape and filter-bypass XSS scenarios.


## Lab 12: Reflected DOM XSS

Topic: Cross-Site Scripting (Reflected DOM) | Difficulty: Practitioner

## Vulnerability
This lab combines both server-side and client-side behavior. The server
takes the search term from the request and echoes it back inside a JSON
response (`search-results`). A separate client-side script
(`searchResults.js`) then takes that JSON response and processes it using
`eval()` — a JavaScript function that executes any string passed to it as
actual code. The server does attempt to escape double-quote characters in
the response, but critically does **not** escape backslash characters,
creating an exploitable gap.

## What is "Reflected DOM" XSS?
This is a hybrid category: the vulnerability starts on the server side
(user input is reflected back in a response, like traditional reflected
XSS), but the actual dangerous execution happens client-side, when
JavaScript unsafely processes that reflected data via a sink like `eval()`.
This differs from Labs 3, 4, 6, and 10 (pure DOM XSS), where the data
never touched the server at all (e.g. `location.hash`, `location.search`
read directly by client-side JS with no server round-trip for that value).

## Steps Taken

### Step 1: Observe the reflection
Enabled Burp Suite's Intercept feature, searched for a test string like
`"XSS"` on the site, and forwarded the request. Observed in the
intercepted response that the search term was reflected inside a JSON
object under a `search-results` response.

### Step 2: Identify the dangerous sink
Opened `searchResults.js` via Burp's Site Map and found that this script
takes the JSON response and passes it directly into an `eval()` call —
meaning the entire JSON response body is executed as JavaScript code, not
just parsed as inert data.

### Step 3: Test the server's escaping behavior
Experimented with different search strings containing special characters
and found that the server escapes double-quote characters (`"` becomes
`\"`) but does **not** escape backslash characters (`\`) themselves.

### Step 4: Exploit the escaping gap
Submitted the following as the search term:
"-alert(1)}//

## How the Payload Works
- The payload begins with a backslash (`\`) followed by a double-quote (`"`)
- Since the server escapes quotes but not backslashes, when the server
  processes this input, it turns the `"` into `\"` — but the backslash I
  already supplied is placed immediately before it, resulting in `\\"`
  in the final response
- A double-backslash (`\\`) in a JSON/JavaScript string is interpreted as
  a single literal backslash character, which means the following quote
  is **no longer escaped** — it now acts as a real, functional
  string-closing quote
- This effectively closes the JSON string early, right where I wanted it to
- `-alert(1)` then uses the subtraction operator (same trick as Lab 9) to
  insert a function call while keeping the surrounding expression
  syntactically parseable by `eval()`
- `}//` closes the JSON object early and comments out anything that would
  have followed, preventing a syntax error from breaking the rest of the
  `eval()` call

The final response body effectively became:
```json
{"searchTerm":"\\"-alert(1)}//", "results":[]}
```
which, once passed through `eval()`, executes `alert(1)` as a side effect
of evaluating the constructed expression.

## Result
Successfully exploited the backslash-escaping gap to break out of the JSON
string inside an `eval()` sink, triggering `alert(1)` and solving the lab.

## What I Learned
This was one of the most intricate labs so far — it required understanding
how escaping works at a character level (specifically, that escaping only
the double-quote character is insufficient if backslash itself isn't also
escaped, since an attacker-supplied backslash can "consume" the server's
defensive escape and neutralize it). It also reinforced why `eval()` is
considered one of the most dangerous possible sinks in JavaScript: passing
untrusted data — even data wrapped in what looks like a safe JSON
structure — into `eval()` means any successful string-escape technique
leads directly to arbitrary code execution. This lab combined multiple
techniques learned earlier (subtraction-operator glueing from Lab 9,
careful analysis of exact escaping behavior) into a single, realistic
attack chain that closely resembles subtle real-world escaping bugs found
in production applications.


## Lab 13: Stored DOM XSS

Topic: Cross-Site Scripting (Stored DOM) | Difficulty: Practitioner

## Vulnerability
The blog comment functionality attempts to prevent XSS by using
JavaScript's `replace()` function to encode angle brackets before storing/
displaying the comment. However, this defense has a critical flaw: when
`replace()` is called with a plain string as its first argument (rather
than a regular expression with the global `/g` flag), it only replaces
the **first occurrence** of that character in the entire string — every
subsequent occurrence is left completely untouched.

## What is "Stored DOM" XSS?
This combines the persistence of stored XSS (Labs 2, 8) with a purely
client-side sink — the comment is saved on the server (like traditional
stored XSS) and later processed unsafely by client-side JavaScript when
rendered, rather than being unsafely inserted directly by server-side code.

## Steps Taken

### Step 1: Understand the flawed filter
Recognized that a `replace()` call without a global flag only affects the
first match. This meant any angle brackets appearing *after* the first
pair in the input would pass through completely unencoded.

### Step 2: Craft a comment that exploits this gap
Posted a comment containing:
<><img src=1 onerror=alert(1)>

## How the Payload Works
- The very first `<` and `>` (forming an empty, meaningless `<>` pair) are
  "sacrificed" — the filter's `replace()` call finds and encodes only this
  first occurrence
- Because `replace()` only processes the first match, the filter considers
  its job "done" after handling this initial pair — even though more angle
  brackets follow
- The real payload, `<img src=1 onerror=alert(1)>`, comes immediately
  after and passes through completely untouched, since the filter never
  looks at it
- The `img` tag itself uses the same broken-image/`onerror` technique from
  Labs 4 and 10 to trigger `alert(1)` without needing a `<script>` tag

## Result
Successfully bypassed the angle-bracket filter by exploiting its
single-replacement limitation, triggering `alert(1)` via the stored
comment, and solving the lab.

## What I Learned
This lab was a strong example of how an XSS defense can be
*conceptually* correct (encoding angle brackets does prevent HTML
injection, in principle) while still being completely broken due to an
implementation detail — here, forgetting that `String.replace()` with a
plain string argument only replaces the first match, not all matches
(which normally requires a regex with the `/g` global flag, or a different
approach entirely). This reinforces a recurring theme across many labs in
this topic (7, 12, and now 13): filter/encoding-based defenses are
fragile and depend entirely on getting every implementation detail
correct, whereas the more robust real-world fix is almost always to use
context-aware output encoding libraries or a Content Security Policy,
rather than hand-rolled string replacement logic.



## Lab 14: Reflected XSS into HTML Context with Most Tags and Attributes Blocked

Topic: Cross-Site Scripting (WAF Bypass) | Difficulty: Practitioner/Expert

## Vulnerability
The search functionality is reflected XSS-vulnerable, but a Web Application
Firewall (WAF) sits in front of it, blocking most common XSS payloads,
tags, and attributes. Solving this required systematically discovering
exactly which tag and which event attribute the WAF does NOT block,
rather than relying on a single known payload.

## Steps Taken

### Step 1: Confirm standard payloads are blocked
Tried a standard payload:
<img src=1 onerror=print()>
````
This was blocked by the WAF, confirming filtering is in place beyond
simple encoding.
Step 2: Systematically fuzz allowed HTML tags using Burp Intruder
Sent a search request to Burp Intruder and set the search term to:
<>
Placed a payload position between the angle brackets (<§§>), then used
PortSwigger's XSS cheat sheet to load a comprehensive list of HTML tags as
the payload set. Ran the attack, testing every tag automatically.
Result: Nearly every tag caused an HTTP 400 (blocked) response,
except body, which returned 200 — confirming the WAF allows the
<body> tag specifically.
Step 3: Systematically fuzz allowed event attributes
Using the confirmed <body> tag, set the search term to:
<body%20=1>
Placed a payload position right before the = sign (<body%20§§=1>),
then loaded the cheat sheet's list of event handler attributes as the
payload set. Ran the attack again.
Result: Most event attributes returned 400, except onresize, which
returned 200 — confirming onresize was the one event handler the WAF
permitted.
Step 4: Construct the working payload
Combined the two discovered allowed elements:
"><body onresize=print()>
This alone triggers print() — but onresize only fires when the
element's containing window/frame is resized, which doesn't happen
naturally on a normal page load. This required a delivery mechanism that
would force a resize event.
Step 5: Build a delivery exploit that triggers the resize event
On the exploit server, created the following payload:
html<iframe src="https://YOUR-LAB-ID.web-security-academy.net/?search=%22%3E%3Cbody%20onresize=print()%3E" onload="this.style.width='100px'">

The iframe's src loads the vulnerable search page with the
URL-encoded XSS payload already embedded in the search parameter
The iframe's own onload handler fires once it finishes loading, and
immediately changes the iframe's width (this.style.width='100px')
Resizing the iframe in this way triggers a resize event on the framed
page itself, which fires the injected onresize handler, executing
print()

Step 6: Deliver to victim
Stored the exploit and clicked "Deliver exploit to victim," successfully
triggering print() in the victim's simulated browser session.
Result
Successfully identified the one tag (body) and one event attribute
(onresize) not blocked by the WAF, then engineered a delivery mechanism
(an iframe that forcibly resizes itself) to reliably trigger that specific
event handler — bypassing the WAF entirely and solving the lab.
What I Learned
This was the most advanced XSS lab completed so far, combining several
skills: automated fuzzing with Burp Intruder to systematically map exactly
what a WAF allows versus blocks (rather than guessing individual payloads
one at a time), understanding that some event handlers (like onresize,
onscroll, onfocus) don't fire automatically on page load and require
specifically engineering a way to trigger the underlying browser event
(here, forcing an iframe resize), and constructing a self-contained
delivery exploit that does all of this automatically the moment a victim
opens the page. This reflects genuinely realistic WAF-bypass methodology:
real-world XSS filter evasion is rarely about finding one clever payload —
it's about systematically testing what a filter actually blocks versus
what it misses, then adapting the exploit to work within those
constraints.



## Lab 15: Reflected XSS into HTML Context with All Tags Blocked Except Custom Ones

Topic: Cross-Site Scripting (WAF Bypass) | Difficulty: Expert

## Vulnerability
This application's filter blocks every recognized/standard HTML tag
(`img`, `body`, `svg`, `script`, etc.) but does not block **custom,
non-standard tag names** — since these aren't real HTML elements, the
filter's blocklist (built around known tags) simply doesn't recognize them
as dangerous. However, a custom tag has no built-in behavior of its own —
it needs an event handler attribute to actually execute JavaScript, and
that event needs a way to fire automatically without user interaction.

## Steps Taken

### Step 1: Construct the custom tag payload
Built a payload using a fictitious tag name (`xss`, which has no special
meaning in HTML) with an `onfocus` event handler:
<xss id=x onfocus=alert(document.cookie) tabindex=1>
````
- `id=x` gives the element a reference name
- `tabindex=1` makes the otherwise non-interactive custom element
  focusable (custom tags aren't focusable by default, since browsers don't
  know what they are)
- `onfocus=alert(document.cookie)` fires the alert whenever this element
  receives focus
Step 2: Use a URL fragment to auto-trigger focus
Appended #x to the target URL — when a URL fragment matches an
element's id, the browser automatically scrolls to and focuses that
element on page load, without requiring the victim to click, hover, or
interact with the page at all.
Step 3: Build and deliver the exploit
On the exploit server, used a <script> tag to redirect the victim's
browser to a URL containing the encoded payload and the #x fragment:
html<script>
location = 'https://YOUR-LAB-ID.web-security-academy.net/?search=%3Cxss+id%3Dx+onfocus%3Dalert%28document.cookie%29%20tabindex=1%3E#x';
</script>
Clicked "Store" and "Deliver exploit to victim."
How the Payload Works

The filter only blocks known/standard tag names, so an invented tag like
<xss> passes through completely unfiltered
Since custom tags have no inherent behavior, tabindex=1 is required to
make the element eligible to receive keyboard/programmatic focus at all
The #x URL fragment causes the browser to automatically jump to and
focus the element with id="x" the moment the page finishes loading —
this is standard browser behavior for URL fragments matching element IDs,
not something requiring JavaScript to trigger manually
The moment focus lands on the element, the onfocus handler fires,
executing alert(document.cookie) with zero victim interaction required

Result
Successfully bypassed a tag-name-based filter using a completely
non-standard/custom tag name, combined with a URL fragment trick to
auto-trigger the onfocus event without any user interaction, solving
the lab.
What I Learned
This lab demonstrated that blocklist-based filters targeting known,
"dangerous" tag names have a fundamental blind spot: browsers will still
parse and render any tag name, even ones with no defined meaning
(<xss>, <foo>, etc.) — the browser simply treats them as generic,
unstyled elements. As long as an event handler attribute is attached and
some way exists to trigger that event, the specific tag name used is
irrelevant to actual exploitation. Using a URL fragment (#id) to
auto-focus an element is also a valuable, non-obvious technique for
triggering onfocus-based payloads automatically on page load, without
requiring the victim to click or interact with anything — reinforcing
that many event handlers assumed to require user interaction can actually
be triggered programmatically through browser behaviors like this.



## Lab 16: Reflected XSS with Some SVG Markup Allowed

Topic: Cross-Site Scripting (Filter Bypass) | Difficulty: Practitioner

## Vulnerability
The application's filter blocks common HTML tags typically associated
with XSS (`script`, `img`, `body`, etc.), but does not account for
SVG-specific elements and their associated event attributes — a less
commonly known but fully valid part of HTML/SVG markup.

## Steps Taken

### Step 1: Identify the gap in the filter
Recognized that SVG-related tags and animation elements are often
overlooked by filters designed primarily around standard HTML tags, since
SVG has its own set of elements and event attributes distinct from
regular HTML.

### Step 2: Construct the payload
<svg><animatetransform onbegin=alert(1) attributeName=transform>

## How the Payload Works
- `<svg>` opens an SVG container element — often allowed through filters
  since it's a legitimate, common tag used for vector graphics, not
  typically associated with script execution
- `<animatetransform>` is an SVG animation element, used normally to
  animate properties like position, scale, or rotation over time
- `onbegin` is an event attribute specific to SVG animation elements — it
  fires automatically the moment the animation begins, with no user
  interaction required
- `attributeName=transform` specifies which property the (fake/unused)
  animation targets — required for the element to be valid enough for the
  browser to actually process it and fire the `onbegin` event
- The moment the browser parses and begins processing this animation
  element, `onbegin` fires automatically, executing `alert(1)`

## Result
Successfully bypassed the tag filter using SVG-specific markup and an
SVG-specific event attribute, triggering `alert(1)` automatically on page
load, solving the lab.

## What I Learned
This lab reinforced a recurring theme from Labs 14–15: filters built
around a fixed list of "known dangerous" tags and attributes consistently
miss less mainstream but fully valid parts of the HTML/SVG specification.
SVG in particular has its own rich set of elements and event attributes
(`onbegin`, `onend`, `onrepeat`, etc.) that are frequently absent from
XSS blocklists because they're less commonly seen in typical attacks,
despite being just as capable of executing JavaScript. This further
confirms that blocklist-based filtering is fundamentally difficult to get
right, since it requires anticipating every valid tag/attribute
combination across the entire HTML and SVG specifications — reinforcing
why allowlist-based sanitization (or a strict Content Security Policy) is
a far more robust defense than trying to block "known bad" patterns.




## Lab 17: Reflected XSS in Canonical Link Tag

Topic: Cross-Site Scripting (Attribute Injection) | Difficulty: Expert

## Vulnerability
User input is reflected inside a `<link rel="canonical">` tag on the home
page. Angle brackets are escaped, ruling out any new tag injection.
However, since the reflection point sits inside an existing tag, an
attacker can still inject new *attributes* onto that tag — the same
underlying idea as Lab 7, but applied to a tag type (`link`) that isn't
normally associated with XSS at all.

## Steps Taken

### Step 1: Construct the payload
Since angle brackets were escaped, injected new attributes directly onto
the existing `<link>` tag instead of trying to create a new element:
'accesskey='x'onclick='alert(1)
Visited:
https://YOUR-LAB-ID.web-security-academy.net/?%27accesskey=%27x%27onclick=%27alert(1)

### Step 2: Trigger the exploit
Since `accesskey` requires an actual keyboard shortcut to activate (it
doesn't fire automatically), pressed the corresponding key combination for
the browser/OS being used (e.g. `Alt+X` on Linux), which triggered the
injected `onclick` handler.

## How the Payload Works
- The injected `accesskey='x'` attribute defines `X` as a keyboard
  accessibility shortcut for the entire page/element — a legitimate HTML
  feature intended to let users quickly focus/activate an element via the
  keyboard
- `onclick='alert(1)'` is injected as a second new attribute on the same
  tag — this is the actual payload, but it only fires on a click-like
  interaction
- Critically, activating an `accesskey` combination counts as triggering a
  "click" on the associated element, even though no mouse was used — so
  pressing the OS-specific key combination (e.g. `Alt+Shift+X`,
  `Ctrl+Alt+X`, or `Alt+X`) fires the `onclick` handler exactly as if the
  (invisible, non-interactive) `<link>` element had been physically clicked

## Result
Successfully injected `accesskey` and `onclick` attributes onto an
existing `<link>` tag, and triggered `alert(1)` by pressing the
corresponding access-key combination, solving the lab.

## What I Learned
This lab showed that attribute injection (from Lab 7) isn't limited to
"obviously interactive" tags like anchors or images — even a `<link>` tag,
which normally has no visible presence or user interaction at all, can be
turned into a click target using the `accesskey` attribute combined with
an `onclick` handler. This is a good example of abusing a legitimate
accessibility feature for an unintended, malicious purpose — a pattern
worth remembering, since accessibility-related attributes are rarely
considered part of a typical XSS threat model but can still provide a
fully functional interaction trigger. It also reinforced that identifying
*which* tag an injection lands in doesn't limit the attack to that tag's
"expected" behavior — any tag can become interactive if the right
attributes are added.
