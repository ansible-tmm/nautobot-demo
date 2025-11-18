# Nautobot EDA Rulebooks

This directory contains Event-Driven Ansible rulebooks for Nautobot automation.

## 📁 Directory Structure

This directory follows the Ansible Automation Platform (AAP) standard for EDA content. When you create a Project in AAP that points to this repository, AAP will automatically discover and display all rulebooks in this `extensions/eda/rulebooks/` directory.

## 📝 Available Rulebooks

### 1. `nautobot-changelog-test.yml` 🧪 **TESTING**

**Purpose:** Test connectivity and view all Nautobot events

**What it does:**
- Listens for ALL Nautobot changelog events
- Prints each event to debug output with full details
- Perfect for verifying your Nautobot connection and API token
- Shows the exact structure of events you'll work with

**Use case:** Initial setup, troubleshooting, understanding event payloads

**Example output:**
```yaml
Got Nautobot changelog event (object_data):
{
  'host': '192.168.1.170',
  'mask_length': 24,
  'ip_version': 4,
  'status': '59a147a9-e2c7-4cbf-9da8-13ebd7de57b0',
  'dns_name': '',
  'description': '',
  'created': '2025-11-18T20:30:30.669Z',
  'last_updated': '2025-11-18T20:30:30.670Z',
  ...
}
```

**Test it locally:**
```bash
ansible-rulebook \
  -r extensions/eda/rulebooks/nautobot-changelog-test.yml \
  -i localhost, \
  --env-vars NAUTOBOT_URL,NAUTOBOT_TOKEN \
  --verbose
```

---

### 2. `nautobot-ip-address-trigger.yml` ✅ **PRODUCTION**

**Purpose:** Trigger automation when IP addresses are created or updated in Nautobot

**What it does:**
- Listens for IP address creation and update events
- Automatically triggers AAP Job Template: **"Nautobot Job Template"**
- Passes all IP address details as extra vars to your playbook

**Use case:** Automate device configuration, validation, or compliance checks when IP addresses are assigned in Nautobot

**Trigger condition:**
```yaml
event.host is defined
and event.ip_version is defined
and event.mask_length is defined
```

**Extra vars passed to Job Template:**
| Variable | Description | Example |
|----------|-------------|---------|
| `ip_address` | The IP address | `192.168.1.170` |
| `mask_length` | Subnet mask length | `24` |
| `ip_version` | IP version | `4` or `6` |
| `parent_prefix` | Parent prefix UUID | `a4a5d2dc-...` |
| `status_id` | Status UUID | `59a147a9-...` |
| `dns_name` | DNS hostname | `server01.example.com` |
| `description` | IP description | `Web server` |
| `role_id` | Role UUID | `0bc9b0f2-...` |
| `created_timestamp` | Creation time | `2025-11-18T20:30:30.669Z` |
| `updated_timestamp` | Last update time | `2025-11-18T20:30:30.670Z` |

**Example event that triggers this rulebook:**
```yaml
host: 192.168.1.170
mask_length: 24
ip_version: 4
status: 59a147a9-e2c7-4cbf-9da8-13ebd7de57b0
parent: a4a5d2dc-0e73-481f-add4-0620ce5bbe97
dns_name: ''
description: ''
created: '2025-11-18T20:30:30.669Z'
last_updated: '2025-11-18T20:30:30.670Z'
```

---

## 🔧 Common Event Fields

All events from the Nautobot changelog event source contain `object_data` with these fields:

```yaml
event:
  created: "timestamp"           # When object was created
  last_updated: "timestamp"      # When object was last updated
  host: "IP address"             # For IP address objects
  mask_length: 24                # Subnet mask
  ip_version: 4                  # IPv4 or IPv6
  status: "status UUID"          # Status identifier
  parent: "parent UUID"          # Parent object reference
  role: "role UUID"              # Role identifier (can be null)
  tenant: "tenant UUID"          # Tenant identifier (can be null)
  type: "host"                   # Object type
  dns_name: "hostname"           # DNS name
  description: "text"            # Description
  tags: []                       # List of tags
  custom_fields: {}              # Custom field data
  meta:
    source:
      name: "networktocode.nautobot.nautobot_changelog"
    received_at: "timestamp"
    uuid: "event UUID"
```

---

## 🚀 Using in AAP

### Setup Steps

1. **Create a Project** in AAP pointing to this repository
2. AAP will automatically discover these rulebooks
3. **Create a Rulebook Activation**:
   - Select your project
   - Choose `nautobot-ip-address-trigger.yml`
   - Select Decision Environment with `networktocode.nautobot` collection
   - Add credentials with `NAUTOBOT_URL` and `NAUTOBOT_TOKEN`
4. **Create the Job Template** named "Nautobot Job Template"
   - Point it to your playbook (e.g., `playbooks/configure_ip_on_arista_lookup.yml`)
   - Use Execution Environment with `pynautobot` installed
5. **Enable the Rulebook Activation** and monitor!

### Credentials Setup

Create a custom credential type for Nautobot:

**Input Configuration:**
```yaml
fields:
  - id: nautobot_url
    type: string
    label: Nautobot URL
  - id: nautobot_token
    type: string
    label: Nautobot API Token
    secret: true
required:
  - nautobot_url
  - nautobot_token
```

**Injector Configuration:**
```yaml
env:
  NAUTOBOT_TOKEN: '{{ nautobot_token }}'
  NAUTOBOT_URL: '{{ nautobot_url }}'
```

---

## 💡 Customization Tips

### Modify the Condition

Edit `nautobot-ip-address-trigger.yml` to add additional filters:

**Filter by DNS name:**
```yaml
condition: >
  event.host is defined
  and event.dns_name != ''
```

**Filter by status (only active IPs):**
```yaml
condition: >
  event.host is defined
  and event.status == "59a147a9-e2c7-4cbf-9da8-13ebd7de57b0"
```

**Filter by parent prefix:**
```yaml
condition: >
  event.host is defined
  and event.parent == "a4a5d2dc-0e73-481f-add4-0620ce5bbe97"
```

### Change the Job Template Name

Edit line 17 in `nautobot-ip-address-trigger.yml`:
```yaml
name: "Your Custom Job Template Name"
```

---

## 📖 Learn More

See the main [README.md](../../../README.md) for:
- Complete setup instructions
- Environment configuration
- Decision Environment building
- Troubleshooting guide
- API testing commands

---

**Why this directory structure?**

The `extensions/eda/rulebooks/` path is the standard location for EDA content in Ansible collections and repositories. AAP automatically discovers rulebooks in this location when you create a Project.

---

## 📄 License

Licensed under the Apache License, Version 2.0. See [LICENSE](../../../LICENSE) for details.
