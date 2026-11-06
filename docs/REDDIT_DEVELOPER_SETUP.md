# Connect a Reddit account

NexusSphere uses Reddit OAuth 2.0 to read the profile and posts submitted by the Reddit account that authorizes the connection. It does not post, vote, message, or read private messages.

## 1. Create the Reddit app

Open [Reddit app preferences](https://www.reddit.com/prefs/apps) while signed in to the Reddit account that will own the integration. In the **create application** form shown in the setup screen:

1. Choose a recognizable app name, such as `NexusSphere`.
2. Enter a short description, such as `Read-only social feed for my NexusSphere account`.
3. Keep **web app** selected. The integration runs on the server and needs a client secret; do not choose **script** or **installed app**.
4. The **about URL** is optional for local testing. If you enter one, use a real site you control.
5. Set **redirect uri** exactly to:

   ```text
   http://localhost:3000/api/v1/connections/reddit/callback
   ```

   This must match `REDDIT_CALLBACK_URL` exactly. The callback goes through the web app so the browser's OAuth state cookie is preserved. Use `localhost:3000` consistently, not `127.0.0.1`.
6. Complete the CAPTCHA and select **create app**.

If Reddit requires API registration, Data API approval, or additional app details for your account, follow the current instructions displayed by Reddit before testing. App creation by itself does not guarantee production or commercial API access.

## 2. Add the app credentials locally

On the created app card:

- The **client ID** is the short value shown beneath the app name and app type. It is not the app name.
- The **client secret** is labeled `secret`.

Put both values directly into the root `.env` file on your machine:

```dotenv
REDDIT_CLIENT_ID=<client-id>
REDDIT_CLIENT_SECRET=<client-secret>
REDDIT_CALLBACK_URL=http://localhost:3000/api/v1/connections/reddit/callback
REDDIT_USER_AGENT=web:NexusSphere:v1.0.0 (by /u/<your_reddit_username>)
```

Replace the placeholders locally. Use the Reddit username of the app owner in the User-Agent. Keep the `.env` file private; never paste credentials, tokens, authorization codes, or screenshots of secrets into chat or commit them. The `.env.example` file is a template and should not receive real values.

Restart the API after changing `.env` so it loads the new app credentials.

## 3. Authorize the Reddit account

1. Ensure PostgreSQL and Redis are running and the API and web app are both running.
2. Open `http://localhost:3000` and sign in to NexusSphere.
3. Select **Connect an account** and then **Connect Reddit**.
4. Review Reddit's consent page. NexusSphere requests only `identity` and `read`; approve only if these are appropriate for the app.
5. Reddit should return to NexusSphere with an account-connected notice. The first sync is queued automatically.
6. Select the Reddit channel to inspect its status. If it reports a sync failure, use **Retry** once and inspect the API/worker terminal for the sanitized provider status.

The callback must remain the exact registered redirect URI. If the callback is rejected, compare the Reddit app's redirect URI with `REDDIT_CALLBACK_URL` character-for-character, including hostname, port, path, and trailing slash.

## Troubleshooting

- **Reddit says `invalid_client` or token exchange fails:** verify the client ID and secret belong to this Reddit app, that the app type is **web app**, and that the API process was restarted after editing `.env`.
- **Redirect URI mismatch:** use the exact callback above in both Reddit app preferences and `.env`; test through `localhost:3000`.
- **The app connects but no posts appear:** check the connection status and worker logs. The adapter reads the authorized user's submitted posts; it does not aggregate that user's saved feed or subreddit timeline.
- **HTTP 401/403 or API policy errors:** verify the granted `identity` and `read` scopes and check the current Reddit Data API eligibility, rate limits, and terms for the app/account.
- **Do not share raw logs if they contain credentials.** Share only the HTTP status and sanitized error message.

Review Reddit's current [Data API terms and access requirements](https://support.reddithelp.com/hc/en-us/articles/16160319875092-Data-API-Wiki) before testing beyond your own account or inviting other users. Requirements may change and may differ by use case.
