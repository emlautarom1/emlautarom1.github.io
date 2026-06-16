---
title: "Isolated agent identities for Git and GitHub"
date: 2026-06-16T12:14:12-03:00
summary: A quick guide on how to separate your agent's credentials from your personal ones
draft: true
---

With the rise of agent-first software development, it’s necessary to be able to split the credentials and permissions used by real engineers from agents. The following guide provides one of many solutions to this problem specifically for GitHub by leveraging [GitHub Apps](https://docs.github.com/en/apps/overview). It’s expected that forges like GitHub and GitLab will implement a first-class solution eventually, but for now this should do. We'll also set up our agents to have separate Git credentials so that commits are attributed correctly.

## Expected results

- A separate identity for agents, tied to a single real-world GitHub account.
- Optionally, commits are attributed to a different author.
- Agents can use this identity to interact with GitHub using standard tooling (ex. GitHub CLI, REST endpoints, etc.)
- Broad permissions per repository, like managing PRs, issues and comments.
- Prevent destructive operations (ex. deleting issues) even when the associated real-world GitHub account has all permissions granted.
- The new identity will be flagged as a `bot` in all GitHub activities.
- This identity does not consume seats on your repositories.

## Real world experience

I’ve been using this setup quite successfully in projects at my day job, where agents can create issues, add comments and publish PRs with separate credentials.

## Caveats

There is a risk that an agent will try to use real-world credentials if these are present on the same machine where the agent is running. Take the appropriate precautions for this (ex. use a sandbox, hide environment variables, etc.)

## Pre-requisites

- An existing GitHub account.
- Git installed.
- GitHub CLI installed.
- ~25 minutes.

## How-to

### Create a new GitHub App

1. Visit the GitHub settings page, and access the “Developer settings” page.
2. Create a new GitHub App by clicking the “New GitHub App” button.
3. Fill in the form as follows (feel free to make tweaks though):
    1. “GitHub app name”: Your username + “agent”. For example, `emlautarom1-agent`. This is the display name used by the agent account and as such must be unique.
    2. “Homepage URL”: Any URL that you prefer. I’ve set it up to my GitHub profile.
    3. “Webhook”: Do not set it up as active.
    4. “Permissions”: Requires at least the following, but feel free to extend them as needed:
        1. “Contents”: read and write.
        2. “Issues”: read and write.
        3. “Pull requests”: read and write.
        4. “Metadata”: read.
    5. “Where can this GitHub App be installed?”: Select “Any account”, allowing the app to be installed in repositories that you don’t own (ex. organization-wide repositories).
4. Create the GitHub app.
5. Click on “Generate a private key”. This will download a `.pem` file to your machine. Keep it safe since it stores the credentials to be used by the agent. This private key grants full access to the app, so store it with the same care as an SSH private key (e.g. `chmod 600`, and never commit it).
6. Copy the “App ID” from the “About” section.

### Enabling the GitHub App

1. Go to “Install App”
2. Select the account to install the application.
3. Select which repositories to install to. For repositories that you don’t own (ex. organization-wide repositories), you must have the role of “admin” or request Administrator permissions to install the application.
4. Repeat for all repositories/organizations as appropriate.

### Local setup

1. Install the [Link-/gh-token](https://github.com/Link-/gh-token) extension for the GitHub CLI by running `gh extension install Link-/gh-token` in a terminal. This extension is what mints the short-lived installation token that the `gh-agent-auth` skill uses to authenticate as the agent.
2. Add the `gh-agent-auth.sh` script to your `PATH`.
3. Tweak the `gh-agent-auth.sh` script as follows:
   1. Replace the `APP_ID` with the “App ID” value from your application.
   2. Replace the `PRIVATE_KEY_PATH` with the path to the `.pem` file with the private key.
4. Add the `gh-agent-auth` skill to your agent’s global skill directory.

### Adjusting Git credentials (optional)

1. In your agent's configuration, override the following environment variables:
   * `GIT_AUTHOR_NAME`: ex. "emlautarom1-agent[bot]".
   * `GIT_AUTHOR_EMAIL`: ex. "292495798+emlautarom1-agent[bot]@users.noreply.github.com".
   * `GIT_COMMITTER_NAME`: ex. "emlautarom1-agent[bot]".
   * `GIT_COMMITTER_EMAIL`: ex. "292495798+emlautarom1-agent[bot]@users.noreply.github.com".

In case you're wondering, the email is set up as `<ACCOUNT_ID>+<username>@users.noreply.github.com` where `ACCOUNT_ID` is the immutable internal user ID GitHub assigned to the bot account when it was created alongside the GitHub App. Note that the `<username>` portion includes the `[bot]` suffix (e.g. `emlautarom1-agent[bot]`). To fetch this ID you can use the following endpoint from the GitHub API:

```bash
curl -g "https://api.github.com/users/emlautarom1-agent[bot]" | jq .id
```
Replace `emlautarom1-agent[bot]` with your own GitHub App name while keeping the `[bot]` suffix.

### Assets

`gh-agent-auth.sh`

```shell
#!/usr/bin/env bash

# Authentication for the GitHub CLI intended to be used by coding agents.
# Mints a short-lived GitHub App installation token so the agent acts as a
# dedicated bot identity instead of the user's personal account.
#
# Usage:
#   gh-agent-auth [<owner> | <owner>/<repo>]
#
# With no argument the owner is inferred from the `origin` remote of the
# current git repository. The installation matching that owner is resolved
# automatically, so the same function works for personal repos and orgs alike.
#
# Examples:
#   GH_TOKEN=$(gh-agent-auth NethermindEth) gh issue list --repo NethermindEth/pluto
#   GH_TOKEN=$(gh-agent-auth) git push        # owner taken from `origin`

function gh-agent-auth() {
  # Implementation note: every external tool below **must** be invoked via `command` so
  # it runs the real binary, never a like-named shell function.

  # Identity of the GitHub App. The app id is stable; the installation id is
  # per-account (one for the user, one for each org) and resolved at runtime.
  local APP_ID=4019232
  local PRIVATE_KEY_PATH="$HOME/.secrets/gh-agent.private-key.pem"

  # Installation tokens are valid for 60 minutes; refresh a little early.
  local EXPIRATION_TIME_MINS=55

  if [[ ! -r "$PRIVATE_KEY_PATH" ]]; then
    echo "gh-agent-auth: private key not found or unreadable at $PRIVATE_KEY_PATH" >&2
    return 1
  fi

  # 1. Determine the repository owner whose installation we need.
  local owner="${1:-}"
  if [[ -z "$owner" ]]; then
    local remote_url
    if ! remote_url=$(command git remote get-url origin 2>/dev/null); then
      echo "gh-agent-auth: not in a git repository; pass an owner explicitly, e.g. 'gh-agent-auth NethermindEth'" >&2
      return 1
    fi
    # Strip the host prefix (git@host: or scheme://host/) and keep the first path segment.
    owner=$(command sed -E 's#^(git@[^:]+:|[a-z]+://[^/]+/)##; s#/.*##' <<<"$remote_url")
  fi
  # Accept "owner/repo" too and keep just the owner.
  owner="${owner%%/*}"

  if [[ -z "$owner" ]]; then
    echo "gh-agent-auth: could not determine the repository owner" >&2
    return 1
  fi

  # 2. Serve a cached token if it is still fresh (cache is per owner).
  #    A fixed /tmp location (not $TMPDIR) keeps the cache stable and shared
  #    across shells, so tokens are reused instead of re-minted on every call
  #    Because /tmp is world-writable, the directory is verified to be owned
  #    by us, mode 0700, and not a symlink before it is used.
  local cache_dir
  cache_dir="/tmp/gh-agent-$(command id -u)"
  if ! command mkdir -p "$cache_dir"; then
    echo "gh-agent-auth: cannot create cache directory $cache_dir" >&2
    return 1
  fi
  command chmod 700 "$cache_dir" 2>/dev/null
  if [[ -L "$cache_dir" || "$(command stat -c '%u %a' "$cache_dir" 2>/dev/null)" != "$(command id -u) 700" ]]; then
    echo "gh-agent-auth: refusing to use unsafe cache directory $cache_dir (wrong owner, mode, or symlink)" >&2
    return 1
  fi
  local token_path="$cache_dir/${owner}.token"
  # Serve the cached token only if it is younger than the refresh window.
  # The non-empty check additionally guards against a truncated or empty cache file.
  if [[ -n "$(command find "$token_path" -maxdepth 0 -type f -mmin -"$EXPIRATION_TIME_MINS" -print -quit 2>/dev/null)" ]]; then
    local cached
    cached=$(command cat "$token_path" 2>/dev/null)
    if [[ -n "$cached" ]]; then
      printf '%s\n' "$cached"
      return 0
    fi
  fi

  # 3. Resolve the installation id for this owner. The app authenticates here
  #    via the private key (a signed JWT), so this never touches GH_TOKEN
  #    or the user's stored credentials. Errors from `gh` flow to stderr so
  #    transient API/network failures stay visible (the extension is quiet on
  #    success).
  local installations installation_id
  if ! installations=$(command gh token installations --app-id "$APP_ID" --key "$PRIVATE_KEY_PATH"); then
    echo "gh-agent-auth: failed to list GitHub App installations (see error above)" >&2
    return 1
  fi
  installation_id=$(command jq -r --arg owner "$owner" \
    'map(select(.account.login | ascii_downcase == ($owner | ascii_downcase)))[0].id // empty' \
    <<<"$installations")

  if [[ -z "$installation_id" ]]; then
    echo "gh-agent-auth: the GitHub App (id $APP_ID) is not installed for owner '$owner'." >&2
    echo "gh-agent-auth: install it for that account/org and grant it the repo, then retry." >&2
    return 1
  fi

  # 4. Mint a fresh installation token (--token-only prints just the token).
  local token
  if ! token=$(command gh token generate --token-only \
      --app-id "$APP_ID" \
      --installation-id "$installation_id" \
      --key "$PRIVATE_KEY_PATH"); then
    echo "gh-agent-auth: failed to generate an installation token for '$owner' (see error above)" >&2
    return 1
  fi

  if [[ -z "$token" ]]; then
    echo "gh-agent-auth: received an empty token for '$owner'" >&2
    return 1
  fi

  # 5. Cache (owner-private) and print.
  #    Caching is done atomically through write then move.
  local tmp
  if tmp=$(command mktemp "$cache_dir/.${owner}.XXXXXX" 2>/dev/null); then
    if printf '%s' "$token" > "$tmp" 2>/dev/null; then
      command mv -f "$tmp" "$token_path" 2>/dev/null || command rm -f "$tmp"
    else
      command rm -f "$tmp"
    fi
  fi
  printf '%s\n' "$token"
}
```

`SKILL.md`

````markdown
---
name: gh-auth
description: >
  Authenticate every GitHub interaction through a short-lived GitHub App token from the `gh-agent-auth` Bash function — NEVER the user's personal credentials.
  Use BEFORE any `gh` CLI command, any GitHub API call, or any `git` push/pull over HTTPS to a GitHub remote. A bare `gh ...` is intentionally broken and will fail with an auth error.
  Always add the token prefix in the form of `GH_TOKEN=$(gh-agent-auth <owner>)` where `<owner>` is the GitHub user/org owning the repo you are acting on (e.g. `NethermindEth`).
---

# GitHub CLI Authentication

You must **never** use the user's personal GitHub credentials. All interaction
with GitHub (the `gh` CLI and `git` pushes/pulls over HTTPS) must go through a
short-lived GitHub App token produced by the `gh-agent-auth` Bash function.

## Getting a token

`gh-agent-auth` is a Bash function (defined in `~/.bashrc.d/gh-agent-auth.sh`).
If it is not available, stop and report this to the user instead of falling
back to any other credential.

It prints a token to stdout and resolves the right GitHub App installation
automatically from the repository **owner**:

- `gh-agent-auth <owner>` — token for that user/org (e.g. `gh-agent-auth NethermindEth`).
- `gh-agent-auth <owner>/<repo>` — same; the repo part is ignored.
- `gh-agent-auth` — infers the owner from the current repo's `origin` remote.

On failure it prints a message to stderr and exits non-zero (e.g. the App is
not installed for that owner). Surface such errors to the user; do not retry
with personal credentials.

## Usage

Prefix each GitHub command with `GH_TOKEN=$(gh-agent-auth <owner>)`. Pass the
owner explicitly so it matches the repo you are acting on:

```bash
# Issues / PRs / API
GH_TOKEN=$(gh-agent-auth NethermindEth) gh issue create \
  --repo NethermindEth/my-repo --title "Test issue" --body "Test issue body"

# git pushes over HTTPS also authenticate as the App via this token
GH_TOKEN=$(gh-agent-auth NethermindEth) git push origin my-branch
```

## Notes

- Tokens are cached per owner for ~55 minutes, so repeated calls are cheap and
  will not hit GitHub's rate limits — always use the prefix, don't try to reuse
  a captured token across commands yourself.
- A bare `gh ...` (without the prefix) is intentionally broken in this
  environment and will fail with an auth error. That is expected — add the
  prefix; never work around it.
- App installation tokens act on repositories, not on the user account, so
  endpoints like `gh api user` will return "Requires authentication". Use
  repo-scoped commands instead.
````
