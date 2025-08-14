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

<pre><s:element name="LoginRequest">
  <s:sequence>
    <s:element name="username" type="s:string"/>
    <s:element name="password" type="s:string"/>
  </s:sequence>
</s:element></pre>


**Messages** – Input/output formats for each operation.

<pre><wsdl:message name="LoginSoapIn">
  <wsdl:part name="parameters" element="tns:LoginRequest"/>
</wsdl:message></pre>


**Port Type** – Lists operations.

<pre><wsdl:operation name="Login">
  <wsdl:input message="tns:LoginSoapIn"/>
  <wsdl:output message="tns:LoginSoapOut"/>
</wsdl:operation></pre>


**Binding** – How operations are called (SOAP config).

`<soap:operation soapAction="Login" style="document"/>`

**Service** – The actual endpoint.

`<soap:address location="http://localhost:80/wsdl"/>`
