# WHOIS

✅ Scenarios Where WHOIS Helps
`whois facebook.com`

1. Phishing Investigation
Suspicious email links to a fake domain.

WHOIS shows:

- Domain is newly registered.

- Owner is hidden (privacy protection).

- Hosting provider is known for abuse.

- ➜ Red flag → Domain likely used for phishing.

2. Malware Analysis
Malware contacts a command-and-control (C2) server.
WHOIS shows:

- Registered with a free/anonymous email.

- Hosted in a high-risk country.

- Registrar ignores abuse complaints.

- ➜ Helps locate and report malicious infrastructure.

3. Threat Intelligence
Analysts study WHOIS data across attacker-owned domains.
They find patterns:

- Domains registered just before attacks.



# DNS 

| **Type** | **Purpose**                    | **Example**                                  |
| -------- | ------------------------------ | -------------------------------------------- |
| A        | Maps hostname → IPv4           | `www.example.com A 192.0.2.1`                |
| AAAA     | Maps hostname → IPv6           | `www.example.com AAAA 2001:db8::1`           |
| CNAME    | Alias to another hostname      | `blog.example.com CNAME webhost.net`         |
| MX       | Mail server info               | `example.com MX 10 mail.example.com`         |
| NS       | Name server for the domain     | `example.com NS ns1.example.com`             |
| TXT      | Misc. text (SPF, verification) | `example.com TXT "v=spf1 mx -all"`           |
| SOA      | Admin info for the zone        | `SOA ns1.example.com. admin.example.com.`    |
| SRV      | Defines services/ports         | `_sip._udp.example.com SRV 10 5 5060 server` |
| PTR      | IP → hostname (reverse lookup) | `1.2.0.192.in-addr.arpa PTR www.example.com` |

# Useful Tools:

| **Tool**         | **What It Does**                                                                                            |
| ---------------- | ----------------------------------------------------------------------------------------------------------- |
| `dig`            | Powerful DNS query tool. Great for all record types (A, MX, NS, etc.), troubleshooting, and zone transfers. |
| `nslookup`       | Simpler than `dig`. Used for basic A, AAAA, and MX lookups.                                                 |
| `host`           | Quick DNS lookups with short output.                                                                        |
| `dnsenum`        | Automated tool that brute-forces subdomains, attempts zone transfers.                                       |
| `fierce`         | Finds subdomains using recursion and wildcard detection.                                                    |
| `dnsrecon`       | Combines many DNS techniques. Great for deep DNS scans.                                                     |
| `theHarvester`   | OSINT tool that gathers emails and subdomains from public sources (including DNS records).                  |

⚠️ Be careful: excessive DNS queries may trigger blocks.

# dig command options:

| **Command**             | **Description**                       |
| ----------------------- | ------------------------------------- |
| `dig domain.com`        | Default A record lookup               |
| `dig domain.com MX`     | Finds mail servers                    |
| `dig domain.com NS`     | Finds name servers                    |
| `dig domain.com TXT`    | Finds TXT records                     |
| `dig domain.com CNAME`  | Finds canonical names                 |
| `dig +short domain.com` | Clean, short output                   |
| `dig -x 192.168.1.1`    | Reverse IP lookup                     |
| `dig +trace domain.com` | Shows DNS resolution path             |
| `dig ANY domain.com`    | Retrieves all records (often blocked) |

# Dnsenum useful command:

`dnsenum --enum inlanefreight.com -f /usr/share/seclists/Discovery/DNS/subdomains-top1million-110000.txt -r`

- `--enum`: Enables full enumeration (uses multiple techniques).

- `inlanefreight.com`: The target domain to scan.

- `-f <wordlist>`: Uses a specific wordlist to brute-force subdomains.

- `-r`: Recursively tries to find sub-subdomains.
- 

# AXFR Zone Transfer

🧠 DNS Zone Transfers (AXFR) – Simplified
What is it?
A zone transfer is when one DNS server copies all DNS records (like A, MX, CNAME, etc.) from another DNS server. It’s meant for backup and redundancy between trusted servers.

🔓 Why It Can Be a Risk
If a DNS server is misconfigured, anyone can request a zone transfer. That means an attacker could get:

✅ Full list of subdomains

✅ Their IP addresses

✅ Mail and name server info

✅ Hidden environments (dev, staging, etc.)

# Using AXFR:

 First, don't forget to add the IP to `/etc/resolv.conf` like `nameserver 10.10.10.10`. Then use this command: `dig axfr target-domain.htb @10.10.10.10`

 # Virtual Hosts:

 📘 Example: Discovering VHosts with Gobuster

`gobuster vhost -u http://<target_IP> -w <wordlist> --append-domain`

-u: Target URL (IP)

-w: Wordlist (e.g., /usr/share/seclists/...)

--append-domain: Adds domain to each word in the list (e.g., test.inlanefreight.htb)

Other useful flags:

-t: Threads (speed up)

-k: Ignore SSL cert errors

-o: Output results to a file


📝 Notes:
Hidden VHosts may reveal admin panels, staging environments, or test sites.

Combine with `/etc/hosts` if DNS doesn't resolve:

`echo "10.10.10.10 test.inlanefreight.htb" | sudo tee -a /etc/hosts`

# Certificate Transparency (CT) Logs:

| Tool       | Highlights                                                          | Best For                           | Pros                            | Cons                    |
| ---------- | ------------------------------------------------------------------- | ---------------------------------- | ------------------------------- | ----------------------- |
| **crt.sh** | Easy web search by domain, shows certificate details and subdomains | Quick checks, finding subdomains   | Free, no sign-up, simple        | Limited filtering       |
| **Censys** | Advanced search and filters, works with domain, IP, and cert data   | Deep analysis, misconfig detection | Detailed results, API available | Requires (free) account |


🔧 Example: Finding Subdomains with crt.sh API
You can use the crt.sh API in the terminal to find subdomains. Here’s how to find all dev subdomains of facebook.com:

<pre>curl -s "https://crt.sh/?q=facebook.com&output=json" | jq -r '.[] | select(.name_value | contains("dev")) | .name_value' | sort -u</pre>

What this does:

- curl fetches certificate data for facebook.com in JSON format.

- jq filters the results, showing only domains with "dev" in them.

- sort -u removes duplicates and sorts them.

Example Output:

<pre>*.dev.facebook.com  
dev.facebook.com  
secure.dev.facebook.com  
newdev.facebook.com  
...</pre>

This is a quick way to enumerate subdomains using CT logs.


# 🛠️ Common Fingerprinting Techniques

| Technique           | Description                                                  |
| ------------------- | ------------------------------------------------------------ |
| **Banner Grabbing** | Read server banners for software/version info.               |
| **HTTP Headers**    | Look at `Server`, `X-Powered-By`, etc.                       |
| **Custom Probes**   | Send crafted requests for version-specific errors.           |
| **Page Content**    | Detect tech by analyzing HTML/JS, e.g., "wp-" for WordPress. |


| Tool           | Use                                               |
| -------------- | ------------------------------------------------- |
| **Wappalyzer** | Browser extension for identifying web tech.       |
| **BuiltWith**  | Online tool for tech stack analysis.              |
| **WhatWeb**    | CLI tool for identifying CMS and server software. |
| **Nmap**       | With NSE scripts for service/OS detection.        |
| **Netcraft**   | Online site profile and security info.            |
| **wafw00f**    | Detects if a WAF (Web App Firewall) is in place.  |


# Banner Grabbing

`curl -I inlanefreight.com`

# WAF Detection with wafw00f

`pip3 install git+https://github.com/EnableSecurity/wafw00f`

`wafw00f inlanefreight.com`

# Web Scanner with Nikto

<pre>git clone https://github.com/sullo/nikto
cd nikto/program
chmod +x nikto.pl
./nikto.pl -h inlanefreight.com -Tuning b</pre>

# Crawlers:

| Tool                  | Description                                                          |
| --------------------- | -------------------------------------------------------------------- |
| **Burp Suite Spider** | Part of Burp Suite, maps websites and finds hidden content.          |
| **OWASP ZAP Spider**  | Open-source security scanner with built-in crawling.                 |
| **Scrapy**            | Python framework for building custom crawlers. Great for automation. |
| **Apache Nutch**      | Scalable Java-based crawler. Suitable for large-scale crawling.      |

# 🐍 Using Scrapy + ReconSpider

<pre>pip3 install scrapy
wget -O ReconSpider.zip https://academy.hackthebox.com/storage/modules/144/ReconSpider.v1.2.zip
unzip ReconSpider.zip
python3 ReconSpider.py http://inlanefreight.com</pre>

**Replace inlanefreight.com with your target.**

# Output

| Key              | What It Shows                     |
| ---------------- | --------------------------------- |
| `emails`         | Found email addresses.            |
| `links`          | Internal/external links.          |
| `external_files` | PDF/docs/etc. linked on the site. |
| `js_files`       | JavaScript files used.            |
| `images`         | Image links.                      |
| `form_fields`    | Any HTML forms.                   |
| `comments`       | HTML comments in the page source. |
| `videos/audio`   | Media files if found.             |


# Useful Google Dorking:

| Operator           | What It Does                        | Example                         |
| ------------------ | ----------------------------------- | ------------------------------- |
| `site:`            | Search within a specific site       | `site:example.com`              |
| `inurl:`           | Find keywords in URLs               | `inurl:login`                   |
| `filetype:`        | Search for specific file types      | `filetype:pdf`                  |
| `intitle:`         | Keyword in page title               | `intitle:"confidential report"` |
| `intext:`          | Keyword in the page body            | `intext:"password reset"`       |
| `cache:`           | View cached (old) version of a page | `cache:example.com`             |
| `link:`            | Find sites linking to a page        | `link:example.com`              |
| `related:`         | Find similar sites                  | `related:example.com`           |
| `" "`              | Exact phrase match                  | `"admin login"`                 |
| `-`                | Exclude terms                       | `site:example.com -inurl:login` |
| `*`                | Wildcard (any word)                 | `filetype:pdf user* manual`     |
| `..`               | Number range                        | `"price" 100..500`              |
| `AND`, `OR`, `NOT` | Combine or exclude terms            | `linux OR ubuntu NOT debian`    |


**🔐 Find Login Pages:**

<pre>site:example.com inurl:login
site:example.com (inurl:login OR inurl:admin)</pre>

**📄 Find Exposed Files:**

<pre>site:example.com filetype:pdf
site:example.com (filetype:xls OR filetype:docx)</pre>

**⚙️ Find Config Files:**

<pre>site:example.com inurl:config.php
site:example.com (ext:conf OR ext:cnf)</pre>

**💾 Find Database Backups:**

<pre>site:example.com inurl:backup
site:example.com filetype:sql</pre>


# 🕵️‍♂️ Using Web Archives (Wayback Machine)

The Wayback Machine lets you see how websites looked in the past. This is helpful for reconnaissance (OSINT) in the CBBH exam.

**✅ How It Helps in CBBH:**

- Find Old Pages: Discover deleted pages, hidden directories, or subdomains that could contain useful or sensitive info.

- Spot Vulnerabilities: Older versions of sites may expose old technologies or outdated features with known weaknesses.

- Track Changes: Compare how a target website evolved. You might find removed features or leaks.

- Silent Recon: Since you're not scanning the site directly, it’s a stealthy way to gather info without triggering alarms.


# Final Recconning:

**What FinalRecon Can Do:**

- Show server info and tech stack
- Check domain registration details
- Get SSL certificate info
- Crawl website pages and resources
- Enumerate DNS records and subdomains
- Search hidden directories/files
- Get URLs from Wayback Machine snapshots
- Scan ports quickly

**Setting up**
<pre>git clone https://github.com/thewhiteh4t/FinalRecon.git
cd FinalRecon
pip3 install -r requirements.txt
chmod +x ./finalrecon.py</pre>

**Usage**
./finalrecon.py --headers --whois --url http://inlanefreight.com
