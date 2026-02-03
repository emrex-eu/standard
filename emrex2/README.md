# EMREX 2.0 - OpenID iGov Compliant Flow

## Table of Contents
1. [Introduction](#introduction)
2. [Architecture](#architecture)
3. [Technical Flow](#technical-flow)
    - [1. Institution Selection](#1-institution-selection)
    - [2. Authorization Request Preparation](#2-authorization-request-preparation)
    - [3. EMP Client Verification](#3-emp-client-verification)
    - [4. User Authentication & Consent](#4-user-authentication--consent)
    - [5. Token Exchange](#5-token-exchange)
    - [6. Data Retrieval](#6-data-retrieval)
4. [Security Features](#security-features)
5. [Data Formats](#data-formats)
6. [Implementation Details](#implementation-details)
7. [Compliance](#compliance)
8. [Migration from EMREX 1.0](#migration-from-emrex-10)
9. [Testing Instructions](#testing-instructions)

---

## Introduction

EMREX 2.0 implements the **OpenID Connect iGov profile** for secure exchange of educational results between institutions. This version introduces significant security and functionality improvements over EMREX 1.0.

Key features:
- OpenID Connect integration with PKCE
- JWT-based client authentication
- Enhanced security with signed responses
- Standardized data formats (ELMO XML, ELM JSON, PDF)
- User consent management with granular result selection

---

## Architecture

The EMREX 2.0 architecture consists of three main components:

### 1. EMC (EMREX Client)
- **Frontend**: User interface for institution selection
- **Backend**:
    - Handles OpenID flow
    - Token exchange
    - Data retrieval
    - New features:
        - PKCE implementation (iGov 5.2)
        - JWT generation and signing (iGov 5.6.2)
        - Signed response verification

### 2. EMP (EMREX Contact Point)
- **Frontend**: Authentication and result selection interface
- **Backend**: Authorization server and resource server
- **Resource Server**: Provides signed educational data
- New features:
    - OpenID Provider capabilities
    - JWT validation (iGov 5.6.3)
    - Signed response generation (iGov 8.2)

### 3. EMREG (EMREX Registry)
- Central registry for EMP metadata
- Provides public keys for JWT validation
- Extended with OpenID configuration endpoints

---

## Technical Flow

### 1. Institution Selection

**Sequence:**
1. Student initiates transfer in EMC
2. EMC fetches list of available EMPs from EMREG
3. EMC displays selection to student

**Key Points:**
- EMP metadata follows iGov 4.1.1 specification
- Selection is presented to student for choice

### 2. Authorization Request Preparation

**Required Parameters:**


| Parameter                     | Example Value                          | Purpose                          | iGov Ref. |
|-------------------------------|----------------------------------------|----------------------------------|-----------|
| response_type                 | code                                   | Authorization code flow          | 5.1       |
| client_id                     | emc-nl-university1                     | EMC client identifier            | 5.1.1     |
| redirect_uri                  | https://emc.university.nl/callback     | EMC callback URL                 | 5.1.2     |
| scope                         | openid elmo                            | Requested scopes                 | 5.1.3     |
| state                         | xYz123abc456def789                     | CSRF protection                  | 5.1.4     |
| nonce                         | nOnCe1234567890                        | Replay attack protection         | 5.1.5     |
| code_challenge                | E9Melv6jU2FjOq4A2TesOYsX9jpwcKQV86Z5HnXKhtI | PKCE challenge            | 5.2       |
| code_challenge_method         | S256                                   | PKCE method                      | 5.2       |
| client_assertion_type         | urn:ietf:params:oauth:client-assertion-type:jwt-bearer | JWT assertion type | 5.5       |
| client_assertion              | [JWT]                                  | Signed JWT assertion             | 5.6.2     |


**PKCE Preparation (iGov 5.2):**
```
code_verifier = BASE64URL(SHA256(random(32)))
code_challenge = BASE64URL(SHA256(code_verifier))
```

**JWT Assertion Structure (iGov 5.6.2):**
```json
Header:
{
"alg": "RS256",
"typ": "JWT",
"kid": "EMC_KEY_ID"
}

Payload:
{
"iss": "EMC_CLIENT_ID",
"sub": "EMC_CLIENT_ID",
"aud": "EMP_TOKEN_URL",
"jti": "UUID()",
"iat": current_timestamp,
"exp": current_timestamp + 300,
"nbf": current_timestamp,
"azp": "EMC_CLIENT_ID"
}
```

### 3. EMP Client Verification

**Validation Steps:**
1. EMP fetches client metadata from EMREG
2. Verifies JWT signature with EMC public key
3. Validates JWT claims according to iGov 5.6.3

**JWT Validation Rules:**
```
- iss == client_id
- aud == EMP token URL
- exp > current time
- nbf <= current time
- azp == client_id
```

### 4. User Authentication & Consent

**Process:**
1. Student submits credentials to EMP
2. EMP validates user (iGov 6.1)
3. EMP retrieves student results
4. Student selects results for transfer
5. EMP stores selection with reference ID
6. EMP generates authorization code bound to selection (iGov 6.2)

### 5. Token Exchange

**Token Request (iGov 7.1):**
```http
POST /token HTTP/1.1
Host: emp.sweden.edu
Content-Type: application/x-www-form-urlencoded

grant_type=authorization_code &
code=AUTH_CODE_123 &
redirect_uri=https%3A%2F%2Femc.university.nl%2Fcallback &
code_verifier=CODE_VERIFIER &
client_assertion_type=urn%3Aietf%3Aparams%3Aoauth%3Aclient-assertion-type%3Ajwt-bearer &
client_assertion=NEW_JWT
```

**Token Validation (iGov 7.2):**
1. Verify JWT signature with EMC public key
2. Validate code_verifier against stored code_challenge
3. Generate access token and ID token

### 6. Data Retrieval

**Available Endpoints:**


| Endpoint          | Format | Description                     |
|-------------------|--------|---------------------------------|
| /results/elmo     | XML    | ELMO format (iGov 8.1)          |
| /results/pdf      | PDF    | PDF document                    |
| /results/elm      | JSON   | ELM format                      |


**Response Headers:**
```
X-Signature: [base64-encoded signature]
X-Signature-Algorithm: RS256
X-Certificate-Thumbprint: [SHA-256 thumbprint of EMP certificate]
X-Request-ID: [unique request identifier]
```

---
## Security Features

### Authentication & Authorization
- **PKCE**: Protection against code interception attacks
- **JWT Assertions**: Signed client authentication (iGov 5.6.2)
- **State Parameters**: CSRF protection
- **Nonce**: Replay attack prevention

### Data Integrity
- **Signed Responses**: All data responses are signed by EMP
- **Token Binding**: Access tokens bound to specific results
- **Short-lived Tokens**: Default 1-hour expiration

### Validation Rules
1. All required OpenID parameters must be present
2. JWT must be correctly signed with EMC private key
3. redirect_uri must exactly match registered URI
4. Scope must include at least "openid"
5. code_challenge must be correctly generated (S256)
6. State and nonce must be unique per request

---
## Data Formats

### ELMO XML
```xml
<elmo>
<learner>
<id>student123</id>
<name>John Doe</name>
</learner>
<issuer>
<name>University of Sweden</name>
</issuer>
<results>
<!-- Result data -->
</results>
</elmo>
```
### ELM JSON
```json
{
"learner": {
"id": "student123",
"name": "John Doe"
},
"issuer": {
"name": "University of Sweden"
},
"results": [
/* Result data */
]
}
```

### PDF Document
- Human-readable result documents
- Digitally signed for authenticity
- Includes visual representation of results

---
## Implementation Details

### JWT Claims Requirements
```json
{
"iss": "EMC_CLIENT_ID",
"sub": "EMC_CLIENT_ID",
"aud": "EMP_TOKEN_URL",
"jti": "UNIQUE_ID",
"iat": CURRENT_TIMESTAMP,
"exp": CURRENT_TIMESTAMP + 300,
"nbf": CURRENT_TIMESTAMP,
"azp": "EMC_CLIENT_ID"
}
```

### Authorization Code Generation (iGov 6.2)
- Code is bound to:
    - Selected results
    - Client ID
    - Redirect URI
    - Unique reference ID

---
## Compliance

EMREX 2.0 implements the following iGov specifications:

| Section   | Description                          |
|-----------|--------------------------------------|
| 4.1.1     | EMP metadata format                  |
| 5.1-5.7   | Authorization request parameters    |
| 5.2       | PKCE implementation                  |
| 5.6       | JWT client authentication           |
| 6.1-6.3   | User authentication and consent      |
| 7.1-7.3   | Token exchange flow                  |
| 8.1-8.2   | Signed response requirements         |
| 9.1       | User information endpoint            |


---
## Migration from EMREX 1.0

**Key Changes:**
1. **Protocol Change**:
    - From custom HTTP POST to OpenID Connect
    - Added PKCE and JWT requirements

2. **Security Enhancements**:
    - All responses now signed
    - Stronger client authentication
    - Token-based authorization

3. **Data Formats**:
    - Added ELM JSON alternative
    - Standardized PDF generation
    - Enhanced ELMO XML schema

4. **User Experience**:
    - Granular result selection
    - Explicit consent management
    - Improved error handling

---
## Testing Instructions

### Prerequisites
- Registered EMC client in EMREG
- Valid TLS certificates for all components
- Configured OpenID endpoints at EMP

### Test Cases
1. Successful authorization flow
2. PKCE validation failures
3. JWT signature verification
4. Result selection and consent
5. Signed response validation
6. Error scenarios (invalid client, expired tokens)

### Validation Tools
- OpenID Connect debuggers
- [JWT.io](https://jwt.io) for token inspection
- Certificate validation tools