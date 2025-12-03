# Complete AWX Playbook Setup Guide for Zammad Inventory

## Step 1: Create Project Directory Structure

On your AWX server (or local machine), create the following directory structure:

```bash
mkdir -p ~/zammad-inventory-project
cd ~/zammad-inventory-project

# Create subdirectories
mkdir -p roles/zammad_inventory/tasks
mkdir -p roles/zammad_inventory/templates
mkdir -p roles/zammad_inventory/files
mkdir -p inventory
mkdir -p group_vars
mkdir -p host_vars
```

## Step 2: Create the Inventory File

Create file: `inventory/hosts.yml`

```yaml
all:
  children:
    zammad_servers:
      hosts:
        hd-ops.scc.kit.edu:
          ansible_host: 141.52.225.103
          state: Prod
          owner: AH
        eunode-helpdesk.scc.kit.edu:
          ansible_host: 141.52.225.39
          state: Prod
          owner: PW
        bwcloud-helpdesk.scc.kit.edu:
          ansible_host: 129.13.64.28
          state: Prod
          owner: AH
        beyond-helpdesk.scc.kit.edu:
          ansible_host: 141.52.225.8
          state: Prod
          owner: PW
        # Add all other hosts here...

  vars:
    ansible_connection: ssh
    ansible_user: root
    ansible_ssh_private_key_file: ~/.ssh/id_rsa
    ansible_python_interpreter: /usr/bin/python3
```

## Step 3: Create Main Playbook

Create file: `site.yml` (in root of project)

```yaml
---
- name: Collect Zammad Instance Information
  hosts: zammad_servers
  gather_facts: yes
  
  tasks:
    - name: Check if Zammad service exists
      systemd:
        name: zammad
      register: zammad_service
      ignore_errors: yes

    - name: Get Zammad version from package (RedHat)
      shell: rpm -qa zammad 2>/dev/null | grep -oP 'zammad-\K[0-9.]+'
      register: zammad_rpm_version
      ignore_errors: yes
      changed_when: false
      when: ansible_os_family == "RedHat"

    - name: Get Zammad version from package (Debian)
      shell: dpkg -l zammad 2>/dev/null | grep zammad | awk '{print $3}'
      register: zammad_deb_version
      ignore_errors: yes
      changed_when: false
      when: ansible_os_family == "Debian"

    - name: Get Zammad version from API
      uri:
        url: "http://localhost:3000/api/v1/version"
        method: GET
        timeout: 5
      register: zammad_api_version
      ignore_errors: yes
      changed_when: false

    - name: Set Zammad version fact
      set_fact:
        zammad_version: "{{ zammad_rpm_version.stdout or zammad_deb_version.stdout or zammad_api_version.json.version or 'Unknown' }}"

    - name: Create Zammad info dict
      set_fact:
        zammad_info:
          hostname: "{{ inventory_hostname }}"
          ip_address: "{{ ansible_default_ipv4.address | default('Unknown') }}"
          os: "{{ ansible_distribution }} {{ ansible_distribution_version }}"
          os_family: "{{ ansible_os_family }}"
          kernel: "{{ ansible_kernel }}"
          cpu_count: "{{ ansible_processor_vcpus }}"
          ram_mb: "{{ ansible_memtotal_mb }}"
          ram_gb: "{{ (ansible_memtotal_mb / 1024) | int }}"
          zammad_version: "{{ zammad_version }}"
          zammad_active: "{{ zammad_service.state == 'started' }}"
          zammad_enabled: "{{ zammad_service.enabled | default(false) }}"
          last_updated: "{{ ansible_date_time.iso8601 }}"

    - name: Display collected info
      debug:
        msg: "{{ zammad_info }}"

- name: Generate Centralized Reports
  hosts: localhost
  gather_facts: no
  
  tasks:
    - name: Create reports directory
      file:
        path: "/tmp/zammad_reports"
        state: directory
        mode: '0755'

    - name: Generate JSON report
      copy:
        content: "{{ hostvars | to_nice_json }}"
        dest: "/tmp/zammad_reports/zammad_inventory_full.json"
      changed_when: false

    - name: Display summary
      debug:
        msg: |
          ========================================
          Zammad Inventory Collection Complete
          ========================================
          Report Location: /tmp/zammad_reports/
          Total Hosts: {{ groups['zammad_servers'] | length }}
          Timestamp: {{ now(utc=True).iso8601 }}
          ========================================
```

## Step 4: Create CSV Template

Create file: `roles/zammad_inventory/templates/zammad_inventory.csv.j2`

```jinja2
Hostname,IP Address,OS,CPU Count,RAM (GB),Zammad Version,Active,Last Updated
{% for host in groups['zammad_servers'] | sort %}
{% set facts = hostvars[host] %}
{{ facts.inventory_hostname }},{{ facts.zammad_info.ip_address }},{{ facts.zammad_info.os }},{{ facts.zammad_info.cpu_count }},{{ facts.zammad_info.ram_gb }},{{ facts.zammad_info.zammad_version }},{{ facts.zammad_info.zammad_active }},{{ facts.zammad_info.last_updated }}
{% endfor %}
```

## Step 5: Test the Playbook Locally

```bash
# Verify syntax
ansible-playbook site.yml --syntax-check -i inventory/hosts.yml

#you need to install ansible first

# Dry-run (see what would happen)
ansible-playbook -i inventory/hosts.yml site.yml --check -v

# Run the playbook
ansible-playbook -i inventory/hosts.yml site.yml -v
```

## Step 6: Create in AWX (Web UI)

### 6.1 Create a New Project

1. Go to **Projects** in AWX
2. Click **Add** button
3. Fill in:
   - **Name**: `Zammad Inventory`
   - **Organization**: Your org
   - **SCM Type**: Git (if using git) or Manual (if uploading files)
   - **SCM URL**: Your git repo URL (or leave empty for Manual)
   - Click **Save**

### 6.2 Create Inventory

1. Go to **Inventories** → Click **Add** → Select **Inventory**
2. Fill in:
   - **Name**: `Zammad Servers`
   - **Organization**: Your org
   - Click **Save**

3. Go to **Sources** tab → Click **Add source**
4. Fill in:
   - **Name**: `Zammad Hosts`
   - **Source**: `Sourced from a Project`
   - **Project**: `Zammad Inventory`
   - **Inventory file**: `inventory/hosts.yml`
   - Click **Save**

5. Click the refresh icon to sync inventory

### 6.3 Create Machine Credential

1. Go to **Credentials** → Click **Add**
2. Select **Machine** as type
3. Fill in:
   - **Name**: `Zammad SSH Key`
   - **Username**: `root`
   - **SSH Private Key**: Paste your private key
   - Click **Save**

### 6.4 Create Job Template

1. Go to **Templates** → Click **Add** → Select **Job Template**
2. Fill in:
   - **Name**: `Zammad Inventory Collector`
   - **Job Type**: `Run`
   - **Inventory**: `Zammad Servers`
   - **Project**: `Zammad Inventory`
   - **Playbook**: `site.yml`
   - **Credential**: `Zammad SSH Key`
   - **Verbosity**: `Verbose`
   - Click **Save**

### 6.5 Launch the Job

1. Click the **Launch** button (rocket icon)
2. Monitor the execution in real-time
3. View the output in the job details page

## Step 7: Schedule the Job (Optional)

1. Go to your Job Template
2. Click **Schedules** tab
3. Click **Add** schedule
4. Fill in:
   - **Name**: `Daily Zammad Inventory`
   - **Run frequency**: `Daily` at specific time
   - Click **Save**

## Step 8: Verify Reports

After running, check the reports:

```bash
# On the AWX server
cat /tmp/zammad_reports/zammad_inventory_full.json | python -m json.tool

# Or in AWX, check job output
```

## Troubleshooting

### Connection Issues
```bash
# Test SSH connection to a host
ssh -i ~/.ssh/id_rsa root@141.52.225.103

# Test Ansible connectivity
ansible -i inventory/hosts.yml zammad_servers -m ping
```

### Syntax Errors
```bash
# Check YAML syntax
yamllint site.yml
```

### No Zammad Version Found
- Make sure zammad service/package is installed
- Check if API is accessible on port 3000
- Review debug output in job

## File Structure Final

```
zammad-inventory-project/
├── site.yml                          # Main playbook
├── inventory/
│   └── hosts.yml                     # Inventory file
├── roles/
│   └── zammad_inventory/
│       ├── tasks/
│       ├── templates/
│       │   └── zammad_inventory.csv.j2
│       └── files/
└── group_vars/
    └── zammad_servers.yml            # (optional) group variables
```

## Next Steps

1. Add all your hosts to `inventory/hosts.yml`
2. Test locally with `ansible-playbook -i inventory/hosts.yml site.yml -v`
3. Create project in AWX
4. Create job template
5. Run and verify
6. Schedule if needed