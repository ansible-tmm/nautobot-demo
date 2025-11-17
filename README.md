# Nautobot Event-Driven Ansible Demo

This repository demonstrates how to use Event-Driven Ansible (EDA) with Nautobot's changelog event source to automate workflows based on changes in Nautobot.

## 📋 Overview

The demo includes:
- **Basic test rulebook** - Prints all Nautobot changelog events (great for testing connectivity)
- **Advanced AAP rulebook** - Reacts to specific device changes and triggers Ansible Automation Platform job templates
- Collection requirements and configuration examples

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
  -r nautobot-changelog-test.yml \
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

The `nautobot-changelog-aap.yml` rulebook demonstrates real-world automation:

```bash
ansible-rulebook \
  -r nautobot-changelog-aap.yml \
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

## 📝 Rulebook Files

| File | Purpose |
|------|---------|
| `nautobot-changelog-test.yml` | Basic test - prints all events |
| `nautobot-changelog-aap.yml` | Advanced - triggers AAP job templates on device changes |
| `requirements.yml` | Ansible collection dependencies |
| `env-template.txt` | Environment variable template |

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

### 2. Create an EDA Project
- Point to this Git repository
- AAP will automatically detect the rulebooks

### 3. Create Credentials
Either:
- **Option A:** Create an EDA credential with environment variables for `NAUTOBOT_URL` and `NAUTOBOT_TOKEN`
- **Option B:** Pass them as extra vars in the activation

### 4. Create a Rulebook Activation
1. Select your project
2. Choose a rulebook (`nautobot-changelog-test.yml` or `nautobot-changelog-aap.yml`)
3. Select your decision environment
4. Add credentials/environment variables
5. Enable the activation

### 5. Test & Monitor
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

Feel free to fork this repo and customize the rulebooks for your specific use cases!

## 📄 License

MIT