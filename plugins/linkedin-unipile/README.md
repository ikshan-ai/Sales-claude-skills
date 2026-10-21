# linkedin-unipile

Publish LinkedIn posts and fetch every comment and reply on a post, using Unipile.

## What you need
1. A Unipile account (dashboard.unipile.com) with your LinkedIn account connected.
2. Your Unipile **DSN** (shown at the top of the dashboard, e.g. `apiXX.unipile.com:XXXXX`).
3. A Unipile **API key** (dashboard -> Access Tokens -> Generate).

The plugin adds the Unipile connector (used for endpoint docs). Real API calls run as `curl` on your computer, so:
- link your computer to Claude and connect any folder,
- allow `*.unipile.com` in Claude Settings -> Capabilities -> allowed domains,
- paste your DSN and API key in chat when asked. Never commit your API key to this repo.

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
