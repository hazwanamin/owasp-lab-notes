\# 04 - File Inclusion (DVWA, Low Security)



\## What I did

Navigated to the File Inclusion module in DVWA. The page loads a file based

on the `page` URL parameter — default URL looks like:



`http://localhost/dvwa/vulnerabilities/fi/?page=file1.php`



Modified the `page` parameter directly in the browser address bar to traverse

directories and target the Windows hosts file:



`http://localhost/dvwa/vulnerabilities/fi/?page=../../../../../../windows/system32/drivers/etc/hosts`



\## What happened

The application loaded and rendered the contents of the Windows system `hosts`

file directly on the page — a file completely outside the intended web directory.



This confirms Local File Inclusion (LFI) — an attacker can read arbitrary files

from the server's filesystem that the web server process has permission to access.



!\[File Inclusion Result](./screenshots/file-inclusion-result.png)



\## Note on Remote File Inclusion (RFI)

DVWA's File Inclusion module also demonstrates Remote File Inclusion — loading

a file from an external URL rather than the local filesystem. However, RFI

requires `allow\_url\_include` to be enabled in PHP, which was deprecated in

PHP 7.4 and removed entirely in PHP 8.x. This environment runs PHP 8.2.12,

so RFI is not possible here.



This is intentional documentation — in a real engagement, PHP version awareness

matters. RFI is effectively eliminated on any modern PHP stack, but LFI remains

a live threat regardless of PHP version.



\## Root cause

The entire vulnerability is a single line of code:



```php

$file = $\_GET\[ 'page' ];

```



The `page` parameter from the URL is taken directly from user input and passed

to PHP's file inclusion mechanism with zero validation. There is no check that

the value is an expected filename, no restriction to a specific directory, and

no allowlist of permitted files.



This allows an attacker to supply `../` sequences (path traversal) to navigate

up the directory tree and reach any file on the server — configuration files,

credential stores, application source code, or system files.



\*\*LFI vs RFI — the distinction for interviews:\*\*



\- \*\*Local File Inclusion (LFI):\*\* attacker reads or executes files already

present on the server — e.g. `/etc/passwd`, Windows `hosts`, application

config files containing database credentials.



\- \*\*Remote File Inclusion (RFI):\*\* attacker supplies a URL to an external

server hosting malicious code — the vulnerable application fetches and

executes it, giving the attacker remote code execution. Requires

`allow\_url\_include = On` in PHP config, which is disabled by default

and removed in PHP 8.



LFI alone can still lead to Remote Code Execution in advanced scenarios —

for example, by including PHP session files or server log files that the

attacker has previously injected malicious PHP code into (log poisoning).



\## Code-level fix

Never pass user input directly to a file inclusion function. Use an explicit

allowlist of permitted page names mapped to their actual file paths:



```php

$allowed\_pages = \[

&#x20;   'file1' => 'file1.php',

&#x20;   'file2' => 'file2.php',

&#x20;   'file3' => 'file3.php',

];



$page = $\_GET\[ 'page' ];



if( array\_key\_exists( $page, $allowed\_pages ) ) {

&#x20;   include( $allowed\_pages\[ $page ] );

} else {

&#x20;   echo 'File not found.';

}

```



With this fix, the only accepted values are `file1`, `file2`, and `file3` —

anything else, including any path traversal sequence, returns "File not found."

The user's input never reaches a file system operation directly.

