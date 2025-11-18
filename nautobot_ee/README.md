# Nautobot Execution Environment (EE)

Container image for Ansible Automation Platform to run network automation playbooks with Nautobot integration.

## 🏗️ What's Inside

**Base:** Official Red Hat AAP 2.6 EE (`ee-supported-rhel9:latest`)
- Already includes: Ansible Core, network collections (arista.eos, cisco.ios, etc.), network libraries (netmiko, paramiko)

**What We Add:**
- ✅ `networktocode.nautobot` collection
- ✅ `pynautobot` Python library
- ✅ System compile dependencies (via bindep.txt)

## 🚀 Build It

### Prerequisites

```bash
# Install ansible-builder
pip install ansible-builder

# Login to Red Hat registry (free account at developers.redhat.com)
podman login registry.redhat.io
```

### Build (Mac with Apple Silicon)

```bash
cd nautobot_ee
rm -rf context

ansible-builder create -v 3

podman build \
  --platform linux/amd64 \
  --format docker \
  -f context/Containerfile \
  -t nautobot-ee:latest \
  context/

# Verify
podman inspect nautobot-ee:latest | grep Architecture
# Should show: "Architecture": "amd64"
```

### Build (Linux)

```bash
cd nautobot_ee
ansible-builder build -t nautobot-ee:latest -v 3
```

## 📤 Push to Quay

```bash
# Login
podman login quay.io

# Tag (replace YOUR_USERNAME with your Quay username)
podman tag localhost/nautobot-ee:latest quay.io/YOUR_USERNAME/nautobot-ee:latest
podman tag localhost/nautobot-ee:latest quay.io/YOUR_USERNAME/nautobot-ee:1.0.0

# Push
podman push quay.io/YOUR_USERNAME/nautobot-ee:latest --remove-signatures
podman push quay.io/YOUR_USERNAME/nautobot-ee:1.0.0 --remove-signatures
```

## 🎯 Use in AAP

1. **Add Execution Environment:**
   - Administration → Execution Environments → Add
   - Name: `Nautobot Network Automation`
   - Image: `quay.io/YOUR_USERNAME/nautobot-ee:latest`
   - Pull: `Always`

2. **Use in Job Templates:**
   - Create/Edit Job Template
   - Execution Environment: `Nautobot Network Automation`
   - Done!

## 📝 Files Explained

**`execution-environment.yml`** - Main config:
```yaml
version: 3
images:
  base_image:
    name: registry.redhat.io/ansible-automation-platform-26/ee-supported-rhel9:latest
dependencies:
  galaxy: requirements.yml    # Ansible collections
  python: requirements.txt    # Python packages
  system: bindep.txt          # System packages
options:
  package_manager_path: /usr/bin/microdnf
```

**`requirements.yml`** - One collection:
```yaml
collections:
  - name: networktocode.nautobot
    version: ">=6.0.0"
```

**`requirements.txt`** - One Python package:
```txt
pynautobot>=2.0.0
```

**`bindep.txt`** - System dependencies for compilation:
```txt
python3-pip [platform:rpm]
systemd-devel [platform:rpm]
pkgconfig [platform:rpm]
gcc [platform:rpm]
python3-devel [platform:rpm]
libxml2-devel [platform:rpm]
```

That's it! No complex build steps needed - `ansible-builder` handles everything.

## 🐛 Troubleshooting

**"No module named pip"**
- Solution: Add `python3-pip [platform:rpm]` to `bindep.txt`

**"fatal error: libxml/xmlreader.h"**
- Solution: Add `libxml2-devel [platform:rpm]` to `bindep.txt`

**"fatal error: systemd/sd-journal.h"**
- Solution: Add `systemd-devel [platform:rpm]` to `bindep.txt`

**"Exec format error" in AAP**
- Solution: Build with `--platform linux/amd64` on Mac (see above)

**Build fails on some Python package**
- Check error for missing `.h` file
- Find the `-devel` package that provides it
- Add to `bindep.txt`

## 🆚 EE vs DE

| | Execution Environment (EE) | Decision Environment (DE) |
|---|---|---|
| **Purpose** | Run playbooks | Listen for events |
| **Used by** | Job Templates | Rulebook Activations |
| **This folder** | `nautobot_ee/` | `nautobot_de/` |

---

**Related:** See `../nautobot_de/` for the Decision Environment used by EDA rulebooks.

## 📄 License

Apache License 2.0 - See [LICENSE](../LICENSE)
