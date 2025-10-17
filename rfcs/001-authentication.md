# RFC-001: Authentication, Authorization & Subscription Management

**Author:** [Your Name]  
**Status:** Draft  
**Created:** 2025-10-16  
**Last Updated:** 2025-10-16

---

**Note on Authorship:**  
This RFC documents architectural decisions for a SaaS platform's authentication and subscription system. The technical design, technology choices (Firebase, Stripe, Neo4j, Elixir/Phoenix), and system architecture (Ports and Adapters pattern) reflect my engineering decisions. The RFC structure and task breakdown were developed collaboratively with AI assistance to ensure clarity and completeness.

This is a sanitized version of a production RFC, with product-specific details removed to protect competitive advantage.

---

## 1. Context

### Problem

A content management platform needs to authenticate users, determine what features they can access based on subscription tier, and handle payment processing without storing any PII/PCI data.

### Goals

- Secure authentication via Firebase (Google SSO)
- Subscription-based authorization (free vs paid tiers)
- Zero PII/PCI data in our systems
- Fast capability checks on every GraphQL request
- Swappable payment provider implementation

### Non-Goals (V1)

- Social login beyond Google
- Team/organization accounts
- API keys for programmatic access
- Granular per-feature permissions (just tier-based for now)
- Multi-user collaboration features

---

## 2. Requirements

### Must Have

- Validate Firebase JWT on every request
- Store user identity (Firebase UID, email) in Neo4j
- Store subscription metadata (tier, status, expiry) in Neo4j
- Handle Stripe webhooks for subscription events
- Inject user capabilities into GraphQL context
- Enforce capability limits in resolvers
- Bootstrap admin user on first Cloud Run deployment

### Nice to Have (Future)

- Token refresh handling
- Graceful degradation when subscription expires
- Admin override capabilities
- Customer portal integration

### Explicitly Out of Scope

- Storing payment methods (Stripe handles this)
- Handling disputes/chargebacks (Stripe handles this)
- Multi-user collaboration (V1 is single-user only)

---

## 3. High-Level Design

### Architecture Flow

```
SPA → Firebase Auth (Google) → JWT
  ↓
SPA → Phoenix API (JWT in Authorization header)
  ↓
AuthPlug validates JWT → Neo4j lookup User+Subscription
  ↓
Inject capabilities into conn.assigns
  ↓
GraphQL resolver checks capabilities → Execute or reject
```

### Ports and Adapters Pattern

Core principle: Business logic should not depend on Stripe directly. We define contracts (behaviors) and implement adapters.

```
Phoenix GraphQL (Presentation Layer)
       ↓
Accounts Context (Domain Layer)
       ↓
SubscriptionService (Port/Interface)
       ↓
StripeAdapter (Adapter/Implementation)
```

### Data Model (Neo4j)

```cypher
(User {
  firebase_uid: string,
  email: string,
  role: string,           # "user" | "admin"
  created_at: datetime,
  updated_at: datetime
})
  -[:HAS_SUBSCRIPTION]->
(Subscription {
  tier: string,                    # "free" | "paid"
  status: string,                  # "active" | "canceled" | "past_due"
  stripe_customer_id: string,
  stripe_subscription_id: string,
  started_at: datetime,
  current_period_end: datetime,
  created_at: datetime,
  updated_at: datetime
})
```

### Subscription Tiers (V1)

**Free Tier:**
- Limited resource creation
- Basic features only
- No access to premium features

**Paid Tier:**
- Unlimited resource creation
- Access to all premium features
- Usage-based premium feature allocation

**Admin Role:**
- Bypasses all limits
- Full access regardless of subscription

### Stripe Integration

**Webhook Events:**
- `customer.subscription.created` → Create subscription in Neo4j
- `customer.subscription.updated` → Update subscription status/tier
- `customer.subscription.deleted` → Downgrade to free tier

**Flow:**
1. User clicks "Upgrade"
2. Phoenix creates Stripe checkout session
3. User completes payment on Stripe
4. Stripe sends webhook to Phoenix
5. Phoenix validates webhook signature
6. Phoenix updates Neo4j subscription node
7. Next request: user has paid tier capabilities

---

## 4. Architecture - Ports and Adapters

### Behavior Definition

```elixir
# lib/app/subscription_service.ex
defmodule App.SubscriptionService do
  @moduledoc """
  Port/Interface for subscription management.
  Defines the contract for payment provider adapters.
  """
  
  @type subscription_tier :: :free | :paid
  @type checkout_result :: {:ok, checkout_url :: String.t()} | {:error, reason :: term()}
  @type webhook_event :: %{
    type: String.t(),
    customer_id: String.t(),
    subscription_id: String.t(),
    status: String.t(),
    current_period_end: DateTime.t()
  }
  
  @callback create_checkout_session(user_id :: String.t(), tier :: subscription_tier()) :: 
    checkout_result()
  
  @callback handle_webhook(payload :: map(), signature :: String.t()) :: 
    {:ok, webhook_event()} | {:error, term()}
  
  @callback cancel_subscription(subscription_id :: String.t()) :: 
    :ok | {:error, term()}
  
  @callback get_customer_portal_url(customer_id :: String.t()) :: 
    {:ok, String.t()} | {:error, term()}
end
```

### Adapter Configuration

```elixir
# config/config.exs
config :app, :subscription_adapter, App.Adapters.StripeAdapter

# config/test.exs
config :app, :subscription_adapter, App.Adapters.MockSubscriptionAdapter
```

### Domain Usage

```elixir
# lib/app/accounts.ex
defmodule App.Accounts do
  @subscription_adapter Application.compile_env(:app, :subscription_adapter)
  
  def upgrade_to_paid(user) do
    with {:ok, checkout_url} <- @subscription_adapter.create_checkout_session(user.id, :paid) do
      {:ok, checkout_url}
    end
  end
  
  def process_webhook(payload, signature) do
    with {:ok, event} <- @subscription_adapter.handle_webhook(payload, signature),
         {:ok, user} <- get_user_by_customer_id(event.customer_id),
         {:ok, _subscription} <- update_subscription(user, event) do
      :ok
    end
  end
end
```

---

## 5. Task Breakdown (2-hour increments)

### Task 1: Firebase JWT Validation Plug

**Objective:** Create authentication plug that validates Firebase JWTs

**Implementation:**
- Create authentication plug module
- Verify JWT signature using Firebase public keys
- Extract `uid` and `email` from token claims
- Return 401 Unauthorized if invalid/expired
- Handle missing Authorization header gracefully

**Success Criteria:**
- Can call protected endpoint with valid JWT → 200 OK
- Call with invalid JWT → 401 Unauthorized
- Call without JWT → 401 Unauthorized

---

### Task 2: Neo4j User Schema

**Objective:** Define User node structure and CRUD operations

**Implementation:**
- Define User node structure with fields: `firebase_uid`, `email`, `role`, `created_at`, `updated_at`
- Create Cypher queries:
    - `create_user/3` (uid, email, role)
    - `get_user_by_uid/1`
    - `get_user_by_email/1`
- Handle first-time user creation (auto-create on first valid JWT)
- Default new users to `:user` role and free tier subscription

**Success Criteria:**
- New user auto-created on first valid JWT request
- User node stored in Neo4j with correct structure
- Can retrieve user by Firebase UID

---

### Task 2.5: Admin Existence Check

**Objective:** Create query to check if any admin users exist

**Implementation:**
- Create `admin_exists?/0` function
- Cypher query: `MATCH (u:User {role: 'admin'}) RETURN count(u) > 0`
- Return boolean result

**Success Criteria:**
- Returns `true` if any admin exists
- Returns `false` if no admins exist
- Query performs efficiently (indexed on role)

---

### Task 2.6: Bootstrap Admin (Cloud Run)

**Objective:** Auto-create admin user on first Cloud Run deployment

**Implementation:**
- Update `entrypoint.sh` to check for `BOOTSTRAP_ADMIN_UID` env var
- If set AND `admin_exists?()` returns `false` → promote that Firebase UID to admin
- If admin already exists OR env var not set → skip (no-op)
- Add logging for debugging

**Entrypoint script:**
```bash
#!/bin/bash
set -e

# Bootstrap admin if needed (idempotent)
if [ -n "$BOOTSTRAP_ADMIN_UID" ]; then
  echo "Checking for existing admins..."
  mix run -e "
    case App.Accounts.admin_exists?() do
      false -> 
        IO.puts(\"Creating bootstrap admin: #{System.get_env(\"BOOTSTRAP_ADMIN_UID\")}\")
        App.Accounts.promote_to_admin(System.get_env(\"BOOTSTRAP_ADMIN_UID\"))
      true -> 
        IO.puts(\"Admin already exists, skipping bootstrap\")
    end
  "
fi

# Start Phoenix
exec mix phx.server
```

**Cloud Run deployment:**
```bash
gcloud run deploy app-api \
  --image=gcr.io/your-project/app \
  --set-env-vars="BOOTSTRAP_ADMIN_UID=your-firebase-uid,NEO4J_URI=...,NEO4J_PASSWORD=..."
```

**Success Criteria:**
- Deploy with `BOOTSTRAP_ADMIN_UID` set → first container start creates admin
- Subsequent container starts/restarts → skip promotion (idempotent)
- Multiple containers starting simultaneously → only one creates admin
- Deploy without env var → no admin created (safe default)

---

### Task 3: Neo4j Subscription Schema

**Objective:** Define Subscription node structure and relationships

**Implementation:**
- Define Subscription node structure with fields:
    - `tier`: "free" | "paid"
    - `status`: "active" | "canceled" | "past_due"
    - `stripe_customer_id`
    - `stripe_subscription_id`
    - `started_at`
    - `current_period_end`
    - `created_at`
    - `updated_at`
- Create relationship: `(User)-[:HAS_SUBSCRIPTION]->(Subscription)`
- Create Cypher queries:
    - `create_subscription/2` (user_id, attrs)
    - `get_subscription/1` (user_id)
    - `update_subscription/2` (user_id, attrs)
    - `cancel_subscription/1` (user_id)

**Success Criteria:**
- Can create subscription linked to user
- Can retrieve subscription for user
- Can update subscription attributes
- Relationship properly traversable in both directions

---

### Task 4: Define SubscriptionService Behavior

**Objective:** Create interface contract for payment providers

**Implementation:**
- Create subscription service module with `@callback` definitions
- Define types: `subscription_tier`, `checkout_result`, `webhook_event`
- Define callbacks:
    - `create_checkout_session/2`
    - `handle_webhook/2`
    - `cancel_subscription/1`
    - `get_customer_portal_url/1`
- Add configuration for adapter selection
- Document behavior in moduledoc

**Success Criteria:**
- Behavior module compiles without errors
- Can be implemented by adapters
- Type specs are clear and correct
- Configuration allows adapter swapping

---

### Task 5: Mock SubscriptionAdapter (Testing)

**Objective:** Create test adapter for subscription service

**Implementation:**
- Create mock subscription adapter module
- Implement subscription service behavior
- Return hardcoded success responses:
    - Checkout session → fake URL
    - Webhook handling → normalized event struct
    - Cancel → success
    - Portal URL → fake URL
- No external API calls

**Success Criteria:**
- All behavior callbacks implemented
- Can run tests without Stripe API
- Swap adapter via config in test environment
- Mock responses match expected types

---

### Task 6: Stripe Adapter - Checkout Session

**Objective:** Implement Stripe checkout session creation

**Implementation:**
- Create Stripe adapter module
- Implement subscription service behavior
- Implement `create_checkout_session/2`:
    - Call Stripe API to create checkout session
    - Map tier → Stripe price ID
    - Include success/cancel URLs
    - Return checkout URL
- Add Stripe API key to config
- Add Stripe library dependency

**Success Criteria:**
- Can generate valid Stripe checkout URL
- Price ID correctly maps to tier
- Checkout session created in Stripe dashboard
- Error handling for API failures

---

### Task 7: Stripe Adapter - Webhook Handling

**Objective:** Parse and validate Stripe webhooks

**Implementation:**
- Implement `handle_webhook/2` in Stripe adapter:
    - Verify Stripe webhook signature using webhook secret
    - Parse webhook payload
    - Map Stripe events → normalized `webhook_event` struct
    - Handle events:
        - `customer.subscription.created`
        - `customer.subscription.updated`
        - `customer.subscription.deleted`
- Extract relevant fields: customer_id, subscription_id, status, period_end
- Return errors for invalid signatures

**Success Criteria:**
- Webhook signature verified correctly
- Valid webhooks parsed into normalized struct
- Invalid signatures rejected
- All subscription lifecycle events handled

---

### Task 8: Accounts Context - Subscription Management

**Objective:** Implement domain logic for subscriptions

**Implementation:**
- Create accounts domain module
- Implement functions:
    - `upgrade_to_paid/1` - calls adapter, returns checkout URL
    - `process_webhook/2` - calls adapter, updates Neo4j
    - `get_user_subscription/1` - loads from Neo4j
    - `cancel_subscription/1` - calls adapter + updates Neo4j
    - `admin_exists?/0` - checks for admin users
    - `promote_to_admin/1` - sets user role to admin
- Domain logic isolated from payment provider details

**Success Criteria:**
- All subscription operations work through domain layer
- Adapter calls isolated in accounts context
- Neo4j updates happen transactionally
- Error handling propagates cleanly

---

### Task 9: GraphQL Mutations - Subscription

**Objective:** Expose subscription operations via GraphQL

**Implementation:**
- Add mutation: `createCheckoutSession` → returns checkout URL
- Add mutation: `cancelSubscription` → cancels and returns new status
- Add query: `mySubscription` → returns current subscription details
- Add query: `customerPortalUrl` → returns Stripe portal URL
- Require authentication for all operations

**Success Criteria:**
- SPA can trigger checkout via GraphQL
- Users can view subscription status
- Users can cancel subscription
- Unauthenticated requests rejected

---

### Task 10: Webhook Endpoint

**Objective:** Create HTTP endpoint for Stripe webhooks

**Implementation:**
- Create Phoenix webhook controller
- Add route: `POST /webhooks/stripe`
- Call accounts context with raw body and signature
- Return 200 on success
- Return 400 on invalid signature
- Return 500 on processing errors (but acknowledge receipt)
- Log all webhook events for debugging

**Success Criteria:**
- Stripe webhook updates Neo4j subscription within 30s
- Invalid webhooks rejected with 400
- Webhook events logged for audit
- No authentication required (signature validates)

---

### Task 11: Capability Injection

**Objective:** Load user capabilities into request context

**Implementation:**
- Define Capabilities struct with fields for tier-based limits
- Update auth plug to load subscription after user lookup
- Map subscription tier → capabilities
- Inject into connection assigns
- Pass to GraphQL context

**Success Criteria:**
- Every authenticated request has capabilities in context
- GraphQL resolvers can access capabilities
- Admin users have admin flag set
- Free/Paid tiers have correct limits

---

### Task 12: Capability Mapping Logic

**Objective:** Map subscription tiers to feature limits

**Implementation:**
- Create capability mapping function
- Define tier limits with configurable resource constraints
- Handle admin override (all limits → unlimited)
- Calculate remaining usage from current consumption

**Success Criteria:**
- Capabilities struct correctly reflects subscription tier
- Admin users have unlimited everything
- Usage tracking calculated correctly
- Expired subscriptions default to free tier

---

### Task 13: GraphQL Authorization Middleware

**Objective:** Enforce capability limits in GraphQL resolvers

**Implementation:**
- Create authorization middleware module
- Implement capability checking helper
- Use in GraphQL schema to protect mutations
- Return clear, user-friendly error messages

**Success Criteria:**
- Mutations blocked when capability exceeded
- Error messages explain limit and upgrade path
- Admin users bypass all checks
- Middleware composable with other checks

---

### Task 14: Subscription Expiry Handling

**Objective:** Handle expired subscriptions gracefully

**Implementation:**
- In auth plug, check subscription expiry
- If expired AND status is still "active" → log warning
- Rely primarily on Stripe webhooks for status updates
- Expired/canceled subscriptions → load as free tier capabilities
- Grace period handled by Stripe (past_due status)

**Success Criteria:**
- Expired paid users automatically get free tier limits
- No manual cron jobs needed (webhook-driven)
- Graceful degradation (existing data accessible, creation blocked)
- Clear messaging to user about expired status

---

### Task 15: Customer Portal Integration

**Objective:** Allow users to manage subscriptions via Stripe

**Implementation:**
- Implement function to get customer portal URL
- Calls Stripe adapter method
- Stripe API creates portal session
- Add GraphQL query returning portal URL
- Portal allows: cancel, update payment, view invoices

**Success Criteria:**
- User can access subscription management portal
- Redirected to Stripe customer portal
- Can cancel subscription via portal
- Webhook updates Neo4j when changes made

---

## 6. Data Flow Examples

### Upgrade Flow

```
User clicks "Upgrade to Paid"
  ↓
GraphQL mutation: createCheckoutSession
  ↓
Accounts.upgrade_to_paid(user)
  ↓
SubscriptionAdapter.create_checkout_session(user.id, :paid)
  ↓
StripeAdapter calls Stripe API
  ↓
Returns checkout URL to SPA
  ↓
User completes payment on Stripe
  ↓
Stripe webhook → POST /webhooks/stripe
  ↓
WebhookController.handle/2
  ↓
Accounts.process_webhook(payload, signature)
  ↓
StripeAdapter.handle_webhook → normalizes event
  ↓
Accounts updates Neo4j subscription node
  ↓
Next request: AuthPlug loads updated subscription → Paid capabilities
```

### Authorization Flow

```
GraphQL mutation: createResource
  ↓
Authorize middleware checks capabilities
  ↓
Query current resource count from Neo4j
  ↓
Current count < max? → Proceed to resolver
  ↓
Current count >= max? → Return error with upgrade message
  ↓
Admin user? → Bypass limit, proceed
```

---

## 7. Success Criteria

### End-to-End Test

1. Deploy to Cloud Run with `BOOTSTRAP_ADMIN_UID` set
2. Bootstrap admin created on first container start
3. Subsequent restarts skip admin creation (idempotent)
4. Admin logs in → admin privileges, unlimited capabilities
5. New user logs in with Google → auto-created with free tier
6. User creates resources up to free tier limit → succeeds
7. User tries to exceed limit → error with upgrade message
8. User calls `createCheckoutSession` mutation → receives Stripe checkout URL
9. User completes payment on Stripe → Stripe webhook fires
10. Webhook updates Neo4j → user now has paid subscription
11. User can now create unlimited resources (paid capabilities active)
12. User calls `customerPortalUrl` query → can manage subscription
13. User cancels subscription via portal → webhook fires → downgraded to free
14. User tries to create more resources → blocked at free tier limits

### Testing with Mock Adapter

1. Set mock adapter in test config
2. All subscription flows work without Stripe API
3. Can simulate webhook events in tests
4. Tests run fast (no network calls)
5. Swap to Stripe adapter in production config

---

## 8. Open Questions

1. **Usage tracking:** Per-month or rolling 30 days? How do we reset counters?
2. **Grace period:** Stripe marks subscriptions as `past_due` before `canceled` - how do we handle this UI-wise?
3. **Prorated upgrades:** Do we handle mid-month upgrades? (Stripe does this automatically, but document it)
4. **Multiple payment providers:** Future support for alternative providers? (Interface makes this easier)
5. **Team subscriptions:** Out of scope for V1, but how would the interface need to change?
6. **Refunds:** Do we handle refund webhooks? What happens to user data?

---

## 9. Future Enhancements (Post-V1)

- Admin panel for subscription management (view all subscriptions, cancel, issue refunds)
- Usage analytics dashboard (track feature usage per user)
- Additional subscription tiers (free/paid/enterprise)
- Annual billing with discount
- Referral program with credits
- Pause subscription feature (Stripe supports this)
- Lifetime access tier (one-time payment)
- Usage-based billing for premium features

---

## 10. References

- [Stripe API Documentation](https://stripe.com/docs/api)
- [Firebase Auth Documentation](https://firebase.google.com/docs/auth)
- [Elixir Behaviours](https://hexdocs.pm/elixir/behaviours.html)
- [Ports and Adapters Pattern](https://alistair.cockburn.us/hexagonal-architecture/)

---

**End of RFC-001**