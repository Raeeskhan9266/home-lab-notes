
# home-lab-notes
## Lab 13: Stored DOM XSS
Completed: [aaj ki date]
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
