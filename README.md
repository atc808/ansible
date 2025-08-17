# VMware CD/DVD Device Management Playbooks

## Purpose

This Ansible project provides automated management of CD/DVD devices across VMware ESXi virtual machines. It can discover VMs with mounted ISO files or connected CD/DVD drives and optionally disconnect them to prevent conflicts during operations like vMotion, snapshots, or maintenance activities.

## Key Features

- **Discovery Mode**: Scan ESXi hosts for VMs with connected CD/DVD devices
- **Dry Run Mode**: Preview what changes would be made without executing them
- **Enhanced Disconnect**: Two-step process that removes drives with media and adds clean drives
- **Force Disconnect**: Bypass guest OS locks that cause dialog prompts
- **VM Blacklisting**: Exclude critical VMs from operations
- **Comprehensive Reporting**: Detailed success/failure reporting with error handling
- **Connection Validation**: Test ESXi connectivity before operations

## Playbook Hierarchy

```
cdmounted.yml                    # Main playbook entry point
├── tasks/
│   ├── gather_vm_info.yml      # ESXi connection and VM discovery
│   ├── process_vm_hardware.yml  # Individual VM hardware querying
│   ├── find_cd_devices.yml     # CD/DVD device detection logic
│   ├── display_results.yml     # Format and display discovered devices
│   ├── disconnect_cd_devices.yml # Enhanced disconnect operations
│   └── display_summary.yml     # Final operation summary
```

## Parameters

### Required Variables
| Variable | Type | Description | Example |
|----------|------|-------------|---------|
| `ansible_user` | string | vCenter/ESXi username | `administrator@vsphere.local` |
| `ansible_password` | string | vCenter/ESXi password | `SecurePassword123` |
| `inventory_hostname` | string | ESXi host FQDN/IP | `esxi-host01.example.com` |

### Control Variables
| Variable | Type | Default | Description |
|----------|------|---------|-------------|
| `disconnect` | boolean | `true` | Enable/disable disconnect operations |
| `dry_run` | boolean | `false` | Preview mode - show what would change |
| `force_disconnect` | boolean | `true` | Bypass guest OS CD lock dialogs |
| `vm_blacklist` | list | `[]` | VM names to exclude from operations |

### Internal Variables (Auto-managed)
| Variable | Type | Description |
|----------|------|-------------|
| `vms_with_cd_media` | list | VMs discovered with connected CD devices |
| `failed_vm_queries` | list | VMs that failed hardware queries |
| `successful_removals` | list | Successfully removed CD drives |
| `failed_removals` | list | Failed CD drive removals |
| `successful_additions` | list | Successfully added clean CD drives |
| `failed_additions` | list | Failed clean CD drive additions |

## Usage Examples

### Discovery Only (Safe Mode)
```bash
# Scan for VMs with CD devices without making changes
ansible-playbook -i inventory cdmounted.yml -e "disconnect=false"
```

### Dry Run Mode
```bash
# Preview what would be disconnected
ansible-playbook -i inventory cdmounted.yml -e "dry_run=true"
```

### Full Disconnect Operation
```bash
# Disconnect all found CD devices
ansible-playbook -i inventory cdmounted.yml
```

### With VM Blacklist
```bash
# Exclude critical VMs from operations
ansible-playbook -i inventory cdmounted.yml \
  -e 'vm_blacklist=["critical-vm-01","production-db"]'
```

### Disable Force Disconnect
```bash
# Allow guest OS to show lock dialogs (may cause hanging)
ansible-playbook -i inventory cdmounted.yml -e "force_disconnect=false"
```

## Operation Modes

### 1. Discovery Mode (`disconnect=false`)
- Scans ESXi hosts for VMs with connected CD/DVD devices
- Reports findings without making changes
- Useful for inventory and planning

### 2. Dry Run Mode (`dry_run=true`)
- Shows exactly what changes would be made
- No actual modifications performed
- Provides operation plan with device details

### 3. Live Mode (`disconnect=true`, `dry_run=false`)
- Performs actual disconnect operations
- Two-step enhanced process:
  1. Remove existing CD drives with media
  2. Add clean CD drives without media
- Comprehensive success/failure reporting

## Enhanced Disconnect Process

The playbook uses a two-step approach to avoid common VMware issues:

1. **Step 1: Remove CD drives with media**
   - Completely removes the virtual CD drive
   - Eliminates media locks and conflicts
   - Handles both IDE and SATA controllers

2. **Step 2: Add clean CD drives**
   - Adds new CD drive without media attached
   - Preserves VM functionality for future use
   - Ready for clean media mounting

## Device Detection Logic

### Supported Controllers
- **IDE Controllers**: Device keys < 15000
- **SATA Controllers**: Device keys >= 15000

### Detection Criteria
- Device label contains "CD/DVD", "CD", "DVD", or "cdrom"
- Device has connectable properties
- Either connected=true OR has backing file (ISO) attached

### Media Type Classification
- **ISO File**: Backing with fileName property
- **Physical Drive**: Backing with deviceName property  
- **Connected**: Other connected states

## Error Handling

### Connection Failures
- Tests ESXi connectivity before operations
- Validates credentials and host accessibility
- Provides clear error messages for authentication issues

### VM Query Failures
- Individual VM failures don't stop entire operation
- Failed queries tracked in `failed_vm_queries` list
- Detailed error reporting in summary

### Disconnect Operation Failures
- Retry logic with 3 attempts and 5-second delays
- Separate tracking of removal vs addition failures
- Manual intervention guidance for failed operations

## Security Considerations

### Credential Management
- Uses Ansible Automation Platform credential injection
- Supports external credential management systems
- No hardcoded passwords in playbooks

### VM Blacklisting
- Case-sensitive VM name matching
- Prevents accidental operations on critical systems
- Supports flexible exclusion lists

### Force Disconnect
- Enabled by default to prevent dialog hanging
- Can be disabled for environments requiring guest consent
- Bypasses guest OS CD lock mechanisms

## Prerequisites

### Ansible Requirements
- Ansible 2.9 or higher
- `community.vmware` collection
- Python `pyvmomi` library

### VMware Requirements
- ESXi 6.5 or higher
- vCenter Server (optional but recommended)
- Account with VM modification privileges

### Network Requirements
- Ansible control node connectivity to ESXi hosts
- HTTPS (443) access to vCenter/ESXi
- DNS resolution for ESXi hostnames

## Troubleshooting

### Common Issues

**Connection Timeouts**
- Verify network connectivity and DNS resolution
- Check firewall rules for port 443
- Validate credentials and permissions

**Device Not Found**
- Ensure VMs have CD/DVD devices configured
- Check if devices are actually connected/mounted
- Verify device naming conventions

**Disconnect Failures**
- VM may be in use by other operations
- Guest OS may have file locks on media
- Enable `force_disconnect` to bypass locks

**Permission Errors**
- Verify account has "Virtual machine.Configuration.Modify device settings"
- Check ESXi host or vCenter permissions
- Ensure account is not expired or locked

### Debug Mode
Add `-vvv` to ansible-playbook commands for detailed debugging:
```bash
ansible-playbook -i inventory cdmounted.yml -vvv
```

## Best Practices

1. **Always test in dry run mode first**
2. **Use VM blacklists for critical systems**
3. **Schedule during maintenance windows**
4. **Monitor ESXi host performance during operations**
5. **Keep credential management secure**
6. **Review logs for any failed operations**

## Version History

- **v1.0**: Initial release with basic disconnect functionality
- **v2.0**: Enhanced two-step disconnect process
- **v2.1**: Added force disconnect and improved error handling
- **v2.2**: Comprehensive documentation and parameter validation