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
