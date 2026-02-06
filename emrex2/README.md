# EMREX 2.0 - OAuth2 iGov Compliant Flow

## Table of Contents

1. [Introduction](#introduction)
2. [Architecture](#architecture)
3. [Technical Flow](#technical-flow)
    - [1. Institution Selection](#1-institution-selection)
    - [2. Authorization Request](#2-authorization-request)
    - [3. Client Verification](#3-client-verification)
    - [4. User Authentication & Consent](#4-user-authentication--consent)
    - [5. Token Exchange](#5-token-exchange)
    - [6. Data Retrieval](#6-data-retrieval)
4. [Security Features](#security-features)
5. [Data Formats](#data-formats)
6. [Implementation Details](#implementation-details)
7. [Compliance](#compliance)
8. [Migration from EMREX 1.0](#migration-from-emrex-10)
9. [Testing](#testing)

---

## Introduction

EMREX 2.0 implements the **OAuth2 Authorization Code Flow with PKCE** as defined in the iGov specifications for secure
exchange of educational results between institutions.

Key features:

- OAuth2 Authorization Code Flow with PKCE (RFC 7636)
- JWT-based client authentication (RFC 7523)
- Signed responses for all data exchanges
- Standardized data formats (ELMO XML, ELM JSON, PDF)
- Granular user consent management

---

## Architecture

### Components

### 1. EMC (EMREX Client)

- Frontend: Institution selection UI
- Backend: OAuth2 flow handler
- New features:
    - PKCE implementation (iGov 5.2)
    - JWT generation for client authentication (RFC 7523)
    - Response signature verification

### 2. EMP (EMREX Provider)

- Frontend: Authentication and result selection interface
- Backend: OAuth2 Authorization Server
- Resource Server: Provides signed educational data
- New features:
    - JWT validation for client authentication
    - Signed response generation (iGov 8.2)

### 3. EMREG (Registry)

- Central metadata repository
- Public key distribution for JWT validation
- OAuth2 configuration endpoints

---

## Technical Flow

### 1. Institution Selection

**Sequence:**

1. Student initiates transfer in EMC
2. EMC requests available EMPs from EMREG
3. EMREG returns EMP metadata (iGov 4.1.1)
4. EMC displays selection to student

### 2. Authorization Request

**Required Parameters:**

| Parameter             | Example Value                                          | Purpose                                          | iGov Ref. |
|-----------------------|--------------------------------------------------------|--------------------------------------------------|-----------|
| response_type         | code                                                   | Authorization code flow                          | 5.1       |
| client_id             | emc-nl-university1                                     | EMC client identifier                            | 5.1.1     |
| redirect_uri          | https://emc.university.nl/callback                     | EMC callback URL                                 | 5.1.2     |
| ~~scope~~             | ~~elmo~~                                               | ~~Requested scope~~                              | ~~5.1.3~~ |
| state                 | xYz123abc456def789                                     | CSRF protection                                  | 5.1.4     |
| code_challenge        | E9Melv6jU2FjOq4A2TesOYsX9jpwcKQV86Z5HnXKhtI            | PKCE challenge                                   | 5.2       |
| code_challenge_method | S256                                                   | PKCE method                                      | 5.2       |
| client_assertion_type | urn:ietf:params:oauth:client-assertion-type:jwt-bearer | JWT assertion type                               | 5.5       |
| client_assertion      | [JWT]                                                  | Signed JWT assertion: DPoP (Proof-of-Possession) | 5.6.2     |

**PKCE Preparation (iGov 5.2):**

```
code_verifier = BASE64URL(SHA256(random(32)))
code_challenge = BASE64URL(SHA256(code_verifier))
```

**JWT Assertion Structure (RFC 7523):**

```json
{
  "alg": "RS256",
  "typ": "JWT",
  "kid": "EMC_KEY_ID"
}
```

```json
{
  "iss": "EMC_CLIENT_ID",
  "sub": "EMC_CLIENT_ID",
  "aud": "EMP_TOKEN_URL",
  "jti": "unique-id-123",
  "iat": 1672531200,
  "exp": 1672531500,
  "nbf": 1672531200
}
```

### 3. Client Verification

**Validation Steps:**

1. EMP fetches client metadata from EMREG
2. Verifies JWT signature with EMC public key
3. Validates JWT claims:
    - iss matches client_id
    - aud matches token endpoint
    - exp/nbf valid
    - jti is unique

### 4. User Authentication & Consent

**Process:**

1. Student authenticates at EMP
2. EMP validates user credentials
3. EMP retrieves student results
4. Student selects results for transfer
5. EMP stores selection with reference ID
6. EMP generates authorization code bound to selection (iGov 6.2)

### 5. Token Exchange

**Token Request:**

```
POST /token HTTP/1.1
Host: emp.sweden.edu
Content-Type: application/x-www-form-urlencoded

grant_type=authorization_code
&code=AUTH_CODE_123
&redirect_uri=https%3A%2F%2Femc.university.nl%2Fcallback ?
&client_id=emc-nl-university1 ?
&code_verifier=CODE_VERIFIER
&client_assertion_type=urn%3Aietf%3Aparams%3Aoauth%3Aclient-assertion-type%3Ajwt-bearer
&client_assertion=NEW_JWT
```

**Token Response:**

```
{
"access_token": "ACCESS_TOKEN_456",
"token_type": "Bearer",
"expires_in": 60
}
```

### 6. Data Retrieval

**Available Endpoints:**

| Endpoint        | Content Type              | Response Headers                      |
|-----------------|---------------------------|---------------------------------------|
| /results        | emrex/elmo                | X-Signature, X-Certificate-Thumbprint |
| /results        | emrex/PDF                 | X-Signature, X-Certificate-Thumbprint |
| /results        | emrex/elm                 | X-Signature, X-Certificate-Thumbprint |
| /identification | emrex/user_identification | X-Signature, X-Certificate-Thumbprint |

**Response Headers:**

```
X-Signature: [base64-encoded signature]
X-Signature-Algorithm: RS256
X-Request-ID: [unique request identifier]
```

---

## Security Features

### Authentication & Authorization

- **PKCE**: Protection against code interception (RFC 7636)
- **JWT Assertions**: Signed client authentication (RFC 7523)
- **State Parameters**: CSRF protection
- **Short-lived Tokens**: Default 1-minute expiration

### Data Integrity

- All responses signed with EMP private key
- Token binding to specific results
- Certificate thumbprint in headers for validation ?

### Validation Rules

1. All required OAuth2 parameters must be present
2. JWT must be correctly signed with EMC private key
3. redirect_uri must exactly match registered URI
4. code_challenge must be correctly generated (S256)
5. State must be unique per request
6. Tokens must be validated before use

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
        <!-- Educational results data -->
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
  ]
}
```

---

## Implementation Details

### JWT Claims Requirements

```json
{
  "iss": "EMC_CLIENT_ID",
  "sub": "EMC_CLIENT_ID",
  "aud": "EMP_TOKEN_URL",
  "jti": "unique-id-123",
  "iat": 1672531200,
  "exp": 1672531500,
  "nbf": 1672531200
}
```

### Authorization Code Generation

- Code is bound to:
    - Selected results (reference ID)
    - Client ID
    - Redirect URI
    - Code verifier (for PKCE validation)

---

## Compliance

EMREX 2.0 implements the following standards and specifications:

| Standard          | Implementation Details           |
|-------------------|----------------------------------|
| RFC 6749 (OAuth2) | Authorization Code Flow          |
| RFC 7636          | PKCE implementation              |
| RFC 7523          | JWT client authentication        |
| iGov 4.1.1        | EMP metadata format              |
| iGov 5.1-5.7      | Authorization request parameters |
| iGov 5.2          | PKCE implementation              |
| iGov 5.5-5.6      | JWT client authentication        |
| iGov 6.1-6.3      | User authentication and consent  |
| iGov 7.1-7.3      | Token exchange flow              |
| iGov 8.1-8.2      | Signed response requirements     |

---
