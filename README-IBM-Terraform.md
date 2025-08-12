# IBM Cloud VPC Twingate Connector Deployment

This Terraform configuration creates an IBM Cloud VPC virtual server instance running CentOS Stream 9 and automatically installs the Twingate connector on first boot using cloud-init.

## Prerequisites

### 1. IBM Cloud CLI and Authentication
```bash
# Install IBM Cloud CLI
curl -fsSL https://clis.cloud.ibm.com/install/linux | sh

# Login to IBM Cloud
ibmcloud login

# Set target region (optional)
ibmcloud target -r us-east
```

### 2. Terraform Installation
```bash
# Install Terraform (Ubuntu/Debian)
wget -O- https://apt.releases.hashicorp.com/gpg | sudo gpg --dearmor -o /usr/share/keyrings/hashicorp-archive-keyring.gpg
echo "deb [signed-by=/usr/share/keyrings/hashicorp-archive-keyring.gpg] https://apt.releases.hashicorp.com $(lsb_release -cs) main" | sudo tee /etc/apt/sources.list.d/hashicorp.list
sudo apt update && sudo apt install terraform
```

### 3. SSH Key Setup
Create an SSH key in IBM Cloud (required for instance access):
```bash
# Generate SSH key if you don't have one
ssh-keygen -t rsa -b 4096 -C "your_email@example.com"

# Add SSH key to IBM Cloud
ibmcloud is key-create twingate-connector-key @~/.ssh/id_rsa.pub
```

### 4. IBM Cloud API Key
Set up authentication using one of these methods:

#### Option A: Environment Variable (Recommended)
```bash
export IC_API_KEY="your-ibm-cloud-api-key"
```

#### Option B: IBM Cloud CLI Login
```bash
ibmcloud login
```

## Configuration

### 1. Copy and Customize Variables
```bash
# Copy the example variables file
cp terraform.tfvars.example terraform.tfvars

# Edit the variables file with your values
vim terraform.tfvars

# IMPORTANT: Add terraform.tfvars to .gitignore (already included in this repo)
# This prevents accidental commit of sensitive tokens
```

### 2. Required Variables
Update `terraform.tfvars` with your values:
```hcl
# REQUIRED: SSH key name (must exist in IBM Cloud)
ssh_key_name = "twingate-connector-key"

# REQUIRED: Twingate connector tokens (from your tgconnect file)
twingate_access_token = "your-twingate-access-token"
twingate_refresh_token = "your-twingate-refresh-token"

# OPTIONAL: Customize these as needed
region = "us-east"
zone = "us-east-1"
resource_group = "default"
instance_name = "twingate-connector"
instance_profile = "bx2-2x8"
enable_floating_ip = true  # Set to false for private-only access
twingate_network = "mynetwork"  # Your Twingate network name
create_second_vsi = false  # Set to true to create a second VSI without cloud-init
second_instance_name = "second-vsi"  # Name for the second instance
```

#### Getting Twingate Tokens
Extract the tokens from your `tgconnect` file:
- **Access Token**: The value of `TWINGATE_ACCESS_TOKEN`
- **Refresh Token**: The value of `TWINGATE_REFRESH_TOKEN`
- **Network**: The value of `TWINGATE_NETWORK`

#### Complete Setup Example
```bash
# 1. Copy your tgconnect file contents
cat tgconnect

# 2. Extract the tokens (example from your tgconnect file)
export TWINGATE_ACCESS_TOKEN="eyJhbGciOiJFUzI1NiIsImtpZCI6..."
export TWINGATE_REFRESH_TOKEN="_6eWcxuDbyM8rqwfSccQwILAwb5v..."

# 3. Create terraform.tfvars with your values
cat > terraform.tfvars << EOF
ssh_key_name = "twingate-connector-key"
twingate_access_token = "$TWINGATE_ACCESS_TOKEN"
twingate_refresh_token = "$TWINGATE_REFRESH_TOKEN"
twingate_network = "mynetwork"
instance_name = "my-twingate-connector"
EOF

# 4. Deploy with Terraform
terraform init
terraform plan
terraform apply
```

## Deployment

### 1. Initialize Terraform
```bash
terraform init
```

### 2. Plan the Deployment
```bash
# Validate configuration
terraform validate

# Plan the deployment (will prompt for missing variables if not set)
terraform plan
```

### 3. Deploy the Infrastructure
```bash
terraform apply
```

### 4. View Outputs
After deployment, Terraform will output useful information:
```bash
# View all outputs
terraform output

# Get specific outputs
terraform output instance_public_ip
terraform output ssh_command
```

## Accessing the Instance

### SSH Access
Use the SSH command from Terraform output:
```bash
# Get the SSH command
terraform output ssh_command

# Example output: ssh root@169.xx.xx.xx
ssh root@<public_ip>
```

### Check Twingate Installation (Primary VSI)
```bash
# Check installation log
tail -f /var/log/twingate-install.log

# If log file doesn't exist, check cloud-init logs
sudo tail -f /var/log/cloud-init-output.log
sudo tail -f /var/log/cloud-init.log

# Check connector service status
systemctl status twingate-connector

# Check cloud-init status
cloud-init status

# Check if setup script was created
ls -la /opt/twingate-setup.sh

# Manually run setup script if needed
sudo /opt/twingate-setup.sh
```

### Check Podman Installation (Second VSI)
```bash
# Check Podman installation log
tail -f /var/log/podman-setup.log

# Check Podman version and info
podman --version
podman info

# Check Podman service status
systemctl status podman.socket

# Test Podman with hello-world container
podman run --rm hello-world

# Check installed container tools
podman-compose --version
buildah --version
skopeo --version

# Check sample container scripts
ls -la /opt/containers/
/opt/containers/hello-world.sh

# Check rootless container configuration
cat /etc/subuid | grep podman-user
cat /etc/subgid | grep podman-user

# Switch to podman-user and test rootless containers
sudo -u podman-user podman run --rm hello-world
```

## Architecture

This Terraform configuration creates:

### Network Infrastructure
- **VPC**: Isolated virtual network
- **Subnet**: Private subnet with 64 IP addresses
- **Public Gateway**: Internet access for the subnet
- **Security Group**: Firewall rules for SSH and Twingate traffic
- **Floating IP**: Public IP address for external access (enabled by default)

### Compute Resources
- **Primary VSI**: CentOS Stream 9 with Twingate connector and cloud-init automation
- **Second VSI** (optional): CentOS Stream 9 with Podman container runtime and cloud-init automation
- **Cloud-Init**: 
  - Primary instance: Automated Twingate connector installation
  - Second instance: Automated Podman installation and configuration

### Security Groups Rules
- **Inbound SSH (Port 22)**: Access from anywhere
- **Outbound All Traffic**: Unrestricted outbound access to all destinations and ports

## Files Created

- `terraibmvpc.tf` - Main Terraform configuration
- `terraform.tfvars.example` - Example variables file
- `README-IBM-Terraform.md` - This documentation
- `.gitignore` - Prevents sensitive files from being committed to version control

## Operating System

This configuration uses **CentOS Stream 9** (`ibm-centos-stream-9-amd64-11`) which provides:
- Enterprise-grade stability and security
- Full compatibility with Red Hat Enterprise Linux (RHEL)
- Latest features and updates from the CentOS Stream project
- Systemd service management
- DNF package manager (modern replacement for YUM)

The cloud-init script is optimized for CentOS/RHEL using `dnf` for package management.

## Customization

### Instance Profiles
Available IBM Cloud instance profiles:
- `bx2-2x8` - 2 vCPUs, 8 GB RAM (default)
- `bx2-4x16` - 4 vCPUs, 16 GB RAM
- `bx2-8x32` - 8 vCPUs, 32 GB RAM
- `bx2-16x64` - 16 vCPUs, 64 GB RAM

### Regions and Zones
Available regions:
- `us-south` (Dallas) - zones: us-south-1, us-south-2, us-south-3
- `us-east` (Washington DC) - zones: us-east-1, us-east-2, us-east-3
- `eu-gb` (London) - zones: eu-gb-1, eu-gb-2, eu-gb-3
- `eu-de` (Frankfurt) - zones: eu-de-1, eu-de-2, eu-de-3
- `jp-tok` (Tokyo) - zones: jp-tok-1, jp-tok-2, jp-tok-3

### Twingate Configuration
The Twingate connector is configured with:
- **Network**: `mynetwork`
- **Access Token**: From your `tgconnect` file
- **Refresh Token**: From your `tgconnect` file
- **Deployment Label**: `terraform-ibm`

## Troubleshooting

### Common Issues

1. **SSH Key Not Found**
   ```bash
   # List available SSH keys
   ibmcloud is keys
   
   # Create new SSH key
   ibmcloud is key-create my-key @~/.ssh/id_rsa.pub
   ```

2. **Authentication Errors**
   ```bash
   # Check authentication
   ibmcloud target
   
   # Re-login if needed
   ibmcloud login
   ```

3. **Resource Group Issues**
   ```bash
   # List available resource groups
   ibmcloud resource groups
   ```

4. **Cloud-Init Not Running or User-Data Issues**
   
   **Symptoms**: 
   - Missing `/opt/twingate-setup.sh`, no logs in `/var/log/twingate-install.log`
   - Error: "Unhandled non-multipart (text/x-not-multipart) userdata"
   
   **Possible Causes**:
   - Cloud-init service not enabled on the image
   - User-data format not recognized (fixed by removing base64 encoding)
   - YAML formatting issues in cloud-config
   - Terraform variable interpolation problems
   - Cloud-init disabled or failing to start
   
   **Troubleshooting**:
   ```bash
   # Check if cloud-init is installed and enabled
   systemctl status cloud-init
   systemctl is-enabled cloud-init
   
   # Check user-data was received
   sudo cat /var/lib/cloud/instance/user-data.txt
   
   # Check cloud-init logs for any errors
   journalctl -u cloud-init-local --no-pager
   journalctl -u cloud-init --no-pager
   
   # Check if our debug files were created
   ls -la /tmp/cloud-init-*.log
   
   # Verify cloud-init configuration
   cloud-init analyze show
   ```

5. **Twingate Installation Failed**
   ```bash
   # SSH to the instance and check logs
   ssh root@<public_ip>
   
   # Check comprehensive logs
   tail -f /var/log/twingate-install.log
   tail -f /var/log/cloud-init-output.log
   
   # Check if fallback script was created and used
   grep -i fallback /var/log/twingate-install.log
   
   # Manually run setup if needed
   sudo /opt/twingate-setup.sh
   ```

### Debugging Commands

```bash
# Check Terraform state
terraform state list

# Show specific resource
terraform state show ibm_is_instance.twingate_vsi

# Check cloud-init logs on the instance
ssh root@<public_ip> 'tail -f /var/log/cloud-init.log'

# Check system logs
ssh root@<public_ip> 'journalctl -f'
```

### "Unhandled non-multipart userdata" Error

If you see the error "Unhandled non-multipart (text/x-not-multipart) userdata" in cloud-init logs:

**This error has been fixed** in the current configuration by:
- Removing base64 encoding from user_data (IBM Cloud VPC expects plain text)
- Ensuring proper cloud-config YAML format
- Adding cloud-init version logging for debugging

**To verify the fix worked**:
```bash
# Check if cloud-init recognized the format
sudo journalctl -u cloud-init --no-pager | grep -v "non-multipart"

# Check cloud-init status
cloud-init status

# Look for our debug files
ls -la /tmp/cloud-init-*.log
```

### Missing Files Troubleshooting

If `/opt/twingate-setup.sh` or `/var/log/twingate-install.log` are missing:

```bash
# 1. Check cloud-init status and completion
cloud-init status
cloud-init status --wait  # Wait for completion if still running

# 2. Check if cloud-init received user-data
sudo cat /var/lib/cloud/instance/user-data.txt | head -20

# 3. Check cloud-init logs for errors
tail -100 /var/log/cloud-init.log | grep -i error
tail -100 /var/log/cloud-init-output.log

# 4. Check debug files created by the new configuration
ls -la /tmp/cloud-init-debug.log /tmp/cloud-init-runcmd.log
cat /tmp/cloud-init-debug.log 2>/dev/null || echo "Debug log not found"
cat /tmp/cloud-init-runcmd.log 2>/dev/null || echo "Runcmd log not found"

# 5. Verify if write_files section worked
ls -la /opt/twingate-setup.sh
cat /opt/twingate-setup.sh 2>/dev/null || echo "Setup script not found"

# 6. Check cloud-init stages that ran
grep -E "write_files|runcmd" /var/log/cloud-init.log

# 7. If script is missing, check if fallback was created
grep "fallback" /var/log/twingate-install.log 2>/dev/null || echo "No fallback info found"

# 8. Manual troubleshooting - create and run script manually
if [ ! -f /opt/twingate-setup.sh ]; then
  echo "Creating manual setup script..."
  sudo mkdir -p /opt
  sudo cat > /opt/manual-twingate-setup.sh << 'EOF'
#!/bin/bash
dnf update -y
dnf install -y curl wget
export TWINGATE_ACCESS_TOKEN="your-token-here"
export TWINGATE_REFRESH_TOKEN="your-refresh-token-here"  
export TWINGATE_NETWORK="mynetwork"
curl -fsSL "https://binaries.twingate.com/connector/setup.sh" | bash
EOF
  sudo chmod +x /opt/manual-twingate-setup.sh
  echo "Manual script created. Edit tokens and run: sudo /opt/manual-twingate-setup.sh"
fi

# 9. Check cloud-init modules that ran
cloud-init analyze show

# 10. Re-run cloud-init if needed (be careful, this may have side effects)
# sudo cloud-init clean
# sudo cloud-init init
# sudo cloud-init modules --mode=config
# sudo cloud-init modules --mode=final
```

## Cleanup

To destroy all resources:
```bash
terraform destroy
```

## Cost Estimation

Estimated monthly costs (US East region):
- Primary Virtual Server Instance (bx2-2x8): ~$30-40/month
- Second Virtual Server Instance (if enabled): ~$30-40/month
- Floating IP(s): ~$5/month each
- VPC resources: Minimal cost
- **Total**: 
  - Single VSI: ~$35-45/month
  - Two VSIs with floating IPs: ~$70-85/month

## Security Considerations

1. **SSH Access**: Restricted to port 22, consider limiting source IPs
2. **Firewall**: Only necessary ports are open
3. **Updates**: Instance will auto-update packages on first boot
4. **Secrets Management**: Twingate tokens are now stored as Terraform variables

### Handling Sensitive Variables

#### Option 1: Environment Variables (Recommended for CI/CD)
```bash
export TF_VAR_twingate_access_token="your-access-token"
export TF_VAR_twingate_refresh_token="your-refresh-token"
terraform apply
```

#### Option 2: Separate tfvars file for secrets
```bash
# Create a separate file for sensitive values
echo 'twingate_access_token = "your-token"' > secrets.tfvars
echo 'twingate_refresh_token = "your-token"' >> secrets.tfvars

# Apply with multiple tfvars files
terraform apply -var-file="terraform.tfvars" -var-file="secrets.tfvars"

# Add secrets.tfvars to .gitignore
echo "secrets.tfvars" >> .gitignore
```

#### Option 3: Interactive input
```bash
# Terraform will prompt for missing variables
terraform apply
```

#### Option 4: IBM Secret Manager (Production)
For production deployments, consider using IBM Secret Manager to store tokens securely.

## Support

For issues with:
- **Terraform**: Check Terraform documentation
- **IBM Cloud**: Contact IBM Cloud support
- **Twingate**: Contact Twingate support

## Advanced Configuration

### Using IBM Secret Manager
For production deployments, consider storing Twingate tokens in IBM Secret Manager:

```hcl
# Add to terraibmvpc.tf

# Data sources to read secrets from IBM Secret Manager
data "ibm_sm_secret" "twingate_access_token" {
  instance_id = "your-secret-manager-instance-id"
  secret_id   = "twingate-access-token"
}

data "ibm_sm_secret" "twingate_refresh_token" {
  instance_id = "your-secret-manager-instance-id"
  secret_id   = "twingate-refresh-token"
}

# Update the variables to use the secrets
locals {
  twingate_access_token  = data.ibm_sm_secret.twingate_access_token.secret_data
  twingate_refresh_token = data.ibm_sm_secret.twingate_refresh_token.secret_data
}

# Use locals in the cloud-init script instead of var.twingate_*_token
```

### Custom Cloud-Init
You can modify the cloud-init configuration in the `locals` block to add additional setup steps or configurations.

### Floating IP Configuration
The VSI is configured with a floating IP (public IP address) by default for external access:

```hcl
# Enable floating IP (default: true)
enable_floating_ip = true

# Disable floating IP for private-only access
enable_floating_ip = false
```

**With Floating IP Enabled (default):**
- VSI gets a public IP address for external access
- SSH access from the internet
- Twingate connector can be managed remotely
- Additional cost: ~$5/month for the floating IP

**With Floating IP Disabled:**
- VSI only has private IP address
- Access only through VPN, bastion host, or IBM Cloud private network
- Lower cost (no floating IP charges)
- More secure (no direct internet access)

The Terraform outputs will automatically adjust based on your floating IP setting.

### Second VSI Configuration
You can optionally create a second VSI with Podman container runtime:

```hcl
# Enable second VSI (default: false)
create_second_vsi = true
second_instance_name = "podman-server"
```

**With Second VSI Enabled:**
- Creates an additional CentOS Stream 9 instance
- Uses the same VPC, subnet, and security group
- Same SSH key and instance profile as the first VSI
- **Automated Podman installation** via cloud-init user data
- Podman, Podman-Compose, Buildah, and Skopeo pre-installed
- Rootless container support configured
- Optional floating IP (follows the enable_floating_ip setting)
- Separate outputs for second instance details

**Podman Features Installed:**
- **Podman**: Container runtime (Docker alternative)
- **Podman-Compose**: Docker Compose equivalent
- **Buildah**: Container image building tool
- **Skopeo**: Container image management tool
- **Rootless containers**: Secure container execution without root
- **Sample container scripts**: Ready-to-use examples in `/opt/containers/`

**Use Cases for Podman VSI:**
- Container development and testing environment
- Microservices deployment platform
- CI/CD container runner
- Application containerization testing
- Docker alternative evaluation 