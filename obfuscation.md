# ❓ What is Obfuscation?
Obfuscation is the process of making code hard to read, while keeping it functionally the same.
It’s usually done with automated tools.
The goal: confuse humans, but still let computers run it.

**🧠 Example:**
A simple JavaScript program can be turned into messy, unreadable code that works the same.

Tools like `beautifytools.com/javascript-obfuscator` can do this.


# Basic Obfuscation

**▶️ Example Code**
`js console.log('HTB JavaScript Deobfuscation Module');`

Run at: https://jsconsole.com → prints message.

**🔽 Minification**
Removes spaces/formatting → one line.

Tool: https://javascript-minifier.com
File extension: .min.js
Still readable if code is small.

**📦 Packing**
Stronger obfuscation using `eval(function(p,a,c,k,e,d){...})....`

Tool: `https://beautifytools.com/javascript-obfuscator.php`

Converts code to encoded format using dictionary.

Strings may still leak info (e.g. HTB).

| Type      | Description                      | Readability |
| --------- | -------------------------------- | ----------- |
| Cleartext | Normal readable code             | 🟢 Easy     |
| Minified  | One-liner, compressed format     | 🟡 Medium   |
| Packed    | Encoded & structured, unreadable | 🔴 Hard     |


# Advanced Obfuscation

- Basic obfuscation still shows some strings (e.g., "HTB") → can leak functionality.
- Advanced obfuscation hides **all readable content**.

**🔧 Tool: obfuscator.io**

- Go to: `https://obfuscator.io`
- Enable: **String Array Encoding → Base64**
- Paste code → Click **Obfuscate**
- Output: Completely unreadable JavaScript with no cleartext.

**🎭 More Obfuscation Examples**
Example (crazy unreadable obfuscation using logical operators and coercion):

`[][(![]+[])[+[]]+...](...)(...);`
Output: 'HTB JavaScript Deobfuscation Module'
Still runs fine at `https://jsconsole.com`

⚠️ These techniques can slow down performance.

**🛠 Other Obfuscation Tools**
JJ Encode → Converts code using Japanese-style symbols
AA Encode → Converts code using weird ASCII art-like syntax
Good for: bypassing filters or restrictions

⚠️ Not recommended unless truly necessary

# Deobfuscation

**1 - 🧼 Beautify (Prettify Minified Code)**
Minified code is usually on one line.

Use browser DevTools:

Firefox: Press Ctrl+Shift+Z, open the script (e.g., secret.js), click {} to Pretty Print.

Online tools also work:

- Prettier Playground
- Beautifier.io

Example (packed code using eval):

`eval(function(p,a,c,k,e,d){...})` // unreadable initially
Beautify tools format it, but code might still be unreadable due to obfuscation.

**2 - 🔓 Deobfuscate (Unpack Code)**

Use tools like UnPacker - `https://matthewfl.com/unPacker.html`

- Paste packed code
- Click UnPack

⚠️ Make sure there are no empty lines at the top

**3 - 🔧 Manual Reverse Engineering**
If the obfuscation is too complex or custom, tools may fail.

In such cases:

- Manually trace code logic
- Understand encoding/decoding patterns
- Reverse transformations

# Code Analysis

**Deobfuscated Code**

Let's say, we have this code after deobfuscating it:

<pre>'use strict';
function generateSerial() {
  ...SNIP...
  var xhr = new XMLHttpRequest;
  var url = "/serial.php";
  xhr.open("POST", url, true);
  xhr.send(null);
};</pre>



**The file has one function: generateSerial().**

- It creates an XMLHttpRequest object (xhr) and sets a url = `"/serial.php"`
- Then it sends a POST request to `/serial.php` using `xhr.open("POST", url, true)` and `xhr.send(null)`
- No data is sent and no response is processed.

Purpose: Likely meant to trigger something on the server (e.g., generating a serial).
Current Usage: No visible HTML element uses this; may be unused or reserved for future.
**Next Step: Try manually sending a POST to /serial.php — might reveal an unfinished or vulnerable backend feature.**

# Sending POST Request with cURL

`curl -s http://SERVER_IP:PORT/serial.php -X POST -d "param1=sample"`

# Decoding

**Common Encoding Methods:**

**🔹 Base64**
Uses only letters, numbers, +, /, and = padding.
Length is always a multiple of 4.

Encode:

`echo "text" | base64`

Decode:

`echo "base64string" | base64 -d`

**🔹 Hex**
Encodes text into hexadecimal ASCII values.

Uses only 0-9 and a-f.

Encode:

`echo "text" | xxd -p`

Decode:

`echo "hexstring" | xxd -p -r`

**🔹 ROT13 / Caesar Cipher**
ROT13 shifts letters 13 characters. a → n, b → o, etc.
Easy to spot if URL-like patterns remain.

Encode/Decode:

`echo "text" | tr 'A-Za-z' 'N-ZA-Mn-za-m'`

For identifying Unknown Encodings, use tools like Cipher Identifier to detect unknown encoding formats.

⚠️ Some advanced obfuscation uses encryption with keys—decoding is only possible if the key is known or found in the code.
