# VMware CD/DVD Media Management Playbook

A comprehensive Ansible playbook for discovering and managing CD/DVD devices with connected media across VMware ESXi environments. This playbook addresses common issues with stuck ISO files, mounted media conflicts, and provides automated cleanup capabilities.

## Table of Contents

- [Overview](#overview)
- [Features](#features)
- [Architecture](#architecture)
- [Prerequisites](#prerequisites)
- [Installation](#installation)
- [Configuration](#configuration)
- [Usage](#usage)
- [How It Works](#how-it-works)
- [VMware Limitations](#vmware-limitations)
- [Troubleshooting](#troubleshooting)
- [Contributing](#contributing)
- [License](#license)

## Overview

This playbook solves a common VMware administration challenge: identifying and managing VMs that have CD/DVD drives with connected media (typically ISO files). It provides both discovery and remediation capabilities with comprehensive safety features.

### Key Problems Solved

- **Stuck ISO Files**: Automatically disconnect ISOs that can't be manually ejected
- **Media Conflicts**: Prevent conflicts when mounting new media
- **Inventory Management**: Discover all VMs with connected CD media across hosts
- **Compliance**: Ensure VMs don't have unnecessary media attached
- **Automation**: Reduce manual vSphere Client interactions

## Features

### 🔍 Discovery Capabilities
- Scans all VMs on target ESXi hosts
- Identifies CD/DVD drives with connected media
- Supports both IDE and SATA controllers
- Detects ISO files, physical drives, and other media types
- Provides detailed device mapping (controller/unit numbers)

### 🛠️ Management Operations
- **Strategy 1**: Media disconnection for IDE drives on powered-on VMs
- **Strategy 2**: Complete drive removal and clean replacement
- Handles VMware hot-removal limitations intelligently
- Force disconnect bypasses guest OS locks

### 🛡️ Safety Features
- VM blacklist protection for critical systems
- Dry-run mode for planning and validation
- Comprehensive error handling and retry logic
- Detailed operation reporting and logging

### 📊 Reporting
- Pre-operation discovery reports
- Real-time operation progress
- Success/failure categorization
- Final summary with recommendations

## Architecture

### File Structure
```
├── cdmounted.yml                    # Main playbook
└── tasks/
    ├── gather_vm_info.yml          # ESXi connection and VM discovery
    ├── process_vm_hardware.yml     # Individual VM hardware queries
    ├── find_cd_devices.yml         # CD/DVD device detection logic
    ├── display_results.yml         # Results presentation
    ├── disconnect_cd_devices.yml   # Main disconnect operations
    └── display_summary.yml         # Final reporting
```

### Component Overview

| Component | Purpose | Key Functions |
|-----------|---------|---------------|
| **Main Playbook** | Orchestration and flow control | Variable management, task sequencing, safety checks |
| **VM Discovery** | ESXi connectivity and inventory | Connection testing, VM enumeration, error handling |
| **Hardware Processing** | Individual VM analysis | Hardware queries, device enumeration, failure tracking |
| **Device Detection** | CD/DVD identification | Device key mapping, controller analysis, media detection |
| **Disconnect Operations** | Media/drive management | Strategy execution, VMware API calls, result categorization |
| **Reporting** | Information presentation | Progress updates, success/failure reporting, summaries |

## Prerequisites

### Software Requirements
- **Ansible**: 2.9+ (tested with 2.15+)
- **Python**: 3.6+
- **Ansible Collections**:
  - `community.vmware` (required for VMware modules)

### VMware Environment
- **ESXi**: 6.5+ (tested with 7.0+)
- **Permissions**: VM configuration and power management rights
- **Network**: Ansible control node must reach ESXi hosts

### Ansible Automation Platform (AAP)
- AAP 2.0+ for credential management
- Custom credential types support (recommended)

## Installation

### 1. Install Required Collections
```bash
ansible-galaxy collection install community.vmware
```

### 2. Clone Repository
```bash
git clone <repository-url>
cd vmware-cd-management
```

### 3. Verify Dependencies
```bash
ansible-doc community.vmware.vmware_guest
```

## Configuration

### Inventory Setup

Create an inventory file with your ESXi hosts:

```ini
[esxi_hosts]
esxi1.example.com
esxi2.example.com
esxi3.example.com

[esxi_hosts:vars]
ansible_connection=local
```

### Credential Configuration

#### Option 1: AAP Custom Credential Type (Recommended)

1. **Create Custom Credential Type in AAP:**
```yaml
# Input Configuration
fields:
  - id: vmware_host
    type: string
    label: VMware Host
  - id: vmware_username
    type: string
    label: VMware Username
  - id: vmware_password
    type: string
    label: VMware Password
    secret: true

# Injector Configuration
extra_vars:
  vmware_host: '{{ vmware_host }}'
  vmware_username: '{{ vmware_username }}'
  vmware_password: '{{ vmware_password }}'
```

2. **Update playbook variables:**
```yaml
vcenter_hostname: "{{ vmware_host }}"
vcenter_username: "{{ vmware_username }}"
vcenter_password: "{{ vmware_password }}"
```

#### Option 2: Environment Variables
```yaml
vcenter_hostname: "{{ lookup('env', 'VMWARE_HOST') }}"
vcenter_username: "{{ lookup('env', 'VMWARE_USER') }}"
vcenter_password: "{{ lookup('env', 'VMWARE_PASSWORD') }}"
```

#### Option 3: Machine Credentials
```yaml
vcenter_hostname: "{{ inventory_hostname }}"
vcenter_username: "{{ ansible_user }}"
vcenter_password: "{{ ansible_password }}"
```

### Variable Configuration

Key variables in `cdmounted.yml`:

```yaml
# Operation Control
disconnect: true              # Enable/disable disconnect operations
dry_run: false               # Show planned operations without execution
force_disconnect: true       # Bypass guest OS CD locks

# VM Protection
vm_blacklist: []            # VMs to exclude from operations
# Example: ["critical-vm-01", "production-db"]
```

## Usage

### Basic Discovery (Safe Mode)
```bash
# Discover VMs with connected CD media without making changes
ansible-playbook -i inventory cdmounted.yml -e "disconnect=false"
```

### Dry Run Mode
```bash
# Show what would be done without execution
ansible-playbook -i inventory cdmounted.yml -e "dry_run=true"
```

### Full Execution
```bash
# Execute disconnect operations
ansible-playbook -i inventory cdmounted.yml
```

### Target Specific Hosts
```bash
# Run against specific ESXi host
ansible-playbook -i inventory cdmounted.yml --limit esxi1.example.com
```

### With VM Blacklist
```bash
# Protect specific VMs from operations
ansible-playbook -i inventory cdmounted.yml -e '{"vm_blacklist": ["critical-vm", "production-db"]}'
```

## How It Works

### Discovery Phase

1. **Connection Testing**: Validates credentials and connectivity to each ESXi host
2. **VM Enumeration**: Discovers all VMs on the target host
3. **Hardware Analysis**: Queries detailed hardware configuration for each VM
4. **Device Detection**: Identifies CD/DVD drives with connected media using VMware device keys

### Device Key Mapping

The playbook uses VMware's device key system to identify controllers and units:

| Device Key Range | Controller Type | Controller Number | Unit Number |
|------------------|----------------|-------------------|-------------|
| 3000 | IDE | 0 | 0 |
| 3001 | IDE | 0 | 1 |
| 3002 | IDE | 1 | 0 |
| 3003 | IDE | 1 | 1 |
| 15000, 16000 | SATA | 0 | 0 |
| 15001, 16001 | SATA | 0 | 1 |

### Operation Strategies

#### Strategy 1: Media Disconnect (IDE + Powered-On VMs)
- **When**: IDE controllers on powered-on VMs
- **Why**: VMware limitation - IDE drives don't support hot-removal
- **Action**: Set CD drive type to "none" (eject media, keep drive)
- **Result**: Drive remains available for future use

#### Strategy 2: Drive Replacement (SATA or Powered-Off VMs)
- **When**: SATA controllers or any powered-off VM
- **Why**: Full hot-removal supported
- **Action**: 
  1. Remove CD drive completely
  2. Add clean replacement drive without media
- **Result**: Clean state ready for new media

### Error Handling

- **Connection Failures**: Retry logic with exponential backoff
- **Individual VM Failures**: Logged but don't stop overall operation
- **Partial Failures**: Detailed reporting for manual intervention
- **API Timeouts**: Configurable retry attempts with delays

## VMware Limitations

### IDE Controller Limitations
- **Hot-Removal**: Not supported on powered-on VMs
- **Workaround**: Media disconnect only (Strategy 1)
- **Impact**: Drive remains but media is ejected

### SATA Controller Capabilities
- **Hot-Removal**: Fully supported
- **Full Management**: Complete remove/replace operations possible

### Guest OS Integration
- **CD Locks**: Some guest OSes lock CD drives
- **Solution**: Force disconnect bypasses guest locks
- **Risk**: May cause guest OS warnings

## Troubleshooting

### Common Issues

#### Authentication Failures
```
Error: Failed to connect to ESXi host
```
**Solutions**:
- Verify credentials in AAP
- Check network connectivity
- Validate ESXi host FQDN/IP
- Ensure user has required permissions

#### Device Detection Issues
```
Found 0 connected CD devices
```
**Solutions**:
- Verify VMs actually have connected media
- Check device labels (might not match expected patterns)
- Review hardware query results in debug output

#### Hot-Removal Failures
```
CD-ROM attach to IDE controller not support hot-remove
```
**Expected Behavior**: This is normal for IDE drives on powered-on VMs
**Solution**: Use Strategy 1 (media disconnect only)

#### Permission Errors
```
Operation failed: Insufficient privileges
```
**Solutions**:
- Grant "Virtual machine configuration" privileges
- Add "Virtual machine power management" if needed
- Use an administrative account

### Debug Mode

Enable verbose output for troubleshooting:
```bash
ansible-playbook -i inventory cdmounted.yml -vvv
```

### Log Analysis

Key debug information to examine:
- Connection test results
- VM hardware query responses
- Device detection logic output
- Operation result categorization

## Advanced Configuration

### Custom Device Detection

Modify device detection patterns in `find_cd_devices.yml`:
```yaml
# Add custom device label patterns
- ('CD/DVD' in device.deviceInfo.label or 
   'CD' in device.deviceInfo.label or 
   'DVD' in device.deviceInfo.label or 
   'cdrom' in device.deviceInfo.label.lower() or
   'optical' in device.deviceInfo.label.lower())  # Custom addition
```

### Extended Blacklist Management

Use group variables for environment-specific blacklists:
```yaml
# group_vars/production.yml
vm_blacklist:
  - "prod-database-01"
  - "critical-app-server"
  - "backup-appliance"

# group_vars/development.yml
vm_blacklist: []  # No restrictions in dev
```

### Integration with ITSM

Add notification tasks for enterprise integration:
```yaml
- name: Create ITSM ticket for failed operations
  uri:
    url: "{{ itsm_api_url }}/tickets"
    method: POST
    body_format: json
    body:
      title: "VMware CD/DVD Management Failures"
      description: "{{ failed_operations | length }} VMs require manual intervention"
  when: failed_operations | length > 0
```

## Performance Considerations

### Large Environments
- **Parallel Execution**: Use `forks` to control concurrency
- **Batching**: Process hosts in smaller groups if needed
- **Timeouts**: Adjust retry counts for slow environments

### Network Optimization
- **Connection Reuse**: Playbook maintains persistent connections
- **Minimal Data**: Only required VM attributes are queried
- **Retry Logic**: Handles temporary network issues gracefully

## Security Best Practices

### Credential Management
- **Never** store credentials in playbooks or variables files
- Use AAP credential injection or secure vaults
- Implement credential rotation policies
- Audit credential usage through AAP logging

### Access Control
- Grant minimal required VMware permissions
- Use dedicated service accounts
- Implement network access controls
- Monitor ESXi access logs

### Audit Trail
- AAP provides complete execution logging
- VMware vCenter logs all API calls
- Enable detailed logging for compliance requirements

## Contributing

### Development Setup
1. Fork the repository
2. Create a feature branch
3. Install development dependencies
4. Run tests against lab environment

### Code Standards
- Follow Ansible best practices
- Add comprehensive comments for complex logic
- Update documentation for new features
- Test against multiple VMware versions

### Testing
- Test with both IDE and SATA configurations
- Verify behavior on powered-on and powered-off VMs
- Test error conditions and edge cases
- Validate against different ESXi versions

## License

[Add your license information here]

## Support

For issues and questions:
- Check the troubleshooting section above
- Review VMware documentation for API limitations
- Consult Ansible community.vmware collection docs
- Open GitHub issues for bugs or feature requests

---

**⚠️ Important**: Always test in a non-production environment first. This playbook makes direct changes to VM configurations and should be used with appropriate caution and approval processes.