---
name: gpg-key-rotation
description:
  "Generate a new GPG signing key, configure git/jj/sl, export public key, and
  upload to GitHub via gh. Designed for annual rotation via cron."
version: 1.0.0
author: Operator 17O
license: MIT
platforms: [linux, macos, windows]
metadata:
  hermes:
    tags: [GPG, Git, GitHub, Key-Rotation, Security, jj, sapling]
    related_skills: [github-auth]
---

# GPG Key Rotation

Generate a new GPG signing key, configure all VCS tools to use it, export the
public key, and upload it to GitHub.

## When to Run

- Annually (via cron — see `references/cron-setup.md`)
- When the current signing key is expiring or compromised
- When setting up a new machine

## Prerequisites

- `gpg` (GnuPG 2.4+) installed and agent running
- `gh` CLI authenticated (`gh auth status`)
- `git` configured with user name/email
- `jj` (Jujutsu) installed (optional — skip if not present)
- `sl` (Sapling SCM) installed (optional — skip if not present)

## Steps

### 1. Check Current Key Status

```bash
# List current signing keys
gpg --list-secret-keys --keyid-format long

# Check git signing config
git config --global user.signingkey
git config --global commit.gpgsign

# Check jj signing config
jj config list | grep -i 'signing\|gpg\|key'

# Check gh auth
gh auth status
```

### 2. Generate New GPG Key

Generate a clean `ed25519` signing-only key with no subkeys:

```bash
# Set key parameters
REAL_NAME="William Phetsinorath"
EMAIL="william.phetsinorath@shikanime.studio"
COMMENT="GitHub"
EXPIRY="2y"

# Generate (batch mode, no passphrase for headless use)
echo "" | gpg --batch --pinentry-mode loopback --passphrase "" \
  --quick-gen-key "${REAL_NAME} (${COMMENT}) <${EMAIL}>" \
  ed25519 sign "${EXPIRY}"

# Capture the new key fingerprint
NEW_KEY=$(gpg --list-secret-keys --keyid-format long "${EMAIL}" \
  | grep '^sec' | tail -1 | awk '{print $2}' | cut -d'/' -f2)
echo "New key: ${NEW_KEY}"
```

**Windows note:** The batch file method (`Key-Type: ed25519` in a batch file)
fails on GPG 2.4 with "invalid algorithm". Always use `--quick-gen-key` with the
loopback pinentry flags shown above. If `--quick-gen-key` fails with "no tty",
the `--batch --pinentry-mode loopback --passphrase ""` flags resolve it.

### 3. Configure Git

**CRITICAL:** The email on the GPG key MUST match the git committer email.
GitHub will show "The email in this signature doesn't match the committer email"
if they differ. Set both the signing key AND the user email:

```bash
git config --global user.signingkey "${NEW_KEY}"
git config --global commit.gpgsign true
git config --global user.email "${EMAIL}"
git config --global user.name "${REAL_NAME}"
```

Also ensure `${EMAIL}` is a **verified email** on your GitHub account:

```bash
gh auth refresh -h github.com -s user   # may be needed for scope
gh api user/emails --jq '.[].email'     # verify the email appears
```

### 4. Configure Jujutsu (jj)

```bash
jj config set --user signing.backend gpg
jj config set --user signing.key "${NEW_KEY}"
jj config set --user user.signingkey "${NEW_KEY}"
```

### 5. Configure Sapling (sl)

Sapling uses git hooks under the hood. Ensure the git config from step 3 is set.
If sapling has its own username config, update it:

```bash
sl config ui.username "William Phetsinorath
  <william.phetsinorath@shikanime.studio>"
```

### 6. Export Public Key

```bash
gpg --armor --export "${NEW_KEY}"
```

Save the output — this is what gets uploaded to GitHub.

### 7. Upload to GitHub

```bash
# Export to temp file
gpg --armor --export "${NEW_KEY}" > /tmp/gpg-new-key.asc

# Upload via gh CLI
gh gpg-key add /tmp/gpg-new-key.asc --title "GPG signing key ($(date
  +%Y-%m-%d))"

# Verify
gh gpg-key list
```

**Note:** If `gh gpg-key add` returns "the key already exists in your account",
the key is already uploaded — this is informational, not an error. Proceed to
verify with `gh gpg-key list`.

**Windows note:** Use a path under the workspace for the temp file:

```powershell
gpg --armor --export "${NEW_KEY}" >
  "$env:USERPROFILE\.hermes\tmp\gpg-new-key.asc"
gh gpg-key add "$env:USERPROFILE\.hermes\tmp\gpg-new-key.asc" --title "GPG
  signing key ($(Get-Date -Format 'yyyy-MM-dd'))"
```

### 8. Cleanup Old Keys (Optional)

After confirming the new key works and is verified on GitHub:

```bash
# List all keys
gpg --list-secret-keys --keyid-format long

# Delete old key (replace FINGERPRINT)
gpg --delete-secret-keys FINGERPRINT
gpg --delete-keys FINGERPRINT
```

### 9. Verification

```bash
# Verify git signing works
git commit --allow-empty --gpg-sign -m "test: verify new GPG key"
git log -1 --show-signature

# Verify jj signing works
jj git push --dry-run  # or create a test commit
```

## Rewriting Existing Commit History to Use a New Key

When you need to re-sign existing commits with a new GPG key (e.g., after key
rotation or email correction), use the `git read-tree` + `git commit` replay
pattern. This preserves the exact file tree and date while changing the author,
message, and signature.

**Do NOT use `git commit-tree`** — it does not honor `commit.gpgsign` and will
produce unsigned commits.

### Pattern

```bash
# 1. Create an orphan branch
git checkout --orphan re-sign
git reset --hard

# 2. For each commit (oldest to newest), replay with new author/key:
#    a. Read the original commit's tree
git read-tree <original-commit-hash>^{tree}

#    b. Commit with new author, message, date, and signature
git commit --gpg-sign \
  --file=/path/to/new-message.txt \
  --date="YYYY-MM-DDTHH:MM:SS+ZZZZ" \
  --author="Name <email>"

# 3. Replace main and force-push
git checkout main
git reset --hard re-sign
git branch -D re-sign
git push --force-with-lease origin main
```

**Windows note:** Use paths under the workspace for message files (e.g.,
`$env:USERPROFILE\.hermes\tmp\msg1`).

**Important:** After force-pushing, GitHub may still show "Unverified" until:

- The GPG public key is uploaded (`gh gpg-key add`)
- The email is verified on GitHub (`gh api user/emails`)

## Cron Setup

To run annually, create a cron job:

```text
Schedule: 0 9 1 6 *   (June 1st at 9am, adjust as needed)
Prompt: Run the gpg-key-rotation skill. Generate a new GPG signing key,
  configure git/jj/sl, export the public key, upload it to GitHub via gh gpg-key
  add, and report the new key fingerprint.
Skills: [gpg-key-rotation]
```

## Troubleshooting

- **`git commit-tree` does NOT honor `commit.gpgsign`**: When replaying commits
  with `git read-tree` + `git commit-tree`, the config setting is ignored. Use
  `git read-tree` + `git commit --gpg-sign` instead to ensure signatures are
  applied.
- **"gpg: cannot open 'no tty'"**: Use
  `--batch --pinentry-mode loopback --passphrase ""` flags
- **"A key already exists"**: Use a unique comment like `(GitHub-2027)` to
  differentiate
- **Windows Defender blocks gpg**: Add gpg/gpg-agent exclusions in Windows
  Security
- **gh gpg-key add fails**: Ensure `gh auth status` shows you're logged in; the
  API requires `write:public_key` scope
