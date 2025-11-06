# SMART App Launch Authentication Flow

## Overview

This document explains the SMART App Launch authentication flow implemented in the Keycloak configuration. SMART (Substitutable Medical Applications, Reusable Technologies) is the healthcare standard for secure application authorization with patient context.

## Authentication Flow Sequence

```mermaid
sequenceDiagram
  participant User
  participant App as FHIR Application
  participant KC as Keycloak Server
  participant FHIR as FHIR Server
  
  Note over User,FHIR: SMART App Launch Authentication Flow
  
  User->>App: 1. Access application
  App->>KC: 2. Authorization request with scopes
  
  rect rgb(255, 250, 240)
    Note right of KC: SMART App Launch Flow Executes
    
    KC->>KC: 3a. Audience Validation
    KC->>User: 3b. Display login form
    User->>KC: 3c. Enter credentials
    
    KC->>FHIR: 3d. Query patient list
    FHIR-->>KC: 3e. Return patients
    KC->>User: 3f. Display patient selector
    User->>KC: 3g. Select patient
  end
  
  KC->>KC: 4. Generate authorization code
  KC-->>App: 5. Return code via redirect
  
  App->>KC: 6. Exchange code for tokens
  KC-->>App: 7. Access token with patient context
  
  App->>FHIR: 8. API request with token
  FHIR->>FHIR: 9. Validate token and audience
  FHIR-->>App: 10. Return FHIR resources
  App-->>User: 11. Display patient data
```

**Figure 1:** Complete SMART App Launch authentication sequence showing all actors and their interactions.

## Flow Configuration

The authentication flow is defined in the `authenticationFlows` section and set as the `browserFlow` for the realm.

```json
{
  "SMART App Launch": {
    "description": "browser based authentication",
    "providerId": "basic-flow",
    "builtIn": false,
    "authenticationExecutions": {
      "SMART Login": { ... }
    }
  },
  "browserFlow": "SMART App Launch"
}
```

## Execution Steps

### Step 1: Audience Validation

```mermaid
flowchart TB
  classDef startNode fill:#dbeafe,stroke:#3b82f6,color:#000000,stroke-width:2px
  classDef processNode fill:#047857,stroke:#065f46,color:#FFFFFF,stroke-width:2px
  classDef errorNode fill:#b91c1c,stroke:#7f1d1d,color:#FFFFFF,stroke-width:2px
  classDef successNode fill:#d1fae5,stroke:#34d399,color:#000000,stroke-width:2px
  
  Start[Authorization Request Received]:::startNode
  Extract[Extract aud parameter from request]:::processNode
  Validate[Compare against configured audiences]:::processNode
  Valid{Audience Valid?}:::processNode
  Success[Continue to next step]:::successNode
  Error[Reject with error]:::errorNode
  
  Start --> Extract
  Extract --> Validate
  Validate --> Valid
  Valid -->|Yes| Success
  Valid -->|No| Error
```

**Figure 2:** Audience validation process flow.

#### Audience Validator Configuration

```json
{
  "Audience Validation": {
    "authenticator": "audience-validator",
    "requirement": "DISABLED",
    "priority": 10,
    "authenticatorFlow": false,
    "configAlias": "localhost",
    "config": {
      "audiences": "https://localhost:9443/fhir-server/api/v4##http://host.docker.internal:9080/fhir-server/api/v4"
    }
  }
}
```

#### Audience Validator Key Points

- **Authenticator**: `audience-validator` (custom component)
- **Requirement**: `DISABLED` in test config (can be enabled for production)
- **Purpose**: Validates that the requested FHIR server URL is authorized
- **Audiences**: Multiple URLs separated by `##`

### Step 2: Username and Password Authentication

```mermaid
flowchart TB
  classDef formNode fill:#dbeafe,stroke:#3b82f6,color:#000000,stroke-width:2px
  classDef processNode fill:#047857,stroke:#065f46,color:#FFFFFF,stroke-width:2px
  classDef errorNode fill:#b91c1c,stroke:#7f1d1d,color:#FFFFFF,stroke-width:2px
  classDef successNode fill:#d1fae5,stroke:#34d399,color:#000000,stroke-width:2px
  
  Display[Display Login Form]:::formNode
  Submit[User Submits Credentials]:::formNode
  Verify[Verify Username and Password]:::processNode
  Valid{Valid?}:::processNode
  Success[Authentication Success]:::successNode
  Error[Show Error - Retry]:::errorNode
  
  Display --> Submit
  Submit --> Verify
  Verify --> Valid
  Valid -->|Yes| Success
  Valid -->|No| Error
  Error --> Display
```

**Figure 3:** Standard username/password authentication flow.

#### Username Password Configuration

```json
{
  "Username Password Form": {
    "authenticator": "auth-username-password-form",
    "requirement": "REQUIRED",
    "priority": 20,
    "authenticatorFlow": false
  }
}
```

#### Username Password Key Points

- **Authenticator**: Built-in Keycloak username/password form
- **Requirement**: `REQUIRED` - must succeed for authentication to continue
- **Priority**: 20 - executes after audience validation

### Step 3: Patient Selection

```mermaid
flowchart TB
  classDef queryNode fill:#047857,stroke:#065f46,color:#FFFFFF,stroke-width:2px
  classDef displayNode fill:#dbeafe,stroke:#3b82f6,color:#000000,stroke-width:2px
  classDef processNode fill:#7c2d12,stroke:#6b1d0c,color:#FFFFFF,stroke-width:2px
  classDef successNode fill:#d1fae5,stroke:#34d399,color:#000000,stroke-width:2px
  
  Auth[User Authenticated]:::successNode
  Query[Query FHIR Server for Patients]:::queryNode
  Parse[Parse Patient Resources]:::processNode
  Display[Display Patient Selection Form]:::displayNode
  Select[User Selects Patient]:::displayNode
  Store[Store Patient ID in Session]:::processNode
  Continue[Continue to Token Generation]:::successNode
  
  Auth --> Query
  Query --> Parse
  Parse --> Display
  Display --> Select
  Select --> Store
  Store --> Continue
```

**Figure 4:** Patient selection authenticator process showing FHIR server interaction.

#### Patient Selection Configuration

```json
{
  "Patient Selection Authenticator": {
    "authenticator": "auth-select-patient",
    "requirement": "REQUIRED",
    "priority": 30,
    "authenticatorFlow": false,
    "configAlias": "host.docker",
    "config": {
      "internalFhirUrl": "http://host.docker.internal:9080/fhir-server/api/v4"
    }
  }
}
```

#### Patient Selection Key Points

- **Authenticator**: `auth-select-patient` (custom component from this project)
- **Requirement**: `REQUIRED` - user must select a patient
- **Priority**: 30 - executes last in the flow
- **FHIR URL**: Internal URL used to query available patients
- **Template**: Uses `patient-select-form.ftl` for UI rendering

#### Custom Extension Code

This authenticator is implemented in:

```text
keycloak-extensions/src/main/java/org/alvearie/keycloak/PatientSelectionForm.java
keycloak-extensions/src/main/java/org/alvearie/keycloak/PatientSelectionFormFactory.java
```

The form template is located at:

```text
keycloak-extensions/src/main/resources/theme-resources/templates/patient-select-form.ftl
```

## Token Generation

After successful authentication and patient selection, Keycloak generates tokens with the following claims:

### Access Token Claims

```json
{
  "sub": "user-id",
  "aud": "https://fhir-server.example.com/api/v4",
  "patient_id": "Patient/123",
  "scope": "launch/patient patient/Observation.read patient/Condition.read",
  "group": ["fhirUser"]
}
```

### ID Token Claims

```json
{
  "sub": "user-id",
  "fhirUser": "Patient/123",
  "group": ["fhirUser"]
}
```

### Claims Added by Mappers

| Claim        | Source                      | Mapper                  | Token Type         |
| ------------ | --------------------------- | ----------------------- | ------------------ |
| `patient_id` | User attribute `resourceId` | Patient ID Mapper       | Access Token       |
| `fhirUser`   | User attribute `resourceId` | fhirUser Mapper         | ID Token, UserInfo |
| `group`      | User groups                 | Group Membership Mapper | All Tokens         |
| `aud`        | Client scope config         | Audience Mapper         | Access Token       |

## Flow Execution Requirements

```mermaid
stateDiagram-v2
  classDef requiredState fill:#047857,stroke:#065f46,color:#FFFFFF
  classDef optionalState fill:#dbeafe,stroke:#3b82f6,color:#000000
  classDef successState fill:#d1fae5,stroke:#34d399,color:#000000
  
  [*] --> AudienceValidation
  
  state AudienceValidation {
    [*] --> Disabled: requirement=DISABLED
    Disabled --> [*]
  }
  
  AudienceValidation --> UsernamePassword
  
  state UsernamePassword {
    [*] --> Required: requirement=REQUIRED
    Required --> Success: Valid Credentials
    Required --> Failed: Invalid Credentials
    Failed --> Required: Retry
    Success --> [*]
  }
  
  UsernamePassword --> PatientSelection
  
  state PatientSelection {
    [*] --> Required: requirement=REQUIRED
    Required --> Query: Fetch Patients
    Query --> Display: Show Form
    Display --> Selected: User Chooses
    Selected --> [*]
  }
  
  PatientSelection --> [*]: All Steps Complete
  
  class UsernamePassword requiredState
  class PatientSelection requiredState
  class AudienceValidation optionalState
```

**Figure 5:** State diagram showing execution requirements for each authentication step.

## Requirement Types Explained

| Requirement   | Behavior                                        |
| ------------- | ----------------------------------------------- |
| `REQUIRED`    | Must succeed for authentication to continue     |
| `ALTERNATIVE` | At least one alternative execution must succeed |
| `DISABLED`    | Skipped during authentication                   |
| `CONDITIONAL` | Executed based on conditions                    |

## Error Handling

### Authentication Failures

```mermaid
flowchart TB
  classDef errorNode fill:#b91c1c,stroke:#7f1d1d,color:#FFFFFF,stroke-width:2px
  classDef processNode fill:#047857,stroke:#065f46,color:#FFFFFF,stroke-width:2px
  classDef retryNode fill:#fbbf24,stroke:#f59e0b,color:#000000,stroke-width:2px
  
  Error[Authentication Error Occurs]:::errorNode
  Log[Log Event to Keycloak Events]:::processNode
  Type{Error Type}:::processNode
  
  InvalidCreds[Invalid Credentials]:::errorNode
  NoPatient[No Patient Selected]:::errorNode
  FHIRError[FHIR Server Unavailable]:::errorNode
  
  RetryLogin[Return to Login Form]:::retryNode
  RetryPatient[Return to Patient Selection]:::retryNode
  SystemError[Display System Error]:::errorNode
  
  Error --> Log
  Log --> Type
  
  Type -->|Bad Password| InvalidCreds
  Type -->|No Selection| NoPatient
  Type -->|FHIR Down| FHIRError
  
  InvalidCreds --> RetryLogin
  NoPatient --> RetryPatient
  FHIRError --> SystemError
```

**Figure 6:** Error handling flow for authentication failures.

### Logged Events

All authentication events are logged in Keycloak events (when `saveLoginEvents: true`):

- `LOGIN` - Successful authentication
- `LOGIN_ERROR` - Failed authentication
- `CODE_TO_TOKEN` - Successful token exchange
- `CODE_TO_TOKEN_ERROR` - Failed token exchange

## Customization Points

### Adding Additional Authentication Steps

To add new steps to the SMART App Launch flow:

1. Create authenticator implementation (Java)
2. Register in `META-INF/services/org.keycloak.authentication.AuthenticatorFactory`
3. Add to `authenticationExecutions` in config JSON
4. Set appropriate `priority` value

### Modifying Patient Selection

The patient selection logic can be customized by modifying:

- **Backend Logic**: `PatientSelectionForm.java`
- **FHIR Query**: Update query parameters in authenticator
- **UI Template**: Edit `patient-select-form.ftl`
- **Styling**: Add custom CSS to theme

## Security Considerations

### Audience Validation

- Should be `REQUIRED` in production
- Prevents token misuse across FHIR servers
- Validates `aud` parameter matches configured servers

### Patient Context Isolation

- Each token is scoped to a single patient
- User must explicitly select patient
- Patient ID stored in user session and token claims
- FHIR server validates patient context on each request

### Token Lifetime

Configure appropriate token lifetimes based on use case:

- **Access Token**: Short-lived (5-15 minutes)
- **Refresh Token**: Longer-lived (hours to days)
- **SSO Session**: Based on security requirements

## Testing the Flow

### Using Inferno Test Tool

The `inferno` client is configured for testing:

```bash
# Start Keycloak
docker-compose up keycloak

# Access Inferno
http://localhost:4567/inferno

# Configure test:
# - FHIR Server: http://localhost:9080/fhir-server/api/v4
# - Auth URL: http://localhost:55095/auth/realms/test2/protocol/openid-connect/auth
# - Token URL: http://localhost:55095/auth/realms/test2/protocol/openid-connect/token
```

## Next Steps

- [Client Scopes and Permissions](./keycloak-scopes.md)
- [Token Claims and Mappers](./keycloak-token-mappers.md)
- [Custom Authenticator Development](./custom-authenticators.md)
