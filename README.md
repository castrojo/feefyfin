# feefyfin

A custom bootc operating system image based on the lessons from [Universal Blue](https://universal-blue.org/) and [Bluefin](https://projectbluefin.io). This OS is designed for reliability, customization, and ease of use.

> Be the one who moves, not the one who is moved.

## What's Included

### Build System
- Automated builds via GitHub Actions on every commit
- Optional: Signed images with cosign for security (see production setup)
- Version tracking with Renovate for automatic base image updates
- Optional: SBOM generation for supply chain security (see production setup)
- Image validation with `bootc container lint`
- Automatic cleanup of old images (90+ days) to save storage space
- Release workflow with testing branch - test changes before production
  - `testing` branch builds `:testing` images
  - `main` branch builds `:stable` images
  - Automated releases with [Release Please](https://github.com/googleapis/release-please)

### Homebrew Integration
- Pre-configured Brewfiles for easy package installation and customization
- Includes curated collections: development tools, fonts, CLI utilities
- Users install packages at runtime with `brew bundle`, aliased to premade `ujust commands`
- See [custom/brew/README.md](custom/brew/README.md) for details

### Flatpak Support
- Ship your favorite flatpaks
- Automatically installed on first boot after user setup
- See [custom/flatpaks/README.md](custom/flatpaks/README.md) for details

### Rechunker
- Optimizes container image layer distribution for better download resumability
- Based on [hhd-dev/rechunk](https://github.com/hhd-dev/rechunk) v1.2.4
- Disabled by default for faster initial builds
- Enable in `.github/workflows/build.yml` by uncommenting the rechunker steps
- Recommended for production deployments after initial testing

### ujust Commands
- User-friendly command shortcuts via `ujust`
- Pre-configured examples for app installation and system maintenance
- See [custom/ujust/README.md](custom/ujust/README.md) for details

### Build Scripts
- Modular numbered scripts (10-, 20-, 30-) run in order
- Example scripts included for third-party repositories and desktop replacement
- Helper functions for safe COPR usage
- See [build/README.md](build/README.md) for details

## GitHub Setup Instructions

### 1. Enable GitHub Actions

**IMPORTANT: You must enable GitHub Actions for the workflows to run.**

1. Go to your repository on GitHub: https://github.com/castrojo/feefyfin
2. Click on the **"Actions"** tab at the top
3. If you see a message about workflows, click **"I understand my workflows, go ahead and enable them"**
4. Additional settings to verify:
   - Go to **Settings** → **Actions** → **General**
   - Under "Actions permissions", ensure "Allow all actions and reusable workflows" is selected
   - Under "Workflow permissions", ensure "Read and write permissions" is selected
   - Check "Allow GitHub Actions to create and approve pull requests"

Once enabled, your first build will start automatically!

### 2. Create the Testing Branch

The release workflow requires a `testing` branch:

```bash
git checkout -b testing
git push -u origin testing
```

### 3. Verify Workflows Are Running

After enabling Actions and pushing the testing branch:

1. Go to the **Actions** tab in your repository
2. You should see workflows running:
   - **Build container image** - Runs on main branch
   - **Build Testing Image** - Runs on testing branch
3. Click on any workflow run to see the build progress
4. Once complete, your images will be available at: `ghcr.io/castrojo/feefyfin:stable` and `ghcr.io/castrojo/feefyfin:testing`

### 4. Optional: Enable Image Signing with Cosign

Image signing is disabled by default to let you start building immediately. However, signing is **strongly recommended** for production use.

#### Why Sign Images?

- Verify image authenticity and integrity
- Prevent tampering and supply chain attacks
- Required for some enterprise/security-focused deployments
- Industry best practice for production images

#### Setup Instructions

1. **Generate signing keys** (run on your local machine):
   ```bash
   cosign generate-key-pair
   ```
   
   This creates two files:
   - `cosign.key` (private key) - Keep this secret!
   - `cosign.pub` (public key) - Commit this to your repository

2. **Add the private key to GitHub Secrets**:
   - Copy the entire contents of `cosign.key`
   - Go to your repository: https://github.com/castrojo/feefyfin
   - Navigate to **Settings** → **Secrets and variables** → **Actions**
   - Click **"New repository secret"**
   - Name: `SIGNING_SECRET`
   - Value: Paste the entire contents of `cosign.key`
   - Click **"Add secret"**

3. **Replace the placeholder public key**:
   - Open `cosign.pub` in your repository
   - Replace the contents with your actual public key from the `cosign.pub` file you generated
   - Commit and push the change:
     ```bash
     git add cosign.pub
     git commit -m "chore: add cosign public key"
     git push
     ```

4. **Enable signing in the workflows**:
   - Edit `.github/workflows/build.yml`
   - Find the section "OPTIONAL: Image Signing with Cosign" (around line 220)
   - Uncomment the cosign signing steps (remove the `#` from the beginning of each line)
   - Edit `.github/workflows/build-testing.yml` and do the same
   - Commit and push:
     ```bash
     git add .github/workflows/build.yml .github/workflows/build-testing.yml
     git commit -m "feat: enable image signing with cosign"
     git push
     ```

5. **Your next build will produce signed images!**

   **Important:** Never commit `cosign.key` to the repository. It's already in `.gitignore`.

#### Verifying Signed Images

Once signing is enabled, users can verify your images with:
```bash
cosign verify --key cosign.pub ghcr.io/castrojo/feefyfin:stable
```

## Switch to Your Image

Once the GitHub Actions workflow completes successfully:

```bash
sudo bootc switch ghcr.io/castrojo/feefyfin:stable
sudo systemctl reboot
```

Or use the testing image:
```bash
sudo bootc switch ghcr.io/castrojo/feefyfin:testing
sudo systemctl reboot
```

## Customization

### Choose Your Base Image

Edit `Containerfile` (line 24) to select your base image:
```dockerfile
FROM ghcr.io/ublue-os/bluefin:stable
```

Options:
- `bluefin:stable` - Developer-focused with GNOME (default)
- `bazzite:stable` - Gaming-optimized 
- `aurora:stable` - KDE Plasma desktop

### Add Packages

Modify `build/10-build.sh` to install packages:
```bash
dnf5 install -y package-name
```

### Customize Apps

- Add Brewfiles in `custom/brew/` ([guide](custom/brew/README.md))
- Add Flatpaks in `custom/flatpaks/` ([guide](custom/flatpaks/README.md))
- Add ujust commands in `custom/ujust/` ([guide](custom/ujust/README.md))

## Production Features

Ready for production? Enable these optional features for enhanced security and reliability:

### Enable Rechunker (Recommended)

Optimizes image layer distribution for better download resumability:

1. Edit `.github/workflows/build.yml`
2. Find the "Rechunk (OPTIONAL)" section around line 121
3. Uncomment the "Run Rechunker" step
4. Uncomment the "Load in podman and tag" step
5. Comment out the "Tag for registry" step that follows
6. Commit and push

### Enable SBOM Attestation (Recommended)

Generates Software Bill of Materials for supply chain security:

1. First complete image signing setup above
2. Edit `.github/workflows/build.yml`
3. Find the "OPTIONAL: SBOM Attestation" section around line 232
4. Uncomment the "Add SBOM Attestation" step
5. Commit and push

## Local Testing

Test your changes before pushing:

```bash
just build              # Build container image
just build-qcow2        # Build VM disk image
just run-vm-qcow2       # Test in browser-based VM
```

## Detailed Guides

- [Homebrew/Brewfiles](custom/brew/README.md) - Runtime package management
- [Flatpak Preinstall](custom/flatpaks/README.md) - GUI application setup
- [ujust Commands](custom/ujust/README.md) - User convenience commands
- [Build Scripts](build/README.md) - Build-time customization
- [Release Workflow](RELEASE_WORKFLOW.md) - How to manage releases

## Community

- [Universal Blue Forums](https://universal-blue.discourse.group/)
- [Universal Blue Discord](https://discord.gg/WEu6BdFEtp)
- [bootc Discussion](https://github.com/bootc-dev/bootc/discussions)

## Learn More

- [Universal Blue Documentation](https://universal-blue.org/)
- [bootc Documentation](https://containers.github.io/bootc/)
- [Video Tutorial by TesterTech](https://www.youtube.com/watch?v=IxBl11Zmq5wE)

## Security

This OS provides security features for production use:
- Optional SBOM generation (Software Bill of Materials) for supply chain transparency
- Optional image signing with cosign for cryptographic verification
- Automated security updates via Renovate
- Build provenance tracking

These security features are disabled by default to allow immediate testing. When you're ready for production, see the setup instructions above to enable them.

---

Based on [finpilot template](https://github.com/castrojo/finpilot) from [Universal Blue Project](https://universal-blue.org/)
