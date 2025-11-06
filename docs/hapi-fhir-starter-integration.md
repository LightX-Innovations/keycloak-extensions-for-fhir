# HAPI FHIR Starter with Built-in Authorization

## Overview

The HAPI FHIR Starter Project includes **built-in authorization capabilities** that can automatically enforce scope-based access control when configured properly. This document explains how it works and how to integrate it with your Keycloak setup.

## How It Works: The Big Picture

```mermaid
sequenceDiagram
  participant App as FHIR Client App
  participant KC as Keycloak
  participant HAPI as HAPI FHIR Starter
  participant Auth as Built-in AuthorizationInterceptor
  participant DB as FHIR Database
  
  App->>KC: 1. Authenticate with scope request
  Note right of App: POST /auth/token<br/>scope=patient/Observation.read
  
  KC->>KC: 2. Validate user and scopes
  KC->>App: 3. Return access token
  Note right of KC: Token contains:<br/>- scope claim<br/>- patient_id claim<br/>- aud claim
  
  App->>HAPI: 4. FHIR API Request
  Note right of App: GET /fhir/Observation<br/>Authorization: Bearer token
  
  HAPI->>HAPI: 5. Extract Bearer token
  HAPI->>KC: 6. Validate token signature
  KC->>HAPI: 7. Token valid
  
  HAPI->>Auth: 8. Check authorization
  Auth->>Auth: 9. Parse scope claim
  Auth->>Auth: 10. Build access rules
  Note right of Auth: IF scope = patient/Observation.read<br/>THEN allow read Observation<br/>WHERE patient = patient_id
  
  Auth->>HAPI: 11. Authorization decision
  
  alt Authorized
    HAPI->>DB: 12. Query with patient filter
    DB->>HAPI: 13. Filtered results
    HAPI->>App: 14. Return FHIR Bundle
  else Denied
    HAPI->>App: 15. Return 403 Forbidden
  end
```

**Figure 1:** Complete authorization flow in HAPI FHIR Starter with Keycloak.

## The Magic: How Scopes in Tokens Become Permissions

### Step-by-Step Breakdown

#### Step 1: Client Requests Token with Scopes

When your application authenticates with Keycloak, it requests specific scopes:

```http
POST https://keycloak.example.com/auth/realms/test2/protocol/openid-connect/token
Content-Type: application/x-www-form-urlencoded

grant_type=authorization_code
&code=AUTH_CODE_HERE
&client_id=my-fhir-app
&client_secret=secret
&scope=launch/patient patient/Observation.read patient/Condition.read
```

**What happens:**

- Keycloak validates the client
- Checks if user has permission for requested scopes
- Generates access token with granted scopes

#### Step 2: Keycloak Returns Token with Scope Claim

The access token contains the granted scopes in the `scope` claim:

```json
{
  "sub": "user-123",
  "aud": "https://fhir.example.com/fhir",
  "scope": "launch/patient patient/Observation.read patient/Condition.read",
  "patient_id": "Patient/456",
  "iss": "https://keycloak.example.com/auth/realms/test2",
  "exp": 1699300000,
  "iat": 1699296400
}
```

**Key points:**

- `scope` is a **space-separated string** of granted scopes
- `patient_id` identifies which patient's data can be accessed
- `aud` ensures token is for your FHIR server

#### Step 3: Client Makes FHIR Request with Token

```http
GET https://fhir.example.com/fhir/Observation
Authorization: Bearer eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCJ9...
Accept: application/fhir+json
```

**What happens:**

- HAPI FHIR Starter extracts the Bearer token
- Passes it to the authorization interceptor

#### Step 4: HAPI FHIR Starter Validates Token

```mermaid
flowchart TB
  classDef process fill:#047857,stroke:#065f46,color:#FFFFFF,stroke-width:2px
  classDef decision fill:#fbbf24,stroke:#f59e0b,color:#000000,stroke-width:2px
  classDef success fill:#d1fae5,stroke:#34d399,color:#000000,stroke-width:2px
  classDef error fill:#b91c1c,stroke:#7f1d1d,color:#FFFFFF,stroke-width:2px
  
  Extract[Extract Bearer Token]:::process
  Decode[Decode JWT]:::process
  
  ValidSig{Valid Signature?}:::decision
  NotExpired{Not Expired?}:::decision
  CorrectAud{Correct Audience?}:::decision
  
  ParseClaims[Extract scope and patient_id claims]:::success
  Error[Return 401 Unauthorized]:::error
  
  Extract --> Decode
  Decode --> ValidSig
  ValidSig -->|No| Error
  ValidSig -->|Yes| NotExpired
  NotExpired -->|No| Error
  NotExpired -->|Yes| CorrectAud
  CorrectAud -->|No| Error
  CorrectAud -->|Yes| ParseClaims
```

**Figure 2:** Token validation process in HAPI FHIR Starter.

**Built-in validation checks:**

1. ✅ Signature verification (using Keycloak public key)
2. ✅ Expiration check (`exp` claim)
3. ✅ Audience validation (`aud` claim)
4. ✅ Issuer validation (`iss` claim)

#### Step 5: Authorization Interceptor Parses Scopes

The built-in `AuthorizationInterceptor` reads the scope claim:

```text
Input: "launch/patient patient/Observation.read patient/Condition.read"

Parse:
  ↓
["launch/patient", "patient/Observation.read", "patient/Condition.read"]
  ↓
Interpretation:
  - launch/patient: Patient context is available
  - patient/Observation.read: Can READ Observation resources
  - patient/Condition.read: Can READ Condition resources
```

#### Step 6: Build Authorization Rules

HAPI FHIR Starter converts scopes into authorization rules:

```mermaid
flowchart LR
  classDef scope fill:#dbeafe,stroke:#3b82f6,color:#000000,stroke-width:2px
  classDef rule fill:#047857,stroke:#065f46,color:#FFFFFF,stroke-width:2px
  
  Scope1[patient/Observation.read]:::scope
  Scope2[patient/Condition.read]:::scope
  PatientID[patient_id = Patient/456]:::scope
  
  Rule1[Allow READ on Observation<br/>WHERE patient = Patient/456]:::rule
  Rule2[Allow READ on Condition<br/>WHERE patient = Patient/456]:::rule
  Rule3[Deny everything else]:::rule
  
  Scope1 --> Rule1
  Scope2 --> Rule2
  PatientID --> Rule1
  PatientID --> Rule2
  
  Rule1 --> Rule3
  Rule2 --> Rule3
```

**Figure 3:** Scope to authorization rule mapping.

**Rule translation:**

| Scope | Generated Rule | Effect |
|-------|---------------|--------|
| `patient/Observation.read` | `allow().read().resourcesOfType("Observation").inCompartment("Patient", patientId)` | Can GET `/Observation` for this patient |
| `patient/Condition.read` | `allow().read().resourcesOfType("Condition").inCompartment("Patient", patientId)` | Can GET `/Condition` for this patient |
| (no matching scope) | `denyAll()` | Everything else is forbidden |

#### Step 7: Execute Query with Authorization

```mermaid
flowchart TB
  classDef start fill:#dbeafe,stroke:#3b82f6,color:#000000,stroke-width:2px
  classDef process fill:#047857,stroke:#065f46,color:#FFFFFF,stroke-width:2px
  classDef decision fill:#fbbf24,stroke:#f59e0b,color:#000000,stroke-width:2px
  classDef success fill:#d1fae5,stroke:#34d399,color:#000000,stroke-width:2px
  classDef error fill:#b91c1c,stroke:#7f1d1d,color:#FFFFFF,stroke-width:2px
  
  Request[GET /fhir/Observation]:::start
  CheckRules[Check authorization rules]:::process
  
  HasObsScope{Has patient/Observation.read?}:::decision
  AddFilter[Add patient=Patient/456 filter]:::process
  Query[Execute SQL query]:::process
  
  Results[Return filtered results]:::success
  Forbidden[Return 403 Forbidden]:::error
  
  Request --> CheckRules
  CheckRules --> HasObsScope
  
  HasObsScope -->|Yes| AddFilter
  HasObsScope -->|No| Forbidden
  
  AddFilter --> Query
  Query --> Results
```

**Figure 4:** Query execution with automatic patient filtering.

**Database query generated:**

```sql
-- Original request: GET /fhir/Observation
-- Authorization adds patient filter automatically

SELECT * FROM observation
WHERE patient_id = '456'  -- From patient_id claim
AND deleted = false
ORDER BY date DESC
```

## Configuration: Making It Work

### 1. HAPI FHIR Starter Configuration

**File: `application.yaml`**

```yaml
hapi:
  fhir:
    # Server settings
    server_address: https://fhir.example.com/fhir
    version: R4
    
    # Enable SMART authorization
    smart:
      enabled: true
      capabilities:
        - launch-ehr
        - launch-standalone
        - client-public
        - client-confidential-symmetric
        - sso-openid-connect
        - context-ehr-patient
        - permission-patient
        - permission-v2
    
    # OAuth2 Configuration
    security:
      enabled: true
      oauth:
        enabled: true
        # Keycloak endpoints
        issuer_uri: https://keycloak.example.com/auth/realms/test2
        jwk_set_uri: https://keycloak.example.com/auth/realms/test2/protocol/openid-connect/certs
        authorization_endpoint: https://keycloak.example.com/auth/realms/test2/protocol/openid-connect/auth
        token_endpoint: https://keycloak.example.com/auth/realms/test2/protocol/openid-connect/token
        
        # Required claims
        audience: https://fhir.example.com/fhir
        
    # Authorization interceptor settings
    authorization:
      enabled: true
      allow_all_anonymous_access: false
      # Scope-based authorization mode
      authorization_mode: SMART
```

### 2. Scope Mapping Configuration

HAPI FHIR Starter automatically understands SMART scopes following this pattern:

**Pattern: `<context>/<resource>.<permission>`**

```text
Examples:
  patient/Observation.read     → Read Observation in patient context
  patient/Condition.read       → Read Condition in patient context
  patient/MedicationRequest.read → Read MedicationRequest in patient context
  patient/*.read               → Read ALL resources in patient context
  
  user/Observation.read        → Read Observation as practitioner
  system/*.read                → System-level access (admin)
```

### 3. Keycloak Scope Configuration

Your Keycloak is already configured with these scopes! From your `keycloak-config.json`:

```json
{
  "clientScopes": [
    {
      "name": "fhirUser",
      "description": "Permission to retrieve current logged-in user"
    },
    {
      "name": "launch/patient",
      "description": "Patient context at launch time"
    },
    {
      "name": "patient/*.read",
      "description": "Read access to all data in patient compartment"
    },
    {
      "name": "patient/Observation.read",
      "description": "Read access to Observation resources"
    }
    // ... 24 more resource-specific scopes
  ]
}
```

**These scopes are injected into the token automatically!** ✨

## How Scopes Flow Through the System

### Complete Flow Diagram

```mermaid
graph TB
  classDef keycloak fill:#1e40af,stroke:#1e3a8a,color:#FFFFFF,stroke-width:2px
  classDef token fill:#7c2d12,stroke:#6b1d0c,color:#FFFFFF,stroke-width:2px
  classDef hapi fill:#047857,stroke:#065f46,color:#FFFFFF,stroke-width:2px
  classDef result fill:#d1fae5,stroke:#34d399,color:#000000,stroke-width:2px
  
  User[User Authenticates]:::keycloak
  
  KC_Scopes[Keycloak Available Scopes<br/>patient/Observation.read<br/>patient/Condition.read<br/>patient/MedicationRequest.read]:::keycloak
  
  Requested[App Requests Scopes<br/>patient/Observation.read<br/>patient/Condition.read]:::keycloak
  
  Token[Access Token Generated<br/>scope: patient/Observation.read patient/Condition.read<br/>patient_id: Patient/456]:::token
  
  HAPI[HAPI FHIR Receives Request<br/>Authorization: Bearer token]:::hapi
  
  Validate[Validate Token<br/>Check signature, expiration, audience]:::hapi
  
  Parse[Parse Scope Claim<br/>Extract: patient/Observation.read, patient/Condition.read<br/>Extract: patient_id = Patient/456]:::hapi
  
  Rules[Build Authorization Rules<br/>Rule 1: Allow READ Observation for Patient/456<br/>Rule 2: Allow READ Condition for Patient/456<br/>Rule 3: Deny everything else]:::hapi
  
  Execute[Execute FHIR Query<br/>Automatically add patient filter<br/>Query only allowed resources]:::hapi
  
  Response[Return Authorized Data<br/>Only Observations and Conditions<br/>Only for Patient/456]:::result
  
  User --> KC_Scopes
  KC_Scopes --> Requested
  Requested --> Token
  Token --> HAPI
  HAPI --> Validate
  Validate --> Parse
  Parse --> Rules
  Rules --> Execute
  Execute --> Response
```

**Figure 5:** Complete scope flow from Keycloak to data access.

## Practical Examples

### Example 1: Read-Only Access to Observations

#### Step 1: Request Token

```bash
curl -X POST https://keycloak.example.com/auth/realms/test2/protocol/openid-connect/token \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -d "grant_type=password" \
  -d "client_id=my-fhir-app" \
  -d "client_secret=secret" \
  -d "username=john.doe" \
  -d "password=password123" \
  -d "scope=launch/patient patient/Observation.read"
```

**Response:**

```json
{
  "access_token": "eyJhbGciOiJSUzI1NiIs...",
  "token_type": "Bearer",
  "expires_in": 3600,
  "scope": "launch/patient patient/Observation.read"
}
```

**Token decoded:**

```json
{
  "sub": "john.doe",
  "aud": "https://fhir.example.com/fhir",
  "scope": "launch/patient patient/Observation.read",
  "patient_id": "Patient/456",
  "exp": 1699300000
}
```

#### Step 2: Query FHIR Server

```bash
# ✅ ALLOWED: Query Observations
curl -X GET https://fhir.example.com/fhir/Observation \
  -H "Authorization: Bearer eyJhbGciOiJSUzI1NiIs..." \
  -H "Accept: application/fhir+json"
```

**What HAPI FHIR Does:**

1. Validates token ✅
2. Reads scope: `patient/Observation.read` ✅
3. Reads patient_id: `Patient/456` ✅
4. Builds rule: "Allow READ Observation WHERE patient=456" ✅
5. Executes query with automatic filter ✅
6. Returns observations for Patient/456 only ✅

**Response:**

```json
{
  "resourceType": "Bundle",
  "type": "searchset",
  "entry": [
    {
      "resource": {
        "resourceType": "Observation",
        "id": "obs-1",
        "subject": {
          "reference": "Patient/456"
        },
        "code": {
          "coding": [{
            "system": "http://loinc.org",
            "code": "8867-4",
            "display": "Heart rate"
          }]
        }
      }
    }
  ]
}
```

#### Step 3: Try to Access Other Resources

```bash
# ❌ DENIED: No scope for Condition
curl -X GET https://fhir.example.com/fhir/Condition \
  -H "Authorization: Bearer eyJhbGciOiJSUzI1NiIs..."
```

**What HAPI FHIR Does:**

1. Validates token ✅
2. Reads scope: `patient/Observation.read` ✅
3. Checks if scope allows Condition: **NO** ❌
4. Returns 403 Forbidden ❌

**Response:**

```json
{
  "resourceType": "OperationOutcome",
  "issue": [{
    "severity": "error",
    "code": "forbidden",
    "diagnostics": "Access denied: Insufficient scopes to access Condition resources"
  }]
}
```

### Example 2: Wildcard Access to All Patient Data

#### Step 1: Request Token with Wildcard Scope

```bash
curl -X POST https://keycloak.example.com/auth/realms/test2/protocol/openid-connect/token \
  -d "grant_type=password" \
  -d "scope=launch/patient patient/*.read"
  # ... other parameters
```

**Token contains:**

```json
{
  "scope": "launch/patient patient/*.read",
  "patient_id": "Patient/456"
}
```

**What You Can Access:**

```bash
# ✅ ALL of these work:
GET /fhir/Observation     # Observations
GET /fhir/Condition       # Conditions
GET /fhir/MedicationRequest  # Medications
GET /fhir/Procedure       # Procedures
GET /fhir/AllergyIntolerance  # Allergies
# ... any patient-compartment resource
```

**Authorization Rule Built:**

```text
Allow READ on ALL resource types
WHERE resource is in Patient compartment
AND patient = Patient/456
```

### Example 3: Multiple Specific Scopes

**Token Scopes:**

```json
{
  "scope": "launch/patient patient/Observation.read patient/Condition.read patient/MedicationRequest.read",
  "patient_id": "Patient/456"
}
```

**Access Matrix:**

| Request | Scope Required | Has Scope? | Result |
|---------|---------------|------------|--------|
| `GET /Observation` | `patient/Observation.read` | ✅ Yes | 200 OK - Returns data |
| `GET /Condition` | `patient/Condition.read` | ✅ Yes | 200 OK - Returns data |
| `GET /MedicationRequest` | `patient/MedicationRequest.read` | ✅ Yes | 200 OK - Returns data |
| `GET /AllergyIntolerance` | `patient/AllergyIntolerance.read` | ❌ No | 403 Forbidden |
| `GET /Procedure` | `patient/Procedure.read` | ❌ No | 403 Forbidden |
| `POST /Observation` | `patient/Observation.write` | ❌ No | 403 Forbidden |

## Configuration in Your Environment

### Step 1: Install HAPI FHIR Starter

```bash
# Clone the repository
git clone https://github.com/hapifhir/hapi-fhir-jpaserver-starter.git
cd hapi-fhir-jpaserver-starter

# Or use Docker
docker pull hapiproject/hapi:latest
```

### Step 2: Configure Keycloak Integration

**File: `src/main/resources/application.yaml`**

```yaml
spring:
  security:
    oauth2:
      resourceserver:
        jwt:
          # Point to your Keycloak instance
          issuer-uri: https://keycloak.example.com/auth/realms/test2
          jwk-set-uri: https://keycloak.example.com/auth/realms/test2/protocol/openid-connect/certs

hapi:
  fhir:
    # Your FHIR server URL (must match token audience)
    server_address: https://fhir.example.com/fhir
    
    # Enable SMART on FHIR
    smart:
      enabled: true
    
    # Enable scope-based authorization
    authorization:
      enabled: true
      authorization_mode: SMART
      
    # SMART capability statement
    smart_capabilities:
      - launch-ehr
      - launch-standalone
      - client-public
      - sso-openid-connect
      - context-ehr-patient
      - permission-patient
```

### Step 3: Configure Scope Mapping (Optional)

If you need custom scope mapping, create a configuration class:

```java
@Configuration
public class SMARTAuthorizationConfig {
    
    @Bean
    public IAuthorizationService authorizationService() {
        return new SMARTAuthorizationService();
    }
}

public class SMARTAuthorizationService implements IAuthorizationService {
    
    @Override
    public List<IAuthRule> buildRuleList(RequestDetails request) {
        // Extract JWT from request
        String token = extractBearerToken(request);
        DecodedJWT jwt = JWT.decode(token);
        
        // Get claims
        String scope = jwt.getClaim("scope").asString();
        String patientId = jwt.getClaim("patient_id").asString();
        
        // Build rules from scopes
        RuleBuilder builder = new RuleBuilder();
        
        // Parse scopes
        String[] scopes = scope.split(" ");
        for (String s : scopes) {
            if (s.matches("patient/\\w+\\.read")) {
                // Extract resource type: patient/Observation.read → Observation
                String resourceType = s.substring(8, s.length() - 5);
                
                builder.allow()
                    .read()
                    .resourcesOfType(resourceType)
                    .inCompartment("Patient", new IdType("Patient", patientId))
                    .andThen();
            }
            else if (s.equals("patient/*.read")) {
                // Wildcard: allow all patient resources
                builder.allow()
                    .read()
                    .allResources()
                    .inCompartment("Patient", new IdType("Patient", patientId))
                    .andThen();
            }
        }
        
        // Deny everything else
        builder.denyAll();
        
        return builder.build();
    }
}
```

### Step 4: Start the Server

```bash
# Using Maven
mvn spring-boot:run

# Using Docker
docker run -p 8080:8080 \
  -e KEYCLOAK_URL=https://keycloak.example.com \
  -e KEYCLOAK_REALM=test2 \
  hapiproject/hapi:latest
```

### Step 5: Test the Integration

```bash
# 1. Get token from Keycloak
TOKEN=$(curl -X POST https://keycloak.example.com/auth/realms/test2/protocol/openid-connect/token \
  -d "grant_type=password" \
  -d "client_id=my-fhir-app" \
  -d "client_secret=secret" \
  -d "username=testuser" \
  -d "password=password" \
  -d "scope=launch/patient patient/Observation.read" \
  | jq -r '.access_token')

# 2. Query FHIR API
curl -X GET http://localhost:8080/fhir/Observation \
  -H "Authorization: Bearer $TOKEN" \
  -H "Accept: application/fhir+json"
```

## Advanced: Scope Hierarchies

HAPI FHIR Starter supports scope hierarchies:

```mermaid
graph TB
  classDef broad fill:#1e40af,stroke:#1e3a8a,color:#FFFFFF,stroke-width:2px
  classDef specific fill:#047857,stroke:#065f46,color:#FFFFFF,stroke-width:2px
  classDef granular fill:#7c2d12,stroke:#6b1d0c,color:#FFFFFF,stroke-width:2px
  
  System[system/*.read<br/>System-level access]:::broad
  User[user/*.read<br/>Practitioner access]:::broad
  Patient[patient/*.read<br/>Patient compartment]:::specific
  
  ObsRead[patient/Observation.read<br/>Observations only]:::granular
  CondRead[patient/Condition.read<br/>Conditions only]:::granular
  MedRead[patient/MedicationRequest.read<br/>Medications only]:::granular
  
  System --> User
  User --> Patient
  Patient --> ObsRead
  Patient --> CondRead
  Patient --> MedRead
```

**Figure 6:** SMART scope hierarchy from broad to granular permissions.

**Hierarchy Rules:**

1. **System scopes** (`system/*.read`) - Broadest access
   - Access all data across all patients
   - Typically for backend services
   
2. **User scopes** (`user/*.read`) - Practitioner context
   - Access data user can see in their role
   - Based on practitioner relationships
   
3. **Patient scopes** (`patient/*.read`) - Patient context
   - Access one patient's data
   - Most common for patient-facing apps

4. **Resource-specific** (`patient/Observation.read`) - Most granular
   - Access specific resource types only
   - Best for least-privilege applications

## Troubleshooting

### Issue 1: Token Valid but 403 Forbidden

**Symptom**: Token validates but all requests return 403

**Check:**

1. **Scope in token matches requested resource**

```bash
# Decode your token to check scopes
echo "eyJhbGc..." | base64 -d | jq '.scope'

# Should output: "launch/patient patient/Observation.read"
```

**Check Patient ID:**

```bash
# Check for patient_id claim
echo "eyJhbGc..." | base64 -d | jq '.patient_id'

# Should output: "Patient/456"
```

**Check Audience:**

```bash
# Check audience claim
echo "eyJhbGc..." | base64 -d | jq '.aud'

# Should match hapi.fhir.server_address in config
```

### Issue 2: Wildcard Scope Not Working

**Symptom**: `patient/*.read` scope doesn't grant access

**Solution**: Ensure HAPI FHIR interprets wildcards correctly

```yaml
hapi:
  fhir:
    authorization:
      # Must be SMART mode to support wildcards
      authorization_mode: SMART
      wildcard_scopes_enabled: true
```

### Issue 3: Can Access Wrong Patient's Data

**Symptom**: User can see data for patients other than in token

**Check:**

1. **Patient compartment filtering is enabled**

```yaml
hapi:
  fhir:
    authorization:
      enforce_compartment_boundaries: true
```

**Verify Interceptor Order:**

```java
// Token validator must come BEFORE authorization interceptor
server.registerInterceptor(tokenValidator);
server.registerInterceptor(authorizationInterceptor);
```

## Summary

### What Happens Automatically ✨

| Step | HAPI FHIR Starter Handles | You Configure |
|------|---------------------------|---------------|
| 1. Token validation | ✅ Automatic | Keycloak URL |
| 2. Signature verification | ✅ Automatic | JWK Set URI |
| 3. Expiration check | ✅ Automatic | Nothing |
| 4. Audience validation | ✅ Automatic | Server URL |
| 5. Scope parsing | ✅ Automatic | Nothing |
| 6. Authorization rules | ✅ Automatic | Enable SMART mode |
| 7. Patient filtering | ✅ Automatic | Enable compartments |
| 8. Error responses | ✅ Automatic | Nothing |

### What You Need to Configure 🔧

1. **Keycloak connection** (5 lines in application.yaml)
2. **Enable SMART mode** (2 lines in application.yaml)
3. **Server address** (1 line in application.yaml)

**Total configuration: ~10 lines of YAML** 🎉

### Key Takeaways

1. **Scopes in token = Automatic permissions** - HAPI FHIR Starter reads the `scope` claim and enforces access
2. **Patient filtering is automatic** - The `patient_id` claim automatically limits all queries
3. **No custom code needed** - Built-in interceptors handle everything
4. **Just configure and go** - Point to Keycloak, enable SMART mode, done!

## Related Documentation

- [Keycloak Configuration Overview](./keycloak-config-overview.md)
- [Client Scopes and Permissions](./keycloak-scopes.md)
- [HAPI FHIR Integration (Detailed)](./hapi-fhir-integration.md)
- [HAPI FHIR Starter Documentation](https://github.com/hapifhir/hapi-fhir-jpaserver-starter)
- [SMART App Launch Specification](http://hl7.org/fhir/smart-app-launch/)

## Next Steps

1. Install HAPI FHIR Starter
2. Configure Keycloak connection (application.yaml)
3. Enable SMART authorization mode
4. Test with your existing Keycloak scopes
5. Deploy and monitor
