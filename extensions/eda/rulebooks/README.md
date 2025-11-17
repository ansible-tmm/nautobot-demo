# Nautobot EDA Rulebooks

This directory contains Event-Driven Ansible rulebooks for Nautobot automation.

## 📁 Directory Structure

This directory follows the Ansible Automation Platform (AAP) standard for EDA content. When you create a Project in AAP that points to this repository, AAP will automatically discover and display all rulebooks in this `extensions/eda/rulebooks/` directory.

## 📝 Available Rulebooks

### Basic Testing

**`nautobot-changelog-test.yml`**
- Prints all Nautobot changelog events
- Great for testing connectivity and seeing event structure
- Use this first to verify your setup

**Usage:**
```bash
ansible-rulebook -r extensions/eda/rulebooks/nautobot-changelog-test.yml -i localhost, --verbose --env-vars NAUTOBOT_URL,NAUTOBOT_TOKEN
```

### AAP Integration

**`nautobot-changelog-aap.yml`**
- Reacts to device creation/update events
- Triggers AAP job template "Configure device from Nautobot"
- Passes device details as extra vars
- Requires AAP with job templates configured

**Trigger conditions:**
- Device created (`dcim.device`, `action=create`)
- Device updated (`dcim.device`, `action=update`)

### Filtering Examples

**`nautobot-changelog-filtered.yml`**
- Demonstrates filtering by:
  - Tags (production devices)
  - Object types (devices, IP addresses, sites, circuits)
  - Actions (create, update, delete)
- Multiple rules in one rulebook
- Debug output for each type

**Example filters:**
- Production device changes (tagged with "production")
- New IP address creation
- Site changes
- Circuit changes
- Any deletion operations

### Multi-Action Examples

**`nautobot-changelog-multi-action.yml`**
- Shows different action types:
  - Webhooks via `post_event`
  - Multiple job templates for different scenarios
- Device creation → Initial configuration job
- Device update → Compliance check job
- IP address creation → IPAM validation job

## 🔧 Common Event Fields

All events from the Nautobot changelog event source contain `object_data` with these common fields:

```yaml
event:
  created: "timestamp"
  last_updated: "timestamp"
  host: "IP or hostname" (for IP addresses)
  type: "object type"
  status: "status UUID"
  meta:
    source:
      name: "networktocode.nautobot.nautobot_changelog"
    received_at: "timestamp"
    uuid: "event UUID"
```

Additional fields depend on the object type (device, IP address, prefix, etc.).

## 💡 Customization Tips

### Filter by Object Type

```yaml
condition: >
  event.changed_object_type == "dcim.device"        # Devices
  event.changed_object_type == "ipam.ipaddress"    # IP Addresses
  event.changed_object_type == "dcim.site"          # Sites
  event.changed_object_type == "circuits.circuit"   # Circuits
```

### Filter by Action

```yaml
condition: >
  event.action.value == "create"                    # Only creations
  event.action.value == "update"                    # Only updates
  event.action.value == "delete"                    # Only deletions
  event.action.value in ["create", "update"]        # Create OR update
```

### Filter by Tag

```yaml
condition: >
  "production" in event.changed_object.tags         # Has production tag
```

### Filter by Site

```yaml
condition: >
  event.changed_object.site.name == "DC-01"         # Specific site
```

## 🚀 Using in AAP

1. **Create a Project** pointing to this repository
2. AAP will auto-discover these rulebooks
3. **Create a Rulebook Activation**:
   - Select project
   - Choose rulebook
   - Add Decision Environment with `networktocode.nautobot` collection
   - Add credentials for `NAUTOBOT_URL` and `NAUTOBOT_TOKEN`
4. Enable and monitor!

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
