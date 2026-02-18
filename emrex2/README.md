# EMREX 2.0 Protocol: Technical Specification

## Table of Contents

1. [Introduction](#introduction)
2. [Architecture](#architecture)
3. [Protocol Flow](#protocol-flow)
    - [1. Institution Selection](#1-institution-selection)
    - [2. Authorization Request with PKCE (RFC 7636)](#2-authorization-request-with-pkce-rfc-7636)
    - [3. Client Verification (RFC 7523)](#3-client-verification-rfc-7523)
    - [4. User Authentication & Consent](#4-user-authentication--consent)
    - [5. Token Exchange (RFC 6749)](#5-token-exchange-rfc-6749)
    - [6. Data Retrieval with Signed Responses](#6-data-retrieval-with-signed-responses)
4. [Security Mechanisms](#security-mechanisms)
5. [Data Formats](#data-formats)
6. [Endpoints](#endpoints)
7. [Error Handling](#error-handling)
8. [Compliance](#compliance)
9. [Migration from EMREX 1.0](#migration-from-emrex-10)
10. [Testing and Demo](#testing-and-demo)

---

## Introduction

EMREX 2.0 is a **secure, OAuth2-based protocol** for exchanging educational results (e.g., grades, diplomas) between
institutions. It replaces the legacy EMREX 1.0 POST/POST flow with a **modern Authorization Code Flow + PKCE**,
ensuring:

- **End-to-end security** (RFC 6749, RFC 7636, RFC 7523).
- **Data integrity** via signed responses (RFC 7515).
- **GDPR compliance** through granular consent.
- **Interoperability** via standardized data formats (ELMO XML/JSON, PDF).

### Key Components

| Component   | Role                                                       | Standards Used                     |
|-------------|------------------------------------------------------------|------------------------------------|
| **EMC**     | EMREX Client (home institution). Initiates the flow.       | OAuth2, PKCE, JWT                  |
| **EMP**     | EMREX Provider (host institution). Hosts the results.      | OAuth2, JWT, RFC 7515 (signatures) |
| **EMREG**   | Central registry. Stores EMP/EMC metadata and public keys. | JSON, HTTPS                        |
| **Student** | End-user. Selects results and grants consent.              | OAuth2 Redirect Flow               |

---

## Architecture

EMREX 2.0 follows a **decentralized architecture** with three core components:

1. **EMC (EMREX Client)**
    - **Frontend**: UI for institution selection and result review.
    - **Backend**: Handles OAuth2 flow, token exchange, and signature verification.
    - **New in 2.0**:
        - PKCE (RFC 7636) for code interception protection.
        - JWT client authentication (RFC 7523) instead of shared secrets.
        - Signature verification for responses.

2. **EMP (EMREX Provider)**
    - **Frontend**: Student login and result selection UI.
    - **Backend**: OAuth2 Authorization Server + Resource Server.
    - **Resource Server**: Provides signed results in multiple formats.
    - **New in 2.0**:
        - Dynamic client registration via EMREG.
        - Signed responses (RFC 7515) for all data exchanges.

3. **EMREG (Registry)**
    - Central directory for:
        - EMP/EMC metadata (e.g., `redirect_uris`, public keys).
        - Supported data formats and endpoints.
    - **Accessible via REST API** (JSON responses).

---

## Protocol Flow

### 1. Institution Selection

**Purpose**: Student selects the host institution (EMP) from a list.

#### Sequence:

1. Student logs into **EMC Frontend**.
2. **EMC Backend** fetches available EMPs from **EMREG**:
   ```http
   GET /emp-list HTTP/1.1
   Host: emreg.eu
   Authorization: Bearer <EMC_API_TOKEN>
   ```
3. **EMREG** responds with a list of EMPs (JSON):
   ```json
   {
     "emp_list": [
       {
         "acronym": "DUO-NL",
         "country": "NL",
         "institutions": ["University of Amsterdam", "TU Delft"],
         "authorization_url": "https://emp.nl/oauth2/authorize",
         "token_url": "https://emp.nl/oauth2/token",
         "public_key": "-----BEGIN PUBLIC KEY-----..."
       }
     ]
   }
   ```
4. **EMC Frontend** displays the list; student selects an EMP.

#### Standards:

- **RFC 8259**: JSON format for EMP metadata.
- **HTTPS**: Mandatory for all EMREG communications.

---

### 2. Authorization Request with PKCE (RFC 7636)

**Purpose**: Initiate a secure OAuth2 flow with PKCE to prevent code interception.

#### Sequence:

1. **EMC Backend** generates a **PKCE code verifier** and **challenge**:
   ```java
   code_verifier = BASE64URL(SHA256(random(32)));  // RFC 7636
   code_challenge = BASE64URL(SHA256(code_verifier));
   ```
    - `code_verifier` is stored in the session.
    - `code_challenge` is sent to the EMP.

2. **EMC Backend** prepares the authorization URL with:
    - OAuth2 parameters (RFC 6749):
        - `response_type=code`
        - `client_id` (e.g., `emc-nl-uva`).
        - `redirect_uri` (must match EMREG registration).
        - `state` (CSRF protection).
    - PKCE parameters (RFC 7636):
        - `code_challenge` (SHA-256).
        - `code_challenge_method=S256`.

3. **EMC Frontend** redirects the student to the **EMP Authorization Endpoint**:
   ```http
   HTTP/1.1 302 Found
   Location: https://emp.nl/oauth2/authorize?
     response_type=code &
     client_id=emc-nl-uva &
     redirect_uri=https%3A%2F%2Femc.uva.nl%2Fcallback &
     state=xYz123abc456def789 &
     code_challenge=E9Melv6jU2FjOq4A2TesOYsX9jpwcKQV86Z5HnXKhtI &
     code_challenge_method=S256
   ```

#### Standards:

- **RFC 6749**: OAuth2 Authorization Code Flow.
- **RFC 7636**: PKCE for public clients.
- **RFC 6750**: `state` parameter for CSRF protection.

---

### 3. Client Verification (RFC 7523)

**Purpose**: Verify the EMC’s identity using JWT assertions.

#### Sequence:

1. **EMP Backend** fetches the **EMC’s metadata** from **EMREG**:
   ```http
   GET /clients/emc-nl-uva HTTP/1.1
   Host: emreg.eu
   Authorization: Bearer <EMP_API_TOKEN>
   ```
2. **EMREG** responds with:
    - EMC’s **public key** (for JWT validation).
    - Registered `redirect_uris`.
      Example:
   ```json
   {
     "client_id": "emc-nl-uva",
     "redirect_uris": ["https://emc.uva.nl/callback"],
     "public_key": "-----BEGIN PUBLIC KEY-----..."
   }
   ```
3. **EMP Backend** validates:
    - The `redirect_uri` matches the registered URI.
    - The EMC’s identity via `client_id`.

#### Standards:

- **RFC 7523**: JWT-based client authentication.
- **RFC 7515**: JWT signature validation.

---

### 4. User Authentication & Consent

**Purpose**: Authenticate the student and obtain explicit consent for data sharing.

#### Sequence:

1. **EMP Frontend** prompts the student to log in (e.g., SAML, local credentials).
2. After authentication, **EMP Backend** retrieves the student’s results.
3. **EMP Frontend** displays results for selection (e.g., courses, grades).
4. Student selects results and grants consent.
5. **EMP Backend** stores the selection with a **reference ID** (`ref_id`):
   ```json
   {
     "ref_id": "a1b2c3d4-5678-90ef-ghij-klmnopqrstuv",
     "student_id": "s123456",
     "selected_results": ["course1", "course2"],
     "consent_granted": true,
     "consent_timestamp": "2026-02-18T12:00:00Z"
   }
   ```
6. **EMP Backend** generates an **authorization code** bound to:
    - `ref_id`, `client_id`, `redirect_uri`, and `code_challenge`.
7. **EMP Frontend** redirects the student back to the **EMC** with the code:
   ```http
   HTTP/1.1 302 Found
   Location: https://emc.uva.nl/callback?
     code=AUTH_CODE_123 &
     state=xYz123abc456def789
   ```

---

### 5. Token Exchange (RFC 6749)

**Purpose**: Exchange the authorization code for an access token using JWT client authentication.

#### Sequence:

1. **EMC Backend** validates the callback (`state`, `code`).
2. Generates a **signed JWT** (RFC 7523) for client authentication:
    - **Header**:
      ```json
      {
        "alg": "RS256",
        "typ": "JWT",
        "kid": "emc-key-1"
      }
      ```
    - **Payload**:
      ```json
      {
        "iss": "emc-nl-uva",
        "sub": "emc-nl-uva",
        "aud": "https://emp.nl/oauth2/token",
        "jti": "unique-id-123",
        "iat": 1672531200,
        "exp": 1672531500,
        "nbf": 1672531200
      }
      ```
3. Sends a **token request** to the **EMP Token Endpoint**:
   ```http
   POST /oauth2/token HTTP/1.1
   Host: emp.nl
   Content-Type: application/x-www-form-urlencoded

   grant_type=authorization_code &
   code=AUTH_CODE_123 &
   redirect_uri=https%3A%2F%2Femc.uva.nl%2Fcallback &
   code_verifier=CODE_VERIFIER &
   client_assertion_type=urn:ietf:params:oauth:client-assertion-type:jwt-bearer &
   client_assertion=SIGNED_JWT
   ```
4. **EMP Backend**:
    - Validates the JWT signature (using EMC’s public key from EMREG).
    - Validates `code_verifier` against `code_challenge` (PKCE).
    - Returns an **access token** (short-lived, ≤10 minutes):
      ```json
      {
        "access_token": "ACCESS_TOKEN_456",
        "token_type": "Bearer",
        "expires_in": 300,
        "scope": "elmo userinfo"
      }
      ```

#### Standards:

- **RFC 6749**: OAuth2 Token Exchange.
- **RFC 7523**: JWT client authentication.
- **RFC 7636**: PKCE validation.

---

### 6. Data Retrieval with Signed Responses

**Purpose**: Fetch results in a secure, signed format.

#### Sequence:

1. **EMC Backend** requests data using the `access_token`:
   ```http
   GET /results?data_format=elmo&data_format_version=2.1 HTTP/1.1
   Host: emp.nl
   Authorization: Bearer ACCESS_TOKEN_456
   X-Request-ID: req-12345
   ```
    - Supported `data_format` values:
      | Format | Content-Type | Description |
      |----------------------|--------------------|----------------------------------------------|
      | `elmo`               | `application/xml`  | ELMO XML (default, based on EN 15981/15982). |
      | `elm`                | `application/json` | JSON alternative to ELMO. |
      | `userinfo`           | `application/xml`  | Student identification (ELMO Learner). |
      | `eidas`              | `application/json` | eIDAS-compliant user info. |
      | `pdf_metadata`       | `application/json` | List of available PDFs (e.g., diplomas). |
      | `pdf`                | `application/pdf`  | PDF document (requires `pdf_id`). |

2. **EMP Resource Server**:
    - Validates the `access_token` (scope, expiration, `ref_id` binding).
    - Signs the response with its private key:
      ```http
      HTTP/1.1 200 OK
      Content-Type: application/xml
      X-Signature: EMP_SIGNATURE_ABC123
      X-Signature-Algorithm: RS256
      X-Request-ID: req-12345
 
      <elmo>
        <learner>
          <id>s123456</id>
          <name>Jane Doe</name>
        </learner>
        ...
      </elmo>
      ```
    - Returns `HTTP 204 No Content` if no results are available.

3. **EMC Backend**:
    - Verifies the `X-Signature` using the EMP’s public key (from EMREG).
    - Stores the results and notifies the student.

#### Standards:

- **RFC 7515**: JSON Web Signature (JWS) for response signing.
- **EN 15981/15982**: ELMO XML format.

---

## Security Mechanisms

| **Mechanism**                  | **Purpose**                                                       | **Standard**  |
|--------------------------------|-------------------------------------------------------------------|---------------|
| OAuth2 Authorization Code Flow | Prevents token exposure in the frontend.                          | RFC 6749      |
| PKCE                           | Mitigates authorization code interception (RFC 7636 §4.6).        | RFC 7636      |
| JWT Client Authentication      | Secures server-to-server communication (replaces client secrets). | RFC 7523      |
| Short-lived Tokens             | Access tokens expire in ≤10 minutes.                              | RFC 6749 §5.1 |
| Signed Responses               | Ensures data integrity and authenticity (RFC 7515 §3).            | RFC 7515      |
| CSRF Protection (`state`)      | Prevents cross-site request forgery (RFC 6749 §10.12).            | RFC 6749      |
| EMREG Validation               | EMP verifies EMC’s `redirect_uri` and public key.                 | HTTPS + JSON  |
| Granular Consent               | Student selects specific results to share (GDPR compliance).      | GDPR Art. 6   |

---

## Example Data Formats

### ELMO XML

- **Schema**: [ELMO XSD (v2.1.2)](https://github.com/emrex-eu/elmo-schemas/tree/v2.1.2)
- **Description**: XML format based on **EN 15981/15982** for educational records (e.g., courses, grades, diplomas).
  ```

### ELM

- **Schemas**: [ELM Schemas ](https://github.com/european-commission-empl/European-Learning-Model)
- **Description**: The European Learning Model (ELM) is a Data Model for Interoperability of Learning Opportunities,
  Qualifications, Accreditation and Credentials in Europe, developed by the European Commission.

### PDF Metadata

- **Schemas**: TBD
- **Description**: JSON list of available PDF documents (e.g., diplomas, transcripts) with metadata such as title, issue
  date, type, size, and checksum.
- **Example**:
  ```json
  {
    "pdf_list": [
      {
        "pdf_id": "123-456-789",
        "title": "Bachelor's Diploma in Computer Science",
        "issue_date": "2023-07-15",
        "type": "diploma",
        "size_bytes": 102400,
        "checksum": "sha256:abc123..."
      }
    ]
  }
  ```

### PDF Document

- **Description**: Binary PDF file (e.g., diploma, transcript) with a detached signature in the `X-Signature` header.
- **Headers**:
  | Header | Value Example | Description |
  |-------------------|-----------------------------------|--------------------------------------|
  | `Content-Type`    | `application/pdf`                 | MIME type. |
  | `X-Signature`     | `EMP_SIGNATURE_ABC123`            | Base64-encoded signature (RFC 7515). |
  | `X-Signature-Alg` | `RS256`                           | Signature algorithm. |
- **Example Request**:
  ```http
  GET /results?data_format=pdf&pdf_id=123-456-789 HTTP/1.1
  Host: emp.com
  Authorization: Bearer ACCESS_TOKEN_456
  ```

### Userinfo

- **Description**: XML format based on **ELMO Learner** for student identification.
- **Schema**: [ELMO Learner XSD](https://github.com/emrex-eu/elmo-schemas/tree/v2.1.2)
- **Example**:
  ```xml
  <learner>
    <person>
      <family_name>Doe</family_name>
      <given_name>John</given_name>
      <birthdate>2000-01-01</birthdate>
    </person>
  </learner>
  ```

### EIDAS UserInfo

- **Description**: JSON format compliant with **eIDAS Regulation** for electronic identification, including attributes
  such as `sub`, `family_name`, `given_name`, `birthdate`, and eIDAS-specific metadata.
- **Schema**: [eIDAS Attributes](https://ec.europa.eu/digital-building-blocks/wikis/display/DIGITAL/eIDAS+Attributes)
- **Example**:
  ```json
  {
    "sub": "eu.eidas.naturalperson.123456789",
    "family_name": "Doe",
    "given_name": "Jane",
    "birthdate": "1995-05-15",
    "person_identifier": "NL/BSN/123456789",
    "eidas": {
      "level_of_assurance": "substantial",
      "issuer": "https://eidas.nl/identity-provider"
    }
  }
  ```

---

## Endpoints

| **Component**       | **Endpoint**           | **Method** | **Description**                           | **Authentication**          |
|---------------------|------------------------|------------|-------------------------------------------|-----------------------------|
| EMREG               | `/emp-list`            | GET        | List of available EMPs.                   | Bearer Token (EMC)          |
| EMREG               | `/clients/{client_id}` | GET        | EMC metadata (public key, redirect URIs). | Bearer Token (EMP)          |
| EMP                 | `/oauth2/authorize`    | GET        | Authorization endpoint (OAuth2).          | None (user login required)  |
| EMP                 | `/oauth2/token`        | POST       | Token endpoint (OAuth2).                  | JWT Assertion (RFC 7523)    |
| EMP Resource Server | `/results`             | GET        | Fetch results in selected format.         | Bearer Token (access_token) |

---

## Error Handling

| **Scenario**                      | **HTTP Status** | **Response Body**                                                                 | **Recovery**                             |
|-----------------------------------|-----------------|-----------------------------------------------------------------------------------|------------------------------------------|
| Invalid `client_id`               | 400             | `{"error": "invalid_client"}`                                                     | Register EMC in EMREG.                   |
| Mismatched `redirect_uri`         | 400             | `{"error": "invalid_request", "error_description": "redirect_uri mismatch"}`      | Update `redirect_uri` in EMREG.          |
| Expired `code`                    | 400             | `{"error": "invalid_grant", "error_description": "code expired"}`                 | Restart the flow.                        |
| Invalid `code_verifier` (PKCE)    | 400             | `{"error": "invalid_grant", "error_description": "PKCE verification failed"}`     | Restart the flow.                        |
| Invalid JWT signature             | 401             | `{"error": "invalid_client", "error_description": "JWT signature invalid"}`       | Regenerate JWT with correct private key. |
| Unsupported `data_format/version` | 400             | `{"error": "invalid_request", "error_description": "unsupported format/version"}` | See Emreg for EMP supported data formats |
| No results available              | 204             | (Empty body)                                                                      | Notify the student.                      |
| Token expired                     | 401             | `{"error": "invalid_token", "error_description": "token expired"}`                | Request a new token.                     |
| Signature verification failed     | 403             | `{"error": "invalid_signature"}`                                                  | Verify EMP’s public key in EMREG.        |

---

## Compliance

EMREX 2.0 complies with the following standards:

| **Category**         | **Standard**      | **Implementation Details**                                             |
|----------------------|-------------------|------------------------------------------------------------------------|
| **Authentication**   | RFC 6749 (OAuth2) | Authorization Code Flow + PKCE.                                        |
|                      | RFC 7523 (JWT)    | JWT client authentication (RS256).                                     |
| **Security**         | RFC 7636 (PKCE)   | `code_challenge` with `S256`.                                          |
|                      | RFC 7515 (JWS)    | Signed responses (RS256).                                              |
|                      | RFC 6749 §10.12   | CSRF protection via `state` parameter.                                 |
| **Data Formats**     | EN 15981/15982    | ELMO XML format.                                                       |
|                      | RFC 8259 (JSON)   | ELM JSON and metadata responses.                                       |
| **Privacy**          | GDPR Article 6    | Granular consent + explicit user approval.                             |
| **Interoperability** | HTTPS (RFC 2818)  | Mandatory for all communications.                                      |
| **eIDAS**            | eIDAS Regulation  | Support for eIDAS-compliant user identification (`data_format=eidas`). |

---