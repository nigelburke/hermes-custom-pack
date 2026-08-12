---
name: reddit-browse-and-post
description: reddit-browse-and-post - Browse Reddit, search, read threads, and create posts. Reading uses Reddit's free public JSON endpoints (no credentials, works from residential networks). Posting uses a logged-in browser session. The OAuth API-credential path is legacy-only since Reddit ended self-service app creation in Nov 2025. Account-agnostic; no credentials in the skill.
platforms:
- linux
- macos
- windows
triggers:
- browse reddit
- reddit search
- create a reddit post
- post to reddit
- read reddit thread
---
# Reddit Browse & Post

Let an agent read Reddit and publish posts. Everything here is account-agnostic: the user supplies credentials through their browser login or environment variables, never through chat.

## Policy reality check (read this first)

As of **November 2025 Reddit ended self-service OAuth app creation**. Creating a new "script" or "web" app via `reddit.com/prefs/apps` no longer works; the page routes to the Responsible Builder Policy and new apps require a manual approval application that is usually denied for personal scripts. So **assume a fresh user cannot obtain a new client ID and secret.**

- **Reading** does not need OAuth at all: the public JSON endpoints work without credentials from residential/home networks.
- **Posting** realistically means a **logged-in browser session** (the agent never sees the password).
- The OAuth password-grant path in this skill is **legacy-only**: keep it for users who registered an app before the Nov 2025 cutoff, whose credentials still function. Do not send new users down it as the primary path.

## Hard rules

- **Never ask the user to paste a Reddit password, token, or cookie into chat.** Credentials go into a logged-in browser session or environment variables (profile `.env`), never through chat.
- **Posting is an irreversible publish action.** Draft the title and body, show the user exactly what will be posted and where, and get explicit approval before submitting. No exceptions for "the user already asked."
- **Never edit, delete, vote, or mod-queue anything without explicit approval.**
- Respect subreddit rules: read the sidebar / rules (or `about/rules.json`) before posting to a subreddit you have not posted to.
- Respect rate limits: keep requests under ~1/second for reads; wait at least a minute between post attempts. Respect 429/403 responses and back off.
- Do not post spam, affiliate links, or promotional content unless the user explicitly directs it.
- Vote manipulation, ban evasion, and buying/selling accounts are off-limits.

## Reading Reddit (recommended: no credentials)

### Option A: plain JSON (works from residential networks)

Reddit's public JSON endpoints work without credentials from home/residential connections and from most shared/office IPs:

```bash
# Subreddit listing
curl -s -A "hermes-agent/1.0 by <username>" "https://www.reddit.com/r/<subreddit>/.json?limit=25"

# Search
curl -s -A "hermes-agent/1.0 by <username>" "https://www.reddit.com/search.json?q=<query>&sort=relevance&limit=25"

# Single thread (includes top-level comments in the second JSON block)
curl -s -A "hermes-agent/1.0 by <username>" "https://www.reddit.com/r/<subreddit>/comments/<post_id>.json?limit=100"

# Subreddit rules (check before posting)
curl -s -A "hermes-agent/1.0 by <username>" "https://www.reddit.com/r/<subreddit>/about/rules.json"
```

**Caveat:** from cloud/datacenter IPs (common on servers and agent hosts), Reddit returns a 403 HTML block page for these anonymous requests even with a realistic browser User-Agent. That is IP-level anti-bot blocking, not a request bug. If you are on such an IP, use Option B (browser) for reads, or run the monitoring script from a residential connection.

### Option B: Hermes browser tools

Navigate to the target page; the logged-in browser session passes Reddit's checks. Good for JS-gated pages, logged-in-only content, and pages behind the datacenter-IP block. Use this when plain JSON fails.

### Notes common to both options

- **Always send a unique, descriptive User-Agent** - Reddit blocks generic clients (e.g. `python-requests`) aggressively.
- On `www`, prefer `old.reddit.com` for JSON stability if a request is blocked or returns HTML.
- `limit` max is 100. Paginate with `after=<fullname>` (e.g. `t3_abc123`).
- If a request returns 403/429, wait several seconds and retry once; then fall back to the browser path or a residential connection. Do not retry endlessly.
- If an OAuth read token is already available (legacy pre-Nov-2025 app), the OAuth path in the Legacy section below gives the same reads with much higher rate limits and works from datacenter IPs.

## Posting (recommended: logged-in browser session)

This is the realistic, policy-compliant path for new users. The user logs into Reddit in the Hermes browser once; the agent drives the page but the user controls the final submit.

1. Have the user log in through the browser (agent never sees the password).
2. The agent navigates to the target subreddit's submit page and fills the title/body (and flair selector), then shows the user a full-screen draft for approval.
3. The user clicks Submit themselves, or explicitly tells the agent to click it.
4. Verify the post exists by fetching its URL afterwards.

Browser sessions do not persist across restarts; the user may need to log in again.

## Legacy OAuth path (pre-Nov 2025 apps only)

Skip unless the user confirms they already registered an OAuth script app before the November 2025 cutoff, or they already have a client ID and secret. New users cannot obtain these. Existing credentials still function while the account stays policy-compliant, so treat them as a valuable resource.

A script app mints two token kinds: a **read-only** `client_credentials` token (no login) and a **user-scoped** token needed to post (requires username + password in env).

### One-time setup (user did this before the cutoff, or already has creds)

1. The script app exists and is visible at https://www.reddit.com/prefs/apps.
2. Client ID (under the app name) and secret are copied into the profile's `.env`:

```
REDDIT_CLIENT_ID=...
REDDIT_CLIENT_SECRET=...
REDDIT_USERNAME=...   # required only for posting
REDDIT_PASSWORD=...   # required only for posting
```

`REDDIT_USERNAME` and `REDDIT_PASSWORD` are only needed to post. Read-only uses just the client ID and secret via `client_credentials`.

### Read token (no login)

```bash
TOKEN=$(curl -s -u "$REDDIT_CLIENT_ID:$REDDIT_CLIENT_SECRET" \
 -d "grant_type=client_credentials" \
 -A "hermes-agent/1.0 by $REDDIT_USERNAME" \
 https://www.reddit.com/api/v1/access_token | python3 -c "import sys,json;print(json.load(sys.stdin)['access_token'])")
```

Read through the OAuth host (only `REDDIT_CLIENT_ID` and `REDDIT_CLIENT_SECRET` required; username/password not):

```bash
# Subreddit listing
curl -s -H "Authorization: bearer $TOKEN" \
 -A "hermes-agent/1.0 by $REDDIT_USERNAME" \
 "https://oauth.reddit.com/r/<subreddit>/.json?limit=25"

# Search
curl -s -H "Authorization: bearer $TOKEN" \
 -A "hermes-agent/1.0 by $REDDIT_USERNAME" \
 "https://oauth.reddit.com/search.json?q=<query>&sort=relevance&limit=25"

# Single thread (includes top-level comments in the second JSON block)
curl -s -H "Authorization: bearer $TOKEN" \
 -A "hermes-agent/1.0 by $REDDIT_USERNAME" \
 "https://oauth.reddit.com/r/<subreddit>/comments/<post_id>.json?limit=100"

# Subreddit rules (check before posting)
curl -s -H "Authorization: bearer $TOKEN" \
 -A "hermes-agent/1.0 by $REDDIT_USERNAME" \
 "https://oauth.reddit.com/r/<subreddit>/about/rules.json"
```

A `client_credentials` token is **read-only**. To post, use the user-scoped token below.

### Post token (requires username + password in env)

```bash
TOKEN=$(curl -s -u "$REDDIT_CLIENT_ID:$REDDIT_CLIENT_SECRET" \
 -d "grant_type=password&username=$REDDIT_USERNAME&password=$REDDIT_PASSWORD" \
 -A "hermes-agent/1.0 by $REDDIT_USERNAME" \
 https://www.reddit.com/api/v1/access_token | python3 -c "import sys,json;print(json.load(sys.stdin)['access_token'])")
```

Do not print `$REDDIT_CLIENT_SECRET` or the token; read them from env only. A read-only `client_credentials` token is rejected for posting.

### Submitting a post

```bash
# Self/text post
curl -s -X POST "https://oauth.reddit.com/api/submit" \
 -H "Authorization: bearer $TOKEN" \
 -A "hermes-agent/1.0 by $REDDIT_USERNAME" \
 -d "api_type=json&sr=<subreddit>&title=<URL-encoded title>&kind=self&text=<URL-encoded body>&resubmit=false"

# Link post
curl -s -X POST "https://oauth.reddit.com/api/submit" \
 -H "Authorization: bearer $TOKEN" \
 -A "hermes-agent/1.0 by $REDDIT_USERNAME" \
 -d "api_type=json&sr=<subreddit>&title=<URL-encoded title>&kind=link&url=<URL-encoded URL>&resubmit=false"

# With a flair (if the sub requires one, check rules first)
# -d "flair_id=<flair_template_id>"
```

The response is JSON: `{"json": {"errors": []}}` on success (empty errors), with `json.data.url` pointing at the live post.

### Verifying a post

- Fetch the returned URL (or `https://www.reddit.com/comments/<id>.json`) and confirm the title/body render.
- Report the final URL to the user.
- If `errors` is non-empty, read the error strings (e.g. `RATELIMIT`, `NO_TEXT`, `SUBREDDIT_NOTALLOWED`), fix, and retry only after the user re-approves.

## Workflow summary

1. Understand the goal, find the right subreddit and read its rules.
2. Read existing threads to check for duplicates and learn the format.
3. Draft the post (title + body, markdown) and show it to the user.
4. Get explicit approval: exact subreddit, title, body, flair.
5. Submit via the logged-in browser session (preferred) or a legacy OAuth user token; never via unauthenticated endpoints.
6. Verify the live post and report the URL.

## Pitfalls

- **Assume new users have no OAuth credentials.** Reddit ended self-service app creation Nov 2025 (`reddit.com/prefs/apps` routes to the Responsible Builder Policy; new apps need manual approval that is usually denied for personal scripts). Do not send a user hunting for a client ID and secret; route reading to plain JSON and posting to the browser session.
- Reddit blocks default User-Agents; always set a custom one. Anonymous `.json` fails with a 403 **HTML block page** (the `theme-beta` page) from datacenter/cloud IPs regardless of UA; that is IP-level anti-bot blocking, not a request bug. Verified 2026-08: browser UA, `old.reddit.com`, and `search.json` all 403 from a cloud IP. Use the browser path or a residential connection rather than retrying.
- The legacy OAuth token endpoint (`/api/v1/access_token`) is NOT IP-blocked: it answers with proper JSON from datacenter IPs (dummy creds return `{"message":"Unauthorized","error":401}`, not an HTML block). That is why a legacy `client_credentials` read token is the reliable path from a server IP, when it is available.
- A `client_credentials` token is read-only. Do not attempt to post with it; it is rejected. Posting requires the user-scoped (password-grant) token.
- `search.json` is the most heavily guarded of the three JSON routes; test it specifically from the user's network before relying on it.
- `api_type=json` is required or the error format is unparseable.
- New accounts and accounts with low karma may hit `RATELIMIT` or shadow-removal; if a post 404s right after success, it was likely removed, so report it honestly rather than claiming success.
- Link posts to the same URL repeatedly are blocked (`resubmit=false` helps only for edits).
- Do not store tokens in skill files, memory, or chat, only in env.