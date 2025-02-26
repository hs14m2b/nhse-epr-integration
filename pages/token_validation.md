[Home](../readme.md)

[Back to Appendix 2](appendix2.md)

# Token Validation

On receiving a request, Epic must validate the identity of the local patient details against the details provided in the ID token passed in header `NHSD-ID-Token`. The valdation requirements are:

| Step | Rule | Description | Standard Ref | Notes |
|---|---|---|---|---|
| 1 | Claim: Header: alg | This MUST be "RS512". For signature verification to work it would have to be this value but this is for completeness to OIDC standard | [OIDC 1.0 3.1.3.7](https://openid.net/specs/openid-connect-core-1_0.html#IDTokenValidation) | |
| 2 | Verify Signature | Verify the signature in the JWT using the public certificate key published as per "jwks_uri" in https://<Environment FQDN>/.well-known/openid-configuration.<br/><br/>Use of the Claim: Header: 'kid' attribute MUST be used to locate the correct entry in the key set within the [/.well-known/jwks.json](https://auth.login.nhs.uk/.well-known/jwks.json).<br/><br/>This provides the mechanism for key rotation is supported. | | See Section 3.3 in Doc Ref #1<br/><br/>E.G. for production (https://auth.login.nhs.uk/.well-known/openid-configuration) the "jwks_uri" key resolves to https://auth.login.nhs.uk/.well-known/jwks.json where the public certifcicate is held.  The 'kid' attribute in the Header of the token is used to locate the correct key set within the /.well-known/jwks.json |
| 3 | Claim: Token: 'iss' | Compare to the fixed value provided by Aggregator team as part of the environment config/onboarding<br/><br/>(This will be as configured in NHS login) | [OIDC 1.0  - 3.1.3.7](https://openid.net/specs/openid-connect-core-1_0.html#IDTokenValidation) | |
| 4 | Claim: Token: 'exp' | The processing of this parameter requires that the current date/time MUST be before the expiration date/time listed in the value. Implementers MAY provide for some small leeway, usually no more than a few minutes, to account for clock skew | [OIDC 1.0  - 3.1.3.7](https://openid.net/specs/openid-connect-core-1_0.html#IDTokenValidation) | |
| 5 | Claim: Token: 'identity_proofing_level' | This MUST be set to "P9". Only users with Identity Level P9 are permitted to utilise the Aggregator | Doc Ref #1 | |
| 6 | Claim: Token: 'nhs_number' | This MUST match the NHS Number as provided in the URL search parameter. Assures the request from the Aggregator is for the person having performed authentication through NHS login | N/A | |
| 7 | Claim Token: 'birthdate' | This MUST match the date of birth held for the patient within the PEP | N/A | If value not present either as supplied from Aggregator or within PEP, treat as failure HTTP `401 Unauthorised` |

Any failure MUST result in a `401 Unauthorised` response, and details MUST be logged along with the associated `X-Request-Id` and `X-Correlation-Id` headers.

An OperationOutcome response must NOT be returned for steps #1 through #4.

Where validation failure occurs is for a custom claim (step #5, #6 or #7) or where validation failure occurs for a custom reason other than those specified here, an OperationOutcome response must be returned with a description stating the reason for the failure. This is to assist service management with diagnosis. 

The OperationOutcome is based on [HL7 FHIR R4 OperationOutcome](https://hl7.org/fhir/R4/operationoutcome.html)

For custom claim validations the value provided in the "diagnostics" attribute should relate to the detail of the claim, e.g. "Birthdate on token does not match birthdate on subject"

Where validation failure is for a custom reason other than those specified here, the value provided in the "diagnostics" attribute should relate to the detail of the validation but be preceded with "Other: ", e.g. "Other: lastname on token does not match surname on subject"

## An example Operation Outcome
```json
{
    "resourceType": "OperationOutcome",
    "issue": [{
            "severity": "error",
            "code": "security",
            "diagnostics": "Birthdate on token does not match birthdate on subject"
        }
    ]
}
```