# 06. References

## Exam
- [AWS Expands Its Agentic AI Microcredentials (Builder Center, Jorge Rivera)](https://builder.aws.com/content/3KBfi05HDC809sba5a0IqTJllDT/aws-expands-its-agentic-ai-microcredentials-two-new-hands-on-credentials-for-builders). Third-party summary. Its details match the exam facts used in `plans.md`.
- [AWS Launches Free Hands-On Credential for Securing AI Agent Identities (The Cybersecurity Trail)](https://thecybersecuritytrail.com/news/aws-free-securing-agent-identities-microcredential/). Source for the 135-minute length, objectives, 12-month validity, and 25-day retry wait.
- [Prove your agentic AI skills: Two new microcredentials now available (AWS blog)](https://aws.amazon.com/blogs/training-and-certification/prove-your-agentic-ai-skills-two-new-microcredentials-now-available/)
- Official assessment page: search "Securing Agent Identities Demonstrated" on AWS Skill Builder and re-check objectives before the exam.

## AgentCore developer guide
- [Identity overview](https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/identity.html)
- [Inbound and outbound auth (Runtime, Cognito, JWT)](https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/runtime-oauth.html)
- [Configure inbound JWT authorizer](https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/inbound-jwt-authorizer.html)
- [Get workload access token](https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/get-workload-access-token.html)
- [Create and manage workload identities](https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/creating-agent-identities.html)
- [Tagging AgentCore Identity resources](https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/identity-tagging.html)
- [Scope down access to credential providers by workload identity](https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/scope-credential-provider-access.html)
- [Outbound credential providers](https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/identity-outbound-credential-provider.html)
- [OAuth 2.0 authorization URL session binding](https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/oauth2-authorization-url-session-binding.html)
- [Private identity providers](https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/identity-private-idp.html)
- [Runtime security best practices](https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/runtime-security-best-practices.html)
- [Resource-based policies](https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/resource-based-policies.html)
- [Identity observability data (metrics, spans, logs)](https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/observability-identity-metrics.html)
- [Observability and audit logging overview](https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/observability.html)
- [Service Authorization Reference for AgentCore](https://docs.aws.amazon.com/service-authorization/latest/reference/list_amazonbedrockagentcore.html)

## AWS blogs
- [Introducing Amazon Bedrock AgentCore Identity](https://aws.amazon.com/blogs/machine-learning/introducing-amazon-bedrock-agentcore-identity-securing-agentic-ai-at-scale)
- [Secure AI agents with AgentCore Identity on Amazon ECS](https://aws.amazon.com/blogs/machine-learning/secure-ai-agents-with-amazon-bedrock-agentcore-identity-on-amazon-ecs/). Self-hosted pattern.
- [Reference your own Secrets Manager secrets in AgentCore Identity](https://aws.amazon.com/blogs/machine-learning/reference-your-own-aws-secrets-manager-secrets-in-amazon-bedrock-agentcore-identity/)
- [Securing agentic workflows: identity propagation (Builder Center)](https://builder.aws.com/content/3EOwdM2gsKqzXKO7OmyfWcRATWV/securing-agentic-workflows-identity-propagation-with-amazon-bedrock-agentcore)
- [Secure agentic access to AWS with Enterprise-Managed Authorization](https://aws.amazon.com/blogs/security/secure-agentic-access-to-aws-with-enterprise-managed-authorization/)
- [Scope down permissions of existing sessions using STS GetScopedCredentials](https://aws.amazon.com/blogs/security/scope-down-permissions-of-existing-sessions-using-aws-sts-getscopedcredentials/)

## Free hands-on code (replaces paid SimuLearn)
- [Starter toolkit: Identity quickstart](https://github.com/aws/bedrock-agentcore-starter-toolkit/blob/main/documentation/docs/user-guide/identity/quickstart.md)
- [Onboarding repo, module 06_identity](https://github.com/aws-samples/sample-amazon-bedrock-agentcore-onboarding/blob/main/06_identity/README.md)
- [Prototype-to-production workshop (module 5: Identity)](https://github.com/aws-samples/sample-amazon-bedrock-agentcore-prototype-to-production)
- [awslabs/agentcore-samples](https://github.com/awslabs/agentcore-samples)

## Videos (search terms only, not verified links)
Search YouTube. Prefer the AWS Developers and Amazon Web Services channels. Check the upload date, because the service changes quickly.
- "AgentCore Identity re:Invent"
- "AgentCore Identity deep dive"
- "AgentCore workload identity OAuth"
- "JWT explained", "OIDC discovery document"

## Your original training list (optional)
AWS Cloud Quest: Security. AWS Cloud Quest: Generative AI Architect. AWS SimuLearn: Getting Started with AgentCore Runtime. AWS SimuLearn: Identity and Access Management. Building Production-Ready AI Agents with Amazon Bedrock AgentCore. SimuLearn needs payment, so this kit replaces it with the free docs and labs above.
