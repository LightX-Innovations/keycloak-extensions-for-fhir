# Client Scopes and FHIR Permissions

## Overview

Client scopes define the OAuth 2.0 scopes that applications can request and determine what claims are added to tokens. In the FHIR context, scopes represent permissions to access specific types of healthcare data.

## Scope Architecture

```mermaid
graph TB
  classDef coreScope fill:#1e40af,stroke:#1e3a8a,color:#FFFFFF,stroke-width:2px
  classDef wildcardScope fill:#7c2d12,stroke:#6b1d0c,color:#FFFFFF,stroke-width:2px
  classDef resourceScope fill:#dbeafe,stroke:#3b82f6,color:#000000,stroke-width:2px
  classDef mapper fill:#d1fae5,stroke:#34d399,color:#000000,stroke-width:2px
  
  Scopes[Client Scopes System]
  
  Core[Core SMART Scopes]:::coreScope
  Wildcard[Wildcard Access]:::wildcardScope
  Resources[Granular Resource Scopes]:::resourceScope
  
  Scopes --> Core
  Scopes --> Wildcard
  Scopes --> Resources
  
  Core --> fhirUser[fhirUser]:::coreScope
  Core --> launch[launch/patient]:::coreScope
  Core --> online[online_access]:::coreScope
  
  Wildcard --> allRead[patient/*.read]:::wildcardScope
  
  Resources --> r1[patient/Observation.read]:::resourceScope
  Resources --> r2[patient/Condition.read]:::resourceScope
  Resources --> r3[patient/MedicationRequest.read]:::resourceScope
  Resources --> more[... 21 more resource types]:::resourceScope
  
  fhirUser --> m1[Protocol Mappers]:::mapper
  launch --> m1
  allRead --> m1
  r1 --> m1
```

**Figure 1:** Client scope hierarchy showing core, wildcard, and granular resource scopes.

## Core SMART Scopes

### fhirUser Scope

**Purpose**: Retrieve the current logged-in user's FHIR resource identifier.

```json
{
  "fhirUser": {
    "protocol": "openid-connect",
    "description": "Permission to retrieve current logged-in user",
    "attributes": {
      "consent.screen.text": "Permission to retrieve current logged-in user"
    },
    "mappers": {
      "fhirUser Mapper": {
        "protocol": "openid-connect",
        "protocolmapper": "oidc-patient-prefix-usermodel-attribute-mapper",
        "config": {
          "user.attribute": "resourceId",
          "claim.name": "fhirUser",
          "jsonType.label": "String",
          "id.token.claim": "true",
          "access.token.claim": "false",
          "userinfo.token.claim": "true"
        }
      }
    }
  }
}
```

**Key Features**:

- Adds `fhirUser` claim to ID token and UserInfo endpoint
- Format: `Patient/{id}` or `Practitioner/{id}` depending on user type
- Uses custom mapper: `oidc-patient-prefix-usermodel-attribute-mapper`
- Reads from user attribute: `resourceId`

**Token Claim Example**:

```json
{
  "fhirUser": "Patient/123"
}
```

### launch/patient Scope

**Purpose**: Enables patient context during authorization, allowing the app to request patient-specific data.

```json
{
  "launch/patient": {
    "protocol": "openid-connect",
    "description": "Used by clients to request patient scope",
    "attributes": {
      "display.on.consent.screen": "false"
    },
    "mappers": {
      "Patient ID Mapper": { ... },
      "Group Membership Mapper": { ... }
    }
  }
}
```

**Key Features**:

- Required for SMART App Launch with patient context
- Does not display on consent screen (system-level scope)
- Includes two critical mappers

#### Patient ID Mapper

```json
{
  "Patient ID Mapper": {
    "protocol": "openid-connect",
    "protocolmapper": "oidc-usermodel-attribute-mapper",
    "config": {
      "user.attribute": "resourceId",
      "jsonType.label": "String",
      "claim.name": "patient_id",
      "id.token.claim": "false",
      "access.token.claim": "true",
      "userinfo.token.claim": "false"
    }
  }
}
```

- Adds `patient_id` claim to access token only
- Used by FHIR server to filter data to specific patient
- Format: `Patient/{id}` (extracted from `resourceId` user attribute)

#### Group Membership Mapper

```json
{
  "Group Membership Mapper": {
    "protocol": "openid-connect",
    "protocolmapper": "oidc-group-membership-mapper",
    "config": {
      "claim.name": "group",
      "full.path": "false",
      "id.token.claim": "true",
      "access.token.claim": "true",
      "userinfo.token.claim": "true"
    }
  }
}
```

- Adds user's group memberships to all tokens
- Used for role-based access control
- Format: Array of group names

### online_access Scope

**Purpose**: Request a refresh token that remains valid while the user is online.

```json
{
  "online_access": {
    "protocol": "openid-connect",
    "description": "Request a refresh_token that can be used to obtain a new access token to replace an expired one, and that will be usable for as long as the end-user remains online.",
    "attributes": {
      "consent.screen.text": "Retain access while you are online"
    }
  }
}
```

**Key Features**:

- Part of SMART on FHIR specification
- Refresh token invalidated when user logs out
- Requires user consent
- No protocol mappers (affects token lifetime only)

## Wildcard Scope

### patient/*.read

**Purpose**: Grant read access to all patient-compartment resources.

```json
{
  "patient/*.read": {
    "protocol": "openid-connect",
    "description": "Read access to all data",
    "attributes": {
      "consent.screen.text": "Read access to all data for the patient"
    },
    "mappers": {
      "Audience Mapper": {
        "protocol": "openid-connect",
        "protocolmapper": "oidc-audience-mapper",
        "config": {
          "included.custom.audience": "<FHIR_BASE_URL>",
          "access.token.claim": "true"
        }
      }
    }
  }
}
```

**Key Features**:

- Single scope grants access to all FHIR resources
- More convenient but less granular than individual resource scopes
- Includes audience mapper for FHIR server validation
- Requires user consent

**Use Case**: Applications that need comprehensive patient data access.

## Granular Resource Scopes

Each FHIR resource type has its own scope for fine-grained permissions.

### Scope Pattern

All resource scopes follow this pattern:

```json
{
  "patient/{ResourceType}.read": {
    "protocol": "openid-connect",
    "description": "Read access to {ResourceType}",
    "attributes": {
      "consent.screen.text": "Read access to {ResourceType} for the patient"
    },
    "mappers": {
      "Audience Mapper": {
        "protocol": "openid-connect",
        "protocolmapper": "oidc-audience-mapper",
        "config": {
          "included.custom.audience": "<FHIR_BASE_URL>",
          "access.token.claim": "true"
        }
      }
    }
  }
}
```

### Complete Resource Scope List

| Category | Resource Scopes |
|----------|----------------|
| **Allergies & Conditions** | `patient/AllergyIntolerance.read`, `patient/Condition.read` |
| **Medications** | `patient/Medication.read`, `patient/MedicationRequest.read` |
| **Care Plans** | `patient/CarePlan.read`, `patient/CareTeam.read`, `patient/Goal.read` |
| **Clinical Data** | `patient/Observation.read`, `patient/DiagnosticReport.read`, `patient/Procedure.read` |
| **Encounters** | `patient/Encounter.read`, `patient/Location.read` |
| **Documents** | `patient/DocumentReference.read`, `patient/Provenance.read` |
| **Patient Info** | `patient/Patient.read`, `patient/RelatedPerson.read` |
| **Care Providers** | `patient/Practitioner.read`, `patient/PractitionerRole.read`, `patient/Organization.read` |
| **Devices** | `patient/Device.read` |
| **Immunizations** | `patient/Immunization.read` |
| **Financial** | `patient/ExplanationOfBenefit.read` |

**Total**: 25 granular resource scopes defined.

### Audience Mapper

Every resource scope includes an audience mapper to ensure tokens are only valid for the intended FHIR server.

```mermaid
flowchart TB
  classDef scopeNode fill:#dbeafe,stroke:#3b82f6,color:#000000,stroke-width:2px
  classDef mapperNode fill:#047857,stroke:#065f46,color:#FFFFFF,stroke-width:2px
  classDef tokenNode fill:#d1fae5,stroke:#34d399,color:#000000,stroke-width:2px
  classDef validationNode fill:#7c2d12,stroke:#6b1d0c,color:#FFFFFF,stroke-width:2px
  
  Scope[Resource Scope Granted]:::scopeNode
  Mapper[Audience Mapper Executes]:::mapperNode
  Config[Read FHIR_BASE_URL from config]:::mapperNode
  Token[Add aud claim to access token]:::tokenNode
  
  App[App receives token]:::tokenNode
  Request[App sends request to FHIR server]:::tokenNode
  Validate[FHIR server validates aud claim]:::validationNode
  Match{aud matches server URL?}:::validationNode
  
  Accept[Accept request]:::tokenNode
  Reject[Reject with 401 Unauthorized]:::validationNode
  
  Scope --> Mapper
  Mapper --> Config
  Config --> Token
  
  Token --> App
  App --> Request
  Request --> Validate
  Validate --> Match
  Match -->|Yes| Accept
  Match -->|No| Reject
```

**Figure 2:** Audience claim flow from scope to FHIR server validation.

## Scope Assignment to Clients

### Default Client Scopes

Scopes automatically included for all clients:

```json
{
  "defaultDefaultClientScopes": ["launch/patient"]
}
```

- Always included in authorization requests
- No user consent required
- Essential for SMART App Launch

### Optional Client Scopes

Scopes that clients can request but require user consent:

```json
{
  "defaultOptionalClientScopes": [
    "fhirUser",
    "offline_access",
    "online_access",
    "profile",
    "patient/*.read",
    "patient/AllergyIntolerance.read",
    "patient/CarePlan.read",
    ...
  ]
}
```

### Client-Specific Configuration

The `inferno` client overrides realm defaults:

```json
{
  "inferno": {
    "defaultClientScopes": ["launch/patient"],
    "optionalClientScopes": [
      "fhirUser",
      "offline_access",
      "online_access",
      "profile",
      "patient/*.read",
      "patient/AllergyIntolerance.read",
      ...
    ]
  }
}
```

## Scope Request Flow

```mermaid
sequenceDiagram
  participant App as FHIR Application
  participant KC as Keycloak
  participant User
  
  App->>KC: Authorization request with scope parameter
  Note right of App: scope=launch/patient patient/Observation.read patient/Condition.read
  
  KC->>KC: Validate requested scopes
  KC->>KC: Check default vs optional scopes
  
  alt Has optional scopes
    KC->>User: Display consent screen
    User->>KC: Grant or deny scopes
  end
  
  KC->>KC: Generate authorization code
  KC->>App: Return code with granted scopes
  
  App->>KC: Exchange code for tokens
  KC->>KC: Add claims via protocol mappers
  KC->>App: Access token with scope and claims
  
  Note right of App: Token contains:<br/>scope: launch/patient patient/Observation.read<br/>patient_id: Patient/123<br/>aud: https://fhir-server.com
```

**Figure 3:** Scope request and token generation sequence.

## Token Claim Mapping Summary

| Scope | Mapper | Claim Name | Token Type | Source |
|-------|--------|------------|------------|--------|
| `fhirUser` | fhirUser Mapper | `fhirUser` | ID Token, UserInfo | User attribute `resourceId` |
| `launch/patient` | Patient ID Mapper | `patient_id` | Access Token | User attribute `resourceId` |
| `launch/patient` | Group Membership Mapper | `group` | All tokens | User groups |
| All resource scopes | Audience Mapper | `aud` | Access Token | Scope config `included.custom.audience` |

## Best Practices

### Scope Selection

```mermaid
flowchart TB
  classDef startNode fill:#dbeafe,stroke:#3b82f6,color:#000000,stroke-width:2px
  classDef decisionNode fill:#fbbf24,stroke:#f59e0b,color:#000000,stroke-width:2px
  classDef optionNode fill:#d1fae5,stroke:#34d399,color:#000000,stroke-width:2px
  
  Start[Application needs FHIR data access]:::startNode
  
  Q1{Need all resource types?}:::decisionNode
  Q2{Need multiple specific types?}:::decisionNode
  
  Wildcard[Use patient/*.read]:::optionNode
  Multiple[Use multiple granular scopes]:::optionNode
  Single[Use single resource scope]:::optionNode
  
  Start --> Q1
  Q1 -->|Yes| Wildcard
  Q1 -->|No| Q2
  Q2 -->|Yes| Multiple
  Q2 -->|No| Single
  
  Wildcard --> Note1[Simpler but less privacy-focused]
  Multiple --> Note2[Better privacy, more consent steps]
  Single --> Note3[Minimal access, best practice]
```

**Figure 4:** Decision tree for selecting appropriate scopes.

### Recommendations

1. **Request Minimum Necessary Scopes**: Only request access to data actually needed
2. **Use Granular Scopes When Possible**: Better for privacy and user trust
3. **Always Include `launch/patient`**: Required for patient context
4. **Consider User Experience**: Too many consent prompts can hurt UX
5. **Document Scope Usage**: Explain why each scope is needed

## Custom Scope Development

To add new custom scopes:

1. Define scope in configuration JSON
2. Add protocol mapper configuration
3. Implement custom mapper if needed (Java)
4. Register in `META-INF/services/org.keycloak.protocol.ProtocolMapper`
5. Add to default or optional client scopes lists

## Related Documentation

- [Authentication Flow](./keycloak-auth-flow.md)
- [Token Claims and Mappers](./keycloak-token-mappers.md)
- [SMART on FHIR Specification](http://hl7.org/fhir/smart-app-launch/)
