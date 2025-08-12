# Authentication Types

Three Main Authentication Factor Types:

- Knowledge — something you know (e.g., password, PIN, security question).
- Ownership — something you have (e.g., ID card, security token, authenticator app).
- Inherence — something you are (e.g., fingerprint, face, voice).

# Brute Forcing 

**What is User Enumeration?**

User enumeration happens when a web app reveals whether a username exists based on different responses for valid and invalid usernames. This often happens during login, registration, or password reset processes.

Although usernames might seem harmless, they can be sensitive because they’re often the primary identifier for authentication. Attackers can gather valid usernames from one service (like a web app) and use them to target other services (FTP, SSH, RDP, etc.) on the same network.

**Why Does This Happen?**

Web developers sometimes reveal different error messages for invalid vs. valid usernames to help legit users correct mistakes. However, this behavior unintentionally helps attackers confirm which usernames are valid.

Example: WordPress login

- Invalid username → Error: “Unknown username. Check again or try your email address.”
- Valid username + wrong password → Error: “The password you entered for the username editor is incorrect.”

Such differences make it easy to enumerate users.

**When is User Enumeration a Feature?**

Some apps, like chat platforms, allow searching users by username. While this can help legitimate users find others, it also allows attackers to enumerate usernames.
Still, user enumeration should be minimized when possible — for example, by requiring login with an email instead of a username.

**How to Enumerate Users: Using Error Messages**

Attackers need a username list (wordlist) to test against login forms. Usernames are usually simpler than passwords, with fewer special characters. Common sources include public info, social media, company websites, or wordlists like SecLists.

**Example with ffuf (fuzzing tool):**

You try logging in with usernames from a wordlist and observe server responses. For invalid users, the server might say “Unknown user.” For valid users, you get a different error like “Invalid credentials.”
By filtering out responses containing “Unknown user,” you get a list of valid usernames.


`ffuf -w /opt/useful/seclists/Usernames/xato-net-10-million-usernames.txt -u http://<SERVER_IP>/index.php -X POST -H "Content-Type: application/x-www-form-urlencoded" -d "username=FUZZ&password=invalid" -fr "Unknown user"`

Result: You find valid usernames, like consuelo, to attack further.

# Brute-Forcing Passwords

After identifying valid users, the next authentication barrier is the password. Since many users pick easy-to-remember or reused passwords, attackers can often guess or brute-force them to gain access.

**Password Policies Matter**

Successful brute-forcing depends on how many guesses you can make and how fast you can try them. A good, targeted wordlist is key.
If the target app enforces a password policy, your wordlist should only contain passwords that comply. Otherwise, you waste time trying passwords the system would reject anyway.

**Example: Filtering a Wordlist to Match a Password Policy**

Suppose the login page says:

Password must have at least one uppercase letter, one lowercase letter, one digit, and be at least 10 characters long.

The popular `rockyou.txt` wordlist contains over 14 million passwords:

`wc -l /opt/useful/seclists/Passwords/Leaked-Databases/rockyou.txt`

**Output: 14344391**

We can filter this list to passwords that match the policy using grep:

`grep '[[:upper:]]' rockyou.txt | grep '[[:lower:]]' | grep '[[:digit:]]' | grep -E '.{10,}' > custom_wordlist.txt`
`wc -l custom_wordlist.txt`

**Output: ~151647 passwords**

This reduces the wordlist by about 99%, saving time during the attack.

# Brute-Forcing Password Reset Tokens


Many web applications offer a password recovery feature that uses a one-time reset token sent to the user via email or SMS. 
This token lets the user authenticate temporarily to reset their password.
If these reset tokens are weak or predictable, attackers can brute-force them to hijack accounts.

**Typical Password Reset Flow:**

- User forgets password and requests a reset
- Application generates a reset token and sends it (via email or SMS)
- User presents the token to verify identity
- User sets a new password
- User gains access with the new password

**Identifying Weak Tokens**

To find weak tokens, create an account on the target app, request a password reset, and analyze the received token.

For example, a password reset email might look like this:

<pre>Hello,

We received a request to reset your password. Use the link below to reset it:

http://weak_reset.htb/reset_password.php?token=7351

This link expires in 24 hours. If you did not request this, ignore this email.

Thank you.</pre>
  
Here, the reset token 7351 is only 4 digits long. This means there are only 10,000 possible tokens (0000 to 9999). An attacker can brute-force all possible tokens to hijack accounts.

**Attacking Weak Reset Tokens**

We will brute-force the reset token using ffuf. First, generate a wordlist of all 4-digit tokens:

`seq -w 0 9999 > tokens.txt`

The -w flag zero-pads numbers, ensuring all tokens have 4 digits.

Verify with:

`head tokens.txt`

Output:

<pre>0000
0001
0002
...
0009
</pre>

**Brute-Forcing with ffuf**

`ffuf -w ./tokens.txt -u http://weak_reset.htb/reset_password.php?token=FUZZ -fr "The provided token is invalid"`

# Attacking 2FA

The account is protected with 2FA, prompting for a 4-digit one-time password after login:

<pre>http://<SERVER_IP>:<PORT>/2fa.php
Welcome admin. Please provide your 4-digit One-Time Password (OTP).</PORT>

Since the 2FA code is only 4 digits long, there are 10,000 possible combinations (0000 to 9999), which can be brute-forced.
To brute-force all 4-digit OTPs, generate a wordlist of all possible codes

`seq -w 0 9999 > tokens.txt`

The -w flag zero-pads numbers to maintain a consistent 4-digit length.


