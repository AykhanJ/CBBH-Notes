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

'curl -s "https://crt.sh/?q=facebook.com&output=json" | jq -r '.[] | select(.name_value | contains("dev")) | .name_value' | sort -u'

What this does:

- curl fetches certificate data for facebook.com in JSON format.

- jq filters the results, showing only domains with "dev" in them.

- sort -u removes duplicates and sorts them.

Example Output:

'
*.dev.facebook.com  
dev.facebook.com  
secure.dev.facebook.com  
newdev.facebook.com  
...
'

This is a quick way to enumerate subdomains using CT logs.
