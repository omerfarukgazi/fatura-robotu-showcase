# Fatura Robotu: Shopify → GİB e-Archive invoice automation

A multi-tenant web app for Turkish Shopify stores that sell with **cash on delivery (COD)**.
It turns confirmed orders into e-Archive invoice drafts on the Turkish Revenue Administration (GİB) portal, prints them in one batch for the parcels, and signs only the orders that were actually delivered.

> **Source code is private** because I plan to offer this as a paid Shopify app.
> This repository explains the problem, the design and the engineering decisions. Code walkthroughs are available on request.

---

## The problem

COD sellers in Turkey have to put an invoice in every parcel, but many parcels come back unpaid. If you sign every invoice up front, each returned parcel leaves you with an invoice to cancel. If you create invoices one by one on the GİB portal, it takes several minutes per order.

The existing paid integrators solve this, but they cost too much for a small store. I ran into this in my own store, so I built a tool for it.

## How it works

```mermaid
flowchart LR
    A[Seller confirms order<br/>by phone] -->|adds 'onaylandi' tag| B[Shopify]
    B -->|GraphQL API| C[Fatura Robotu]
    C -->|unsigned draft| D[GİB e-Archive portal]
    C -->|one printable page| E[Invoices go into parcels]
    B -->|delivered / returned| C
    C -->|"SMS code → batch sign<br/>(delivered only)"| D
    C -->|delete draft<br/>(returned only)| D
```

1. The seller calls the customer and tags the confirmed order `onaylandi` in Shopify.
2. In the dashboard, the seller selects orders. The app creates an **unsigned draft** for each one on the GİB portal.
3. All drafts are printed on one page and go into the parcels.
4. The **awaiting signature** screen pulls delivery and return status from Shopify.
5. The seller selects the delivered orders and types in the SMS code that GİB sends. All selected invoices are signed in a single step.
6. If a parcel comes back, its draft is deleted from GİB with one click, so there is nothing to cancel.

## Engineering highlights

| Area | What I did |
|---|---|
| **Multi-tenancy** | Several stores share one server. Every query is filtered by store ID, and tests check that one store can't see, sign or delete another store's invoices. |
| **Secrets at rest** | GİB passwords, Shopify access tokens and invoice images are encrypted with Fernet (AES). The encryption key is kept outside the database. |
| **Shopify integration** | OAuth install flow with `state` and HMAC verification. GraphQL Admin API for orders and tags. HMAC-verified webhooks. Shopify Billing for subscriptions. |
| **Privacy compliance** | Handles Shopify's mandatory GDPR/KVKK webhooks (`customers/redact`, `shop/redact`, `customers/data_request`). Secrets and customer names are wiped when the app is uninstalled. |
| **Web security** | CSRF tokens on every state-changing request. `HttpOnly`, `Secure` and `SameSite` cookies. Invoice previews render in a sandboxed iframe. |
| **GİB portal client** | GİB has no official API, so the client uses the same endpoints as the portal's own web page. It works against both the test portal and the live portal. |
| **Money handling** | All amounts use `Decimal`, with VAT split and half-up rounding to the kuruş. Invoice dates use Istanbul time, not the server's clock. |
| **Testing** | 27 automated tests (pytest) run against fake Shopify and fake GİB services. They cover OAuth, CSRF, signing, expiry, returns, webhooks and store isolation. |

## Tech stack

Python 3.13 · Flask · SQLite · `cryptography` (Fernet) · Shopify GraphQL Admin API · Gunicorn · Docker · pytest

## Project structure

```
fatura_saas/
  __init__.py    app factory, security headers, error pages
  gorunum.py     pages, Shopify OAuth, dashboard API
  islem.py       invoice logic: draft, sync, SMS signing, delete
  gib.py         GİB e-Archive portal client
  shopify.py     Shopify API: orders, tags, billing, webhooks
  kanca.py       webhooks (uninstall, billing, GDPR)
  guvenlik.py    encryption and Shopify signature checks
  vt.py          database schema
tests/           27 tests with fake Shopify and fake GİB
Dockerfile
```

## Status

- [x] Core flow: draft → print → sign → delete returned drafts
- [x] Multi-tenant SaaS version with Shopify Billing
- [x] 27 passing tests against fake services
- [ ] End-to-end run against the GİB **test** portal
- [ ] Pilot with my own store, then 3–5 other COD sellers
- [ ] Embedded Shopify admin app (App Bridge) and App Store submission
- [ ] Cargo carrier APIs for automatic delivery status

## About

Built by **Ömer Faruk Gazi**, Computer Engineering student and founder of GAZIOFG LTD.
I built it with an AI coding assistant (Claude). I made the product and architecture decisions and reviewed and tested the code.

[LinkedIn](https://www.linkedin.com/in/omerfarukgazi/) · [GitHub](https://github.com/omerfarukgazi)
