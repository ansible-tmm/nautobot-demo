# Ansible Playbooks for Nautobot EDA

This directory contains Ansible playbooks that are triggered by EDA rulebook activations.

## 📁 Playbooks

### `configure_ip_on_arista.yml`

A complete end-to-end example that demonstrates the full EDA → Nautobot → Device workflow.

**Flow:**
1. **Receives event data** from EDA (IP address, mask length)
2. **Queries Nautobot** to get full IP address details
3. **Extracts device information** (which device/interface owns the IP)
4. **Looks up device details** (management IP, platform)
5. **Connects to Arista device** and configures the interface
6. **Verifies configuration** was applied

**Required Variables:**
- `ip_address`: The IP address (e.g., "192.168.1.160")
- `mask_length`: The subnet mask length (e.g., 24)

**Environment Variables:**
- `NAUTOBOT_URL`: Your Nautobot instance URL
- `NAUTOBOT_TOKEN`: Your Nautobot API token
- `ARISTA_USERNAME`: Username for Arista devices (default: admin)
- `ARISTA_PASSWORD`: Password for Arista devices

**Usage in AAP:**

1. **Create a Job Template** in AAP:
   - **Name**: `Configure IP on Arista`
   - **Job Type**: `Run`
   - **Inventory**: Your network inventory
   - **Project**: This repository
   - **Playbook**: `playbooks/configure_ip_on_arista.yml`
   - **Credentials**:
     - Nautobot API credential
     - Network device credential (for Arista)
   - **Extra Variables**: (will be passed from EDA)
     ```yaml
     ip_address: ""
     mask_length: ""
     ```

2. **Update the EDA Rulebook** to call this job template:

```yaml
- name: IP address created - configure device
  condition: >
    event.host is defined
    and event.created is defined
  action:
    run_job_template:
      name: "Configure IP on Arista"
      organization: "Default"
      extra_vars:
        ip_address: "{{ event.host }}"
        mask_length: "{{ event.mask_length }}"
```

3. **Test the flow:**
   - Create a new IP address in Nautobot
   - Assign it to a device interface
   - Watch EDA trigger the playbook
   - Verify configuration on the Arista device

## 🔧 Customization

### For Other Vendors

To adapt for Cisco IOS, Juniper, etc., change the network modules:

**Cisco IOS:**
```yaml
- name: Configure IP on Cisco IOS
  cisco.ios.ios_l3_interfaces:
    config:
      - name: "{{ interface_name }}"
        ipv4:
          - address: "{{ ip_address }}/{{ mask_length }}"
```

**Juniper:**
```yaml
- name: Configure IP on Juniper
  junipernetworks.junos.junos_l3_interfaces:
    config:
      - name: "{{ interface_name }}"
        ipv4:
          - address: "{{ ip_address }}/{{ mask_length }}"
```

### Add Configuration Validation

Add tasks to verify the configuration was applied:

```yaml
- name: Ping the IP address
  ansible.builtin.command:
    cmd: "ping -c 4 {{ ip_address }}"
  register: ping_result
  ignore_errors: true

- name: Report back to Nautobot
  ansible.builtin.uri:
    url: "{{ nautobot_url }}/api/ipam/ip-addresses/{{ ip_obj.id }}/"
    method: PATCH
    headers:
      Authorization: "Token {{ nautobot_token }}"
      Content-Type: "application/json"
    body_format: json
    body:
      custom_fields:
        last_validated: "{{ ansible_date_time.iso8601 }}"
    validate_certs: false
```

### Add Rollback on Failure

Implement automated rollback if configuration fails:

```yaml
- name: Configure IP address
  arista.eos.eos_l3_interfaces:
    config:
      - name: "{{ interface_name }}"
        ipv4:
          - address: "{{ ip_address }}/{{ mask_length }}"
    state: merged
  register: config_result

- name: Rollback on failure
  arista.eos.eos_config:
    commands:
      - "no interface {{ interface_name }}"
      - "interface {{ interface_name }}"
  when: config_result is failed
```

## 📋 Prerequisites

### Collections Required

Install these collections for the playbook to work:

```bash
ansible-galaxy collection install arista.eos
ansible-galaxy collection install networktocode.nautobot
ansible-galaxy collection install ansible.netcommon
```

Or add to your `requirements.yml`:

```yaml
collections:
  - name: arista.eos
    version: ">=6.0.0"
  - name: networktocode.nautobot
    version: ">=6.0.0"
  - name: ansible.netcommon
    version: ">=5.0.0"
```

### AAP Setup

1. **Decision Environment** must include:
   - `arista.eos` collection
   - `networktocode.nautobot` collection
   - Python `pynautobot` library

2. **Credentials** needed:
   - Nautobot API credential (custom type)
   - Machine/Network credential for devices

3. **Inventory** must include your network devices

## 🧪 Testing Locally

You can test the playbook locally (outside of EDA):

```bash
# Set environment variables
export NAUTOBOT_URL="http://your-nautobot:8080"
export NAUTOBOT_TOKEN="your-token"
export ARISTA_USERNAME="admin"
export ARISTA_PASSWORD="your-password"

# Run the playbook
ansible-playbook playbooks/configure_ip_on_arista.yml \
  -e "ip_address=192.168.1.160" \
  -e "mask_length=24" \
  -v
```

## 🔍 Troubleshooting

### "IP address not found in Nautobot"

- Verify the IP address exists in Nautobot
- Check it's assigned to a device interface
- Verify your API token has read permissions

### "Failed to connect to device"

- Check device management IP is set in Nautobot
- Verify network connectivity to the device
- Confirm credentials are correct
- Check platform is set correctly (Arista EOS)

### "Module not found: arista.eos"

- Install the collection: `ansible-galaxy collection install arista.eos`
- In AAP: ensure your Execution Environment includes the collection

## 📚 Learn More

- [Arista EOS Collection Docs](https://docs.ansible.com/ansible/latest/collections/arista/eos/index.html)
- [Nautobot Ansible Collection](https://docs.nautobot.com/projects/ansible/en/latest/)
- [AAP Job Templates](https://docs.ansible.com/automation-controller/latest/html/userguide/job_templates.html)

---

## 📄 License

Licensed under the Apache License, Version 2.0. See [LICENSE](../LICENSE) for details.
