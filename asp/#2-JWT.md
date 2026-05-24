# JWT (JSON Web Token) - Complete Technical Guide

## 1. What is JWT?

A JWT is a compact, self-contained string used to securely transmit claims (data) between parties.

### Common Uses

- Authentication
- Authorization
- Stateless sessions

### What JWT Proves

1. Who issued the token
2. That the token has not been modified

**Important:** JWT does not hide data by default—it only protects integrity.

---

## 2. JWT Structure

A JWT contains three parts separated by dots:

```
HEADER.PAYLOAD.SIGNATURE
```

### Example

```
eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9
.
eyJ1c2VyX2lkIjo0Miwicm9sZSI6InVzZXIifQ
.
Xh4Wv2...
```

### Part 1: Header

Metadata describing how the token is signed.

```json
{
  "alg": "HS256",
  "typ": "JWT"
}
```

### Part 2: Payload

Contains claims (data about the user or session).

```json
{
  "sub": 42,
  "role": "user",
  "exp": 1710000000,
  "iat": 1709999000
}
```

#### Common Claims

| Claim | Meaning                                  |
| ----- | ---------------------------------------- |
| `sub` | Subject (user ID)                        |
| `exp` | Expiration time (Unix timestamp)         |
| `iat` | Issued at (Unix timestamp)               |
| `iss` | Issuer (who created the token)           |
| `aud` | Audience (intended recipient)            |
| `nbf` | Not before (token valid after this time) |
| `jti` | JWT ID (unique identifier)               |

#### Custom Claims

You can add application-specific data:

```json
{
  "user_id": 42,
  "username": "johndoe",
  "permissions": ["read", "write"],
  "department": "engineering"
}
```

**Warning:** Payload is visible to anyone with the token. Never store passwords, credit cards, or secrets in it.

### Part 3: Signature

Ensures token integrity and authenticity.

If someone modifies the payload, the signature becomes invalid.

---

## 3. Encoding vs Signing vs Encryption

This is where confusion typically occurs.

| Mechanism  | Purpose                | Key Required | Reversible | Secure |
| ---------- | ---------------------- | ------------ | ---------- | ------ |
| Encoding   | Formatting (Base64)    | No           | Yes        | No     |
| Signing    | Integrity/Authenticity | Yes          | No         | Yes    |
| Encryption | Confidentiality        | Yes          | No         | Yes    |

### Encoding (Base64URL)

Header and payload are Base64URL encoded, not encrypted.

```
Plain:    {"user_id":42}
Encoded:  eyJ1c2VyX2lkIjo0Mn0
```

**Properties:**

- Reversible (anyone can decode it)
- No key required
- Not secure

**Purpose:**

- Make token safe for URLs and HTTP headers

### Signing (The Real Security)

The server signs the token using a secret key.

**Formula:**

```
SIGNATURE = HASH(
  base64(header) + "." + base64(payload),
  SECRET_KEY
)
```

**Example with HS256:**

```
HMACSHA256(header.payload, secret_key)
```

**Result:**

```
HEADER.PAYLOAD.SIGNATURE
```

---

## 4. What Happens When Server Receives JWT

When a request arrives with:

```
Authorization: Bearer eyJhbGciOiJIUzI1NiIs...
```

The server performs two steps:

### Step 1: Decode

Extract and decode header and payload.

```
No key required.
No validation yet.
```

### Step 2: Verify Signature

Server recomputes the signature:

```
expected_signature = HMACSHA256(header.payload, SECRET_KEY)
```

Then compares:

```
expected_signature == token_signature ?
```

**If equal:** Token is valid  
**If not equal:** Token is rejected

**Key Point:** The secret key is used ONLY for signature verification, not for decoding.

---

## 5. Why Modifying a Token Fails

An attacker can:

1. Decode the payload
2. Modify it (e.g., `role: user` → `role: admin`)
3. Re-encode it

But they **cannot** generate a valid signature because they don't have the secret key.

**Result:**

```
Signature mismatch → Token rejected
```

This is why JWT is secure—the signature proves authenticity.

---

## 6. Signing Algorithms

### Symmetric Algorithms (HMAC)

Examples: `HS256`, `HS384`, `HS512`

**One shared secret key used for both signing and verification.**

```
Server signs JWT with secret_key
Server verifies JWT with secret_key
```

**Use case:** Single backend system

**Example:**

```
Algorithm: HS256 (HMAC SHA-256)
Secret: very_long_random_string_min_256_bits
```

### Asymmetric Algorithms (RSA, ECDSA)

Examples: `RS256`, `RS384`, `ES256`, `ES384`

**Two keys exist:**

- Private key → Sign tokens
- Public key → Verify tokens

```
Auth Server: Signs JWT with PRIVATE_KEY
API Servers: Verify JWT with PUBLIC_KEY
```

**Benefits:**

- Verification servers don't need the private key
- Public key can be safely distributed
- Scalable for microservices

**Example:**

```
Algorithm: RS256 (RSA SHA-256)
Private Key: /config/jwt/private.pem (kept secret)
Public Key: /config/jwt/public.pem (distributed)
```

---

## 7. Where Keys Are Stored

Keys are **never** stored inside the JWT.

### Typical Storage Locations

**Environment Variables:**

```
JWT_SECRET=very_long_random_string_minimum_256_bits
```

**Key Files:**

```
config/jwt/private.pem
config/jwt/public.pem
```

**Example Private Key:**

```
-----BEGIN PRIVATE KEY-----
MIIEvQIBADANBgkqhkiG9w0BAQEFAASCBKcwggSjAgEAAoIBAQC7...
...
-----END PRIVATE KEY-----
```

### Rules

- Private key must stay secret (secure server storage)
- Public key can be distributed safely
- Never commit keys to version control
- Rotate keys periodically

---

## 8. Encryption (Optional - JWE)

### Standard

JSON Web Encryption (JWE)

### Structure

```
header.encrypted_key.iv.ciphertext.tag
```

### Properties

- Payload is encrypted and cannot be read
- Decryption key required to access payload

### Important

Most systems use **signed tokens only** (JWS), not encrypted tokens (JWE).

Encryption adds complexity and size. Use it only if payload confidentiality is critical.

---

## 9. Typical Authentication Flow

### Login

1. User sends credentials (username/password)
2. Server validates credentials against database
3. Server creates JWT with user info
4. Server signs JWT with secret key
5. Server returns JWT to client

### Subsequent Requests

1. Client stores JWT (localStorage, sessionStorage, or secure cookie)
2. Client sends JWT in Authorization header:

```
   Authorization: Bearer <JWT>
```

3. Server receives request
4. Server decodes and verifies JWT signature
5. If valid, request is processed
6. If invalid, request is rejected (401 Unauthorized)

### No Server-Side Session Storage Needed

The token itself proves authentication—no database lookup required for every request.

---

## 10. Token Expiration & Refresh

### Access Tokens (Short-Lived)

```json
{
  "user_id": 42,
  "exp": 1710000000 // Expires in 15 minutes
}
```

### Refresh Tokens (Long-Lived)

```json
{
  "user_id": 42,
  "exp": 1720000000 // Expires in 7 days
}
```

### Flow

```
1. User logs in → Receives access token + refresh token
2. Access token expires → Use refresh token to get new access token
3. Refresh token expires → User must re-authenticate
```

**Benefits:**

- Access tokens can be short-lived (more secure)
- Refresh tokens allow extended sessions
- Compromised access token has limited damage window

---

## 11. What JWT Protects vs Doesn't Protect

### JWT PROTECTS

- ✅ Token integrity (detects tampering)
- ✅ Token authenticity (proves who created it)
- ✅ Data consistency

### JWT DOES NOT PROTECT

- ❌ Payload confidentiality (data is visible)
- ❌ Token theft (if stolen, attacker can use it)
- ❌ Replay attacks (token can be reused until expiration)
- ❌ Man-in-the-middle attacks (without HTTPS)

### Best Practices

- Use HTTPS always
- Set short expiration times
- Store tokens securely (avoid localStorage if possible)
- Implement token revocation/blacklist
- Validate expiration on every request
- Use secure cookies with HttpOnly flag
- Implement refresh token rotation

---

## 12. JWT vs Session-Based Authentication

| Feature         | JWT                          | Session        |
| --------------- | ---------------------------- | -------------- |
| Storage         | Client-side                  | Server-side    |
| Stateless       | Yes                          | No             |
| Scalability     | High                         | Lower          |
| Revocation      | Difficult (until expiration) | Easy           |
| Size            | Larger                       | Smaller        |
| CORS-friendly   | Yes                          | Requires setup |
| Database lookup | Not needed                   | Every request  |

---

## 13. Token Revocation (Blacklist)

Since tokens cannot be revoked before expiration, implement a blacklist:

### Implementation

1. When user logs out → Add token to blacklist
2. On each request → Check if token is in blacklist
3. After expiration → Remove token from blacklist

### Storage Options

- In-memory cache (fast, lost on restart)
- Redis (fast, persistent)
- Database (slower, persistent)

---

## 14. Where to Send JWT

### Authorization Header (Recommended)

```
Authorization: Bearer eyJhbGciOiJIUzI1NiIs...
```

### Query Parameter (Less Secure)

```
GET /api/products?token=eyJhbGciOiJIUzI1NiIs...
```

Avoid—token appears in logs and browser history.

### Cookie

```
Set-Cookie: jwt=eyJhbGciOiJIUzI1NiIs...; HttpOnly; Secure
```

**HttpOnly flag:** JavaScript cannot access (prevents XSS attacks)  
**Secure flag:** Only sent over HTTPS

---

## 15. Common Use Cases

- **Authentication** - Login/logout
- **Authorization** - Role-based access control (RBAC)
- **Information Exchange** - Secure data transfer between services
- **Mobile Apps** - Stateless API authentication
- **Single Sign-On (SSO)** - Cross-domain authentication
- **Microservices** - Inter-service communication
- **API Access** - Third-party API tokens

---

## 16. JWT vs OAuth

**JWT** is a token format (how data is structured)  
**OAuth** is an authorization protocol (how authentication works)

OAuth often uses JWT tokens. They're complementary, not competing standards.

---

## 17. Key Mental Model

Think of JWT as:

```
DATA + CRYPTOGRAPHIC SIGNATURE
```

Like a document with a tamper-proof seal:

- Anyone can read the document (Base64 is not encryption)
- But if someone modifies it, the seal breaks
- Server verifies the seal (signature) on every request

---

## 18. Security Summary

### What Makes JWT Secure

- Cryptographic signing prevents tampering
- Secret key is never transmitted
- Server can verify authenticity without a database

### What You Must Do

- Use HTTPS always
- Protect secret keys
- Set short expiration times
- Implement token refresh
- Store tokens securely
- Validate signatures on every request
- Use strong algorithms (RS256 or HS256 minimum)

---

## 19. Key Insights

1. **JWT payload is not encrypted by default** — Anyone with the token can read it
2. **Encoding is not security** — Base64URL is just formatting
3. **Security comes from the signature** — The cryptographic proof
4. **Keys are used only for signing/verification** — Not included in the JWT
5. **Tokens can't be revoked before expiration** — Implement blacklist if needed
6. **Stateless authentication** — No session storage needed on server
7. **Payload is self-contained** — All user info is in the token

---

**JWT is powerful for stateless, scalable authentication. Use it correctly by understanding what it protects and what it doesn't.**
