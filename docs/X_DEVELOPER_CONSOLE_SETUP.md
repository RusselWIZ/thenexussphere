# Configure X Developer Console authentication

This project uses X OAuth 2.0 Authorization Code with PKCE. X sends the user back to the Next.js web origin, which proxies the callback to the API while forwarding the browser-bound OAuth cookie.

## Before editing settings

The `.env` file shared during setup contained credentials. Treat the X client secret as exposed: regenerate it in the X Developer Portal, replace it in your local ignored `.env`, and do not paste it into chat, Markdown, screenshots, or source control. The shared file also contained the app's JWT and token-encryption keys. Replace the JWT secret. Replace the token-encryption key only if no provider tokens have been stored under it; otherwise those tokens must first be re-encrypted with a planned key-rotation migration.

## Console settings for local development

In the X Developer Portal, open your Project, select the app, and open **User authentication settings**.

Set the options shown in the supplied authentication-settings screen as follows:

1. **App permissions:** choose **Read**. Do not select Read and write, or Direct message permissions. This integration only reads the authorized user's profile and posts.
2. **Request email from users:** leave **off**. NexusSphere does not need X email access; users create their NexusSphere login separately.
3. **Type of App:** choose **Web App, Automated App or Bot** (confidential client). This is a server-side web application. Do not choose Native App or a public browser-only client.
4. **Callback URI / Redirect URL:** enter this exact local URL:

   ```text
   http://localhost:3000/api/v1/connections/x/callback
   ```

   It must match `X_CALLBACK_URL` exactly in the root `.env`. Do not use port 4000 here: the callback must arrive through the web app so the OAuth state cookie is sent. Open NexusSphere consistently at `http://localhost:3000`; do not start at `127.0.0.1` and then return to `localhost`, as browser cookies are host-specific.
5. **Website URL:** for local development, use `http://localhost:3000` if the console accepts it. This is profile/app metadata; it is not the callback URL.
6. **Organization name/URL:** optional for local testing unless X requires them for your account.
7. **Terms of Service / Privacy Policy:** use actual public policy pages if the console requires them. Do not enter placeholder or invented URLs. Publish real policy pages before inviting external beta testers.
8. Save the settings.

The permissions selector in this screen refers to app permissions and may mention OAuth 1.0a. The **Type of App** selection enables the OAuth 2.0 authentication used by this project. The API scopes requested by this project are configured separately as OAuth scopes below.

## Add X scopes and credentials locally

The current root `.env.example` uses these least-privilege scopes:

```text
users.read tweet.read offline.access
```

- `users.read` lets the app read the authorized user's profile.
- `tweet.read` lets it read posts available to that user.
- `offline.access` requests a refresh token so scheduled sync can continue after the short-lived access token expires.

Do not add `tweet.write`, message, email, or other scopes for this read-only launch.

In the X Developer Portal, open the app's **Keys and tokens** section. Copy the app's **Client ID** and newly generated **Client Secret** into the root `.env` locally:

```dotenv
X_CLIENT_ID=<client-id>
X_CLIENT_SECRET=<new-client-secret>
X_CALLBACK_URL=http://localhost:3000/api/v1/connections/x/callback
X_SCOPES=users.read tweet.read offline.access
```

The placeholders above are instructions, not literal values. Do not send the values back in chat or commit them. The server uses the Client Secret only for server-side token exchange/refresh; it must never be put in a `NEXT_PUBLIC_*` variable or web-client code.

Restart `npm run dev` after saving `.env`. The X app's API access tier must also permit the user-context profile and posts endpoints used by the adapter; OAuth setup alone does not grant API product access.

## Test the connection

1. Start PostgreSQL and Redis and ensure the database migration has been applied.
2. Start the app from the repository root with `npm run dev`.
3. Open `http://localhost:3000`, register/sign in to NexusSphere, and select **Connect X**.
4. Review the X authorization page. It should request only the configured read scopes. Approve with an X account allowed by the app's current access mode.
5. Expect a return to the NexusSphere dashboard with a connected notice; the first sync is queued.
6. Check the API readiness endpoint at `http://localhost:4000/health/ready`. Check the API/worker terminal for a sync result if posts do not appear.
7. Disconnect the account in NexusSphere to revoke its provider authorization and delete that connection's ingested posts.

The callback is validated using the one-time server-side OAuth state and the browser-bound state cookie; it does not depend on the NexusSphere session cookie still being valid after the user leaves for X. Starting OAuth still requires a signed-in NexusSphere user.

## Common failures

- **Redirect URI mismatch:** compare the portal Callback URI and `X_CALLBACK_URL` character-for-character, including scheme, hostname, port, path, and trailing slash.
- **Connection returns an error:** verify OAuth 2.0 is enabled by the selected app type, the regenerated Client ID/Secret pair belongs to this app, and the callback is being opened through `localhost:3000`.
- **Missing refresh token:** confirm `offline.access` is enabled/requested and reauthorize the account after saving the scope.
- **401/403 from X API after consent:** check the app's API product/access and endpoint entitlement in addition to OAuth scopes.
- **402 from X API during sync:** OAuth succeeded, but X rejected the data request as payment required. Check the Developer Console's API credits, billing status, and spending limits; add credits or raise the limit if needed, then select **Retry** in NexusSphere. Reconnecting does not resolve an API billing/credit limit. X currently documents its API as pay-per-usage; see the official [X API pricing and credits](https://docs.x.com/x-api/getting-started/pricing) page.
- **No feed posts:** verify the authorized account has posts available through the enabled endpoint, inspect rate-limit/API entitlement errors, and use the manual sync button. The integration reads posts; it does not publish.
- **Provider app is in restricted/development mode:** allowlist the test account if X provides that option. Do not invite accounts that are not permitted by the app's current access approval.

For current provider behavior and available scopes, consult the official [X OAuth 2.0 Authorization Code with PKCE documentation](https://docs.x.com/fundamentals/authentication/oauth-2-0/authorization-code) and the X Developer Portal. X access tiers and approval requirements can change; confirm them in the portal before beta launch.

## Production settings later

When deploying, change all three URLs consistently:

- Callback URI and `.env` `X_CALLBACK_URL`: `https://<web-domain>/api/v1/connections/x/callback`
- Website URL: `https://<web-domain>`
- `WEB_ORIGIN`: `https://<web-domain>`

The callback must remain on the web hostname because the web rewrite forwards the OAuth callback and its cookie to the API. Put X credentials only in the API/worker secret manager, use HTTPS, use a production X app with the required approved API access, and publish actual Terms and Privacy pages before a public launch.
