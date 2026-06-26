\# 06 - Burp Suite — Manual SQL Injection Testing



\## Tool

Burp Suite Community Edition v2026.4.3



\## Target

DVWA SQL Injection module — Low security

`http://localhost/dvwa/vulnerabilities/sqli/`



\## What I did



\### Step 1 — Intercepted the request

Configured Burp Suite as a proxy and navigated to the DVWA SQL 

Injection module. Submitted User ID `1` — Burp intercepted the 

outgoing GET request before it reached the server:



GET /dvwa/vulnerabilities/sqli/?id=1\&Submit=Submit HTTP/1.1



Host: localhost



\### Step 2 — Sent to Repeater

Right-clicked the intercepted request and sent it to Burp Repeater 

for manual testing — allowing repeated modification and replay of 

the request without using the browser.



\### Step 3 — Normal response

Sent the original request in Repeater. Server returned the expected 

result for User ID 1:



ID: 1



First name: admin



Surname: admin



\### Step 4 — Injected SQL payload

Modified the `id` parameter to:



1' OR '1'='1



This needs to be encoded to make to acceptable as a http request



URL-encoded for the GET request:



GET /dvwa/vulnerabilities/sqli/?id=1%27+OR+%271%27%3D%271\&Submit=Submit



ID: 1' OR '1'='1 — First name: admin, Surname: admin



ID: 1' OR '1'='1 — First name: Gordon, Surname: Brown



ID: 1' OR '1'='1 — First name: Hack, Surname: Me



ID: 1' OR '1'='1 — First name: Pablo, Surname: Picasso



ID: 1' OR '1'='1 — First name: Bob, Surname: Smith





!\[Burp Repeater SQL Injection](./screenshots/burp-repeater-sqli-injection.png)



\## Why this works

The `id` parameter is concatenated directly into the SQL query 

without sanitisation. The payload `1' OR '1'='1` closes the 

original string and adds a condition that is always true — causing 

the query to return every row in the users table.



\## What Burp Suite adds over browser testing

\- Intercepts and displays the raw HTTP request including all headers

\- Allows modification of any parameter before it reaches the server

\- Repeater enables rapid retesting without reloading the browser

\- Builds an HTTP history log of every request made during testing



\## Fix

Parameterised queries — keep user input separate from query logic 

so it can never be interpreted as SQL syntax. See `01-sql-injection.md` 

for the full code-level fix.

