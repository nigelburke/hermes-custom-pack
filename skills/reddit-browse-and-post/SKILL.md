---
name: reddit-browse-and-post
description: reddit-browse-and-post - Browse Reddit, search, read threads and comments, and create posts through the agent's own OAuth app or authenticated session. Read-only works with a no-login client_credentials token; posting needs the account's user-scoped token. Account-agnostic; no credentials in the skill.
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

Let an agent read Reddit and publish posts using the user's own Reddit account. Everything here is account-agnostic: the user supplies their credentials through environment variables or a browser login, never through chat.

## Hard rules

- **Never ask the user to paste a Reddit password, token, or cookie into chat.** Credentials go into environment variables (profile `.env`) or the user logs in via the browser themselves.
- **Posting is an irreversible publish action.** Draft the title and body, show the user exactly what will be posted and where, and get explicit approval before submitting. No exceptions for "the user already asked."
- **Never edit, delete, vote, or mod-queue anything without explicit approval.**
- Respect subreddit rules: read the sidebar / rules (or `about/rules.json`) before posting to a subreddit you have not posted to.
- Respect rate limits: keep requests under ~1/second for reads; wait at least a minute between post attempts. Respect 429/403 responses and back off.
- Do not post spam, affiliate links, or promotional content unless the user explicitly directs it.
- Vote manipulation, ban evasion, and buying/selling accounts are off-limits.

## Reading Reddit

Reddit's public JSON endpoints (`/r/<sub>/.json`, `/search.json`, `/comments/<id>.json`) work without credentials from residential networks. From cloud or datacenter IPs (common on servers and agent hosts), Reddit returns a 403 HTML block page for those anonymous requests even with a realistic browser User-Agent, so do not rely on anonymous JSON off a home connection.

The reliable, credential-light option is a **no-login read token**: an OAuth `client_credentials` grant from a free script app. No username or password. It works from any IP including datacenter/cloud, and gets far higher rate limits than anonymous `.json`. It only reads, never posts.

### Option A: OAuth read token (recommended, works from any network)

Once the script app exists (Setup below), mint a read-only token with no login:

```bash
TOKEN=$(curl -s -u "$REDDIT_CLIENT_ID:$REDDIT_CLIENT_SECRET" \
 -d "grant_type=client_credentials" \
 -A "hermes-agent/1.0 by $REDDIT_USERNAME" \
 https://www.reddit.com/api/v1/access_token | python3 -c "import sys,json;print(json.load(sys.stdin)['access_token'])")
```

Then read through the OAuth host (only `REDDIT_CLIENT_ID` and `REDDIT_CLIENT_SECRET` are required for this; username and password are not):

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

A `client_credentials` token is **read-only**. To post, you need the user-scoped token in the Posting section below.

### Option B: plain JSON (only if it works from the user's network)

If a plain-curl probe does work (typical on residential connections), anonymous `.json` is the simplest:

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

### Option C: Hermes browser tools

Navigate to the target page; the browser session passes Reddit's checks. Good for JS-gated pages and logged-in work. Use this when the OAuth token path itself is unavailable.

### Notes common to all options

- **Always send a unique, descriptive User-Agent** - Reddit blocks generic clients (e.g. `python-requests`) aggressively.
- On `www`, prefer `old.reddit.com` for JSON stability if a request is blocked or returns HTML.
- `limit` max is 100. Paginate with `after=<fullname>` (e.g. `t3_abc123`).
- If a request returns 403/429, wait several seconds and retry once; then fall back to the OAuth or browser path. Do not retry endlessly.

## Authenticated posting (user-scoped OAuth token)

Your free script app mints two kinds of tokens. A **read-only** `client_credentials` token (no login, used in Reading Option A) and a **user-scoped** token needed to post, which requires the account's username and password in env.

### One-time setup (user does this)

1. Create a **script** app at https://www.reddit.com/prefs/apps (name anything; type: script; redirect uri: `http://localhost:8080` - unused for script apps).
2. Copy the client ID (under the app name) and secret.
3. Put them in the profile's `.env`:

```
REDDIT_CLIENT_ID=...
REDDIT_CLIENT_SECRET=...
REDDIT_USERNAME=...   # required only for posting
REDDIT_PASSWORD=...   # required only for posting
```

`REDDIT_USERNAME` and `REDDIT_PASSWORD` are only needed to post. Read-only browsing uses just the client ID and secret via `client_credentials` (Reading, Option A).

### Getting a post token (from the agent)

```bash
TOKEN=$(curl -s -u "$REDDIT_CLIENT_ID:$REDDIT_CLIENT_SECRET" \
 -d "grant_type=password&username=$REDDIT_USERNAME&password=$REDDIT_PASSWORD" \
 -A "hermes-agent/1.0 by $REDDIT_USERNAME" \
 https://www.reddit.com/api/v1/access_token | python3 -c "import sys,json;print(json.load(sys.stdin)['access_token'])")
```

Do not print `$REDDIT_CLIENT_SECRET` or the token; read them from env only. A read-only `client_credentials` token will be rejected for posting.

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

## Browser-session option (no API app)

If the user does not want an OAuth app, they can log into Reddit in the Hermes browser once:

1. Have the user log in through the browser (agent never sees the password).
2. The agent uses that logged-in session: navigate to the target subreddit's submit page, fill title/body (or flair selector), and show the user a full-screen draft for approval.
3. The user clicks Submit themselves, or explicitly tells the agent to click it.
4. Verify the post exists by fetching its URL afterwards.

Browser sessions do not persist across restarts; the user may need to log in again.

## Workflow summary

1. Understand the goal, find the right subreddit and read its rules.
2. Read existing threads to check for duplicates and learn the format.
3. Draft the post (title + body, markdown) and show it to the user.
4. Get explicit approval: exact subreddit, title, body, flair.
5. Submit via OAuth or browser; never via unauthenticated endpoints.
6. Verify the live post and report the URL.

## Pitfalls

- Reddit blocks default User-Agents; always set a custom one. But anonymous `.json` fails with a 403 **HTML block page** (the `theme-beta` page) from datacenter/cloud IPs regardless of UA; that is IP-level anti-bot blocking, not a request bug. Verified 2026-08: browser UA, `old.reddit.com`, and `search.json` all 403 from a cloud IP. Switch to the OAuth token (Reading Option A) rather than retrying.
- The OAuth token endpoint (`/api/v1/access_token`) is NOT IP-blocked: it answers with proper JSON from datacenter IPs (dummy creds return `{"message":"Unauthorized","error":401}`, not an HTML block). This is why the no-login `client_credentials` read token is the reliable path.
- A `client_credentials` token is read-only. Do not attempt to post with it; it is rejected. Posting requires the user-scoped (password-grant) token.
- `search.json` is the most heavily guarded of the three JSON routes; test it specifically from the user's network before relying on it.
- `api_type=json` is required or the error format is unparseable.
- New accounts and accounts with low karma may hit `RATELIMIT` or shadow-removal; if a post 404s right after success, it was likely removed, so report it honestly rather than claiming success.
- Link posts to the same URL repeatedly are blocked (`resubmit=false` helps only for edits).
- Do not store tokens in skill files, memory, or chat, only in env.
