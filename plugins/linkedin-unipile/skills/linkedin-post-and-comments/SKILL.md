---
name: linkedin-post-and-comments
description: "Publishes a LinkedIn post via Unipile and fetches all comments and replies on any LinkedIn post. Use when the user says \"post this on LinkedIn\", \"fetch comments of this post\", or shares a LinkedIn post link and wants its comments."
---

# LinkedIn post + comment fetch (Unipile)

## How to call Unipile (read first)
Always build the full URL from the user's DSN: `B = https://<DSN>/api/v1` (e.g. `https://apiXX.unipile.com:XXXXX/api/v1`). NEVER use the connector's default server `api1.unipile.com:13111` - it returns `503 no_client_session`. If the DSN isn't in the conversation, ask for it first (Unipile dashboard, top of the page). The DSN is not a secret.

Route 1 - Unipile connector (try first):
- Call `execute-request` with the full `B/...` URL and header `accept: application/json`.
- If the user added `X-API-KEY` as a Request header when adding the connector in Claude, the key is sent automatically - don't ask for it.
- If the user pasted a key in chat, also pass it as an `X-API-KEY` header in the request (user-authorized; do it).
- `503 no_client_session` -> wrong URL/DSN. `401` -> key missing/wrong or not forwarded -> go to Route 2.
- Connector shows `needs_reconnect` / tools missing -> ask the user to reconnect Unipile in Settings -> Connectors.

Route 2 - curl on the user's computer (fallback):
- Use `device_bash` (computer linked + any folder connected) with `curl -sS "$B/..." -H "X-API-KEY: $K" -H 'accept: application/json'`. Needs the key in chat.
- Needs `*.unipile.com` allowed in Claude Settings -> Capabilities -> allowed domains; `403 from proxy` means it isn't.
- The cloud sandbox `Bash` can't reach Unipile's port - don't use it.
- Each `device_bash` call is a fresh shell: set `K`, `B`, `A` at the top every time; temp JSON in `$HOME` (not `$HOME/mnt/`), delete after.

The request shapes in the steps below are the same for both routes (curl syntax shown; translate to execute-request: multipart params for POST /posts, query string for GETs).

## Step 0 - Pick the account
- Before anything, try Route 1 directly - don't ask for the API key upfront.
- If the user's memory/preferences name a default Unipile account (e.g. "default account: <name>"), use it WITHOUT asking for reading posts/comments. Find its `id` via `GET $B/accounts` (match by name) or use the saved id.
- Only if there is no default: for reading someone else's public post, use the first LinkedIn account with status OK and say which one you used; ask only if the user wants a specific one.
- For POSTING, always confirm which account the post will go from, even with a default.
- Keep the account `id` as `A` and `connection_params.im.id` as its provider id.

## Step 1 - Draft & confirm the post
- Draft in the user's tone; confirm the post language.
- Show the exact final text AND which account it will post from, and wait for an explicit OK. Posting is public and can't be undone from here.

## Step 2 - Publish
`curl -sS -X POST "$B/posts" -H "X-API-KEY: $K" -H 'accept: application/json' -F "account_id=$A" -F "text=<text>"`
- Optional: `-F external_link=...`, `-F as_organization=<org id>` (company page), images via `-F attachments=@/path/to/file` (file must be in a connected folder).
- Response `{object: PostCreated, post_id}` -> report `post_id`.

## Step 3 - Resolve social_id
Comments need the post's `social_id` (`urn:li:activity:<digits>`).
- From a LinkedIn URL: take the digits after `activity-` or `urn:li:activity:` -> `urn:li:activity:<digits>`.
- After publishing: `curl "$B/posts/<post_id>?account_id=$A"` -> `social_id`.
- To list the user's own recent posts: `curl "$B/users/<provider id>/posts?account_id=$A&limit=10"` (shows social_id, comment_counter, text).

## Step 4 - Fetch ALL comments
`curl -sS "$B/posts/<url-encoded social_id>/comments?account_id=$A&limit=100" ...` (encode `:` as `%3A`).
- Pagination: while `cursor` is non-null, call again with `&cursor=<value>`.
- Replies: for each comment with `reply_counter > 0`, call again with `&comment_id=<comment id>` (also paginate).
- Use a short python script in the same `device_bash` call to loop and collect.

Keep per comment: id, parent_comment_id (replies), author, author_details.headline, author_details.profile_url, network_distance, date, text, reaction_counter, reply_counter.

## Step 5 - Output
- Chat summary: total comments + replies, top commenters, main themes/questions.
- CSV (one row per comment/reply) written into the user's connected folder as `linkedin_comments_<activity-id>_<YYYY-MM-DD>.csv`; tell them the file name.
- Never invent missing fields - leave blank.

## Errors
- `403 from proxy` -> Unipile domain not allowed in Claude settings.
- 401 `invalid_credentials` -> wrong key, or key and DSN from different Unipile workspaces.
- 401 `disconnected_account` -> reconnect that LinkedIn account in the Unipile dashboard.
- 403 `account_restricted` / `action_required` -> the account owner must open LinkedIn and resolve it.
- New post with 0 comments is normal - offer to re-check later.
