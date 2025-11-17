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

3. **Red Hat Registry Login** (for base image):
   ```bash
   podman login registry.redhat.io
   # Enter your Red Hat credentials
   ```

### Build the Image

```bash
# Navigate to this directory
cd nautobot_de

# Build with ansible-builder
ansible-builder build -t nautobot-de:latest -v 3
```

This will create a container image named `nautobot-de:latest`.

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
podman run --rm -it \
  -e NAUTOBOT_URL="http://your-nautobot-url" \
  -e NAUTOBOT_TOKEN="your-token" \
  -v $(pwd)/../nautobot-changelog-test.yml:/tmp/rulebook.yml:Z \
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

### 2. Create an EDA Credential

1. Go to **Credentials** → **Add**
2. Name it `Nautobot API`
3. Credential Type: **Generic**
4. Add environment variables:
   ```
   NAUTOBOT_URL: http://your-nautobot-instance
   NAUTOBOT_TOKEN: your-api-token
   ```

### 3. Create an EDA Project

1. Go to **Projects** → **Add**
2. Point to your Git repository containing the rulebooks
3. Save

### 4. Create a Rulebook Activation

1. Go to **Rulebook Activations** → **Add**
2. Select:
   - **Project**: Your nautobot-demo project
   - **Rulebook**: `nautobot-changelog-test.yml` or `nautobot-changelog-aap.yml`
   - **Decision Environment**: `Nautobot Decision Environment`
   - **Credentials**: `Nautobot API`
3. Enable the activation
4. Monitor the logs!

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

### Build Fails - Registry Login
```bash
# Make sure you're logged in to Red Hat registry
podman login registry.redhat.io
```

### Java Not Found in Container
- Check that `bindep.txt` includes `java-17-openjdk-headless`
- Rebuild the image

### Collection Not Found
```bash
# Test if collection is installed in the image
podman run --rm nautobot-de:latest ansible-galaxy collection list
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
