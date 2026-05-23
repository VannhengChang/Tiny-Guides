# SSH Key — Quick Reference

> Copy-paste commands only. macOS. For explanations, see the [full guide](./ssh-key-generate.md).

---

## Single account (GitHub or GitLab)

**1. Generate an Ed25519 key**

Replace the email and key path if needed. Press Enter for no passphrase, or set one.

```bash
ssh-keygen -t ed25519 -C "your_email@example.com" -f ~/.ssh/id_ed25519
```

**2. Optional — remember passphrase in macOS Keychain**

Skip if you chose no passphrase at step 1.

```bash
eval "$(ssh-agent -s)"
ssh-add --apple-use-keychain ~/.ssh/id_ed25519
```

**3. Copy the public key**

```bash
pbcopy < ~/.ssh/id_ed25519.pub
```

**4. Add the key to your account**

- GitHub: [Settings → SSH and GPG keys](https://github.com/settings/keys)
- GitLab: [Preferences → SSH Keys](https://gitlab.com/-/user_settings/ssh_keys)

Paste and save.

**5. Test the connection**

```bash
ssh -T git@github.com
```

```bash
ssh -T git@gitlab.com
```

**6. Clone with SSH**

```bash
git clone git@github.com:username/repo.git
```

```bash
git clone git@gitlab.com:username/repo.git
```

---

## Multiple accounts (same machine)

Use a **unique key file** and **Host alias** per account.

**1. Generate a key per account**

```bash
ssh-keygen -t ed25519 -C "personal@gmail.com" -f ~/.ssh/id_ed25519_github_personal
```

```bash
ssh-keygen -t ed25519 -C "work@company.com" -f ~/.ssh/id_ed25519_github_work
```

```bash
ssh-keygen -t ed25519 -C "you@example.com" -f ~/.ssh/id_ed25519_gitlab_personal
```

**2. Copy each public key (paste into the matching account)**

```bash
pbcopy < ~/.ssh/id_ed25519_github_personal.pub
```

```bash
pbcopy < ~/.ssh/id_ed25519_github_work.pub
```

```bash
pbcopy < ~/.ssh/id_ed25519_gitlab_personal.pub
```

- GitHub: https://github.com/settings/keys
- GitLab: https://gitlab.com/-/user_settings/ssh_keys

**3. Create `~/.ssh/config`**

```bash
nano ~/.ssh/config
```

```ssh
Host *
    AddKeysToAgent yes
    UseKeychain yes

Host github-personal
    HostName github.com
    User git
    IdentityFile ~/.ssh/id_ed25519_github_personal
    IdentitiesOnly yes

Host github-work
    HostName github.com
    User git
    IdentityFile ~/.ssh/id_ed25519_github_work
    IdentitiesOnly yes

Host gitlab-personal
    HostName gitlab.com
    User git
    IdentityFile ~/.ssh/id_ed25519_gitlab_personal
    IdentitiesOnly yes
```

**4. Fix permissions**

```bash
chmod 600 ~/.ssh/config
```

**5. Test each alias**

```bash
ssh -T git@github-personal
```

```bash
ssh -T git@github-work
```

```bash
ssh -T git@gitlab-personal
```

**6. Clone using the alias (not `github.com`)**

```bash
git clone git@github-personal:personal-username/repo.git
```

```bash
git clone git@github-work:work-org/repo.git
```

```bash
git clone git@gitlab-personal:username/repo.git
```

**7. Fix an existing repo remote**

```bash
git remote set-url origin git@github-work:work-org/repo.git
```

---

## Troubleshooting

**Permission denied (publickey)**

```bash
ssh-add -l
```

```bash
ssh -vT git@github-personal
```

**Bad permissions**

```bash
chmod 700 ~/.ssh
chmod 600 ~/.ssh/config
chmod 600 ~/.ssh/id_ed25519_*
```
