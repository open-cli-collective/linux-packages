# Open CLI Collective Linux Packages

APT and RPM repositories for Open CLI Collective command-line tools.

## Installation

### Debian / Ubuntu (APT)

```bash
# Add the GPG key
curl -fsSL https://open-cli-collective.github.io/linux-packages/keys/gpg.asc | sudo gpg --dearmor -o /usr/share/keyrings/open-cli-collective.gpg

# Add the repository
echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/open-cli-collective.gpg] https://open-cli-collective.github.io/linux-packages/apt stable main" | sudo tee /etc/apt/sources.list.d/open-cli-collective.list

# Update and install
sudo apt update
sudo apt install cfl  # or any package from the table below
```

### Fedora / RHEL / CentOS (DNF)

```bash
# Add the repository
sudo tee /etc/yum.repos.d/open-cli-collective.repo << 'EOF'
[open-cli-collective]
name=Open CLI Collective
baseurl=https://open-cli-collective.github.io/linux-packages/rpm
enabled=1
gpgcheck=1
gpgkey=https://open-cli-collective.github.io/linux-packages/keys/gpg.asc
EOF

# Install
sudo dnf install cfl  # or any package from the table below
```

## Available Packages

| Package | Description | Source Repo |
|---------|-------------|-------------|
| `cfl` | Confluence Cloud CLI | [confluence-cli](https://github.com/open-cli-collective/confluence-cli) |

---

## Maintainer Guide

### Adding a New Package

Follow these steps to add a new Open CLI Collective tool to the Linux repositories.

#### 1. Add nfpms config to the source repo

In the source repo's `.goreleaser.yml`, add:

```yaml
nfpms:
  - id: <package-name>
    package_name: <package-name>
    vendor: Open CLI Collective
    homepage: https://github.com/open-cli-collective/<repo-name>
    maintainer: Open CLI Collective <https://github.com/open-cli-collective>
    description: <description>
    license: MIT
    formats:
      - deb
      - rpm
    bindir: /usr/bin
    contents:
      - src: LICENSE
        dst: /usr/share/licenses/<package-name>/LICENSE
```

#### 2. Add dispatch step to release workflow

In the source repo's `.github/workflows/release.yml`, add a job:

```yaml
linux-packages:
  needs: goreleaser
  runs-on: ubuntu-latest
  steps:
    - name: Trigger linux-packages repo update
      uses: peter-evans/repository-dispatch@v3
      with:
        token: ${{ secrets.LINUX_PACKAGES_DISPATCH_TOKEN }}
        repository: open-cli-collective/linux-packages
        event-type: package-release
        client-payload: |-
          {
            "package": "<package-name>",
            "version": "${{ github.ref_name }}",
            "repo": "open-cli-collective/<repo-name>"
          }
```

#### 3. Configure the required secret

Add `LINUX_PACKAGES_DISPATCH_TOKEN` to the source repo's secrets:
- Create a Personal Access Token (PAT) with `repo` scope
- Add it as a repository secret

#### 4. Update this README

Add the new package to the "Available Packages" table above.

### Architecture

```
┌─────────────────────┐
│   Source Repo       │
│ (e.g. confluence-cli)│
└─────────┬───────────┘
          │ 1. Release tagged
          ▼
┌─────────────────────┐
│    GoReleaser       │
│ - Builds .deb/.rpm  │
│ - Attaches to release│
└─────────┬───────────┘
          │ 2. repository_dispatch
          ▼
┌─────────────────────┐
│  linux-packages     │
│ - Downloads packages│
│ - Signs with GPG    │
│ - Updates repo index│
└─────────┬───────────┘
          │ 3. Push to main
          ▼
┌─────────────────────┐
│   GitHub Pages      │
│ - Serves APT repo   │
│ - Serves RPM repo   │
└─────────────────────┘
```

### GPG Key Setup (One-Time)

The repositories are signed with a GPG key. This setup only needs to be done once.

#### Generate the key

```bash
gpg --full-generate-key
```

When prompted:
- Key type: `RSA and RSA` (option 1)
- Key size: `4096`
- Expiration: `0` (does not expire) or set an expiry
- Real name: `Open CLI Collective`
- Email: `open-cli-collective@users.noreply.github.com`
- Passphrase: Choose a strong passphrase

#### Get the key ID

```bash
gpg --list-secret-keys --keyid-format LONG
```

Output:
```
sec   rsa4096/ABC123DEF456789 2025-01-23 [SC]
      FULLFINGERPRINTHERE
uid   Open CLI Collective <...>
```

The key ID is `ABC123DEF456789` (after `rsa4096/`).

#### Export and configure

```bash
# Export private key (for GitHub secret)
gpg --armor --export-secret-keys <KEY_ID> > private-key.asc

# Export public key (for this repo)
gpg --armor --export <KEY_ID> > keys/gpg.asc
```

#### Add secrets to GitHub

Go to [org secrets](https://github.com/organizations/open-cli-collective/settings/secrets/actions) or repo secrets and add:

| Secret | Value |
|--------|-------|
| `LINUX_PACKAGES_GPG_PRIVATE_KEY` | Contents of `private-key.asc` |
| `LINUX_PACKAGES_GPG_PASSPHRASE` | The passphrase you chose |

#### Clean up

```bash
rm private-key.asc  # Delete the private key file - it now lives only in GitHub Secrets
```

Commit the public key (`keys/gpg.asc`) to this repository.

### Manual Package Addition

To manually add a package (for testing or one-off releases):

```bash
# APT
reprepro -b apt includedeb stable /path/to/package.deb

# RPM
cp /path/to/package.rpm rpm/packages/
createrepo_c rpm/packages
```

Then commit and push. GitHub Pages will update automatically.
