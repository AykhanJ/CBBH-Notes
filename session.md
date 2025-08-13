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

cookie name: PHPSESSID
cookie value: <sessionID>

**Java (Tomcat):**

Session manager stores sessions in SESSIONS.ser

**.NET:**

Session data in:

aspnet_wp.exe (InProc)

StateServer (OutProc)

## SQL Server

**3. Post-Exploitation (Database Access)**

If you have database access (via SQL injection or credentials), check for session tables:


show databases;
use project;
show tables;
select * from users;
select * from all_sessions;
select * from all_sessions where id=3;

Extracted session IDs allow logging in as users without credentials.
