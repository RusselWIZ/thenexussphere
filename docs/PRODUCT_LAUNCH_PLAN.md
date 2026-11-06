# NexusSphere first launch plan

## Launch decision

Start with a **small, invite-only, no-charge beta** for read-only aggregation from X and Reddit. Do not promise other platforms, publishing, messaging, comment unification, real-time webhooks, or paid Pro features in this launch.

The goal is to learn whether account connection, reliable sync, and a useful unified feed solve a real user problem before adding more providers or operating a public paid service.

## Product boundary

### Include in the first beta

- NexusSphere email/password account creation and sign-in.
- User-authorized X and Reddit connections, limited to the approved read scopes.
- Periodic/manual sync of the user's own supported posts into the unified feed.
- Disconnecting a social account and deleting the posts ingested for that connection.
- Clear connection/sync errors and a channel for pilot feedback.

### Explicitly defer

- Instagram, Facebook, LinkedIn, TikTok, and any provider not implemented and approved.
- Cross-posting, scheduling/publishing, DMs, comment aggregation, workflow automation, and webhooks.
- Charging pilot users, unless the owner later elects to complete and launch Stripe billing.
- Public/open registration before the account recovery, verification, privacy, and abuse controls below are complete.

The code currently has X and Reddit adapters; the other providers are UI placeholders. The app does not yet have email verification, password reset, a complete account-deletion flow, or public-facing privacy/terms pages. Treat it as a **closed technical beta**, not a finished public consumer launch.

## Phases and exit criteria

### Phase 0 — Provider and hosting readiness

- Complete `REAL_ACCOUNT_INTEGRATION_INTAKE.md` for X and/or Reddit.
- Confirm each provider permits the intended app, OAuth scopes, read endpoints, user cohort, data retention, and commercial use.
- Configure separate development/staging apps and exact HTTPS callback URLs.
- Choose hosting, managed PostgreSQL, managed Redis, domain, and a person responsible for production support.
- Store provider secrets only in local ignored `.env` for development or the hosting secret manager for hosted environments.

**Exit:** each provider's access is approved for the exact read use case; staging credentials and a test account are controlled by the owner; no credentials have been put in Markdown, email, chat, source control, or logs.

### Phase 1 — Internal alpha

- Run database migrations against a disposable/staging database and validate restore from backup.
- Exercise signup/sign-in, OAuth consent and denial, token refresh, manual and scheduled sync, rate limiting, disconnect, and data deletion.
- Confirm users can only see their own connections and posts.
- Verify provider limits from actual approved accounts and tune sync intervals to stay within those limits.
- Add basic error tracking/alerts and ensure credentials and post bodies are not logged.

**Exit:** the owner can repeat the end-to-end flow in staging; failures are visible and recoverable; backup restore and token-key recovery are documented.

### Phase 2 — Invite-only beta

- Invite a small cohort of consenting testers (suggestion: 5–10 people, subject to provider approval and API limits).
- Keep registration private; do not post a public signup link until an access-control mechanism is deployed.
- Publish privacy/terms and explain which data is fetched, stored, retained, and deleted.
- Collect structured feedback on connection completion, sync usefulness, missing posts, and provider errors.
- Review support/incident reports weekly; pause new invites if sync failures, provider limit errors, or privacy issues rise.

**Exit:** at least two weeks without unresolved data-isolation/security incidents; reliable sync for the approved sample; user value and provider access costs are understood.

### Phase 3 — Public or paid launch (separate decision)

Before opening registration, implement and test email verification, password reset, account deletion/export, abuse controls, support/contact process, public legal pages, monitoring/backup recovery, incident response, and capacity/load behavior. Before charging, test Stripe Checkout, customer portal, webhook signature and retry/idempotency behavior, cancellation, refunds/support handling, and subscription access changes.

Do not treat passing typecheck/build as a production approval. A human owner must review legal, provider-policy, privacy, operational, and cost gates.

## Minimum production deployment topology

- Web: Next.js service, public HTTPS domain.
- API: Express service, private database/Redis egress, readiness endpoint `/health/ready`.
- Worker: independent, continuously running BullMQ consumer.
- Data: managed PostgreSQL with tested backups; managed Redis with TLS and appropriate persistence/availability.
- Secrets: platform secret manager; distinct staging and production values. API and worker share the same JWT and token-encryption keys.
- Release: CI installs with `npm ci`, generates Prisma client, runs typecheck/tests/build/audit; a one-off release job runs `npm run db:deploy -w @nexussphere/api` before deploying API/worker.

The repository README has current service build/start commands and environment variables.

## Owner decisions still needed

Fill out the integration intake document and return the completed Markdown. It requests non-secret facts only. Also decide:

- Preferred host/region and public domain (or that these are undecided).
- Whether the first beta is X-only, Reddit-only, or both.
- Approximate invite count and intended start date.
- Whether Stripe should remain disabled during beta (recommended).
- Data retention target for synced posts and the country/region in which beta users will be located.

These are launch inputs, not passwords or credentials. If unsure, write `undecided`.
