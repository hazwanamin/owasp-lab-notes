\# 01 - SQL Injection (DVWA, Low Security)



\## What I did

Navigated to the SQL Injection module in DVWA and entered the following payload into the User ID field:



`1' OR '1'='1`



\## What happened

Instead of returning the details for a single user, the application returned the first name and surname for every user in the database — admin, Gordon Brown, Hack Me, Pablo Picasso, and Bob Smith.



!\[SQL Injection Result](./screenshots/sqli-result.png)



\## Root cause

The application builds its SQL query by directly concatenating the raw `$id` value from user input into the query string without using parameters or sanitization:



```php

$id = $\_REQUEST\[ 'id' ];

$query = "SELECT first\_name, last\_name FROM users WHERE user\_id = '$id';";

$result = mysqli\_query($GLOBALS\["\_\_\_mysqli\_ston"], $query);

```



Submitting `1' OR '1'='1` changes the query's actual logic to:



```sql

SELECT first\_name, last\_name FROM users WHERE user\_id = '1' OR '1'='1';

```



`'1'='1'` always evaluates to true, so the WHERE clause matches every row in the table. The database has no way to distinguish "data" from "part of the query" because they were never separated.



\## Code-level fix

Use a parameterised prepared statement — this keeps user input strictly as data and never lets it become part of the query logic:



```php

$id = $\_REQUEST\[ 'id' ];

$stmt = mysqli\_prepare($GLOBALS\["\_\_\_mysqli\_ston"], 

&#x20;   "SELECT first\_name, last\_name FROM users WHERE user\_id = ?");

mysqli\_stmt\_bind\_param($stmt, "s", $id);

mysqli\_stmt\_execute($stmt);

$result = mysqli\_stmt\_get\_result($stmt);



while ($row = mysqli\_fetch\_assoc($result)) {

&#x20;   $first = $row\["first\_name"];

&#x20;   $last  = $row\["last\_name"];

&#x20;   echo "<pre>ID: {$id}<br />First name: {$first}<br />Surname: {$last}</pre>";

}

```



The `?` placeholder reserves the position for the value. No matter what the user types — including `1' OR '1'='1` — it is treated as a literal string to match against, not executable SQL syntax.

