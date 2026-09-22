# TODO

Human-only steps — each one needs a vendor console, a browser OAuth flow,
or a credential only Jared holds. Source of truth for anything that
changes is `inframanager/docs/social-publishing.md`; update that first,
then this list.

## Credential the social-publish pipeline (mineralsaga)

The pipeline (`inframanager/.github/workflows/social-publish.yml`, hourly
cron) is built and already running, but every platform secret is unset,
so every run fails. Four independent one-time setups:

1. **Meta (Instagram + Facebook)**
   - Instagram account must be Business/Creator, connected to a Facebook
     Page (Instagram → Settings → Business tools → Connect a Page).
   - developers.facebook.com → Create app → type Business → add products
     Instagram and Facebook Login for Business → leave in Development mode.
   - Graph API Explorer → User Token → permissions `pages_show_list`,
     `pages_manage_posts`, `pages_read_engagement`, `instagram_basic`,
     `instagram_content_publish`, `business_management` → Generate → copy
     the short-lived token. App settings → Basic → copy app id + secret.
   - Run:
     ```bash
     cd ~/inframanager
     python3 scripts/social_auth.py meta
     ```
     Stores `META_PAGE_TOKEN`, `META_PAGE_ID`, `META_IG_USER_ID`.

2. **TikTok**
   - developers.tiktok.com → Create app → add Login Kit + Content Posting
     API (enable Direct Post) → scopes `user.info.basic`, `video.publish`,
     `video.upload`.
   - Login Kit → Redirect URI `https://mineralsaga.com/oauth/tiktok`
     (404 is expected, the auth code is in the address bar).
   - Content Posting API → verify domain `mineralsaga.com`.
   - Submit the app for review ("audit") — posts are `SELF_ONLY` until it
     passes (already the default).
   - Run:
     ```bash
     cd ~/inframanager
     python3 scripts/social_auth.py tiktok
     ```
     Stores `TIKTOK_CLIENT_KEY`, `TIKTOK_CLIENT_SECRET`,
     `TIKTOK_REFRESH_TOKEN` (365-day refresh token — calendar reminder).

3. **YouTube**
   - console.cloud.google.com → existing GCP project → enable YouTube Data
     API v3.
   - OAuth consent screen → External → app name + email → scope
     `.../auth/youtube.upload` → Publish app, status **In production**
     (Testing tokens die after 7 days).
   - Credentials → Create → OAuth client ID → Desktop app → copy id + secret.
   - Run:
     ```bash
     cd ~/inframanager
     python3 scripts/social_auth.py youtube
     ```
     Stores `YOUTUBE_CLIENT_ID`, `YOUTUBE_CLIENT_SECRET`,
     `YOUTUBE_REFRESH_TOKEN`. Videos stay locked private until the app
     passes YouTube's compliance audit, regardless of privacy setting.

4. **The repo side**
   - Fine-grained PAT `SOCIAL_REPOS_PAT`: owner `jaredpsloan`, scoped to
     `mineralsaga` only, Contents: read and write.
     ```bash
     gh secret set SOCIAL_REPOS_PAT -R jaredpsloan/inframanager
     ```
   - Repo variables:
     ```bash
     gh variable set SOCIAL_REPOS -R jaredpsloan/inframanager -b "jaredpsloan/mineralsaga"
     gh variable set META_API_VERSION -R jaredpsloan/inframanager -b "v22.0"
     gh variable set TIKTOK_PRIVACY -R jaredpsloan/inframanager -b "SELF_ONLY"
     gh variable set YOUTUBE_PRIVACY -R jaredpsloan/inframanager -b "public"
     ```
   - Deploy the inbox-drop `/post` route update to the box:
     ```bash
     cd ~/inframanager/services/inbox-drop
     source ~/.config/jaredpsloan/infra.env
     bash install.sh
     ```
   - Bind the bot in `agentmanager`
     (`ops/drafts/create-bot-mineralsaga-social-poster.md` is ready to
     paste) and add it to `DROP_TOKENS` in `/etc/inbox-drop.env` on the box.

Once all four are done, dry-run before the first real post:

```bash
gh workflow run social-publish.yml -R jaredpsloan/inframanager -f dry_run=true
```

Ask Claude to run the `gh secret set` / `gh variable set` / box-deploy
steps once the browser parts are done and tokens are in hand — those are
copy-paste from a terminal, not a browser.
