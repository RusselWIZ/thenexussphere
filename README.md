# NexusSphere

NexusSphere is a full-stack social account dashboard. It currently supports account registration, a unified feed, and OAuth connections for X and Reddit. Provider access tokens are encrypted at rest; background workers normalize provider posts into the shared feed. Stripe-backed Pro entitlements are optional.

## Requirements

- Node.js 22.18 or later and npm
- Docker with Docker Compose, or compatible PostgreSQL 16+ and Redis 7+ services
- Developer applications for each social provider you want to connect
- A Stripe account and Pro Price ID only if you want to exercise paid subscriptions

## Run locally

For the recommended phased beta scope, see [the first launch plan](./docs/PRODUCT_LAUNCH_PLAN.md). To prepare your real X/Reddit developer apps and test accounts, fill in [the non-secret integration intake](./docs/REAL_ACCOUNT_INTEGRATION_INTAKE.md); never put API secrets or tokens in that file.

For the X Developer Portal fields shown in the authentication settings screen, use [the X OAuth setup guide](./docs/X_DEVELOPER_CONSOLE_SETUP.md).
For the Reddit app creation form and local OAuth test, use [the Reddit developer app setup guide](./docs/REDDIT_DEVELOPER_SETUP.md).

From the repository root:

1. Install workspace dependencies:

   ```sh
   npm install
   ```

2. Create the local environment file and generate unique secrets:

   ```sh
   cp .env.example .env
   openssl rand -hex 32
   openssl rand -base64 32
   ```

   Put the first value in `JWT_SECRET` and the second in `TOKEN_ENCRYPTION_KEY` in `.env`. The encryption key must decode to exactly 32 bytes. Keep `.env` private and never commit or share it.

3. Start PostgreSQL and Redis:

   ```sh
   docker compose up -d
   ```

   The compose file binds both services to loopback. Its default database credentials are for local development only.

4. Create the database schema and generate the Prisma client:

   ```sh
   npm run db:migrate -w @nexussphere/api
   ```

5. Start the API, worker, and web application:

   ```sh
   npm run dev
   ```

   Open <http://localhost:3000>. Register with an email and password, then connect a provider from the dashboard. The API health endpoints are available at <http://localhost:4000/health/live> and <http://localhost:4000/health/ready>.

## Connect social accounts

Provider credentials are optional for running the dashboard, but are required for live account connections. Register OAuth applications with the exact callback URLs below, then set the corresponding values in `.env` and restart the API:

| Provider | Callback URL | Required configuration |
|---|---|---|
| X | `http://localhost:3000/api/v1/connections/x/callback` | `X_CLIENT_ID`, `X_CLIENT_SECRET` |
| Reddit | `http://localhost:3000/api/v1/connections/reddit/callback` | `REDDIT_CLIENT_ID`, `REDDIT_CLIENT_SECRET`, `REDDIT_USER_AGENT` |

Use the read-only scopes configured in `.env.example`. The provider must grant the required identity and post-reading scopes; otherwise the connection is rejected. Provider API access is subject to each provider's app approval, API plan, rate limits, and terms. NexusSphere does not bypass those restrictions.

For production, register the callback on the **web app host** (for example, `https://app.example.com/api/v1/connections/x/callback`) and set the matching `X_CALLBACK_URL` or `REDDIT_CALLBACK_URL`. The web app rewrites that callback path to the API while preserving the browser session cookie.

The OAuth callback uses authorization state, PKCE, and an HTTP-only browser-bound state cookie. Connection credentials are encrypted with AES-256-GCM. Do not reuse the development secrets in production. For production, use HTTPS, managed secret storage, a production database and Redis instance, and provider-approved production callback URLs.

## Optional Stripe subscriptions

Set `STRIPE_SECRET_KEY`, `STRIPE_WEBHOOK_SECRET`, and `STRIPE_PRO_PRICE_ID` in `.env` to enable checkout and subscription management. Set `STRIPE_SUCCESS_URL` and `STRIPE_CANCEL_URL` to your public web app root (for example, `https://app.example.com/?checkout=success` and `https://app.example.com/?checkout=cancelled`). Configure the Stripe webhook endpoint to:

```text
https://<your-api-host>/api/v1/billing/webhook
```

Subscribe it to `checkout.session.completed`, `customer.subscription.created`, `customer.subscription.updated`, `customer.subscription.deleted`, `invoice.paid`, and `invoice.payment_failed`. Stripe values are deliberately optional; billing endpoints report that billing is unavailable until configured.

## Useful commands

```sh
npm run typecheck
npm test
npm run build
npm run db:studio -w @nexussphere/api
```

`npm run dev` starts the API with its sync worker and the Next.js app. To scale them independently, run the API with `ROLE=api` and run the compiled worker with `npm run start:worker -w @nexussphere/api`.

## Architecture and data flow

```text
Browser → Next.js dashboard → Express REST API → PostgreSQL
                                      ↘ Redis/BullMQ → sync worker → provider APIs
```

- `apps/web` contains the responsive Next.js dashboard. Its rewrites keep browser API calls on the web origin, including OAuth callbacks.
- `apps/api/src/routes` exposes auth, connections, feed, subscription, and billing endpoints. Express middleware handles session validation, rate limiting, and entitlement checks.
- `apps/api/src/social` contains the provider adapter contract, X/Reddit adapters, normalized post types, and ingestion service. The adapters map provider responses to the same author/content/media/metrics/timestamp model.
- `apps/api/src/worker.ts` runs queue jobs and scheduled syncs. PostgreSQL stores users, encrypted provider credentials, cursors, normalized posts, subscriptions, and processed Stripe event IDs. Redis carries jobs and serializes syncs per social connection.
- `apps/api/prisma/migrations` contains the initial PostgreSQL migration. The `prisma` CLI uses the PostgreSQL driver adapter; app and worker instances use the same `DATABASE_URL` and Redis queue.

### OAuth connection flow

1. A signed-in user chooses **Connect X** or **Connect Reddit** in the dashboard.
2. The API creates a random OAuth `state`; X also uses PKCE. It stores a hash of the state and an encrypted verifier, then sets a short-lived, HTTP-only, `SameSite=Lax` browser cookie.
3. The user authorizes the app at the provider. The provider redirects to the configured callback, which is routed through the web origin to the API.
4. The callback validates the session, provider, expiry, one-time state record, and browser cookie before exchanging the authorization code.
5. The adapter fetches the provider identity and required scopes. The API encrypts access and refresh tokens with AES-256-GCM and stores them against the signed-in NexusSphere user.
6. A BullMQ job is queued. The worker refreshes expiring credentials, fetches posts, maps them to the shared model, and upserts by provider post ID.

### Ingestion and access behavior

The worker polls on a one-minute scheduler and enqueues accounts only when their plan's interval is due (60 minutes on Free; 5 minutes on Pro, subject to provider limits). Redis prevents concurrent sync jobs for the same account. Rate limits, provider 5xx responses, and network timeouts are retried with exponential backoff; invalid/expired authorization is marked for reconnection without stopping other accounts.

The user-facing feed is `GET /api/v1/feed`; provider/source selection is available to all users, while text search is entitlement-gated. `GET /api/v1/analytics` and `GET /api/v1/archive` are Pro-gated server-side. `GET /api/v1/check-subscription` returns the current plan entitlements. Stripe webhook signatures are verified against the raw request body, and duplicate event IDs are ignored.

The current ingestion mechanism is polling; webhook subscriptions are not yet implemented. Provider-native posting, DMs, comments, and workflow automation are also outside the current adapter scope.

## Deployment outline

For production, deploy the Next.js web app, Express API, and BullMQ worker as separately scalable services. Use managed PostgreSQL and Redis, HTTPS, a shared secret manager, and distinct least-privilege credentials per environment. Configure provider callback URLs on the public web host, set `WEB_ORIGIN` and `API_ORIGIN`, and use production provider and Stripe secrets; never use the local Docker credentials.

With a Node.js host that supports multiple services from one Git repository, configure these three services (all use Node.js 22.18+ and `npm ci` from the repository root):

| Service | Build command | Start command |
|---|---|---|
| Web | `npm run build -w @nexussphere/web` | `npm run start -w @nexussphere/web` |
| API | `npm run build -w @nexussphere/api` | `npm run start -w @nexussphere/api` with `ROLE=api` |
| Worker | `npm run build -w @nexussphere/api` | `npm run start:worker -w @nexussphere/api` |

Set `API_ORIGIN` on the web service to the API's reachable HTTPS origin. It is read by `next.config.ts` for `/api/*` and `/health/*` rewrites, so provide it when building the web app as well as at runtime. Set `WEB_ORIGIN`, `API_ORIGIN`, `DATABASE_URL`, `REDIS_URL`, `JWT_SECRET`, and `TOKEN_ENCRYPTION_KEY` on both API and worker. Use the same JWT and token-encryption keys on those two services. The API listens on the host's `PORT` automatically; set `API_PORT` only if you need to override it. For managed Redis with TLS, use its `rediss://` connection URL.

Before deploying the API/worker versions, run this one-off release command with the API's production environment:

```sh
npm run db:deploy -w @nexussphere/api
```

Attach persistent/managed PostgreSQL and Redis before starting the services. Configure the platform's web health check as `/` and API readiness check as `/health/ready`. Register OAuth redirect URLs on the public **web** hostname (not the API hostname), because the web rewrite proxies OAuth callbacks and preserves the browser-bound cookie. Add the provider secrets only to API/worker, and keep production values out of Git.

A standard GitHub Actions pipeline should run `npm ci`, `npm run db:generate`, `npm run typecheck`, `npm test`, and `npm run build` for pull requests. After approval, deploy to staging, apply `npm run db:deploy -w @nexussphere/api`, run health checks and provider/Stripe smoke tests, then promote the same build to production. Run schema migrations as a one-off release job before rolling out API or worker instances.

## Current scope

The current adapters ingest posts from X and Reddit and normalize them to the shared post model. Instagram, Facebook, LinkedIn, and TikTok are shown in the connection menu as unavailable future integrations, with an inert adapter base template and implementation checklist in `apps/api/src/social/PROVIDER_ADAPTERS.md`. OAuth connect/disconnect and manual/background synchronization are implemented for X and Reddit only. Other provider integrations, publishing, direct messages, comment aggregation, and webhook-driven ingestion are not implemented yet; they require separate provider APIs, permissions, and adapters.
