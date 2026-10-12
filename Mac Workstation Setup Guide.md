# Mac Workstation Setup Guide

Complete setup procedure for a macOS (Apple Silicon) workstation running `network-mt-eos`. Covers machine prerequisites, GitHub SSH/SSO access, environment build, credential files, first dry run, and GitKraken integration.

  **Before you start:** each new setup requires a GitHub account that is a member of the **McKesson organization** with access to the `network-mt-eos` repository. Request access through McKesson IDM at https://idm.mckesson.com/ui/d/my-requests/access-requests - the grant to request is **GitHubNAMCK-AD: DSSALM_GITHUB_MCKESSON** (Access Profile: "Access to the MCKESSON GitHub Org"). A completed grant looks like this:

  ![Completed McKesson GitHub org access grant in IDM](images/github-org-access-grant.png)

  The SSH/SSO steps in Phase 2 cannot be completed until this grant shows Completed.

---

## Phase 1: Machine Prerequisites

### 1. Homebrew

Skip if already installed (check with `brew --version`):

```
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"
```

### 2. Tools and libraries

```
brew install pyenv pipenv xz
```

  **Important:** `xz` must be installed **before** Python is built. If it is missing, pyenv builds Python without lzma support - the install still reports success, but later package operations fail with `ModuleNotFoundError: No module named '_lzma'`.

### 3. Python 3.13.3 via pyenv

```
pyenv install 3.13.3
```

### 4. Verify lzma compiled in

```
~/.pyenv/versions/3.13.3/bin/python -c "import lzma; print('lzma OK')"
```

Expected output: `lzma OK`

If it errors instead:

```
brew install xz
pyenv uninstall 3.13.3
pyenv install 3.13.3
```

Then re-run the verification.

---

## Phase 2: GitHub SSH Key + SSO

HTTPS cloning does **not** work against the org's SAML SSO. SSH is required.

### 1. Generate an SSH key

```
ssh-keygen -t ed25519 -C "your.name@mckesson.com"
```

Accept defaults; passphrase optional. Then display the public key:

```
cat ~/.ssh/id_ed25519.pub
```

### 2. Add the key to GitHub

In a browser: **Settings → SSH and GPG keys → New SSH key**

| Field | Value |
|---|---|
| Title | e.g. `McKesson MacBook (<hostname>)` |
| Key type | **Authentication Key** (the default) |
| Key | The entire one-line output of the `cat` command above |

  Paste only the `.pub` file contents. The file without `.pub` is the private key - never share it.

### 3. Authorize the key for SSO (required)

Next to the new key, open the **Configure SSO** dropdown → find **mckesson** → click **Authorize** → complete the SAML sign-in.

If `mckesson` does not appear in the dropdown, your GitHub account is not yet a member of the org - submit the IDM access request described at the top of this guide before continuing.

### 4. Test

```
ssh -T git@github.com
```

Expected: `Hi <username>! You've successfully authenticated, but GitHub does not provide shell access.` - this message is a **pass**, not an error.

---

## Phase 3: Clone the Repo

### 1. Clone via SSH

```
cd ~/repo        # or your preferred parent directory
git clone git@github.com:McKesson/network-mt-eos.git
cd network-mt-eos
```

**If you see** `The 'mckesson' organization has enabled or enforced SAML SSO` - the key authenticated but was not SSO-authorized. Return to Phase 2, step 3.

---

## Phase 4: Build the Python / Ansible Environment

  **Note:** Ansible is **not** installed separately. It is installed by `pipenv sync` in step 3 below, pinned to the version in `Pipfile.lock`. Do not `brew install ansible` or `pip install ansible` globally.

### 1. Activate the project environment

From the repo root:

```
pipenv shell
```

Creates the virtualenv on first run (using the pyenv 3.13.3). The prompt changes to show `(network-mt-eos)`.

### 2. Compile ansible-pylibssh

Mac-specific step - must run **after** `pipenv shell`, **before** `pipenv sync`:

```
./macosx_pylibssh.sh
```

Installs `libssh` via Homebrew, upgrades pip, compiles `ansible-pylibssh` against the correct headers, and configures the repo's git hooks.

### 3. Install all dependencies

```
pipenv sync
```

Takes several minutes. Success = `All dependencies are now up-to-date!`

**Known failure:** `fatal error: 'libssh/libssh.h' file not found` - the Homebrew headers are not on the compiler's include path. Fix:

```
export LDFLAGS="-L/opt/homebrew/lib"
export CFLAGS="-I/opt/homebrew/include"
export C_INCLUDE_PATH="/opt/homebrew/include"
pipenv sync
```

### 4. Verify

Still inside the pipenv shell:

```
ansible --version
ansible-playbook --version
```

Both must print version information.

---

## Phase 5: Credential Files

Two local files are required before any run. Both are gitignored - **never commit them**. Obtain the values from a team member over a secure channel.

### 1. Create the files

From the repo root:

```
echo the_vault_password > ansible/ansible_vault_pw
echo the_cvaas_token > ansible/cvaas.token
chmod 600 ansible/ansible_vault_pw ansible/cvaas.token
```

Replace the placeholders with the real values - no quotes, single line.

| File | Purpose |
|---|---|
| `ansible/ansible_vault_pw` | Decrypts the vaulted device credentials in the repo |
| `ansible/cvaas.token` | CVaaS service-account token (labeled "Ansible Based Automa..." in the CVaaS portal) |

---

## Phase 6: Dry Run

### 1. Optional offline warm-up

Renders templates only - no device connections, no credentials required:

```
cd ansible
ansible-playbook tasks/configure_switches.yml --tags renderconfig
```

### 2. The real dry run

**Safe - no-commit is the default behavior.** Renders configs, connects to devices read-only via eAPI, and shows diffs against running configs. Requires network reachability to the devices.

```
ansible-playbook tasks/configure_switches.yml
```

Success looks like every device in the `PLAY RECAP` showing:

```
ok=5  changed=0  failed=0
```

The `skipping` lines are the commit / CVaaS tasks - expected in a dry run. `changed=0` means the repo matches the live configs.

  **NOTE:** **Never pass** `-e '{"device_commit": true}'` unless you are executing an approved change. That flag is what commits changes to the production network. Without it, NAPALM stages the config in a session, computes the diff, and discards - devices are never modified.

---

## Phase 7: GitKraken Integration

GitKraken provides a visual client for git operations (history, branches, commits, PRs). It rides on top of the SSH/SSO setup from Phase 2 - no new authentication is created. The automation itself (`pipenv shell`, `ansible-playbook`) still runs in the terminal.

### 1. Install GitKraken

```
brew install --cask gitkraken
```

Or download from gitkraken.com. Launch it and sign in / create a GitKraken account on first run.

### 2. Point GitKraken at the system SSH key

This is the step that matters. By default GitKraken may generate and manage its own internal SSH key - that key would **not** be SSO-authorized for the org, and fetch/pull/push would fail.

**GitKraken → Preferences (`Cmd+,`) → SSH**, then either:

- **Option A (simplest):** enable **Use local SSH agent** - GitKraken uses whatever the `ssh` command itself uses, i.e., the key that already works.
- **Option B:** manually set the key paths:
  - Private key: `~/.ssh/id_ed25519`
  - Public key: `~/.ssh/id_ed25519.pub`

Do **not** click any "Generate new key" button.

### 3. Open the repo

**`Cmd+O` → Open a Repository →** navigate to the cloned repo (e.g. `~/repo/network-mt-eos`).

Because the repo was cloned over SSH, its remote is already the `git@github.com:...` URL, so GitKraken inherits working authentication.

### 4. Test

Click **Fetch / Pull** in GitKraken.

- Completes silently → integration done.
- Authentication error → GitKraken is using the wrong key. Return to step 2 of this phase and force the local SSH agent / correct key paths.

### Optional: GitHub integration for PR visibility

**Preferences → Integrations → GitHub → Connect.** Adds pull request visibility inside GitKraken - useful for the PR review/merge workflow. This uses browser OAuth and is separate from the git/SSH authentication above; it is not required for fetch, pull, push, or commit.

  If GitKraken ever prompts to authenticate with GitHub over **HTTPS**, decline/skip - HTTPS auth fails against the org's SAML SSO. All git transport goes over SSH.

---

## Daily Usage After Setup

```
cd ~/repo/network-mt-eos
pipenv shell
cd ansible
ansible-playbook tasks/configure_switches.yml
```

Type `exit` to leave the pipenv shell when done. Git operations can be done from the terminal or GitKraken interchangeably - both use the same SSH key and repo.
