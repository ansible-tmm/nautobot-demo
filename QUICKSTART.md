# Quick Start Guide

Get up and running with Nautobot EDA in 5 minutes!

## 1️⃣ Install Java (Required)

ansible-rulebook needs Java to run:

**macOS:**
```bash
# Install Java 17
brew install openjdk@17

# Create symlink for system Java wrappers
sudo ln -sfn /usr/local/opt/openjdk@17/libexec/openjdk.jdk /Library/Java/JavaVirtualMachines/openjdk-17.jdk

# Add to PATH and set JAVA_HOME permanently
echo 'export PATH="/usr/local/opt/openjdk@17/bin:$PATH"' >> ~/.zshrc
echo 'export JAVA_HOME=$(/usr/libexec/java_home -v 17)' >> ~/.zshrc
source ~/.zshrc
```

**Linux (RHEL/Fedora):**
```bash
sudo dnf install java-17-openjdk-devel
```

**Linux (Ubuntu/Debian):**
```bash
sudo apt install openjdk-17-jdk
```

Verify: `java -version`

## 2️⃣ Install Ansible Dependencies

```bash
ansible-galaxy collection install -r requirements.yml
```

## 3️⃣ Set Environment Variables

```bash
export NAUTOBOT_URL="https://your-nautobot-instance.com"
export NAUTOBOT_TOKEN="your-api-token-here"
```

💡 **Get your API token**: Nautobot → User → API Tokens → Create Token

## 4️⃣ Run the Test Rulebook

```bash
ansible-rulebook -r nautobot-changelog-test.yml -i localhost, --verbose --env-vars NAUTOBOT_URL,NAUTOBOT_TOKEN
```

## 5️⃣ Test It!

1. Keep the rulebook running
2. Open Nautobot in a browser
3. Make any change (edit a device, create an IP, etc.)
4. Watch the event appear in your terminal! 🎉

## ✅ Success!

You should see output like:
```
Got Nautobot changelog event:
{
  "id": "...",
  "changed_object_type": "dcim.device",
  "action": {"value": "update"},
  ...
}
```

## 🎯 Next Steps

- **Try filtering**: `ansible-rulebook -r nautobot-changelog-filtered.yml -i localhost, --verbose --env-vars NAUTOBOT_URL,NAUTOBOT_TOKEN`
- **Test AAP integration**: Use `nautobot-changelog-aap.yml` (requires AAP setup)
- **Customize**: Edit the rulebooks to match your workflow

📖 See [README.md](README.md) for full documentation!
