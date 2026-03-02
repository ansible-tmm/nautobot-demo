# Nautobot Dynamic Inventory for Ansible Automation Platform

This directory contains the inventory source file used to pull hosts dynamically from Nautobot into AAP.

## Why "Sourced from a Project"?

Nautobot is not available as a built-in inventory source type in AAP (unlike AWS, Azure, GCE, etc.). Instead, the `networktocode.nautobot` collection provides inventory plugins that AAP can use through the **"Sourced from a Project"** inventory source type. This means:

- The inventory plugin configuration lives as a YAML file in a git repo
- AAP syncs that repo as a Project
- An Inventory Source points to the YAML file inside that Project
- The Execution Environment must have the `networktocode.nautobot` collection installed

## Prerequisites

1. **Nautobot EE** — An Execution Environment with `networktocode.nautobot` installed.  
   The `nautobot_ee/` directory in this repo contains the EE build definition. The built image is at `quay.io/acme_corp/nautobot-ee`.

2. **Nautobot API access** — You need the Nautobot URL and an API token.

3. **AAP Project** — This git repo must be synced as a Project in AAP.

## Walkthrough: Setting Up the Dynamic Inventory

### Step 1: Create a Custom Credential Type

AAP needs a way to inject `NAUTOBOT_URL` and `NAUTOBOT_TOKEN` as environment variables into the inventory sync job. A Custom Credential Type handles this.

1. In AAP, go to **Administration** → **Credential Types** → **Add**
2. Set the **Name** to `Nautobot`
3. Paste the following into **Input Configuration**:

```yaml
fields:
  - id: nautobot_url
    type: string
    label: Nautobot URL
  - id: nautobot_token
    type: string
    label: Nautobot Token
    secret: true
required:
  - nautobot_url
  - nautobot_token
```

4. Paste the following into **Injector Configuration**:

```yaml
env:
  NAUTOBOT_URL: '{{ nautobot_url }}'
  NAUTOBOT_TOKEN: '{{ nautobot_token }}'
```

5. Click **Save**.

### Step 2: Create a Credential

1. Go to **Resources** → **Credentials** → **Add**
2. Set the **Name** to `Nautobot Credential`
3. Set **Credential Type** to the `Nautobot` type you just created
4. Fill in:
   - **Nautobot URL**: Your Nautobot instance URL (e.g. `https://nautobot.example.com`)
   - **Nautobot Token**: Your Nautobot API token
5. Click **Save**

### Step 3: Ensure the Project Is Synced

1. Go to **Resources** → **Projects**
2. Verify the project pointing to this git repo exists (e.g. "Nautobot Example Project")
3. Click the **Sync** button to pull the latest code (including the `inventory/nautobot_inventory.yml` file)
4. Wait for the sync to complete successfully

### Step 4: Create the Inventory (if not already created)

1. Go to **Resources** → **Inventories** → **Add** → **Add Inventory**
2. Set the **Name** to `Nautobot Inventory`
3. Set the **Organization**
4. Click **Save**

### Step 5: Add an Inventory Source

This is the key step that ties everything together.

1. Open your **Nautobot Inventory**
2. Go to the **Sources** tab → **Add**
3. Fill in the following:

| Field                     | Value                                                     |
|---------------------------|-----------------------------------------------------------|
| **Name**                  | `Nautobot Source`                                         |
| **Source**                 | `Sourced from a Project`                                  |
| **Credential**             | `Nautobot Credential` (the one created in Step 2)         |
| **Project**                | `Nautobot Example Project` (or your project name)         |
| **Inventory file**         | `nautobot_inventory.yml` (at project root)                |
| **Execution Environment**  | `Nautobot EE`                                             |

4. Optionally check **Update on launch** if you want fresh inventory data every time a job runs
5. Click **Save**

### Step 6: Sync the Inventory Source

1. On the Inventory Source you just created, click the **Sync** button
2. Watch the sync job output — it should connect to Nautobot and pull in devices
3. Go back to the **Hosts** tab on your inventory to verify hosts were imported
4. Check the **Groups** tab to see groups created by `group_by` (location, status, role)

## How It Works

```
┌─────────────────────┐
│  Git Repo            │
│  (this repo)         │
│                      │
│  inventory/          │
│    nautobot_         │
│    inventory.yml     │ ◄── inventory plugin config
└──────────┬──────────┘
           │ git sync
           ▼
┌─────────────────────┐
│  AAP Project         │
│  "Nautobot Example   │
│   Project"           │
└──────────┬──────────┘
           │ references
           ▼
┌─────────────────────┐     ┌─────────────────────┐
│  Inventory Source    │     │  Nautobot EE         │
│  "Sourced from a    │────►│  (has networktocode. │
│   Project"          │     │   nautobot collection)│
│                     │     └─────────────────────┘
│  + Nautobot         │
│    Credential       │     ┌─────────────────────┐
│    (injects env     │────►│  Nautobot API        │
│     vars)           │     │  (returns devices,   │
└──────────┬──────────┘     │   VMs, groups)       │
           │                └─────────────────────┘
           ▼
┌─────────────────────┐
│  AAP Inventory       │
│  "Nautobot Inventory"│
│                      │
│  Hosts + Groups      │
│  populated from      │
│  Nautobot            │
└─────────────────────┘
```

## Inventory File Details

The `nautobot_inventory.yml` file uses the **GraphQL-based** inventory plugin (`networktocode.nautobot.gql_inventory`), which is more efficient than the REST-based plugin because it fetches only the fields you request in a single query.

Key configuration:

- **`plugin`**: Must be `networktocode.nautobot.gql_inventory` (or `networktocode.nautobot.inventory` for REST)
- **`api_endpoint`** / **`token`**: Pulled from environment variables injected by the custom credential
- **`query`**: Defines what device/VM fields to fetch from Nautobot via GraphQL
- **`group_by`**: Creates Ansible inventory groups based on device attributes (location, status, role)

### Switching to the REST Inventory Plugin

If you prefer the REST-based plugin, replace `nautobot_inventory.yml` with:

```yaml
---
plugin: networktocode.nautobot.inventory
api_endpoint: "{{ lookup('env', 'NAUTOBOT_URL') }}"
token: "{{ lookup('env', 'NAUTOBOT_TOKEN') }}"
validate_certs: false
group_by:
  - location
  - role
  - platform
  - status
config_context: false
interfaces: false
```

## Troubleshooting

- **"No inventory plugin found"**: The Execution Environment does not have the `networktocode.nautobot` collection. Make sure you select the **Nautobot EE**.
- **Inventory file not showing in dropdown**: Sync the Project first. The inventory file must exist in the repo and the project must be up to date.
- **Authentication errors**: Verify the credential has the correct Nautobot URL (include `https://`) and a valid API token. Check that the custom credential type injectors are set up correctly.
- **Empty inventory after sync**: Check the `query` filters in the inventory file. Make sure devices in Nautobot have a `name` and `primary_ip4` assigned.
- **SSL errors**: Set `validate_certs: false` in the inventory file if your Nautobot instance uses self-signed certificates.

## References

- [Nautobot Ansible Collection - Inventory Plugin](https://github.com/nautobot/nautobot-ansible/tree/develop/plugins/inventory)
- [Nautobot Ansible Collection - GraphQL Inventory Plugin](https://docs.nautobot.com/projects/ansible/en/latest/plugins/gql_inventory_inventory.html)
- [AAP Documentation - Inventory Sources](https://docs.redhat.com/en/documentation/red_hat_ansible_automation_platform/)
