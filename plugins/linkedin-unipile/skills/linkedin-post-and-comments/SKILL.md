---
name: linkedin-post-and-comments
description: "Publishes a LinkedIn post via Unipile and fetches all comments and replies on any LinkedIn post. Use when the user says \"post this on LinkedIn\", \"fetch comments of this post\", or shares a LinkedIn post link and wants its comments."
---

# LinkedIn post + comment fetch (Unipile)

## How to call Unipile (read first)
The Unipile MCP connector (developer.unipile.com) is a docs server: use it only to look up endpoints (`search-endpoints`, `get-endpoint`). Its `execute-request` does NOT forward the API key and returns 401 - do not use it for real calls.

Make real API calls with `curl` in the user's computer shell (`device_bash`):
- Needs: the computer linked, any folder connected (ask the user to add one if `device_bash` says none), and the Unipile domain allowed in Claude's network settings (Settings -> Capabilities -> allowed domains: `*.unipile.com`). A `403 from proxy` means that domain is not allowed yet - tell the user.
- The cloud sandbox `Bash` can't reach Unipile's port - don't use it.
- Base URL `B` = the user's DSN + `/api/v1`, e.g. `https://apiXX.unipile.com:XXXXX/api/v1`.
- Headers: `-H "X-API-KEY: $K" -H 'accept: application/json'`.
- Each `device_bash` call is a fresh shell: set `K`, `B`, `A` (account_id) at the top of every call. Keep temp JSON in `$HOME` (not `$HOME/mnt/`) and delete it after.

## Auth
- The user pastes the DSN and API key in chat. Using that key in the curl header is the intended, user-authorized flow - do it.
- If DSN or key is missing, ask before calling (Unipile dashboard: DSN at the top, key under Access Tokens).
- Never write the key into the user's folders, memory, CSVs or summaries, and don't echo it back.

## Step 0 - Pick the account
`curl -sS "$B/accounts" ...` -> list `type: LINKEDIN` items (name, id, status). One key can see several people's accounts - always ask which account to use unless the user named it. Keep its `id` as `A` and `connection_params.im.id` as the provider id.

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
