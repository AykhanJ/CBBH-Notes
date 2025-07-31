# Basic directory fuzzing:

`ffuf -w /opt/useful/seclists/Discovery/Web-Content/directory-list-2.3-small.txt:FUZZ -u http://SERVER_IP:PORT/FUZZ`

# Extension Fuzzing:

`ffuf -w /opt/useful/seclists/Discovery/Web-Content/web-extensions.txt:FUZZ -u http://SERVER_IP:PORT/blog/indexFUZZ`

# Page fuzzing after determining the extension:

`ffuf -w /opt/useful/seclists/Discovery/Web-Content/directory-list-2.3-small.txt:FUZZ -u http://SERVER_IP:PORT/blog/FUZZ.php`

# Recursive Scan:

- **Recursive Fuzzing**
So far, we’ve been fuzzing directories, then going inside them to find files. But if there are many folders with subfolders, this process becomes very slow. To fix this, we can use recursive fuzzing, which automates the process.

- **What Is Recursive Fuzzing?** 
Recursive fuzzing means that once a tool finds a folder, it will automatically scan inside it, and keep doing this for every new folder it finds.

Some websites have deep folder paths like `/login/user/content/uploads/...`, which can make scans very long. To avoid wasting time, we can set a depth limit—this tells the tool how many levels deep it should scan.

# Using ffuf for Recursive Fuzzing:
- Use **-recursion** to turn on recursive scanning.

- Use **-recursion-depth 1** to scan only main folders and their direct subfolders.

For example: it will scan `/login` and `/login/user`, but not `/login/user/content`.

- Use **-e .php** to tell ffuf to look for .php pages.

- Add **-v** to show full URLs in the output (helpful for seeing where each file is).

`ffuf -w /opt/useful/seclists/Discovery/Web-Content/directory-list-2.3-small.txt:FUZZ -u http://SERVER_IP:PORT/FUZZ -recursion -recursion-depth 1 -e .php -v`

# Writing IP and Domain to /etc/hosts:

`sudo sh -c 'echo "SERVER_IP  academy.htb" >> /etc/hosts'`

# Subdomain Fuzzing:

`ffuf -w /opt/useful/seclists/Discovery/DNS/subdomains-top1million-5000.txt:FUZZ -u https://FUZZ.inlanefreight.com/`

# Reminder: To fuzz hidden/internal subdomains, add possible subdomains manually to /etc/hosts.

# Vhosts Fuzzing:

`ffuf -w /opt/useful/seclists/Discovery/DNS/subdomains-top1million-5000.txt:FUZZ -u http://academy.htb:PORT/ -H 'Host: FUZZ.academy.htb'`

# Filtering Results:

- **Size Filtering:**  `ffuf -w /opt/useful/seclists/Discovery/DNS/subdomains-top1million-5000.txt:FUZZ -u http://academy.htb:PORT/ -H 'Host: FUZZ.academy.htb' -fs 900`
As a result, we have a subdomain like: admin.academy.htb. This is obviously not a public website ,so we need to consider these notes:

**Note 1: Don't forget to add "admin.academy.htb" to "/etc/hosts".**

**Note 2: If your exercise has been restarted, ensure you still have the correct port when visiting the website.**

# GET Parameter Fuzzing:

Goal is to find hidden GET parameters that might give access to content (like a flag) on:
http://admin.academy.htb:PORT/admin/admin.php

**🧠 Background:**
When visiting the page, we see “You don't have access to read the flag.”

Note that we're not logged in and have no cookies.

There may be a key parameter we can pass via GET (e.g. ?key=value) to access the flag.

**⚙️ Tool: ffuf**

Wordlist:
/opt/useful/seclists/Discovery/Web-Content/burp-parameter-names.txt

Command:
`ffuf -w /opt/useful/seclists/Discovery/Web-Content/burp-parameter-names.txt:FUZZ -u http://admin.academy.htb:PORT/admin/admin.php?FUZZ=key -fs xxx`

?FUZZ=key means testing parameters like ?auth=key, ?token=key, etc.

**✅ Result:**
We get one valid hit, but visiting:

`http://admin.academy.htb:PORT/admin/admin.php?REDACTED=key`

shows:

"This method is deprecated."

So the parameter exists, but it’s no longer useful.

**💡 Tip:**
Fuzzed parameters may reveal undocumented and potentially vulnerable functionality. Always test these for common web vulnerabilities.

# POST Parameter Fuzzing:

`ffuf -w /opt/useful/seclists/Discovery/Web-Content/burp-parameter-names.txt:FUZZ -u http://admin.academy.htb:PORT/admin/admin.php -X POST -d 'FUZZ=key' -H 'Content-Type: application/x-www-form-urlencoded' -fs xxx`

# Value Fuzzing:

First, we need to create a file to fuzz id value: `for i in $(seq 1 1000); do echo $i >> ids.txt; done`

After that, command: `ffuf -w ids.txt:FUZZ -u http://admin.academy.htb:PORT/admin/admin.php -X POST -d 'id=FUZZ' -H 'Content-Type: application/x-www-form-urlencoded' -fs xxx`
