\# 02 - Reflected XSS (DVWA, Low Security)



\## What I did

Navigated to the XSS (Reflected) module in DVWA and entered the following payload

into the "What's your name?" input field:



`<script>alert('XSS')</script>`



\## What happened

The browser executed the input as JavaScript and displayed an alert popup with "XSS" —

confirming the script ran in the victim's browser context rather than being displayed

as plain text.



!\[Reflected XSS Result](./screenshots/xss-reflected-result.png)



\## Root cause

The application takes the `name` parameter directly from the GET request and reflects

it back into the HTML page with no output encoding:



```php

header ("X-XSS-Protection: 0");



if( array\_key\_exists( "name", $\_GET ) \&\& $\_GET\[ 'name' ] != NULL ) {

&#x20;   echo '<pre>Hello ' . $\_GET\[ 'name' ] . '</pre>';

}

```



Two problems here:



1\. `$\_GET\['name']` is concatenated directly into the HTML output with no encoding —

whatever the user sends gets rendered as raw HTML by the browser.



2\. `X-XSS-Protection: 0` explicitly disables the browser's built-in XSS filter,

removing the last line of defence. In a real application this header would never

be set to 0 — DVWA sets it deliberately to ensure the attack works for learning

purposes.



Because the input is reflected straight back into the page, an attacker can craft

a malicious URL like:



`http://localhost/dvwa/vulnerabilities/xss\_r/?name=<script>alert('XSS')</script>`



Anyone who clicks this link executes the attacker's script in their own browser —

against their own session, cookies, and account.



\## Code-level fix

Encode all output before rendering it into the page using `htmlspecialchars()` —

this converts special characters like `<` and `>` into their HTML entity equivalents

(`\&lt;` and `\&gt;`), so the browser displays them as text instead of executing them

as code:



```php

if( array\_key\_exists( "name", $\_GET ) \&\& $\_GET\[ 'name' ] != NULL ) {

&#x20;   $name = htmlspecialchars( $\_GET\[ 'name' ], ENT\_QUOTES, 'UTF-8' );

&#x20;   echo '<pre>Hello ' . $name . '</pre>';

}

```



With this fix, submitting `<script>alert('XSS')</script>` renders as the literal

text `<script>alert('XSS')</script>` on screen — the browser never sees it as

executable code.



\### Defence-in-depth

Beyond output encoding, a Content Security Policy (CSP) header adds a second layer:



`Content-Security-Policy: script-src 'self'`



This tells the browser to only execute scripts loaded from the same origin — inline

scripts injected via XSS are blocked entirely, even if output encoding is missed

somewhere.

