# API Security

## Zero Trust Security

### Philosophy
The philosophy behind a Zero Trust network assumes that there are attackers both within and outside of the network, so no users or machines should be automatically trusted. 
### Definition
Zero Trust security is an IT security model that requires strict identity verification for every person and device trying to access resources on a private network, regardless of whether they are sitting **within or outside of the network perimeter.**
Zero Trust security means that no one is trusted by default from inside or outside the network, and verification is required from everyone trying to gain access to resources on the network. 
### Principles
1. Grants access only to specific authorized resources
2. Least Privileges: Users or devices get minimum access needed
3. Continuous Validation: Logins and connections time out periodically once established, forcing users and devices to be continuously re-verified.

## Type of Risks
1. Weak Authentication
2. Man in Middle: Steal or modifies data in flight
3. Vulnerabilties: Code or script injection in request body
4. Deniai of Service Attack: Jamming API to crash the server
5. Broken Object : Using Ids in API route to gain access

## Development Time Security Monitoring 
- Analyze source code for security flaws , vulnerabilties before deployment using tools like SonarQube,
- Scan third-party libraries and dependencies for known vulnerabilities using tools like Snyk or OWASP Dependency-Check.
- Continuous security scanning in CI/CD pipelines
- Keep dependencies updated: Regularly check for security patches and update dependencies to the latest stable versions
- Only include essential libraries to reduce the attack surface and potential security risks.
https://www.securitycompass.com/blog/application-security-best-practices/

## App to API - B2C
- Enforce strong pwd, MFA
- Use secure authentication tokens: Implement JSON Web Tokens (JWT) with proper expiration and encryption to prevent token theft and replay attacks.
- Enable role-based access control (RBAC): Restrict user permissions based on their role or attributes to enforce the principle of **least privilege**
- Short lived sessions , Logout inacive sessions

### CORS
browsers enforce that clients can only send requests to a resource with the same origin as the client's URL. The protocol, port, and hostname of the client's URL should all match the server it requests.
- You need CORS when you want to pull data from external APIs that are public or authorized. 
- You also need CORS if you want to allow authorized third-party access to your own server resources.
#### Pre-Flight with OPTION
- Why: Can browser send cross-domain request?
Browser creates pre-flight request
    OPTIONS /data HTTP/1.1
    Origin: https://example.com
    Access-Control-Request-Method: DELETE
Server responds with
    HTTP/1.1 200 OK
    Access-Control-Allow-Headers: Content-Type
    Access-Control-Allow-Origin: https://news.example.com
    Access-Control-Allow-Methods: GET, DELETE, HEAD, OPTIONS

### Token based Auth
#### Two-Legged Authentication
- Api to API- Client credential
- Device to API - Device Credentia;
#### Three-Legged Authentication
-- Auth code where user is involved

#### Nonce in OIDC
- The server side client generates random nonce before forwarding Auth request to IDP
- When IDP issues token, it includes the same nonce in the token
- The client app matches nonce in the token with the nonce in session memory
- if matching fails, the token is not valid.

## API to API - B2B
### WAF Web Application Forewall
- Vulnerability Scanning:WAF security software can analyze incoming API requests and block malicious traffic before reaching enterprise infrastructure.

### API Gateway
- Single entry point for all security policies
- Offload B2C and forward B2B security

### Rate Limiter & DDoS Protection
This protects resources against brute force and DoS attacks by restricting the number of requests a user or IP address can make in a certain amount of time.

### Schema and Data Validation
- Validate and sanitize user input
- Schema validation identifies invalid requests and responses. 
- Prevent SQL injection by using prepared statements rather than concatenating user input directly into database queries.

### PII Protection Encrypt data at rest: 
- Always encrypt sensitive data, such as passwords, credit card numbers, and personal identification information, both at rest and in transit.

### Hash sensitive data : 
- Hash Password and personal identfiers- Never store plain text, save Hash created out of hash dunction and compare aginst the hash
- Digital Segnature: this is to ensure that email message or file or payload or JWT is not beng currupted while transit
    - Apply a hash function to JWT to create a hash value 
    - Sender Encrypts the hash using private key to create digital signature
    - The receipient decrypt the digital signature with the public key in the cert
    - The receipient then create hash out of the recieved JWT to the same hash function . If two hashes match, that means JWT is not tampered.

**Hashing vs Encryption**
 - Hashing is one way, once applied , can not be recovered
- Encryption can be decrypted

### API Key Auth
The client is assigned a key — a unique string of characters that only they and the API service know. The key is attached to each API request. The API server checks for the key when it receives an API request to make sure it is from an authenticated client.

- API key can be stolen, not safe
- Good for lower environment
- Create separate keys for read, write and admin operations. Rotate keys often.


### TLS - Encryption Protocal Encrypt data in transit:
TLS is the encryption protocol that creates an encrypted, authenticated connection between client and server when loading webpages. 

### TLS Handshake
Client connects to server
Server presents its TLS certificate
Client verifies the server's certificate
Client and server exchange information over encrypted TLS connection

### mTLS
1. Both client and server possess their own own copy of TLS certs.
2. The client initiates a reqeust, the server challenges presenting returning the cert
3. The client validates the cert with CA
4. The client responds with its own cert , which then the server validates with the CA 
5. Once both certs are validated the encrypted channel is created

#### Why mTLS?
in API Security, mTLS ensures that API requests come from legitimate, authenticated users only
On-path attacks: On-path attackers place themselves between a client and a server and intercept or modify communications between the two.When mTLS is used, on-path attackers cannot authenticate to either the client or the server, making this attack almost impossible to carry out.


