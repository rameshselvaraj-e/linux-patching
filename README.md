Linux-Patching

 Linux Patching Automation with Ansible & Jenkins

Automated Linux patching solution for **Ubuntu/Debian** and **RedHat/CentOS/Rocky/AlmaLinux** servers using Ansible playbooks orchestrated through a Jenkins CI/CD pipeline.

## Features

- **Pre-patch checks**: OS info, disk space validation, package manager lock detection, pending update count, critical service health, package inventory snapshot
- **OS-aware patching**: Automatic detection of Debian vs RedHat family, uses `apt` or `dnf`/`yum` accordingly
- **Safety controls**: Disk threshold abort, lock detection, serial batching (patch one host at a time), production approval gate
- **Post-patch validation**: Service health re-check, disk space check, failed systemd unit detection, reboot-required check, package diff (pre vs post)
- **Automated reboot**: Optional auto-reboot if kernel updates require it, with configurable timeout
- **Reporting**: Per-host pre-patch and post-patch reports, package diffs, archived in Jenkins
- **Jenkins pipeline**: Parameterized builds with environment selection, dry-run mode, approval gates, report archiving

## Repository Structure

```
linux-patching/
├── ansible.cfg                    # Ansible configuration
├── Jenkinsfile                    # Jenkins pipeline definition
├── README.md                      # This file
├── .gitignore                     # Git ignore rules
├── inventory/
│   ├── hosts.yml                  # Ansible inventory (edit with your hosts)
│   └── group_vars/
│       ├── all.yml                # Global variables (thresholds, services)
│       ├── ubuntu.yml             # Ubuntu-specific variables
│       └── redhat.yml             # RedHat-specific variables
├── playbooks/
│   ├── pre_patch.yml              # Pre-patch: collect state, validate readiness
│   ├── patch_linux.yml            # Patching: apt/dnf update, reboot handling
│   └── post_validate.yml          # Post-patch: validate health, generate reports
├── templates/
│   └── patch_report.j2            # (Reserved for Jinja2 report template)
└── reports/                       # Generated reports (gitignored)
    ├── pre_patch_<host>.txt
    ├── post_patch_<host>.txt
    ├── pre_packages_<host>.txt
    ├── post_packages_<host>.txt
    └── package_diff_<host>.txt
```

## Prerequisites

### Jenkins Agent
1. **Ansible** installed: `pip3 install ansible` (or use `apt install ansible`)
2. **Ansible plugin** for Jenkins (optional, provides syntax highlighting)
3. **SSH key** credential stored in Jenkins with ID `ansible-ssh-key`
4. Jenkins agent must have SSH access to all target servers

### Target Servers
1. SSH access configured for the Jenkins user (or a dedicated `devops` user)
2. Passwordless `sudo` configured for the SSH user
3. Python 3 installed (required by Ansible)

## Setup Instructions

### 1. Clone the Repository

```bash
git clone git@github.com:your-org/linux-patching.git
cd linux-patching
```

### 2. Configure Inventory

Edit `inventory/hosts.yml` with your server details:

```yaml
all:
  children:
    ubuntu:
      hosts:
        ubuntu-server-01:
          ansible_host: 192.168.1.10
          ansible_user: devops
    redhat:
      hosts:
        rhel-server-01:
          ansible_host: 192.168.1.20
          ansible_user: devops
```

### 3. Configure Variables

Edit `inventory/group_vars/all.yml` to customize:
- `disk_threshold_percent` - abort if disk usage exceeds this
- `critical_services` - list of services that must be running
- `auto_reboot` - enable/disable automatic reboot
- `serial_batch` - number of hosts to patch simultaneously

Edit `inventory/group_vars/ubuntu.yml` for Ubuntu-specific settings (autoremove, dist-upgrade).

Edit `inventory/group_vars/redhat.yml` for RedHat-specific settings (dnf vs yum, security-only updates).

### 4. Test Locally (Optional)

Run the playbooks manually before integrating with Jenkins:

```bash
# Dry run (check mode - no changes)
ansible-playbook playbooks/pre_patch.yml -i inventory/hosts.yml --check
ansible-playbook playbooks/patch_linux.yml -i inventory/hosts.yml --check

# Full run on a single host first
ansible-playbook playbooks/pre_patch.yml -i inventory/hosts.yml -l ubuntu-server-01
ansible-playbook playbooks/patch_linux.yml -i inventory/hosts.yml -l ubuntu-server-01
ansible-playbook playbooks/post_validate.yml -i inventory/hosts.yml -l ubuntu-server-01
```

### 5. Push to GitHub

```bash
git add .
git commit -m "Add Linux patching automation with Ansible and Jenkins pipeline"
git push origin main
```

### 6. Configure Jenkins Pipeline

1. Go to **Jenkins** > **New Item**
2. Enter a name (e.g., `linux-patching`)
3. Select **Pipeline** and click OK
4. Under **Pipeline** section:
   - Definition: **Pipeline script from SCM**
   - SCM: **Git**
   - Repository URL: `git@github.com:your-org/linux-patching.git`
   - Branch: `*/main`
   - Script Path: `linux-patching/Jenkinsfile`
5. Under **Credentials**, add an SSH key credential with ID `ansible-ssh-key`
6. Save and click **Build with Parameters**

### 7. Configure Jenkins Credentials

1. Go to **Jenkins** > **Manage Credentials** > **System** > **Global credentials**
2. Add a new credential:
   - Kind: **SSH Username with private key**
   - ID: `ansible-ssh-key`
   - Username: `devops` (or your SSH user)
   - Private Key: Enter directly or upload your SSH private key

## Jenkins Pipeline Parameters

| Parameter | Description | Default |
|-----------|-------------|---------|
| `TARGET_GROUP` | Ansible inventory group to patch | `all` |
| `ENVIRONMENT` | Environment (dev/staging/prod) - affects approval gates | `dev` |
| `AUTO_REBOOT` | Auto-reboot if kernel updates require it | `false` |
| `SERIAL_BATCH` | Hosts to patch in parallel | `1` |
| `DRY_RUN` | Ansible check mode (no changes) | `false` |
| `SKIP_APPROVAL` | Skip manual approval gate | `false` |
| `EXTRA_ANSIBLE_ARGS` | Additional ansible-playbook arguments | (empty) |
| `ANSIBLE_VERSION` | Pin Ansible version | (empty = system default) |

## Pipeline Stages

1. **Checkout** - Pull latest code from Git
2. **Setup Ansible** - Ensure Ansible is installed on the agent
3. **Syntax Check** - Validate playbook syntax
4. **Connectivity Check** - Verify SSH access to all target hosts
5. **Pre-Patch Checks** - Disk space, locks, services, package inventory
6. **Approval Gate** - Manual approval for production (30 min timeout)
7. **Apply Patches** - Run apt/dnf updates, handle reboots
8. **Post-Patch Validation** - Service health, disk, failed units, reports

## Reports

After each pipeline run, the following reports are generated in `reports/`:

- `pre_patch_<host>.txt` - System state before patching
- `post_patch_<host>.txt` - Validation results after patching
- `pre_packages_<host>.txt` - Package list before patching
- `post_packages_<host>.txt` - Package list after patching
- `package_diff_<host>.txt` - Diff of package changes

Reports are archived in Jenkins build artifacts and published as HTML.

## Security Best Practices

- Store SSH keys in Jenkins credentials, never in the repository
- Use Ansible Vault for sensitive variables if needed
- Restrict Jenkins job permissions using `submitter` on approval gates
- Review reports after each patching cycle
- Use `serial_batch: 1` in production to patch one host at a time
- Test in dev/staging before applying to production

## Troubleshooting

| Issue | Solution |
|-------|----------|
| SSH connection failed | Verify SSH key is correct and target hosts are reachable |
| apt/dpkg lock held | Another package operation is running. Wait or kill the process |
| Disk space threshold exceeded | Free up space on the affected filesystem before patching |
| Service not running after patch | Check `systemctl status <service>` and logs in `/var/log/` |
| Reboot required but not done | Enable `AUTO_REBOOT` or reboot manually |
| Ansible module errors | Ensure Python 3 is installed on target hosts |
