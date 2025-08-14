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


<wsdl:message name="LoginSoapIn">
  <wsdl:part name="parameters" element="tns:LoginRequest"/>
</wsdl:message>
