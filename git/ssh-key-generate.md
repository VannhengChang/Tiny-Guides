# SSH Key — GitHub & GitLab (Single & Multiple Accounts)

> ~10 min read

Use SSH keys to clone, push, and pull without typing your password every time. This guide covers a **single account** on GitHub and GitLab, then **multiple accounts** on the same machine (e.g. personal + work).

---

## Prerequisites

- Git installed (`git --version`)
- Terminal access
- A GitHub and/or GitLab account

All paths below assume macOS. On Linux, swap `pbcopy` for `cat` and copy the output manually; skip `--apple-use-keychain` when adding keys to the agent.

---

## Part 1 — Single account (GitHub or GitLab)

### 1. Generate an SSH key

Use **Ed25519** (recommended).

```bash
ssh-keygen -t ed25519 -C "your_email@example.com" -f ~/.ssh/id_ed25519
```

#### What `-C "your_email@example.com"` means

`-C` adds a **comment** — a short label stored inside the public key so you can tell keys apart later.

- Replace `your_email@example.com` with something **you** will recognize (often the email on that GitHub/GitLab account).
- It is **not** used to log you in. GitHub and GitLab authenticate the key itself, not this string.
- It does **not** have to match your account email exactly, but using the account email is a common convention.
- You will see this comment in:
  - The last part of `~/.ssh/id_ed25519.pub`
  - GitHub/GitLab SSH key lists (helps when you have several keys)

**Examples:**

```bash
-C "personal@gmail.com"          # personal GitHub
-C "you@company.com"             # work GitHub
-C "gitlab-personal"             # any label works if it helps you identify the key
```

#### What `-f ~/.ssh/id_ed25519` means

`-f` sets the **file path** (and base name) for the key pair. SSH writes two files:

| File | Purpose |
|------|---------|
| `~/.ssh/id_ed25519` | Private key — keep on your machine only |
| `~/.ssh/id_ed25519.pub` | Public key — paste into GitHub/GitLab |

Breaking down the path:

- `~` — your home directory (e.g. `/Users/yourname` on macOS)
- `.ssh/` — hidden folder SSH expects for keys and config
- `id_ed25519` — the key name; `.pub` is added automatically for the public key

If you **omit** `-f`, `ssh-keygen` defaults to `~/.ssh/id_ed25519` (or prompts if that file already exists). Using `-f` explicitly avoids surprises and lets you pick a **unique name per account** — required for multiple GitHub/GitLab accounts on one machine:

```bash
-f ~/.ssh/id_ed25519_github_personal
-f ~/.ssh/id_ed25519_github_work
-f ~/.ssh/id_ed25519_gitlab_personal
```

Every `-f` path you use later must match the `IdentityFile` in `~/.ssh/config` (covered in Part 2).

#### Finish key generation

- Press **Enter** for no passphrase, or set one for extra security.
- This creates the two files listed above.

### 2. Start the SSH agent and add the key

**macOS:**

```bash
eval "$(ssh-agent -s)"
ssh-add --apple-use-keychain ~/.ssh/id_ed25519
```

**Linux:**

```bash
eval "$(ssh-agent -s)"
ssh-add ~/.ssh/id_ed25519
```

### 3. Copy the public key

```bash
pbcopy < ~/.ssh/id_ed25519.pub   # macOS
# cat ~/.ssh/id_ed25519.pub      # Linux — copy the output
```

### 4. Add the key to your account

| Platform | Where to paste |
|----------|----------------|
| GitHub | [Settings → SSH and GPG keys → New SSH key](https://github.com/settings/keys) |
| GitLab | [Preferences → SSH Keys](https://gitlab.com/-/user_settings/ssh_keys) |

Give the key a title (e.g. `MacBook Pro`), paste the public key, and save.

### 5. Test the connection

```bash
ssh -T git@github.com
ssh -T git@gitlab.com
```

**Expected output (GitHub):**

```text
Hi username! You've successfully authenticated, but GitHub does not provide shell access.
```

**Expected output (GitLab):**

```text
Welcome to GitLab, @username!
```

This is **not an error** — it means SSH works and the key is linked to that account.

### 6. Clone with SSH

```bash
git clone git@github.com:username/repo.git
git clone git@gitlab.com:username/repo.git
```

---

## Part 2 — Multiple accounts on one machine

### The problem

Both GitHub and GitLab use the same host for SSH:

- `git@github.com`
- `git@gitlab.com`

If you add keys for a personal account and a work account, SSH may always pick the **wrong** key (usually the first one in `~/.ssh/` or the agent). You need **one key per account** and an **SSH config** that tells the agent which key to use.

### Naming convention

Use descriptive filenames so you know which key belongs to which account:

| Account | Suggested private key path |
|---------|---------------------------|
| GitHub personal | `~/.ssh/id_ed25519_github_personal` |
| GitHub work | `~/.ssh/id_ed25519_github_work` |
| GitLab personal | `~/.ssh/id_ed25519_gitlab_personal` |
| GitLab work | `~/.ssh/id_ed25519_gitlab_work` |

You can reuse one key on both GitHub and GitLab if you add the same `.pub` to both accounts — but separate keys per account give clearer control when using multiple accounts on the same platform.

---

### Step 1 — Generate a key for each account

Example: second GitHub account (work):

```bash
ssh-keygen -t ed25519 -C "work@company.com" -f ~/.ssh/id_ed25519_github_work
```

Example: GitLab account:

```bash
ssh-keygen -t ed25519 -C "you@example.com" -f ~/.ssh/id_ed25519_gitlab_personal
```

Repeat for every account that needs its own identity.

### Step 2 — Add all keys to the SSH agent

**macOS:**

```bash
eval "$(ssh-agent -s)"
ssh-add --apple-use-keychain ~/.ssh/id_ed25519_github_personal
ssh-add --apple-use-keychain ~/.ssh/id_ed25519_github_work
ssh-add --apple-use-keychain ~/.ssh/id_ed25519_gitlab_personal
```

**Linux:**

```bash
eval "$(ssh-agent -s)"
ssh-add ~/.ssh/id_ed25519_github_personal
ssh-add ~/.ssh/id_ed25519_github_work
ssh-add ~/.ssh/id_ed25519_gitlab_personal
```

Optional — persist macOS keychain loading. Add to `~/.ssh/config` (see next step):

```ssh
Host *
    AddKeysToAgent yes
    UseKeychain yes
    IdentityFile ~/.ssh/id_ed25519_github_personal
```

Only list `IdentityFile` entries here for keys you want loaded by default; account-specific hosts below override per connection.

### Step 3 — Add each public key to the right account

Copy each `.pub` file and paste it into the matching GitHub or GitLab account (while logged into **that** account):

```bash
pbcopy < ~/.ssh/id_ed25519_github_personal.pub
pbcopy < ~/.ssh/id_ed25519_github_work.pub
pbcopy < ~/.ssh/id_ed25519_gitlab_personal.pub
```

| Platform | URL |
|----------|-----|
| GitHub | https://github.com/settings/keys |
| GitLab | https://gitlab.com/-/user_settings/ssh_keys |

---

### Step 4 — Configure `~/.ssh/config`

Create or edit the file:

```bash
nano ~/.ssh/config
```

Use **Host aliases** — short names you use instead of `github.com` or `gitlab.com` in clone URLs. Each alias maps to one identity file.

**Example: two GitHub accounts + one GitLab account**

```ssh
# Default macOS agent behavior (optional)
Host *
    AddKeysToAgent yes
    UseKeychain yes

# GitHub — personal
Host github-personal
    HostName github.com
    User git
    IdentityFile ~/.ssh/id_ed25519_github_personal
    IdentitiesOnly yes

# GitHub — work
Host github-work
    HostName github.com
    User git
    IdentityFile ~/.ssh/id_ed25519_github_work
    IdentitiesOnly yes

# GitLab — personal
Host gitlab-personal
    HostName gitlab.com
    User git
    IdentityFile ~/.ssh/id_ed25519_gitlab_personal
    IdentitiesOnly yes
```

Save in nano: **Ctrl+O**, **Enter**, **Ctrl+X**.

Set restrictive permissions (SSH ignores the config if it is too open):

```bash
chmod 600 ~/.ssh/config
```

**What `IdentitiesOnly yes` does:** SSH only offers the key listed in `IdentityFile` for that host — it won't try every key in the agent and confuse GitHub/GitLab.

---

### Step 5 — Test each account

Use the **Host alias**, not `github.com` / `gitlab.com`:

```bash
ssh -T git@github-personal
ssh -T git@github-work
ssh -T git@gitlab-personal
```

Expected:

```text
Hi personal-username! You've successfully authenticated...
Hi work-username! You've successfully authenticated...
Welcome to GitLab, @gitlab-username!
```

If you see the wrong username, check that the public key was added to the correct account and that `IdentityFile` points to the right private key.

---

### Step 6 — Clone using Host aliases

Replace `github.com` / `gitlab.com` with your alias in the remote URL.

**GitHub personal:**

```bash
git clone git@github-personal:personal-username/repo.git
```

**GitHub work:**

```bash
git clone git@github-work:work-org/repo.git
```

**GitLab:**

```bash
git clone git@gitlab-personal:username/repo.git
```

The part after `:` is still `owner/repo.git` — only the host alias changes.

---

### Step 7 — Fix an existing repo’s remote

If you already cloned with `git@github.com:...` and pushes use the wrong account:

```bash
cd /path/to/repo
git remote -v
git remote set-url origin git@github-work:work-org/repo.git
git remote -v
```

Same pattern for GitLab: `git@gitlab-personal:username/repo.git`.

---

### Step 8 — Set Git user name/email per repo (optional)

SSH picks the **account**; Git still uses **global** `user.name` and `user.email` unless you override per repo:

```bash
cd /path/to/work-repo
git config user.name "Work Name"
git config user.email "work@company.com"
```

For many work repos under one folder, use conditional includes in `~/.gitconfig`:

```gitconfig
[includeIf "gitdir:~/Work/"]
    path = ~/.gitconfig-work
```

In `~/.gitconfig-work`:

```gitconfig
[user]
    name = Work Name
    email = work@company.com
```

---

## Quick reference

| Task | Command |
|------|---------|
| Generate key | `ssh-keygen -t ed25519 -C "email" -f ~/.ssh/key_name` |
| Copy public key (macOS) | `pbcopy < ~/.ssh/key_name.pub` |
| Test alias | `ssh -T git@github-personal` |
| Clone | `git clone git@github-personal:user/repo.git` |
| Change remote | `git remote set-url origin git@github-work:org/repo.git` |

---

## Troubleshooting

### `Permission denied (publickey)`

1. Key not added to agent: `ssh-add -l` (list loaded keys), then `ssh-add ~/.ssh/your_key`.
2. Wrong key offered: add `IdentitiesOnly yes` under the Host block in `~/.ssh/config`.
3. Public key not on the account: re-copy `.pub` and add it while logged into the correct account.
4. Test with verbose output: `ssh -vT git@github-personal` and read which keys are tried.

### Authenticates as the wrong GitHub user

- You are probably testing `git@github.com` instead of your alias — use `git@github-personal` or `git@github-work`.
- Update clone URLs and `git remote set-url` to use the alias.

### `Bad permissions` on `~/.ssh/config`

```bash
chmod 700 ~/.ssh
chmod 600 ~/.ssh/config
chmod 600 ~/.ssh/id_ed25519_*
```

### Agent forgets keys after reboot (macOS)

Use `ssh-add --apple-use-keychain` and `UseKeychain yes` in `~/.ssh/config`.

### Self-hosted GitLab

Use the same pattern with your instance hostname:

```ssh
Host gitlab-work
    HostName gitlab.company.com
    User git
    IdentityFile ~/.ssh/id_ed25519_gitlab_work
    IdentitiesOnly yes
```

Clone: `git clone git@gitlab-work:group/project.git`

---

## Security notes

- Never share or commit **private** keys (`*.pub` is safe to share; files without `.pub` are not).
- Use a passphrase on keys for laptops shared or easily lost.
- Remove old keys from GitHub/GitLab when you retire a machine: account SSH key settings → delete unused keys.

---

**See also:** [GitHub SSH docs](https://docs.github.com/en/authentication/connecting-to-github-with-ssh) · [GitLab SSH docs](https://docs.gitlab.com/ee/user/ssh.html) · [GitHub: multiple accounts](https://docs.github.com/en/authentication/connecting-to-github-with-ssh/managing-deploy-keys)
