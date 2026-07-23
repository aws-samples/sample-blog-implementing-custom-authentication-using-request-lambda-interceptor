# Custom Authentication with Request Lambda Interceptor

## Introduction

When deploying AI agents with [Amazon Bedrock AgentCore](https://aws.amazon.com/bedrock/agentcore/), organizations benefit from built-in support for OAuth 2.0, [AWS Identity and Access Management (IAM)](https://aws.amazon.com/iam/), and API key authentication through [Amazon Bedrock AgentCore Gateway](https://aws.amazon.com/blogs/machine-learning/introducing-amazon-bedrock-agentcore-gateway-transforming-enterprise-ai-agent-tool-development/). However, some enterprise environments still utilize older authentication mechanisms such as HTTP Basic Authentication ([RFC 7617](https://datatracker.ietf.org/doc/html/rfc7617)). AgentCore Gateway's extensible architecture makes it straightforward to support these authentication mechanisms through [request Lambda interceptors](https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/gateway-interceptors.html) — custom code that runs each time an agent calls a tool.

This tutorial shows how to use a request Lambda interceptor to authenticate to a downstream tool API using system credentials — retrieving a service account credential from [AWS Secrets Manager](https://aws.amazon.com/secrets-manager/) and constructing a Basic Auth header on behalf of the calling agent. This design keeps credentials isolated from the agent runtime, preventing exposure through model-driven behavior such as prompt injection.

**Important**: HTTP Basic Authentication is an antiquated technology that transmits credentials as Base64-encoded text and should not be used as a long-term authentication strategy. AWS recommends modernizing to OAuth 2.0, SAML, or OpenID Connect where possible. However, some organizations with legacy workloads choose to decouple authentication modernization from their agentic AI adoption, addressing each on independent timelines. If your environment requires Basic Auth integration as an interim measure, consult your AWS Solutions Architect to evaluate the security trade-offs before proceeding.

![Architecture Flow](images/Figure-1.png)

### How the Interceptor Works

When a request flows through the gateway, the interceptor Lambda validates the inbound JWT as a defense-in-depth measure, retrieves the system service account credential from Secrets Manager, and constructs the Basic Auth header for the downstream tool:

```
Agent (Bearer token) → Gateway (validates JWT) → Interceptor Lambda → Target Tool API
                                                       ↓
                                                 1. Validates JWT (defense-in-depth)
                                                 2. Retrieves system credential
                                                    from Secrets Manager (KMS-encrypted)
                                                 3. Constructs Basic Auth header
```

### Tutorial Details

| Information | Details |
|:---------------------|:-------------------------------------------------------------|
| Tutorial type | Interactive |
| AgentCore components | AgentCore Gateway |
| Gateway Target type | AWS Lambda |
| Inbound Auth IdP | Amazon Cognito (can be adapted to work with OIDC providers) |
| Outbound Auth | Request Lambda Interceptor (system credential) |
| Tutorial components | Gateway, Interceptor, Tool API, Secrets Manager, KMS |
| Tutorial vertical | Cross-vertical |
| Example complexity | Intermediate |
| SDK used | boto3, PyJWT |

### Key Features

* **JWT validation (defense-in-depth)** — Interceptor independently validates the JWT signature, expiration, and issuer, protecting against bypass scenarios
* **System credential isolation** — The agent runtime never has access to Secrets Manager; only the interceptor retrieves credentials
* **KMS encryption** — System credential encrypted with a customer-managed KMS key
* **Trusted subsystem pattern** — The interceptor authenticates to the downstream tool as a service account; the tool authenticates the credential against Active Directory

## Tutorial

- [Custom Authentication with Request Lambda Interceptor](custom-auth-interceptor.ipynb)

## Resources

* [Gateway Request Interceptor — AWS Documentation](https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/gateway-interceptors.html)
* [AWS Secrets Manager — Developer Guide](https://docs.aws.amazon.com/secretsmanager/latest/userguide/intro.html)
* [Rotate Active Directory credentials stored in AWS Secrets Manager](https://aws.amazon.com/blogs/modernizing-with-aws/rotate-active-directory-credentials-stored-in-aws-secrets-manager/)
* [Header Propagation with Gateway — AWS Documentation](https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/gateway-headers.html)

## Conclusion

A request Lambda interceptor in AgentCore Gateway can bridge the gap between the authentication patterns supported by Gateway and the authentication requirements of legacy tool APIs that have not yet migrated to modern authentication standards. As demonstrated in this tutorial, the interceptor validates the inbound JWT, retrieves system credentials from Secrets Manager, and constructs the downstream tool's Basic Auth header — without modifying tool schemas or agent implementation.

This approach is an interim integration pattern, not a target architecture. It introduces a credential that must be synchronized between Secrets Manager and the tool's identity store (such as Active Directory), adding operational overhead for rotation, drift detection, and lifecycle management. The recommended path is to modernize the downstream tool to accept OAuth 2.0, SAML, or OpenID Connect — eliminating stored credentials entirely. Until that modernization is complete, the interceptor isolates credential handling from the agent runtime, ensuring that the agent — a non-deterministic system influenced by user prompts — never has access to authentication secrets.
