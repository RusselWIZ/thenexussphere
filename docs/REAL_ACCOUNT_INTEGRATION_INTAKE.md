# NexusSphere real-account integration intake

Complete and return this Markdown file to plan/test the first read-only X and Reddit integrations. This is **not** a place to send credentials. `Unknown` or `undecided` is an acceptable answer.

## Security — do not include secrets

Do **not** put any of the following in this file, a chat, a screenshot, or source control:

- OAuth client secrets, API keys, passwords, access tokens, refresh tokens, session cookies, or recovery codes.
- Database/Redis URLs if they contain credentials.
- Unredacted provider responses, private messages, private post text, email addresses, or personal data belonging to other people.

Keep actual app secrets in your local ignored `.env` or the hosting provider's secret manager. If a debug response is requested, remove tokens, authorization headers, names, email addresses, post text, media URLs, and user IDs first. We need the status/error code and field names, not someone's private content.

## A. Product and pilot

- First launch provider(s): `X` / `Reddit` / `both` / `undecided`
- Intended launch: `private alpha` / `invite-only beta` / `public` (public is not recommended yet)
- Approximate initial tester count:
- Intended tester countries/regions:
- Preferred hosting provider and region, if known:
- Initial monthly hosting/provider API budget range (or `undecided`):
- Public web domain, if already owned (do not include account credentials):
- Should Stripe remain disabled during the beta? (recommended: yes):
- Desired retention period for ingested posts, if decided:
- Who will own provider accounts, deployment, and incident response? (role/name is enough):

## B. X developer app

Complete if X is in scope. Use `not applicable` otherwise.

- Developer portal account/app already exists? `yes/no`
- Are you the app owner/admin? `yes/no`
- App name (non-secret):
- Current API access product/tier shown in the developer portal (exact label):
- Does the app have access to the user-context endpoints required for reading the authorized user's profile and posts? `yes/no/unknown`
- App OAuth client type shown in settings: record the exact portal label:
- OAuth 2.0 Authorization Code with PKCE enabled? `yes/no/unknown`
- Exact approved/granted read scopes as shown by the provider (scope names only; do not paste credentials):
- Is `offline.access` available and approved for this app? `yes/no/unknown`
- Redirect/callback URL currently registered (URL only):
- Is the app restricted to development/test users, or approved for general users? (exact portal wording):
- Published rate/usage limits relevant to profile and user-post reads (include time window and tier, or `unknown`):
- Sanitized error code/status if a test call failed (no headers/body containing personal data):
- Anything the app portal says about review, user limits, attribution, retention, or commercial use:

### How to collect X information

1. Sign in to the [X Developer Portal](https://developer.x.com/en/portal/dashboard) with the account that owns the developer app.
2. Open the project/app and record the displayed API product/access tier and its current usage/rate-limit information. Check that the product includes the user-context profile and user-post read operations needed by this launch; do not assume that OAuth approval alone grants API endpoint access.
3. Open app **User authentication settings**. Record the client type and whether OAuth 2.0 Authorization Code with PKCE is enabled. The current server adapter supports a client ID with PKCE and can use a client secret when the provider issues one; record the exact available type and whether a secret is issued, but do not copy it here.
4. Record the exact callback URL configured for the environment. Local development default: `http://localhost:3000/api/v1/connections/x/callback`. Staging/production callback must use the HTTPS web-app domain and match exactly.
5. Record the exact read scopes the portal allows for the app. The current implementation requests `users.read`, `tweet.read`, and `offline.access`; no write scopes are needed for this read-only launch.
6. Confirm whether test users are allowlisted/development-only or whether X has approved access for the intended beta cohort. Note any restrictions shown.
7. If a test fails, provide only the endpoint path (without query secrets), HTTP status, provider error code, and a manually redacted message. Never export tokens or paste the Authorization header.

Official implementation reference: [X OAuth 2.0 Authorization Code with PKCE](https://docs.x.com/fundamentals/authentication/oauth-2-0/authorization-code). Confirm portal requirements and current API limits directly because provider plans and approvals can change.

## C. Reddit developer app

Complete if Reddit is in scope. Use `not applicable` otherwise.

- Reddit developer app exists? `yes/no`
- Are you the app owner/admin? `yes/no`
- App name (non-secret):
- App type shown in app settings (for example, web app/installed app/script; copy exact label):
- Current Data API access/review status (exact portal wording):
- Exact requested/approved OAuth scopes (scope names only):
- Is user-authorized read access to the account's own submitted posts approved/available? `yes/no/unknown`
- Redirect/callback URL registered (URL only):
- Production/live-user approval or review still required? `yes/no/unknown`
- Current published rate or usage limits relevant to this app (or `unknown`):
- Required User-Agent format / app identification shown by Reddit:
- Sanitized error code/status from a test call, if any:
- Any restrictions/terms on commercial use, retention, attribution, or display:

### How to collect Reddit information

1. Sign in to Reddit using the account that will own the application, then open [Reddit app preferences](https://www.reddit.com/prefs/apps).
2. Create or open the app you intend to use. Record the app type and whether it is approved for the intended use; do not copy the secret displayed beneath the app.
3. Set the OAuth redirect URI to the exact environment callback. Local development default: `http://localhost:3000/api/v1/connections/reddit/callback`. Staging/production must use the HTTPS web-app domain and match exactly.
4. Review the current [Reddit Data API information](https://support.reddithelp.com/hc/en-us/articles/16160319875092-Data-API-Wiki) and the requirements displayed in the developer account. Record any access approval, user limits, rate limits, commercial-use conditions, or retention constraints.
5. The current adapter requests only `identity` and `read`, and reads the authorized user's submitted posts. Confirm these scopes and this endpoint are allowed for your app and intended beta.
6. Provide the required non-secret User-Agent identification format; use a product/app version and the app owner's Reddit username if Reddit requires it. Do not include passwords or tokens.
7. If a test fails, share only status/error code and a redacted error message; do not paste API headers or unredacted response bodies.

Reddit may restrict API access, commercial use, or app distribution. A locally-created app is not evidence that production access is approved; confirm requirements with the current official Reddit developer/support guidance before inviting users.

## D. Test accounts and authorized data

Do not provide account credentials or export another person's data. The safest integration test is for the app owner/tester to connect their own account through the app's OAuth button, review the provider consent screen, and authorize only the requested read scopes.

- Do you control an X account that can be used to authorize the app? `yes/no`
- Do you control a Reddit account that can be used to authorize the app? `yes/no`
- Are these accounts permitted as test accounts under the provider app's current status? `yes/no/unknown`
- After you run the local/staging app, what exact step fails (if any)?
- Sanitized provider error code/status:
- Expected observable result (for example, profile connected, recent authored posts appear):
- Actual observable result:
- Approximate number of posts you expect the app to fetch (a range is enough):

Please do not paste real post text, personal messages, private media links, or raw JSON. If a schema example is needed, use a synthetic example with fake IDs and text.

## E. Environment and readiness (non-secret)

- Node version used locally:
- Operating system:
- Can PostgreSQL and Redis be started locally or provisioned in staging? `yes/no/unknown`
- Host plan supports separate web, API, and continuously running worker services? `yes/no/unknown`
- Managed PostgreSQL provider and backup/restore option (provider/product name only):
- Managed Redis provider and TLS support (provider/product name only):
- Domain/DNS/TLS status:
- Support email/contact process ready? `yes/no/unknown`
- Privacy policy and terms URLs, if available:
- Known launch blocker(s):

## F. Handoff

- Any additional provider-console requirements or review text (redacted):
- What would you like integrated/tested first?
- Preferred test window/time zone:

When returning this file, include only the completed non-secret fields. Once the details are reviewed, I can identify which access is available, adjust the adapters/configuration if needed, and give you exact instructions to enter credentials locally or into your host's secret manager.
