# PayAlly

Agentic commerce for merchants. PayAlly lets a brand stand up its own AI-agent storefront — customers shop, pay, and track orders by talking to an AI assistant, presented entirely as the merchant's brand.

This is the umbrella repository. The two components are included as git submodules:

| Submodule | Repo | What it is |
|---|---|---|
| [`payally-partner`](https://github.com/varshney-ansh/payally-partner) | partner portal | AWS Amplify Gen2 + Next.js (Cloudscape) merchant dashboard. Merchants onboard and **one-click deploy** their own branded MCP storefront. |
| [`mcp-app`](https://github.com/varshney-ansh/payally-mcp-app) | per-brand MCP server | The agentic-commerce MCP server deployed **once per merchant** (branded via env). Product discovery, cart, real PayPal checkout, and order history, exposed as MCP tools + inline UI widgets. |

## How they fit together

The **partner portal** provisions, per merchant, a container running the **mcp-app** image (CloudFormation → ECS/Fargate behind a shared ALB), on the brand's own subdomain `https://<brand>.mcp.payally.world`. At deploy it injects the brand's identity (name, store noun, product nouns, categories), mints a per-brand AWS API Gateway key, and provisions a per-brand Auth0 Resource Server. The customer's AI agent then talks to a fully brand-native storefront — never PayAlly.

Both components are at **v1.0.0**.

## Clone with submodules

```bash
git clone --recurse-submodules https://github.com/varshney-ansh/payally.git

# or, after a plain clone:
git submodule update --init --recursive
```

To move a submodule to its latest commit:

```bash
git -C mcp-app pull origin master
git -C payally-partner pull origin master
git add mcp-app payally-partner && git commit -m "Bump submodules"
```
