# Cloud Platforms and Enterprise Deployment: Bedrock, Vertex AI, Auth, and Data Governance

## What you will learn

- The ways to reach Claude: the first-party Claude API, Amazon Bedrock, and Google Cloud Vertex AI.
- How the SDK client, model IDs, and authentication change between platforms (and what stays the same).
- How to choose a platform based on governance, billing, region, and feature needs.
- Enterprise patterns: gateways, least-privilege credentials, secrets handling, and data retention.
- Common exam traps about "same model, different platform".

## Explanation

### One model family, several doors

Claude is available through more than one provider:

| Platform | Typical reason to choose it |
|---|---|
| Claude API (first party) | Newest features first, simplest setup, direct billing |
| Amazon Bedrock | Company already standardizes on AWS: IAM, VPC endpoints, CloudWatch, consolidated AWS billing |
| Google Cloud Vertex AI | Company already standardizes on GCP: IAM, Cloud Logging, committed spend |

The model is the same family on each, and the core request shape (messages, system prompt, tools, `max_tokens`) is the same. What differs is the *wrapper*: authentication, model identifiers, region handling, and sometimes the availability date of newer or beta features.

### What changes in code

The Anthropic SDKs ship platform-specific clients. The conversation logic stays identical; only client construction changes.

```python
from anthropic import Anthropic, AnthropicBedrock, AnthropicVertex

# First party: API key from ANTHROPIC_API_KEY
client = Anthropic()

# Bedrock: uses AWS credentials (env, profile, or IAM role) and an AWS region
client = AnthropicBedrock(aws_region="us-east-1")

# Vertex AI: uses Google Application Default Credentials, a project, and a region
client = AnthropicVertex(project_id="my-project", region="us-east5")

# The rest of the code is the same on all three:
resp = client.messages.create(
    model=MODEL_ID,          # the ID format depends on the platform
    max_tokens=512,
    messages=[{"role": "user", "content": "Summarize this ticket."}],
)
```

Things to remember:

- **Model IDs differ per platform.** Bedrock and Vertex use their own identifier formats, so look the ID up in that platform's catalog. Never hard-code a first-party ID into a Bedrock deployment.
- **Authentication differs.** First party uses an API key. Bedrock uses AWS IAM (roles, policies). Vertex uses Google IAM (service accounts). On the cloud platforms you normally avoid long-lived keys and rely on workload identity or attached roles.
- **Feature availability can lag.** A new beta feature (for example a new tool type or header) may reach the first-party API before the cloud platforms. Always check the platform's documentation before designing around a feature.
- **Region matters.** Model availability, quotas, and latency are per region. Some platforms offer cross-region routing to improve throughput, which interacts with data residency rules.

### Choosing a platform

Ask these questions in order:

1. **Governance:** Must traffic stay inside an existing cloud account, VPC, or private network? Must access be controlled by existing IAM policies and audited in the existing logging system? That favors Bedrock or Vertex.
2. **Procurement and billing:** Is spend committed to a cloud provider? Then using that provider's marketplace can consume the commitment.
3. **Features:** Do you need a feature that is only on the first-party API today? Then either use it, or wait.
4. **Data residency:** Does the law or contract require processing in a given region? Pick a platform and region that guarantee it, and verify cross-region routing settings.
5. **Operations:** Who will run it? A team fluent in AWS will be safer on Bedrock than on a platform they have never operated.

There is rarely a *quality* difference to weigh; it is an operational and governance decision.

### Enterprise patterns

**Central gateway.** Instead of every service holding credentials and calling Claude directly, route calls through one internal gateway (or LLM proxy). It gives you a single place for:

- authentication of internal callers and per-team quotas,
- logging and cost attribution (ties back to observability, lesson 021),
- retries and fallbacks (lesson 016),
- input/output guardrails and PII redaction (lesson 008),
- swapping platform or model without changing every application.

**Least privilege.** Grant the calling role permission to invoke only the specific models it needs, nothing broader. Separate roles for production and experimentation.

**Secrets.** Keep API keys in a secrets manager, never in code, prompts, CLAUDE.md, or the repository. On cloud platforms prefer role-based identity so there is no key to leak.

**Data handling.** Decide what may be sent to the model at all. Minimize: send only the fields the task needs. Mask or tokenize identifiers where possible. Understand the retention and training-use terms of the platform you choose, and check them in the provider's current policy rather than assuming. Logs of prompts and responses are themselves sensitive data and need the same access control and retention limits.

**Claude Code and the Agent SDK on cloud platforms.** Both can be configured to use Bedrock or Vertex instead of the first-party API, typically through environment variables that select the provider plus the usual cloud credentials. This lets an enterprise run coding agents and CI workers (see `ci-notes.md`) under its existing cloud identity and billing.

### Small example: a platform-agnostic factory

```python
import os

def make_client():
    provider = os.environ.get("CLAUDE_PROVIDER", "anthropic")
    if provider == "bedrock":
        return AnthropicBedrock(aws_region=os.environ["AWS_REGION"])
    if provider == "vertex":
        return AnthropicVertex(project_id=os.environ["GCP_PROJECT"],
                               region=os.environ["GCP_REGION"])
    return Anthropic()
```

Keep the model ID in configuration, per provider, so switching is a config change, not a code change.

## Common exam traps

- **"Bedrock/Vertex run a different, weaker model."** No: same model family; differences are in wrapper, IDs, auth, and feature timing.
- **Reusing a first-party model ID or API key on another platform.** Each platform has its own ID format and its own authentication.
- **Assuming every new feature is on every platform on day one.** Check availability.
- **Putting API keys in the prompt, repo, or CLAUDE.md.** Use a secrets manager or role-based identity.
- **Treating platform choice as a quality decision.** It is mainly governance, billing, residency, and operations.
- **Ignoring that prompt logs are sensitive data.** Logging everything without access control creates a new leak path.
- **Overlooking cross-region routing** when strict data residency is required.

## Practice questions

<details>
<summary>Show questions and answers</summary>

**1. A company runs everything on AWS and requires that all model access be governed by IAM and audited in its existing logging. Which option fits best, and what changes in the code?**

Amazon Bedrock. Construct the Bedrock client with an AWS region and use IAM role credentials instead of an API key, and use Bedrock's model identifier. The messages, system prompt, and tool logic stay the same.

**2. A developer copies a working first-party model ID into a Vertex AI deployment and gets a "model not found" error. What is the most likely cause, and how do you avoid it?**

Model identifiers are platform-specific. Look up the ID in Vertex AI's model catalog (and check it is enabled in the chosen region), and store model IDs per provider in configuration.

**3. Your team wants per-team quotas, cost attribution, PII redaction, and the ability to switch providers without touching every app. What architecture do you propose?**

A central internal gateway (LLM proxy) in front of Claude. Applications call the gateway; it authenticates callers, enforces quotas, logs usage, applies guardrails and redaction, handles retries/fallbacks, and holds the only provider credentials (preferably role-based identity).

</details>

## Key takeaways

- Claude is reachable via the first-party API, Amazon Bedrock, and Google Cloud Vertex AI. It is the same model family with a different wrapper.
- Client class, authentication, model IDs, regions, and feature timing change per platform. The messages logic does not.
- Choose by governance, billing, data residency, features, and operations, not by model quality.
- Prefer role-based identity and a secrets manager. Never embed keys in code or prompts.
- A central gateway gives one place for auth, quotas, logging, guardrails, and provider switching.
- Treat prompt and response logs as sensitive data.
