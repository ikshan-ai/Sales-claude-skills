# linkedin-unipile

Publish LinkedIn posts and fetch every comment and reply on a post, using Unipile.

## What you need
1. A Unipile account (dashboard.unipile.com) with your LinkedIn account connected.
2. Your Unipile **DSN** (shown at the top of the dashboard, e.g. `apiXX.unipile.com:XXXXX`).
3. A Unipile **API key** (dashboard -> Access Tokens -> Generate).

Setup:
1. In Claude, Settings -> Connectors -> Unipile: when adding it, choose **No sign-in** and add a Request header `X-API-KEY` = your Unipile API key.
2. In chat, tell Claude your DSN once, e.g. "my Unipile DSN is https://apiXX.unipile.com:XXXXX/api/v1".
3. Fallback if the connector returns 401: link your computer, connect any folder, allow `*.unipile.com` in Settings -> Capabilities, and paste the key in chat - Claude will use curl.

Never commit your API key to this repo.

## Usage
- "Post this on LinkedIn: ..." - Claude drafts, shows the final text, and posts only after you confirm.
- "Fetch all comments of this post: <link>" - returns a summary plus a CSV of every comment and reply.

## Limits
- Image posts are not supported through the connector (text and link posts only).
- Fetching many posts in bulk can get a LinkedIn account restricted; keep it light.

## Install
Claude desktop -> Customize -> Plugins -> Add marketplace -> `RohitMakaniProfile/Sales-claude-skills` -> install **linkedin-unipile**.

Claude Code:
```
/plugin install linkedin-unipile --marketplace RohitMakaniProfile/Sales-claude-skills
```
