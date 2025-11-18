# Nautobot Event-Driven Ansible Demo

This repository demonstrates how to use Event-Driven Ansible (EDA) with Nautobot's changelog event source to automate workflows based on changes in Nautobot.

⚡ **[Quick Start Guide →](QUICKSTART.md)** | Skip straight to the 5-minute setup!

---

## Table of Contents

- [Nautobot Event-Driven Ansible Demo](#nautobot-event-driven-ansible-demo)
  - [Table of Contents](#table-of-contents)
  - [📋 Overview](#-overview)
  - [📁 Repository Structure](#-repository-structure)
  - [🚀 Prerequisites](#-prerequisites)
    - [1. Required Software](#1-required-software)
      - [Installing Java](#installing-java)
    - [2. Nautobot API Token](#2-nautobot-api-token)
  - [⚙️ Setup](#️-setup)
    - [Step 1: Install the Collection](#step-1-install-the-collection)
    - [Step 2: Configure Environment Variables](#step-2-configure-environment-variables)
  - [🧪 Testing](#-testing)
    - [Basic Test (Recommended First Step)](#basic-test-recommended-first-step)
  - [🎯 Advanced Usage with AAP](#-advanced-usage-with-aap)
  - [📝 Rulebook Files](#-rulebook-files)
  - [🔧 Customization Ideas](#-customization-ideas)
    - [Filter by Different Object Types](#filter-by-different-object-types)
    - [Filter by Action](#filter-by-action)
    - [Add Additional Filters](#add-additional-filters)
  - [🏢 Integrating with Ansible Automation Platform](#-integrating-with-ansible-automation-platform)
    - [1. Create a Decision Environment](#1-create-a-decision-environment)
    - [2. Create a Custom Credential Type (Nautobot)](#2-create-a-custom-credential-type-nautobot)
    - [3. Create an EDA Project](#3-create-an-eda-project)
    - [4. Create Nautobot Credentials](#4-create-nautobot-credentials)
    - [5. Create a Rulebook Activation](#5-create-a-rulebook-activation)
    - [6. Test \& Monitor](#6-test--monitor)
  - [📚 Event Structure Reference](#-event-structure-reference)
  - [🔍 Troubleshooting](#-troubleshooting)
    - [Java Not Found](#java-not-found)
    - [Events Not Appearing](#events-not-appearing)
    - [Collection Not Found](#collection-not-found)
    - [Connection Errors](#connection-errors)
  - [📖 Additional Resources](#-additional-resources)
  - [🤝 Contributing](#-contributing)
  - [📄 License](#-license)

---

## 📋 Overview

The demo includes:
- **Basic test rulebook** - Prints all Nautobot changelog events (great for testing connectivity)
- **Advanced AAP rulebook** - Reacts to specific device changes and triggers Ansible Automation Platform job templates
- **Decision Environment** - Container image configuration for AAP EDA
- Collection requirements and configuration examples

## 📁 Repository Structure

```
nautobot-demo/
├── extensions/eda/rulebooks/          # Rulebooks (AAP EDA standard location)
│   ├── nautobot-changelog-test.yml    # Basic test rulebook
│   ├── nautobot-changelog-aap.yml     # Device automation (job templates)
│   ├── nautobot-changelog-ipam.yml    # IPAM automation (IP address changes)
│   ├── nautobot-changelog-filtered.yml # Filtering examples
│   ├── nautobot-changelog-multi-action.yml # Multiple actions
│   └── README.md                      # Rulebooks documentation
├── playbooks/                          # Ansible playbooks (triggered by EDA)
│   ├── configure_ip_on_arista.yml     # Configure IP on Arista devices
│   └── README.md                      # Playbook documentation
├── nautobot_de/                        # Decision Environment (for EDA)
│   ├── execution-environment.yml      # ansible-rulebook, Java, EDA plugins
│   ├── requirements.yml
│   ├── bindep.txt
│   └── README.md
├── nautobot_ee/                        # Execution Environment (for playbooks)
│   ├── execution-environment.yml      # Network collections, pynautobot
│   ├── requirements.yml
│   ├── requirements.txt
│   ├── bindep.txt
│   └── README.md
├── requirements.yml                    # Ansible collection dependencies
├── env-template.txt                    # Environment variable template
├── QUICKSTART.md                       # 5-minute setup guide
├── LICENSE                             # Apache 2.0 license
└── README.md                          # This file
```

**Why `extensions/eda/rulebooks/`?**

Ansible Automation Platform follows a standard directory structure for EDA content. When you create a Project in AAP that points to this repository, AAP will automatically discover and display rulebooks located in `extensions/eda/rulebooks/`. This is the same structure used by Ansible collections that include EDA content.

**Container Images:**

This repository includes configurations for **two different container images**:

1. **Decision Environment (DE)** - `nautobot_de/`
   - Used by: **EDA Rulebook Activations**
   - Contains: ansible-rulebook, Java, EDA event source plugins
   - Purpose: Listen for Nautobot events and trigger automation
   - Build guide: `nautobot_de/README.md`

2. **Execution Environment (EE)** - `nautobot_ee/`
   - Used by: **AAP Job Templates**
   - Contains: Base AAP EE + networktocode.nautobot collection + pynautobot
   - Purpose: Run playbooks that configure network devices
   - Build guide: `nautobot_ee/README.md`

## 🚀 Prerequisites

### 1. Required Software
- **Java JDK 17+** (required by ansible-rulebook)
- **Ansible EDA** (`ansible-rulebook` CLI) installed
- Access to a Nautobot instance
- (Optional) Ansible Automation Platform for the advanced rulebook

#### Installing Java

**macOS (using Homebrew):**
```bash
# Install Java 17
brew install openjdk@17

# Create symlink for system Java wrappers
sudo ln -sfn /usr/local/opt/openjdk@17/libexec/openjdk.jdk /Library/Java/JavaVirtualMachines/openjdk-17.jdk

# Add to your PATH and set JAVA_HOME
echo 'export PATH="/usr/local/opt/openjdk@17/bin:$PATH"' >> ~/.zshrc
echo 'export JAVA_HOME=$(/usr/libexec/java_home -v 17)' >> ~/.zshrc

# Load the new settings
source ~/.zshrc
```

**RHEL/Fedora/CentOS:**
```bash
sudo dnf install java-17-openjdk-devel
```

**Ubuntu/Debian:**
```bash
sudo apt update
sudo apt install openjdk-17-jdk
```

Verify Java installation:
```bash
java -version
```

### 2. Nautobot API Token
You'll need a Nautobot API token with at least read access to the Object Changes API:
1. Log into Nautobot
2. Navigate to **User** → **API Tokens**
3. Create a new token with appropriate permissions
4. Save the token securely

## ⚙️ Setup

### Step 1: Install the Collection

```bash
ansible-galaxy collection install -r requirements.yml
```

This installs the `networktocode.nautobot` collection which provides the `nautobot_changelog` event source plugin.

### Step 2: Configure Environment Variables

Copy and customize the environment template:

```bash
# Set your Nautobot instance URL and token
export NAUTOBOT_URL="https://your-nautobot-instance.com"
export NAUTOBOT_TOKEN="your-api-token-here"
```

Or reference `env-template.txt` for a template.

## 🧪 Testing

### Basic Test (Recommended First Step)

Start with the basic test rulebook to verify connectivity:

```bash
ansible-rulebook \
  -r extensions/eda/rulebooks/nautobot-changelog-test.yml \
  -i localhost, \
  --verbose \
  --env-vars NAUTOBOT_URL,NAUTOBOT_TOKEN
```

**What it does:**
- Polls Nautobot every 5 seconds for changelog events
- Prints every event to the console
- Helps verify your token and connectivity are working

**Test it:**
1. Run the command above
2. In Nautobot, make a change (edit a device, IP address, etc.)
3. Watch for the event to appear in the rulebook logs

Example output:
```
Got Nautobot changelog event:
{
  "id": "abc-123",
  "display": "Device changed",
  "changed_object_type": "dcim.device",
  "action": {"value": "update"},
  "changed_object": {
    "name": "router-01",
    "id": "device-uuid"
  },
  ...
}
```

## 🎯 Advanced Usage with AAP

### Device Automation

The `nautobot-changelog-aap.yml` rulebook demonstrates device automation:

```bash
ansible-rulebook \
  -r extensions/eda/rulebooks/nautobot-changelog-aap.yml \
  -i localhost, \
  --verbose \
  --env-vars NAUTOBOT_URL,NAUTOBOT_TOKEN
```

**What it does:**
- Listens for device creation or update events
- Triggers an AAP job template called "Configure device from Nautobot"
- Passes device details (name, ID, change ID) as extra vars to the job

**Conditions:**
```yaml
condition: >
  event.changed_object_type == "dcim.device"
  and event.action.value in ["create", "update"]
```

### IPAM Automation

The `nautobot-changelog-ipam.yml` rulebook demonstrates IP address automation:

```bash
ansible-rulebook \
  -r extensions/eda/rulebooks/nautobot-changelog-ipam.yml \
  -i localhost, \
  --verbose \
  --env-vars NAUTOBOT_URL,NAUTOBOT_TOKEN
```

**What it does:**
- Listens for IP address creation or update events
- Triggers different job templates based on the action:
  - **New IP**: Runs "IPAM - New IP Address Validation"
  - **Updated IP**: Runs "IPAM - IP Address Compliance Check"
- Passes IP details (address, prefix, status, DNS name, role) as extra vars

**Conditions:**
- New IP: `event.host is defined and event.created is defined`
- Updated IP: `event.last_updated != event.created`

**How it works:**
1. EDA detects IP address change in Nautobot
2. Rulebook triggers "Configure IP on Arista" job template
3. Playbook queries Nautobot for full device/interface details
4. Playbook connects to the Arista device
5. Playbook configures the IP address on the interface
6. Playbook verifies the configuration

See `playbooks/configure_ip_on_arista.yml` for the complete implementation.

## 📝 Rulebook Files

All rulebooks are located in `extensions/eda/rulebooks/` (AAP EDA standard location):

| File | Purpose |
|------|---------|
| `nautobot-changelog-test.yml` | Basic test - prints all events |
| `nautobot-changelog-aap.yml` | Device automation - triggers job templates on device changes |
| `nautobot-changelog-ipam.yml` | IPAM automation - triggers job templates on IP address changes |
| `nautobot-changelog-filtered.yml` | Examples with filters (tags, sites, object types) |
| `nautobot-changelog-multi-action.yml` | Multiple actions (webhooks, different job templates) |

**Other files:**
| File | Purpose |
|------|---------|
| `requirements.yml` | Ansible collection dependencies |
| `env-template.txt` | Environment variable template |
| `nautobot_de/` | Decision Environment build configuration |

## 🔧 Customization Ideas

### Filter by Different Object Types

Change `changed_object_type` to react to different Nautobot objects:

```yaml
# IP addresses
event.changed_object_type == "ipam.ipaddress"

# Sites
event.changed_object_type == "dcim.site"

# Circuits
event.changed_object_type == "circuits.circuit"
```

### Filter by Action

```yaml
# Only creations
event.action.value == "create"

# Only deletions
event.action.value == "delete"

# Create or update
event.action.value in ["create", "update"]
```

### Add Additional Filters

```yaml
# Only devices in a specific site
event.changed_object_type == "dcim.device"
and event.changed_object.site.name == "DC-01"

# Only devices with a specific tag
event.changed_object_type == "dcim.device"
and "production" in event.changed_object.tags
```

## 🏢 Integrating with Ansible Automation Platform

To use these rulebooks in AAP EDA:

### 1. Create a Decision Environment
Ensure your decision environment includes:
- `ansible-rulebook`
- `networktocode.nautobot` collection (via requirements.yml)

### 2. Create a Custom Credential Type (Nautobot)

Before creating credentials, you need to create a custom credential type for Nautobot:

1. Go to **Administration** → **Credential Types** → **Add**
2. Fill in:
   - **Name**: `Nautobot API`
   - **Description**: `Credential for Nautobot API access`

3. **Input Configuration:**
```yaml
fields:
  - id: nautobot_url
    type: string
    label: Nautobot URL
    help_text: The URL of your Nautobot instance (e.g., https://nautobot.example.com)
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

### 3. Create an EDA Project
- Point to this Git repository
- AAP will automatically detect the rulebooks in `extensions/eda/rulebooks/`

### 4. Create Nautobot Credentials

Now create the actual credential using your custom type:

1. Go to **Resources** → **Credentials** → **Add**
2. Fill in:
   - **Name**: `Nautobot Production`
   - **Credential Type**: `Nautobot API` (the type you just created)
   - **Nautobot URL**: `http://your-nautobot-instance:8080`
   - **Nautobot API Token**: `your-actual-token-here`
3. Click **Save**

### 5. Create a Rulebook Activation

1. Go to **Automation Decisions** → **Rulebook Activations** → **Add**
2. Fill in:
   - **Name**: `Nautobot Changelog Monitor`
   - **Project**: Your nautobot-demo project
   - **Rulebook**: Choose from discovered rulebooks:
     - `nautobot-changelog-test.yml` (basic testing)
     - `nautobot-changelog-aap.yml` (device automation)
     - `nautobot-changelog-ipam.yml` (IPAM/IP address automation)
     - `nautobot-changelog-filtered.yml` (filtering examples)
     - `nautobot-changelog-multi-action.yml` (multiple actions)
   - **Decision Environment**: `Nautobot Decision Environment`
   - **Credentials**: Select `Nautobot Production` (your Nautobot credential)
   - **Restart policy**: `On failure`
   - **Log level**: `Info` (or `Debug` for troubleshooting)
3. Click **Create rulebook activation**
4. **Enable** the activation

### 6. Test & Monitor
- Make changes in Nautobot
- Monitor the EDA activation logs
- Verify job templates are triggered with correct variables

## 📚 Event Structure Reference

The `nautobot_changelog` event source provides events with this structure:

```json
{
  "id": "unique-change-id",
  "display": "Human readable description",
  "changed_object_type": "dcim.device",
  "action": {
    "value": "create|update|delete"
  },
  "changed_object": {
    "id": "object-uuid",
    "name": "object-name",
    // ... other object fields
  },
  "object_data": {
    // Full object data
  },
  "changed_data": {
    // What changed (for updates)
  },
  "user": {
    "username": "user-who-made-change"
  },
  "time": "timestamp"
}
```

## 🔍 Troubleshooting

### Java Not Found
If you see "Java executable or JAVA_HOME environment variable not found":

**macOS:**
```bash
# Install Java (see Prerequisites section)
brew install openjdk@17

# Create symlink
sudo ln -sfn /usr/local/opt/openjdk@17/libexec/openjdk.jdk /Library/Java/JavaVirtualMachines/openjdk-17.jdk

# Add to shell config
echo 'export PATH="/usr/local/opt/openjdk@17/bin:$PATH"' >> ~/.zshrc
echo 'export JAVA_HOME=$(/usr/libexec/java_home -v 17)' >> ~/.zshrc
source ~/.zshrc
```

**Linux:**
```bash
# Set JAVA_HOME
export JAVA_HOME=/usr/lib/jvm/java-17-openjdk
export PATH="$JAVA_HOME/bin:$PATH"
```

**Verify:**
```bash
java -version
echo $JAVA_HOME
```

### Events Not Appearing

First, verify your token works outside of ansible-rulebook:
```bash
# Test the API directly
curl -H "Authorization: Token $NAUTOBOT_TOKEN" \
  "$NAUTOBOT_URL/api/extras/object-changes/?limit=1"
```

If that fails:
- Verify `NAUTOBOT_TOKEN` is set correctly: `echo $NAUTOBOT_TOKEN`
- Verify `NAUTOBOT_URL` is set correctly: `echo $NAUTOBOT_URL`
- Check token has read permissions for Object Changes in Nautobot UI
- Check token is not expired in Nautobot UI
- Make sure the token was created properly (re-create if needed)

If curl works but ansible-rulebook shows "Error 403":
- The token may have been truncated or has special characters
- Try exporting the token in quotes: `export NAUTOBOT_TOKEN="your-token-here"`
- Make sure you sourced your shell config if you added it there

### Collection Not Found
```bash
# Verify collection is installed
ansible-galaxy collection list | grep nautobot

# Reinstall if needed
ansible-galaxy collection install networktocode.nautobot --force
```

### Connection Errors

**Test Nautobot API manually:**
```bash
curl -H "Authorization: Token $NAUTOBOT_TOKEN" \
  "$NAUTOBOT_URL/api/extras/object-changes/?limit=1"
```

**Expected responses:**
- ✅ **Success**: `{"count":0,"next":null,"previous":null,"results":[]}` or similar JSON
- ❌ **Invalid token**: `{"detail":"Invalid token"}` - Token is wrong or expired, create a new one
- ❌ **403 Forbidden**: Token lacks permissions to view Object Changes
- ❌ **Connection refused**: Check `NAUTOBOT_URL` is correct and Nautobot is running

## 📖 Additional Resources

- [Nautobot Ansible Collection Documentation](https://docs.nautobot.com/projects/ansible/en/latest/)
- [Event-Driven Ansible Documentation](https://ansible.readthedocs.io/projects/rulebook/en/stable/)
- [Nautobot API Documentation](https://docs.nautobot.com/projects/core/en/stable/user-guide/platform-functionality/rest-api/overview/)

## 🤝 Contributing

Contributions are welcome! Feel free to:
- Fork this repository
- Submit pull requests with improvements
- Report issues or suggest enhancements
- Share your use cases and customizations

When contributing, please:
1. Test your changes locally
2. Update documentation as needed
3. Follow existing code style and structure
4. Provide clear commit messages

## 📄 License

Licensed under the Apache License, Version 2.0 (the "License");
you may not use this file except in compliance with the License.
You may obtain a copy of the License at

    http://www.apache.org/licenses/LICENSE-2.0

Unless required by applicable law or agreed to in writing, software
distributed under the License is distributed on an "AS IS" BASIS,
WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
See the License for the specific language governing permissions and
limitations under the License.

---

**Disclaimer:** This project is provided as-is for demonstration and educational purposes. It is not officially supported by Red Hat or Network to Code. For production use, please review and test thoroughly in your environment.