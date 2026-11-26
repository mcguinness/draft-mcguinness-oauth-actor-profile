# Service-to-Service Delegation Example

This example shows this profile in a same-domain service-to-service delegation flow that involves no AI agent.  A payroll batch processor acts on behalf of a human payroll administrator to call the Payroll API; the Payroll API then exchanges the inbound access token for an internal Transaction Token used to write an audit record.

## Scenario

| Party | Identifier |
|-------|------------|
| Payroll Administrator | `https://idp.example.com/users/pat` |
| Enterprise AS | `https://as.example.com` |
| Payroll Batch Processor | `https://services.example.com/payroll-batch` |
| Payroll API | `https://services.example.com/payroll-api` |
| Audit TTS | `https://tts.example.com` |
| Audit Service | `https://internal.example.com/audit` |

The batch processor is both the OAuth client and the acting service.  The `client_id` claim identifies the client registration, and `act.sub` explicitly identifies the acting service.

## Access Token

The Enterprise AS issues the following JWT access token for the Payroll API:

~~~json
{
  "iss": "https://as.example.com",
  "sub": "https://idp.example.com/users/pat",
  "sub_profile": "user",
  "client_id": "payroll-batch-client",
  "aud": "https://services.example.com/payroll-api",
  "scope": "payroll:run",
  "cnf": { "jkt": "SvcJKT-123" },
  "act": {
    "sub": "https://services.example.com/payroll-batch",
    "iss": "https://as.example.com",
    "sub_profile": "service"
  }
}
~~~

This token carries a single-hop actor object: the `act` claim is present but contains no nested `act` object.  The `sub` claim identifies the payroll administrator, while `act.sub` identifies the service exercising that administrator's authorization.

## Transaction Token

After processing the payroll request, the Payroll API exchanges the inbound access token at the Audit TTS to call the internal Audit Service, presenting its workload credential, issued by `https://as.example.com`, as `actor_token`.  The Payroll API is the requesting workload (`req_wl`).  In presenter-rebind mode, the TTS validates the inbound delegation chain and the workload credential, preserves the chain as an inner `act` object, and adds a new outermost actor for the Payroll API:

~~~json
{
  "iss": "https://tts.example.com",
  "sub": "https://idp.example.com/users/pat",
  "sub_profile": "user",
  "req_wl": "https://services.example.com/payroll-api",
  "aud": "https://internal.example.com/audit",
  "scope": "audit:create",
  "txn": "550e8400-e29b-41d4-a716-446655440099",
  "cnf": { "jkt": "ApiJKT-456" },
  "act": {
    "sub": "https://services.example.com/payroll-api",
    "iss": "https://as.example.com",
    "sub_profile": "service",
    "act": {
      "sub": "https://services.example.com/payroll-batch",
      "iss": "https://as.example.com",
      "sub_profile": "service"
    }
  }
}
~~~

The administrator remains the subject.  The Payroll API becomes the outermost actor, and the batch processor remains the inner actor.
