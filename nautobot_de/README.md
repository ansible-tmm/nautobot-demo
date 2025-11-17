# Nautobot Decision Environment

This directory contains the files needed to build a Decision Environment (DE) for Ansible Automation Platform Event-Driven Ansible with Nautobot support.

## 📋 What's a Decision Environment?

A Decision Environment is a container image that includes:
- `ansible-rulebook`
- Required Ansible collections (`networktocode.nautobot`)
- Python dependencies (`aiohttp`, etc.)
- System dependencies (Java for Drools rules engine)

## 📁 Files

| File | Purpose |
|------|---------|
| `execution-environment.yml` | Main config for ansible-builder |
| `requirements.yml` | Ansible collections to include |
| `requirements.txt` | Python packages to install |
| `bindep.txt` | System-level dependencies (Java) |
| `README.md` | This file |

## 🔨 Building the Decision Environment

### Prerequisites

1. **Install ansible-builder:**
   ```bash
   pip install ansible-builder
   ```

2. **Container runtime** (Docker or Podman):
   ```bash
   # macOS
   brew install podman
   podman machine init
   podman machine start

   # Or use Docker Desktop
   ```

3. **For Mac Users with Apple Silicon:**

   AAP runs on x86_64/AMD64 servers. If you're on Apple Silicon (M1/M2/M3), you need to build for the correct architecture:

   ```bash
   # Set podman to use AMD64 emulation
   podman machine set --rootful

   # Or ensure your podman machine supports multi-arch
   podman machine stop
   podman machine rm
   podman machine init --now --cpus 4 --memory 8192
   ```

### Build the Image

```bash
# Navigate to this directory
cd nautobot_de

# Build with ansible-builder
# IMPORTANT: Use --build-arg to specify the platform for AAP (linux/amd64)
ansible-builder build \
  -t nautobot-de:latest \
  -v 3 \
  --container-runtime podman \
  --build-arg EE_BASE_IMAGE=quay.io/ansible/ansible-rulebook:latest \
  --build-arg PKGMGR_PRESERVE_CACHE=always
```

**⚠️ Important for Mac Users (Apple Silicon):**

If you're building on a Mac with Apple Silicon (M1/M2/M3), you MUST build for the linux/amd64 platform since AAP runs on x86_64 servers:

```bash
# For Apple Silicon Macs - build for x86_64/amd64
podman build \
  --platform linux/amd64 \
  -f context/Containerfile \
  -t nautobot-de:latest \
  context/

# Or use buildx with docker
docker buildx build \
  --platform linux/amd64 \
  -f context/Containerfile \
  -t nautobot-de:latest \
  context/
```

This will create a container image named `nautobot-de:latest` compatible with AAP servers.

## 🚀 Pushing to Quay.io

### 1. Create a Repository on Quay

1. Go to [quay.io](https://quay.io)
2. Sign in or create an account
3. Click **Create New Repository**
4. Name it `nautobot-de`
5. Make it **Public** (or Private if preferred)
6. Click **Create Public Repository**

### 2. Login to Quay

```bash
podman login quay.io
# Enter your Quay.io username and password
```

### 3. Tag and Push

```bash
# Tag the image for Quay (replace YOUR_USERNAME with your Quay username)
podman tag localhost/nautobot-de:latest quay.io/YOUR_USERNAME/nautobot-de:latest

# Push to Quay
podman push quay.io/YOUR_USERNAME/nautobot-de:latest

# Optional: Also push with a version tag
podman tag localhost/nautobot-de:latest quay.io/YOUR_USERNAME/nautobot-de:1.0.0
podman push quay.io/YOUR_USERNAME/nautobot-de:1.0.0
```

## 🧪 Testing the Image Locally

Test your Decision Environment before pushing:

```bash
# Run a test rulebook
# Note: Rulebooks are in extensions/eda/rulebooks/ directory
podman run --rm -it \
  -e NAUTOBOT_URL="http://your-nautobot-url" \
  -e NAUTOBOT_TOKEN="your-token" \
  -v $(pwd)/../extensions/eda/rulebooks/nautobot-changelog-test.yml:/tmp/rulebook.yml:Z \
  nautobot-de:latest \
  ansible-rulebook \
    -r /tmp/rulebook.yml \
    -i localhost, \
    --verbose \
    --env-vars NAUTOBOT_URL,NAUTOBOT_TOKEN
```

## 🏢 Using in Ansible Automation Platform

### 1. Add the Decision Environment to AAP

1. In AAP Controller, go to **Administration** → **Execution Environments**
2. Click **Add**
3. Fill in:
   - **Name**: `Nautobot Decision Environment`
   - **Image**: `quay.io/YOUR_USERNAME/nautobot-de:latest`
   - **Pull**: `Always` or `If not present`
4. Click **Save**

### 2. Create a Custom Credential Type

First, create a custom credential type for Nautobot:

1. Go to **Administration** → **Credential Types** → **Add**
2. **Name**: `Nautobot API`
3. **Input Configuration:**
```yaml
fields:
  - id: nautobot_url
    type: string
    label: Nautobot URL
    help_text: The URL of your Nautobot instance
  - id: nautobot_token
    type: string
    label: Nautobot API Token
    secret: true
    help_text: API token for Nautobot authentication
required:
  - nautobot_url
  - nautobot_token
```

4. **Injector Configuration:**
```yaml
env:
  NAUTOBOT_URL: '{{ nautobot_url }}'
  NAUTOBOT_TOKEN: '{{ nautobot_token }}'
```

5. Click **Save**

### 3. Create the Credential

1. Go to **Resources** → **Credentials** → **Add**
2. **Name**: `Nautobot Production`
3. **Credential Type**: `Nautobot API`
4. **Nautobot URL**: `http://your-nautobot-instance:8080`
5. **Nautobot API Token**: `your-actual-token`
6. Click **Save**

### 4. Create an EDA Project

1. Go to **Resources** → **Projects** → **Add**
2. **Name**: `Nautobot EDA Demo`
3. **Source Control Type**: `Git`
4. **Source Control URL**: Your Git repository URL
5. **Options**: Check `Update Revision on Launch`
6. Click **Save**
7. AAP will automatically discover rulebooks in `extensions/eda/rulebooks/`

### 5. Create a Rulebook Activation

1. Go to **Automation Decisions** → **Rulebook Activations** → **Add**
2. Fill in:
   - **Name**: `Nautobot Changelog Monitor`
   - **Project**: `Nautobot EDA Demo`
   - **Rulebook**: Choose from discovered rulebooks:
     - `nautobot-changelog-test.yml` (basic testing)
     - `nautobot-changelog-aap.yml` (device automation)
     - `nautobot-changelog-filtered.yml` (filtering examples)
     - `nautobot-changelog-multi-action.yml` (multiple actions)
   - **Decision Environment**: `Nautobot Decision Environment`
   - **Credentials**: `Nautobot Production`
   - **Restart policy**: `On failure`
   - **Log level**: `Info` (or `Debug` for troubleshooting)
3. Click **Create rulebook activation**
4. **Enable** the activation
5. Monitor the logs!

**Note:** AAP automatically discovers rulebooks from `extensions/eda/rulebooks/` directory.

## 🔧 Customization

### Add More Collections

Edit `requirements.yml`:
```yaml
collections:
  - name: networktocode.nautobot
    version: ">=6.0.0"
  - name: ansible.eda
    version: ">=1.4.0"
  - name: community.general
```

### Add More Python Packages

Edit `requirements.txt`:
```
aiohttp>=3.8.0
requests>=2.28.0
pynautobot>=2.0.0
jmespath
```

### Change Base Image

Edit `execution-environment.yml` to use a different base:
```yaml
images:
  base_image:
    name: quay.io/ansible/ansible-rulebook:latest
```

Then rebuild:
```bash
ansible-builder build -t nautobot-de:latest -v 3
```

## 📚 Additional Resources

- [Ansible Builder Documentation](https://ansible.readthedocs.io/projects/builder/en/latest/)
- [Event-Driven Ansible Documentation](https://access.redhat.com/documentation/en-us/red_hat_ansible_automation_platform/2.4/html/using_event-driven_ansible/index)
- [Quay.io Documentation](https://docs.quay.io/)
- [Nautobot Ansible Collection](https://docs.nautobot.com/projects/ansible/en/latest/)

## 🐛 Troubleshooting

### ⚠️ "Exec format error" in AAP

**Error:** `exec container process '/opt/builder/bin/entrypoint': Exec format error`

**Cause:** Architecture mismatch - image was built for ARM64 (Apple Silicon) but AAP needs AMD64/x86_64.

**Solution:** Rebuild the image for the correct platform:

```bash
cd nautobot_de

# Method 1: Use podman with platform flag
podman build \
  --platform linux/amd64 \
  -f context/Containerfile \
  -t nautobot-de:latest \
  context/

# Method 2: Use docker buildx
docker buildx build \
  --platform linux/amd64 \
  -f context/Containerfile \
  -t nautobot-de:latest \
  context/

# Then re-tag and push
podman tag localhost/nautobot-de:latest quay.io/YOUR_USERNAME/nautobot-de:latest
podman push quay.io/YOUR_USERNAME/nautobot-de:latest --remove-signatures
```

**Verify the architecture:**
```bash
# Check what platform the image is
podman inspect nautobot-de:latest | grep Architecture
# Should show: "Architecture": "amd64"
```

### Build Fails - Missing Context
```bash
# Make sure ansible-builder has created the context
cd nautobot_de
rm -rf context
ansible-builder build -t nautobot-de:latest -v 3
```

### Java Not Found in Container
- The base image `quay.io/ansible/ansible-rulebook:latest` includes Java
- No additional installation needed

### Collection Not Found
```bash
# Test if collection is installed in the image
podman run --rm --platform linux/amd64 nautobot-de:latest ansible-galaxy collection list
```

### Push Permission Denied
```bash
# Make sure you're logged in and the repo is created
podman login quay.io
```

## ✅ Verification Checklist

After building and pushing:

- [ ] Image builds successfully with `ansible-builder build`
- [ ] Can test rulebook locally with `podman run`
- [ ] Image pushed to Quay.io successfully
- [ ] Image is visible on Quay.io repository
- [ ] Added Decision Environment in AAP Controller
- [ ] Created EDA Activation using the DE
- [ ] Activation runs and picks up Nautobot events

---

**Questions?** Check the main [README.md](../README.md) in the parent directory for the complete demo documentation.

---

## 📄 License

Licensed under the Apache License, Version 2.0. See [LICENSE](../LICENSE) for details.
