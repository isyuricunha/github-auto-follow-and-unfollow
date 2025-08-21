# github-auto-follow-and-unfollow

I use this repo to automatically follow back people who follow me, and unfollow people who don't follow back. It runs on a schedule via GitHub Actions, talks to the GitHub REST API, and sends me a Telegram notification of what changed.

[![Script](https://github.com/isyuricunha/github-auto-follow-and-unfollow/actions/workflows/main.yml/badge.svg)](https://github.com/isyuricunha/github-auto-follow-and-unfollow/actions/workflows/main.yml)

## What this does

- __Syncs following with followers__: 
  - If someone follows me and I don't follow them, it follows them.
  - If I'm following someone who doesn't follow me back, it unfollows them.
- __Sends a Telegram message__ summarizing actions taken (follow/unfollow with links).
- __Updates the README__ with the last run time and simple counters (rate limit and counts).

All of this logic lives in `script.php` and is executed by `.github/workflows/main.yml` on a schedule.

## How it works (quick overview)

The script uses the GitHub REST API with basic auth (username + token):

- `GET /user` to read my profile and rate limit headers.
- `GET /user/followers` and `GET /user/following` to fetch up to 30 pages (100 per page) of users.
- Compares the two sets and calls `PUT /user/following/{username}` to follow and `DELETE /user/following/{username}` to unfollow.
- Sends a Telegram message via `https://api.telegram.org/bot<token>/sendMessage`.
- Writes a small run summary back into `README.md`.

All requests use a mobile Chrome user-agent and respect GitHub's rate limit headers which the script parses (`X-RateLimit-Used` and `X-RateLimit-Limit`).

## Repo structure

- `script.php`: main PHP script with all logic (GitHub API calls, diffing, Telegram notify, README update).
- `.github/workflows/main.yml`: GitHub Actions workflow that runs the script every 5 minutes and on pushes to `main`.
- `license.md`: license file.

## Setup

You can fork and run this under your own account. You'll need a few secrets.

1) __Create a GitHub token__ (classic or fine‑grained) for API auth

- Minimum scopes (classic token):
  - `read:user`
  - `user:follow`
- For fine‑grained tokens, allow read on user and write on following (User follow/unfollow) for your account.

2) __Create a Telegram bot and get your chat id__

- Create a bot with @BotFather and grab the bot token.
- Get your chat id (e.g., by messaging the bot and using a helper bot like @userinfobot or via a small script). This script sends messages in HTML parse mode.

3) __Add repo secrets__ (Settings → Secrets and variables → Actions → New repository secret)

- `flx_token`: your GitHub token created above.
- `telegram_api`: your Telegram bot token (the value after `bot`).
- `chat_id`: your chat id (where the bot will send messages).

4) __Update the username in the workflow__

The workflow currently runs:

```yaml
run: |
  php script.php isyuricunha ${{ secrets.flx_token }} ${{ secrets.telegram_api }} ${{ secrets.chat_id}}
```

Replace `isyuricunha` with your GitHub username if you're using this in your own repo.

## Running locally

Prereq: PHP with cURL enabled.

```bash
php script.php <github_username> <github_token> <telegram_bot_token> <telegram_chat_id>
```

It will:

- Read your followers/following.
- Follow/unfollow to sync.
- Send a Telegram summary.
- Overwrite `README.md` with a run summary (see caveats below).

## GitHub Actions schedule

The workflow is configured to run every 5 minutes and on pushes to `main`:

```yaml
on:
  push:
    branches:
      - main
  schedule:
    - cron: "*/5 * * * *"
```

It uses `franzliedke/gh-action-php@0.3.0` to provide PHP, then commits and pushes any changes (like the README update) back to the `main` branch.

## Caveats and notes

- __README is auto‑generated__: `script.php` writes the run summary to `README.md` at the end of each run. If you want to keep a hand‑written README, either:
  - Remove or comment out the `file_put_contents("README.md", ...)` at the end of `script.php`, or
  - Move the auto‑generated stats to a separate file and adjust the script accordingly.
- __Pagination limit__: the script fetches up to 30 pages at 100 entries each (max ~3000) for followers and following. Accounts with more than that won't be fully processed.
- __Rate limits__: the script reads and logs GitHub rate‑limit headers. Running every 5 minutes may be fine for small changes, but if your account is very active, consider relaxing the cron schedule.
- __Error handling__: if the API returns a `message`, the script stops fetching more pages. Telegram messages are attempted regardless; you may see a notification even if no change occurred.
- __Auth__: the script uses basic auth (`username:token`). Use a token, not a password.
- __Scope__: this strictly mirrors followers ↔ following; it doesn't maintain custom allow/deny lists.

## Customizing behavior

If you want to add rules (e.g., never unfollow certain users, only follow accounts with criteria, etc.), you'd edit `script.php` around the comparison logic:

- Unfollow block: where `doAction($username, $password, "DELETE", $fl["login"])` is called for accounts you're following that don't follow back.
- Follow block: where `doAction($username, $password, "PUT", $fl["login"])` is called for followers you don't follow yet.

You can gate those calls with your own conditions.

## Security

- Keep your token in GitHub Actions secrets. Never commit tokens to the repo.
- Telegram messages go through the bot API; don't put sensitive data in the message body.

## License

See `license.md`.

