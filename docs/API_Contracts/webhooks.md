# Webhooks API Contract

This document defines the webhook endpoint Venix exposes to receive payment provider events.

---

# 1. Stripe Webhook

## Request

**POST** `/webhooks/stripe/{tenant_slug}`

- `tenant_slug` (string, path parameter) — the unique slug of the tenant whose Stripe account sent this event.
- No authentication header required — Stripe sends this directly. The tenant is identified from the path, not from a JWT or API key.
- This route is bypassed by `TenantResolverMiddleware` (prefix `/webhooks/stripe/`).

Headers set by Stripe:
- `stripe-signature`: used for payload signature verification.

Body: raw Stripe event payload (JSON, read as bytes).

## Response (200 OK)

```json
{
  "status": "ok"
}
```

## Processing

1. Tenant is fetched from DB by `tenant_slug`. If not found or `stripe_webhook_secret` is null → 400.
2. Signature is verified using the tenant's `stripe_webhook_secret` via `stripe.Webhook.construct_event`. If invalid → 400.
3. Event is routed to the appropriate handler inside `StripeService`:
   - `checkout.session.completed` — confirms the order, creates order items, decrements stock, clears cart, sends confirmation email.
   - `payment_intent.payment_failed` — marks order payment status as FAILED.
   - `charge.refunded` — marks order payment status as REFUNDED.
   - All other event types are ignored (return 200 immediately).
4. Idempotency: each `event.id` is recorded in `processed_webhook_events`. Duplicate delivery is a no-op.

## Notes

- The webhook URL a tenant must configure in their Stripe dashboard is: `https://<your-venix-host>/webhooks/stripe/{tenant_slug}`
- Tenant's `stripe_secret_key` is fetched from DB (never from Redis cache) to perform any Stripe API calls (e.g. refunds).
- Stripe keys are read directly from the `tenants` table — never from env vars.

## Errors

- `400 Bad Request` — tenant slug not found, `stripe_webhook_secret` not configured, or signature verification failed.
