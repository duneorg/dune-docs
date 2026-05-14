---
title: "Payments"
published: true
visible: true
taxonomy:
  audience: [developer]
  difficulty: [intermediate]
  topic: [extending, payments]
metadata:
  description: "Accept payments and manage subscriptions with Dune's Stripe integration"
---

# Payments

Dune's payment module wires Stripe checkout, webhook handling, and customer portal into your site with minimal configuration. When a user completes a purchase, Dune automatically assigns the configured role to their site user account.

Payments require [public site authentication](../../17.authentication/01.public-auth) to be configured — the payment flow reads the current `SiteUser` from the session.

## Configuration

```yaml
# site.yaml
payments:
  provider: "stripe"
  secret_key: "$STRIPE_SECRET_KEY"
  webhook_secret: "$STRIPE_WEBHOOK_SECRET"
  products:
    - id: "membership"
      name: "Monthly Membership"
      price_id: "price_1Abc123"       # Stripe Price ID from your dashboard
      role: "member"                  # Dune role assigned after payment
      mode: "subscription"            # "subscription" | "payment"
    - id: "lifetime"
      name: "Lifetime Access"
      price_id: "price_1Xyz789"
      role: "member"
      mode: "payment"
```

`secret_key` and `webhook_secret` support `"$ENV_VAR"` expansion — keep them out of committed config.

## Routes

When `payments:` is configured, three routes are registered:

| Route | Description |
|-------|-------------|
| `POST /payments/checkout/:productId` | Create a Stripe Checkout session and redirect to it |
| `POST /payments/webhook` | Receive Stripe webhook events (must be registered in Stripe dashboard) |
| `GET /payments/portal` | Redirect to Stripe Customer Portal for subscription management |

The checkout and portal routes require a logged-in `SiteUser`. Unauthenticated requests get a `401`.

## Checkout flow

1. User clicks a "Subscribe" button that POSTs to `/payments/checkout/membership`
2. Dune creates a Stripe Checkout Session with the configured price and `success_url` / `cancel_url` pointing back to your site
3. Dune returns `303 See Other` to the Stripe-hosted checkout page
4. User completes payment on Stripe
5. Stripe sends a `checkout.session.completed` event to `/payments/webhook`
6. Dune verifies the webhook signature, finds or creates the `SiteUser`, and assigns the configured `role`

## Webhook setup

Register your webhook in the [Stripe dashboard](https://dashboard.stripe.com/webhooks):

- **Endpoint URL**: `https://your-site.com/payments/webhook`
- **Events to listen for**: `checkout.session.completed`, `customer.subscription.deleted`

Copy the webhook signing secret (`whsec_…`) to `STRIPE_WEBHOOK_SECRET`.

Dune verifies the `Stripe-Signature` header on every webhook request using HMAC-SHA256. Requests with an invalid or missing signature are rejected with `400`.

## Role assignment

After a successful `checkout.session.completed` event, Dune adds the product's `role` to the `SiteUser.roles` array. This role can be used in [content gating](../../17.authentication/02.content-gating) rules:

```yaml
# In content frontmatter:
roles: member
```

For `subscription` mode products, Dune also handles `customer.subscription.deleted` — when a subscription is cancelled, the role is removed from the user.

## Customer portal

`GET /payments/portal` creates a Stripe Billing Portal session and redirects the user to it. The portal lets users manage their subscription, update payment methods, and view invoices.

Requires a logged-in user. The portal is pre-configured for the customer ID stored on the user's record after their first purchase.

## `@dune/core/ui` — SubscriptionForm

Use the built-in `SubscriptionForm` component to trigger checkout:

```tsx
import { SubscriptionForm } from "@dune/core/ui";

export default function PricingPage({ Layout, ...props }) {
  return (
    <Layout {...props}>
      <h1>Become a member</h1>
      <SubscriptionForm productId="membership" label="Subscribe — $10/month" />
    </Layout>
  );
}
```

The component POSTs to `/payments/checkout/{productId}` and shows a loading state while the redirect is in progress.

## Testing with Stripe test mode

Use Stripe test keys (`sk_test_…`, `whsec_…` from the test mode dashboard) during development. The Stripe CLI can forward webhooks to your local server:

```bash
stripe listen --forward-to localhost:3000/payments/webhook
```

Use [Stripe test card numbers](https://stripe.com/docs/testing#cards) (`4242 4242 4242 4242`) to simulate successful payments.
