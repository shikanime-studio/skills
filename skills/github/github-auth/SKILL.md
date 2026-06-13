---
name: github-auth
description: "GitHub auth setup: HTTPS tokens, SSH keys, gh CLI login."
version: 1.1.0
author: Operator 21O
license: MIT
platforms: [linux, macos, windows]
metadata:
  hermes:
    tags: [GitHub, Authentication, Git, gh-cli, SSH, Setup]
    related_skills:
      [
        github-pr-workflow,
        github-code-review,
        github-issues,
        github-repo-management,
      ]
---

# GitHub Authentication Setup

This skill sets up authentication so the agent can work with GitHub
repositories, PRs, issues, and CI. It covers two paths:

- **`git` (always available)** — uses HTTPS personal access tokens or SSH keys
- **`gh` CLI (if installed)** — richer GitHub API access with a simpler auth
  flow

## Detection Flow

When a user asks you to work with GitHub, run this check first:

```bash
# Check what's available
git --version
gh --version 2>/dev/null || echo "gh not installed"

# Check if already authenticated
gh auth status 2>/dev/null || echo "gh not authenticated"
git config --global credential.helper 2>/dev/null || echo "no git credential
  helper"
```

**Decision tree:**

1. If `gh auth status` shows authenticated → you're good, use `gh` for
   everything
2. If `gh` is installed but not authenticated → use "gh auth" method below
3. If `gh` is not installed → use "git-only" method below (no sudo needed)

---

## Method 1: Git-Only Authentication (No gh, No sudo)

This works on any machine with `git` installed. No root access needed.

### Option A: HTTPS with Personal Access Token (Recommended)

This is the most portable method — works everywhere, no SSH config needed.

## Step 1: Create a personal access token

Tell the user to go to: **<https://github.com/settings/tokens**>

- Click "Generate new token (classic)"
- Give it a name like "hermes-agent"
- Select scopes:
  - `repo` (full repository access — read, write, push, PRs)
  - `workflow` (trigger and manage GitHub Actions)
  - `read:org` (if working with organization repos)
- Set expiration (90 days is a good default)
- Copy the token — it won't be shown again

## Step 2: Configure git to store the token

```bash
# Set up the credential helper to cache credentials
# "store" saves to ~/.git-credentials in plaintext (simple, persistent)
git config --global credential.helper store

# Now do a test operation that triggers auth — git will prompt for credentials
# Username: <their-github-username>
# Password: <paste the personal access token, NOT their GitHub password>
git ls-remote https://github.com/<their-username>/<any-repo>.git
```

After entering credentials once, they're saved and reused for all future
operations.

## Alternative: cache helper (credentials expire from memory)

```bash
# Cache in memory for 8 hours (28800 seconds) instead of saving to disk
git config --global credential.helper 'cache --timeout=28800'
```

## Alternative: set the token directly in the remote URL (per-repo)

```bash
# Embed token in the remote URL (avoids credential prompts entirely)
git remote set-url origin
  https://<username>:<token>@github.com/<owner>/<repo>.git
```

## Step 3: Configure git identity

```bash
# Required for commits — set name and email
git config --global user.name "Their Name"
git config --global user.email "their-email@example.com"
```

## Step 4: Verify

```bash
# Test push access (this should work without any prompts now)
git ls-remote https://github.com/<their-username>/<any-repo>.git

# Verify identity
git config --global user.name
git config --global user.email
```

### Option B: SSH Key Authentication

Good for users who prefer SSH or already have keys set up.

## Step 1: Check for existing SSH keys

```bash
ls -la ~/.ssh/id_*.pub 2>/dev/null || echo "No SSH keys found"
```

## Step 2: Generate a key if needed

```bash
# Generate an ed25519 key (modern, secure, fast)
ssh-keygen -t ed25519 -C "their-email@example.com" -f ~/.ssh/id_ed25519 -N ""

# Display the public key for them to add to GitHub
cat ~/.ssh/id_ed25519.pub
```

Tell the user to add the public key at: **<https://github.com/settings/keys>**

- Click "New SSH key"
- Paste the public key content
- Give it a title like "hermes-agent-machine-name"

## Step 3: Test the connection

```bash
ssh -T git@github.com
# Expected: "Hi <username>! You've successfully authenticated..."
```

## Step 4: Configure git to use SSH for GitHub

```bash
# Rewrite HTTPS GitHub URLs to SSH automatically
git config --global url."git@github.com:".insteadOf "https://github.com/"
```

## Step 5: Configure git identity

```bash
git config --global user.name "Their Name"
git config --global user.email "their-email@example.com"
```

---

## Method 2: gh CLI Authentication

If `gh` is installed, it handles both API access and git credentials in one
step.

### Interactive Browser Login (Desktop)

```bash
gh auth login
# Select: GitHub.com
# Select: HTTPS
# Authenticate via browser
```

### Token-Based Login (Headless / SSH Servers)

```bash
echo "<THEIR_TOKEN>" | gh auth login --with-token

# Set up git credentials through gh
gh auth setup-git
```

### Verify

```bash
gh auth status
```

---

## Using the GitHub API Without gh

When `gh` is not available, you can still access the full GitHub API using
`curl` with a personal access token. This is how the other GitHub skills
implement their fallbacks.

### Setting the Token for API Calls

```bash
# Option 1: Export as env var (preferred — keeps it out of commands)
export GITHUB_TOKEN="<token>"

# Then use in curl calls:
curl -s -H "Authorization: token $GITHUB_TOKEN" \
  https://api.github.com/user
```

### Extracting the Token from Git Credentials

If git credentials are already configured (via credential.helper store), the
token can be extracted:

```bash
# Read from git credential store
grep "github.com" ~/.git-credentials 2>/dev/null | head -1 | sed
  's|https://[^:]*:\([^@]*\)@.*|\1|'
```

### Helper: Detect Auth Method

Use this pattern at the start of any GitHub workflow:

```bash
# Try gh first, fall back to git + curl
if command -v gh &>/dev/null && gh auth status &>/dev/null; then
  echo "AUTH_METHOD=gh"
elif [ -n "$GITHUB_TOKEN" ]; then
  echo "AUTH_METHOD=curl"
elif [ -f ~/.hermes/.env ] && grep -q "^GITHUB_TOKEN=" ~/.hermes/.env; then
  export GITHUB_TOKEN=$(
    grep "^GITHUB_TOKEN=" ~/.hermes/.env |
      head -1 |
      cut -d= -f2 |
      tr -d '\n\r'
  )
  echo "AUTH_METHOD=curl"
elif grep -q "github.com" ~/.git-credentials 2>/dev/null; then
  export GITHUB_TOKEN=$(grep "github.com" ~/.git-credentials | head -1 | sed
    's|https://[^:]*:\([^@]*\)@.*|\1|')
  echo "AUTH_METHOD=curl"
else
  echo "AUTH_METHOD=none"
  echo "Need to set up authentication first"
fi
```

---

## SSH Commit Signing

GitHub can enforce **verified signatures** on protected branches. GPG keys must
be uploaded to GitHub to pass this check. An alternative is **SSH commit
signing**, which reuses an SSH key already registered on the user's GitHub
account.

### When to Use SSH Signing

- GPG signatures are rejected by GitHub (`GH013: Repository rule violations`)
- The user already has SSH keys on GitHub and wants a simpler setup
- The user prefers not to manage separate GPG keys

### Step 1: Add an SSH Signing Key to GitHub

```bash
# Show available public keys
ls ~/.ssh/*.pub
cat ~/.ssh/id_ed25519.pub
```

Tell the user to go to **<https://github.com/settings/keys>**:

- Click "New SSH key"
- Set **Key type** to "Signing Key" (not "Authentication Key")
- Paste the public key content
- Give it a title like "commit-signing-machine-name"

### Step 2: Configure jj for SSH Signing (Per-Repo)

```bash
# Set SSH as the signing backend for this repo
jj config set --repo signing.backend ssh

# Point to the public key (jj derives the private key path)
jj config set --repo signing.key ~/.ssh/id_ed25519.pub
```

### Step 3: Re-Sign Existing Commits

Existing commits signed with GPG (or unsigned) must be re-signed. The simplest
approach is to rebase:

```bash
# Rebase all local commits onto the last signed/accepted ancestor
# This re-creates commits with the new signing backend
jj rebase -d <last-accepted-commit>
```

Or for a single bookmark:

```bash
jj rebase -d main -r 'main..@'
```

### Step 4: Verify

```bash
# Check that new commits are signed
git log --format='%H %G? %s' -3
# Should show 'G' for good signature after the next commit
```

### Step 5: Push

```bash
jj git push -b main
```

### Notes

- `gh` CLI may need `admin:ssh_signing_key` scope to manage signing keys via
  API: `gh auth refresh -h github.com -s admin:ssh_signing_key`
- SSH signing and SSH authentication are **separate keys** on GitHub — a key
  used for `git push` auth is not automatically available for signing
- The `signing.key` value is the **public key** path; jj finds the private key
  by replacing `.pub` with the private key path

## Troubleshooting

| Problem                                                       | Solution                                                                                                                      |
| ------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------- |
| `git push` asks for password                                  | GitHub disabled password auth. Use a personal access token as the password, or switch to SSH                                  |
| `remote: Permission to X denied`                              | Token may lack `repo` scope — regenerate with correct scopes                                                                  |
| `fatal: Authentication failed`                                | Cached credentials may be stale — run `git credential reject` then re-authenticate                                            |
| `ssh: connect to host github.com port 22: Connection refused` | Try SSH over HTTPS port: add `Host github.com` with `Port 443` and `Hostname ssh.github.com` to `~/.ssh/config`               |
| Credentials not persisting                                    | Check `git config --global credential.helper` — must be `store` or `cache`                                                    |
| Multiple GitHub accounts                                      | Use SSH with different keys per host alias in `~/.ssh/config`, or per-repo credential URLs                                    |
| `gh: command not found` + no sudo                             | Use git-only Method 1 above — no installation needed                                                                          |
| `GH013: Commits must have verified signatures`                | GPG key not on GitHub, or commits not signed. Switch to SSH signing (see above) or upload GPG key at github.com/settings/keys |
