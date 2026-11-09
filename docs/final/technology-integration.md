# Technology Integration

## Module 1: PayPal Checkout

| Tool / API | What It Does in Our Project |
|---|---|
| **PayPal Orders API (v2)** | Creates and manages checkout orders. Cart items are mapped to PayPal's `item` schema (`unit_amount`, `quantity`, `sku`, `category`) and sent to the PayAlly checkout endpoint, which calls PayPal to create the order. |
| **PayPal Merchant ID** | Each brand gets its own `PAYPAL_MERCHANT_ID` linked to their store. This is stored server-side (env var) and **never exposed** to the AI agent or customer — ensuring the white-label promise. |
| **PayPal Checkout Flow** | The `checkout` tool reads the user's cart server-side (prices cannot be manipulated by the AI), charges the buyer via PayPal, and returns the order confirmation. Cart is auto-discarded after successful payment. |

---

## Module 2: PayPal Customer Vault (Advanced Vaulting)

| Tool / API | What It Does in Our Project |
|---|---|
| **PayPal Vault Setup Tokens API** (`/v3/vault/setup-tokens`) | Creates a setup token to initiate the vaulting flow. Customers authorize PayAlly to store their PayPal payment method for automatic future payments inside AI apps. |
| **PayPal Payment Tokens API** (`/v3/vault/payment-tokens`) | After customer authorization, creates a vaulted payment token linked to the customer's email + merchant. Enables one-click checkout on repeat purchases. |
| **ADVANCED_VAULTING Product** | Enabled per merchant via PayPal partner onboarding. Supports `VAULT` and `BILLING_AGREEMENT` usage types with `IMMEDIATE` usage pattern. |
| **PayPal Partner Attribution** | Uses `PayPal-Partner-Attribution-Id` header to track PayAlly as the partner, enabling revenue sharing and partner reporting. |
| **Customer Lookup** | On checkout, looks up the buyer's vaulted PayPal details by email + merchant ID before creating the payment. |

### Vault Flow

1. **Customer logs in** to the brand's MCP app via Auth0
2. **PayAlly generates** a PayPal setup token via `/v3/vault/setup-tokens`
3. **Customer authorizes** PayAlly to vault their PayPal account — redirected to PayPal, returns with authorization
4. **PayAlly creates** the payment token via `/v3/vault/payment-tokens` — stored linked to customer email
5. **On repeat purchases**: checkout uses the vaulted token directly — no re-authentication required

---

## Module 3: PayPal Merchant Onboarding (Partner Flow)

| Tool / API | What It Does in Our Project |
|---|---|
| **PayPal Partner Referrals API** (`/v2/customer/partner-referrals`) | Creates a referral link for new merchants to sign up as PayPal partners. Includes `API_INTEGRATION` with `THIRD_PARTY` details for payment, refund, partner fee, vault, and billing agreement capabilities. |
| **PayPal Platform Access Token** | PayAlly obtains a platform-level access token via `PayPalPlatformAccessTokenFunction` (AWS Lambda) to act as the partner. Cached for 8 hours. |
| **PayPal Merchant Access Token** | Generates merchant-scoped JWT tokens via `PayPalMerchantAccessTokenFunction` for API calls on behalf of the merchant. |
| **PayPal Partner Attribution** | Uses `PayPal-Partner-Attribution-Id` header (`PAYALLY_PARTNER_ATTRIBUTION_ID`) in all API calls to track partner attribution and revenue share. |
| **Merchant Status Tracking** | Saves merchant signup info (email, PayPal merchant ID, capabilities) to the PayAlly database for tracking onboarding completion. |

### Onboarding Flow

1. **Merchant signs up** in PayAlly Partner Portal
2. **PayAlly generates** a partner referral link via `/v2/customer/partner-referrals`
3. **Merchant completes** PayPal onboarding — receives their `merchant_id`
4. **PayAlly tracks** onboarding status and saves merchant info
5. **Merchant is enabled** for EXPRESS_CHECKOUT + ADVANCED_VAULTING products
6. **Customers can now** vault their PayPal accounts and shop via AI

## AI Tool / Model / Platform

| Tool / API | What It Does in Our Project |
|---|---|
| **MCP (Model Context Protocol)** | The core interoperability layer. Every brand deploys a dedicated MCP server that exposes product search, cart, and checkout as tools. Any AI assistant that speaks MCP — ChatGPT, Claude, Gemini, etc. — can discover and use the brand's store. |
| **@modelcontextprotocol/server** | SDK used to build the branded MCP server. Registers tools (`explore-products`, `get-product-by-id`, `cart-add-items`, `cart-view`, `checkout`, `list-orders`) and handles the JSON-RPC transport. |
| **@modelcontextprotocol/node** | Provides `NodeStreamableHTTPServerTransport` to expose the MCP server over HTTP, enabling AI assistants to connect via standard HTTP POST. |
| **Auth0 (with AI Agents add-on)** | Secures agent-to-store authentication. The MCP server uses Auth0's OAuth 2.0 resource server flow to validate bearer tokens from AI agents, extract the buyer's identity (`sub`), and resolve their email via the Auth0 Management API. |

## How It Works Together

1. **Brand deploys** their white-labeled MCP server via the PayAlly Partner Portal
2. **Customer chats** with Claude/Gemini/ChatGPT: *"Find me a silk saree for a party"*
3. **AI assistant** calls the MCP `explore-products` tool → returns the brand's products
4. **Customer adds** items to cart via `cart-add-items` → cart stored in Valkey
5. **Customer checks out** via `checkout` → PayAlly API creates a PayPal order → customer pays → order confirmed
6. **Entire flow** happens inside the AI chat — no brand website visit required
---

## Testing Instructions for Judges

### Demo Merchant Credentials

| Field | Value |
|---|---|
| **Email** | amitsinghwilko9@gmail.com |
| **Password** | Anshvar9090@ |

### How Judges Can Test

1. **Log in** to the PayAlly Partner Portal at [payally.world](https://payally.world) with the demo merchant credentials above

2. **Connect the merchant's MCP endpoint** in any AI chat app:
   - **Claude**: Use the MCP configuration to connect to the merchant's store
   - **ChatGPT**: Configure the merchant's MCP server as a custom GPT
   - **Gemini**: Connect via MCP for Google AI Studio

3. **As a customer**, use a PayPal sandbox account to:
   - Browse products via AI chat
   - Add items to cart
   - Complete checkout with PayPal

4. **Verify the white-label experience**: The customer should only see the merchant's brand, not "PayAlly" — from product listings through checkout.

### What Judges Will See

- ✅ A working end-to-end flow from AI chat → product search → cart → PayPal checkout
- ✅ Real PayPal sandbox transactions
- ✅ 100% white-labeled experience (merchant brand only)
- ✅ Customer vault in action (optional — enables one-click repeat purchases)
---

## Overall Experience Rating & Feedback

### Challenges Faced

While integrating the PayPal Developer Platform, we encountered several obstacles:

1. **Unexpected API Responses**: PayPal API endpoints sometimes returned unexpected responses, including `Not Authorized`, `403 Forbidden`, and `Invalid syntax` errors. Each required individual investigation and debugging to resolve.

2. **API Version Confusion**: PayPal uses multiple API versions (v2, v3) across different endpoints, which created confusion about which version to integrate for each use case. For example:
   - Orders API uses `/v2/`
   - Vault/Customer Vaulting uses `/v3/vault/`
   - Partner referrals use `/v2/customer/partner-referrals`

3. **Documentation Navigation**: The PayPal REST API documentation, while comprehensive, was sometimes hard to navigate — especially for newer features like Advanced Vaulting and Partner Attribution.

### How We Resolved These Issues

- Spent considerable hours debugging each API response and tracing the request/response flow
- Consulted multiple PayPal developer resources and community forums
- Implemented proper error handling and logging to catch and diagnose unexpected responses
- Carefully mapped each feature to its correct API version and endpoint

### Recommendation for PayPal

Simplifying the API version landscape and providing clearer migration guides between v2/v3 would significantly improve the developer experience. Additionally, more detailed error messages in one place in sandbox mode would help developers diagnose issues faster.

---

*Despite these challenges, we successfully built a complete end-to-end integration covering merchant onboarding, customer vault, and checkout — all working in the PayPal sandbox environment.*