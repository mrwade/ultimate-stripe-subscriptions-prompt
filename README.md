# The Ultimate Stripe Subscriptions Prompt

Use the following as a basis for prompting Cursor (or other code generation tools) to implement Stripe subscriptions in your code base. I recommend you prompt one step at a time.

The following is based on how we implemented billing in our project management app, [Baton](https://www.usebaton.com/). You can see full walkthrough of implementation via Cursor in this [YouTube video](https://www.youtube.com/watch?v=Lfbu5dwaW3s).

Highly recommended: Read Theo's [Stripe Recommendations](https://github.com/t3dotgg/stripe-recommendations] notes.

## Overview

- Subscription is at Workspace level (our top-level entity)
- Use subscription quantity to bill per user
- Plans:
  - Hobby (free)
  - Pro ($15/month/user)
- Workspace can be "comped" (Pro but unbilled) - for internal workspaces, demos
- Use Stripe Checkout links to start subscriptions
- Use Stripe Billing Portal to manage subscriptions

## Implementation

1. Update database model
2. Add stripe checkout sessions
3. Use webhooks to process events and sync subscription state
4. Add billing mode and plan fields to graphql
5. Upon user add/archive, use background jobs to update subscription quantity to stripe
6. Add stripe billing portal sessions

### 1. Update database model

- Database (packages/db)
  - Update `Workspace` model:
    - Add `stripeCustomerId` string
    - Add `billingMode` string, default to `standard` (either `standard` or `comped`)
    - Add `plan` string, default to `hobby` (either `hobby` or `pro`)
  - Add `subscriptions` table
    - A Workspace has one subscription
    - Add fields to support `subData` below
    ```js
    const subData = {
      subscriptionId: subscription.id,
      status: subscription.status,
      priceId: subscription.items.data[0].price.id,
      currentPeriodEnd: subscription.current_period_end,
      currentPeriodStart: subscription.current_period_start,
      cancelAtPeriodEnd: subscription.cancel_at_period_end,
      card:
        subscription.default_payment_method &&
        typeof subscription.default_payment_method !== "string"
          ? {
              brand: subscription.default_payment_method.card?.brand ?? null,
              last4: subscription.default_payment_method.card?.last4 ?? null,
            }
          : null,
    };
    ```

### 2. Add stripe checkout sessions

- API (apps/api)
  - Add `createCheckoutSession` mutation (in `apps/api/src/schema/settings/billing.ts`)
    - Accepts a `workspaceId`
    - Returns a `checkoutUrl`
    - Always create the stripe customer before creating the checkout session (if they don't already have one)
    - Place business logic in `store.billing.createCheckoutSession` (packages/store)
    - Add docs to `apps/api/docs/billing.ts`

### 3. Use webhooks to process events and sync subscription state

- Store (packages/store)
  - Add `syncStripeData` and expose as `store.billing.syncStripeData`
    - Accepts a `customerId` (the stripe customer ID)
    - Fetches the customer's subscription from Stripe
    - Upserts the subscription in our database
- API (apps/api)
  - Add Express to our setup and integrate Yoga with it
  - Add webhook handler for `/webhooks/stripe`
    - Runs the `syncStripeData` store function to sync the subscription state
    - Listen to the following events:
      - checkout.session.completed
      - customer.subscription.created
      - customer.subscription.updated
      - customer.subscription.deleted
      - customer.subscription.paused
      - customer.subscription.resumed
      - customer.subscription.pending_update_applied
      - customer.subscription.pending_update_expired
      - customer.subscription.trial_will_end
      - invoice.paid
      - invoice.payment_failed
      - invoice.payment_action_required
      - invoice.upcoming
      - invoice.marked_uncollectible
      - invoice.payment_succeeded
      - payment_intent.succeeded
      - payment_intent.payment_failed
      - payment_intent.canceled
- Docker (packages/docker-dev)
  - Add stripe service for listening to webhooks

### 4. Add billing mode and plan fields to graphql

- API (apps/api)
  - Add `Workspace` fields (in `apps/api/src/schema/workspaces.ts`):
    - `billingMode`
    - `plan`
  - Add docs to `apps/api/docs/billing.ts`

### 5. Upon user add/archive, use background jobs to update subscription quantity to stripe

- Queue (packages/queue)
  - Add `updateSubscriptionQuantity` job
    - Accepts a `workspaceId`
- Worker (apps/worker)
  - Add background job to update subscription quantity for a workspace
  - Place business logic in `store.billing.updateSubscriptionQuantity` (packages/store)
- API (apps/api)
  - When user accepts invite to join Workspace, trigger background job to update subscription quantity
  - When person is archived, trigger background job to update subscription quantity

### 6. Add stripe billing portal sessions

- API (apps/api)
  - Add `createBillingPortalSession` mutation (in `apps/api/src/schema/settings/billing.ts`)
    - Accepts a `workspaceId`
    - Place business logic in `store.billing.createBillingPortalSession` (packages/store)
    - Add docs to `apps/api/docs/billing.ts`
