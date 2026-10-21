---
name: "linkedin-post-and-comments"
description: "Publishes a LinkedIn post via the Unipile connector and fetches all comments and replies on any LinkedIn post. Use when the user says \"post this on LinkedIn\", \"fetch comments of this post\", or shares a LinkedIn post link and wants its comments."
---

# LinkedIn post + comment fetch (Unipile)

Tools: the Unipile connector's `execute-request` tool (load via ToolSearch with the other Unipile tools if deferred). The connector is a generic API executor, so EVERY request needs:
- Base URL = the user's Unipile DSN: `https://<DSN>` (e.g. `https://apiXX.unipile.com:XXXXX`). The default `api1...:13111` usually returns `no_client_session` - always use the user's own DSN.
- Header `X-API-KEY: <access token>` and `accept: application/json`.

If DSN or API key is not in the conversation, ask the user for them first (Unipile dashboard -> DSN shown top, Access Tokens page for the key). Never write the API key into files or memory.

## Step 0 - Get account_id
GET `{DSN}/api/v1/accounts` -> pick the item with `type: LINKEDIN` (if several, ask the user which one). Keep its `id` as `account_id`.

## Step 1 - Draft & confirm the post
- Write the draft (match the user's tone; confirm the post language).
- Show the exact final text and ask for explicit OK. Posting is public and can't be undone from here - never post without a clear "haan, post kar do".

## Step 2 - Publish
POST `{DSN}/api/v1/posts` with `multipart/form-data` params:
- `account_id` (required), `text` (required)
- optional: `external_link`, `mentions` (use `{{0}}` in text), `as_organization` (company page id)
Images need binary `attachments` - the MCP executor can't upload local files reliably; if images are needed, tell the user to add them manually or skip.
Response: `{object: PostCreated, post_id}`. Save `post_id` and report it.

## Step 3 - Resolve social_id
Comments API needs the post's `social_id` (looks like `urn:li:activity:...`), not always the id from the URL.
GET `{DSN}/api/v1/posts/{post_id}?account_id=...` -> read `social_id`.
If the user gives a LinkedIn post URL instead, extract the activity number from the URL (`activity-<digits>` or `urn:li:activity:<digits>`) and call GET post with that to get `social_id`.

## Step 4 - Fetch ALL comments
GET `{DSN}/api/v1/posts/{social_id}/comments` with query `account_id`, `limit=100`, `sort_by=MOST_RECENT`.
- Pagination: if response `cursor` is non-null, call again with `cursor=<value>` until null. Don't stop at page 1.
- Replies: for every comment with `reply_counter > 0`, call same endpoint with `comment_id=<comment.id>` (also paginate).
- URL-encode the social_id in the path (`urn%3Ali%3Aactivity%3A...`) if the raw one errors.

Fields to keep per comment: id, parent comment id (for replies), author, author_details.headline, author_details.profile_url, network_distance, date, text, reaction_counter, reply_counter.

## Step 5 - Output
- Summary in chat: total comments + replies, top commenters, quick themes/questions asked.
- Full data as a CSV (one row per comment/reply) delivered as a file; if the user's folder is connected, save it there as `linkedin_comments_<post-id>_<date>.csv`.
- Do not invent or fill missing fields - leave blank.

## Errors
- 503 `no_client_session` -> wrong DSN.
- 401 -> bad/missing API key, or LinkedIn account disconnected in Unipile (reconnect in dashboard).
- 403 `account_restricted` / `action_required` -> the user must open LinkedIn in browser and resolve.
- Fresh post with 0 comments is normal - offer to re-check later (can set a scheduled task).