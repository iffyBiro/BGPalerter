# Deploying BGPalerter on Microsoft Azure

This guide provides comprehensive instructions for deploying BGPalerter on Microsoft Azure using security best practices. It covers both basic deployment and enterprise-grade security hardening suitable for production environments.

## Table of Contents

1. [Overview](#overview)
2. [Prerequisites](#prerequisites)
3. [Architecture Overview](#architecture-overview)
4. [Phase 1: Basic Deployment](#phase-1-basic-deployment)
5. [Phase 2: Security Hardening](#phase-2-security-hardening)
6. [Phase 3: Monitoring and Maintenance](#phase-3-monitoring-and-maintenance)
7. [Cost Analysis](#cost-analysis)
8. [Troubleshooting](#troubleshooting)

## Overview

BGPalerter can be deployed on Azure using several approaches. This guide focuses on **Azure Container Instances (ACI)** as the most cost-effective solution, enhanced with comprehensive security measures for production use.

### BGPalerter Requirements
- **Memory**: 4GB RAM minimum (with swap enabled recommended)
- **Storage**: Persistent storage for configuration files and logs
- **Network**: Outbound access to RIPE RIS WebSocket (ws://ris-live.ripe.net)
- **Docker**: Uses official `nttgin/bgpalerter:latest` image
- **Health Check**: Built-in endpoint on port 8011

### Security Considerations
BGPalerter processes sensitive network routing data and generates security alerts, making it a critical infrastructure component that requires proper hardening against:
- BGP data manipulation attacks
- Configuration tampering
- Alert suppression
- Network reconnaissance
- Service disruption

## Prerequisites

### Azure Account Setup
1. Create an Azure account at [azure.microsoft.com](https://azure.microsoft.com)
2. Access the Azure Portal at [portal.azure.com](https://portal.azure.com)
3. Install Azure CLI (optional but recommended):
   ```bash
   curl -sL https://aka.ms/InstallAzureCLIDeb | sudo bash
   az login
   ```

### Required Permissions
Your Azure account needs the following roles:
- **Contributor** on the subscription or resource group
- **Key Vault Administrator** for secrets management
- **Network Contributor** for networking configuration

## Architecture Overview

### Secure Architecture Components

```
Internet → Azure Front Door → Private Endpoint → Container Instance
                                      ↓
                              Azure Key Vault (secrets)
                                      ↓
                              Azure File Share (persistent storage)
                                      ↓
                              Log Analytics (monitoring)
```

### Network Security
- **Private Virtual Network**: Isolated network environment
- **Network Security Groups**: Firewall rules for traffic control
- **Private Endpoints**: Secure access to Azure services
- **No Public IP**: Container accessible only through private network

## Phase 1: Basic Deployment

### Step 1: Create Resource Group

```bash
az group create \
  --name bgpalerter-rg \
  --location eastus
```

Or via Azure Portal:
1. Navigate to **Resource groups** → **Create**
2. **Resource group name**: `bgpalerter-rg`
3. **Region**: Choose closest to your location
4. Click **Review + create** → **Create**

### Step 2: Create Virtual Network

```bash
az network vnet create \
  --resource-group bgpalerter-rg \
  --name bgpalerter-vnet \
  --address-prefix 10.0.0.0/16 \
  --subnet-name bgpalerter-subnet \
  --subnet-prefix 10.0.1.0/24
```

### Step 3: Set Up Secure Storage

#### Create Storage Account
```bash
az storage account create \
  --name bgpalerterstorageXXXX \
  --resource-group bgpalerter-rg \
  --location eastus \
  --sku Standard_LRS \
  --encryption-services blob file \
  --https-only true \
  --min-tls-version TLS1_2
```

#### Create File Share
```bash
az storage share create \
  --name bgpalerter-data \
  --account-name bgpalerterstorageXXXX \
  --quota 5
```

### Step 4: Configure Network Security

#### Create Network Security Group
```bash
az network nsg create \
  --resource-group bgpalerter-rg \
  --name bgpalerter-nsg

# Allow outbound to RIPE RIS (HTTPS)
az network nsg rule create \
  --resource-group bgpalerter-rg \
  --nsg-name bgpalerter-nsg \
  --name AllowRIPEHTTPS \
  --protocol Tcp \
  --direction Outbound \
  --priority 100 \
  --source-address-prefix 10.0.1.0/24 \
  --destination-address-prefix Internet \
  --destination-port-range 443 \
  --access Allow

# Allow outbound to RIPE RIS (WebSocket)
az network nsg rule create \
  --resource-group bgpalerter-rg \
  --nsg-name bgpalerter-nsg \
  --name AllowRIPEWebSocket \
  --protocol Tcp \
  --direction Outbound \
  --priority 110 \
  --source-address-prefix 10.0.1.0/24 \
  --destination-address-prefix Internet \
  --destination-port-range 80 \
  --access Allow

# Deny all other outbound traffic
az network nsg rule create \
  --resource-group bgpalerter-rg \
  --nsg-name bgpalerter-nsg \
  --name DenyAllOutbound \
  --protocol "*" \
  --direction Outbound \
  --priority 4000 \
  --source-address-prefix "*" \
  --destination-address-prefix "*" \
  --destination-port-range "*" \
  --access Deny

# Associate NSG with subnet
az network vnet subnet update \
  --resource-group bgpalerter-rg \
  --vnet-name bgpalerter-vnet \
  --name bgpalerter-subnet \
  --network-security-group bgpalerter-nsg
```

### Step 5: Deploy Container Instance

```bash
az container create \
  --resource-group bgpalerter-rg \
  --name bgpalerter \
  --image nttgin/bgpalerter:latest \
  --cpu 1 \
  --memory 4 \
  --restart-policy OnFailure \
  --vnet bgpalerter-vnet \
  --subnet bgpalerter-subnet \
  --azure-file-volume-account-name bgpalerterstorageXXXX \
  --azure-file-volume-account-key $(az storage account keys list --resource-group bgpalerter-rg --account-name bgpalerterstorageXXXX --query "[0].value" --output tsv) \
  --azure-file-volume-share-name bgpalerter-data \
  --azure-file-volume-mount-path /opt/bgpalerter/volume \
  --command-line "npm run serve -- --d /opt/bgpalerter/volume/" \
  --environment-variables TZ=UTC
```

## Phase 2: Security Hardening

### Step 1: Implement Azure Key Vault

#### Create Key Vault
```bash
az keyvault create \
  --name bgpalerter-kv-XXXX \
  --resource-group bgpalerter-rg \
  --location eastus \
  --enable-soft-delete true \
  --enable-purge-protection true \
  --network-acls-default-action Deny
```

#### Store Secrets
```bash
# Email credentials
az keyvault secret set \
  --vault-name bgpalerter-kv-XXXX \
  --name "smtp-password" \
  --value "your-email-app-password"

# Slack webhook
az keyvault secret set \
  --vault-name bgpalerter-kv-XXXX \
  --name "slack-webhook-url" \
  --value "https://hooks.slack.com/services/..."

# API keys for external services
az keyvault secret set \
  --vault-name bgpalerter-kv-XXXX \
  --name "webhook-api-key" \
  --value "your-api-key"
```

### Step 2: Enable Managed Identity

```bash
# Enable system-assigned managed identity
az container update \
  --resource-group bgpalerter-rg \
  --name bgpalerter \
  --assign-identity [system]

# Get the managed identity principal ID
CONTAINER_IDENTITY=$(az container show \
  --resource-group bgpalerter-rg \
  --name bgpalerter \
  --query identity.principalId -o tsv)

# Grant Key Vault access
az keyvault set-policy \
  --name bgpalerter-kv-XXXX \
  --object-id $CONTAINER_IDENTITY \
  --secret-permissions get list

# Grant storage access
az role assignment create \
  --assignee $CONTAINER_IDENTITY \
  --role "Storage File Data SMB Share Contributor" \
  --scope "/subscriptions/$(az account show --query id -o tsv)/resourceGroups/bgpalerter-rg/providers/Microsoft.Storage/storageAccounts/bgpalerterstorageXXXX"
```

### Step 3: Configure Private Container Registry

#### Create Azure Container Registry
```bash
az acr create \
  --resource-group bgpalerter-rg \
  --name bgpalerterregXXXX \
  --sku Premium \
  --admin-enabled false
```

#### Enable Security Scanning
```bash
# Enable Microsoft Defender for Cloud
az security auto-provisioning-setting update \
  --name default \
  --auto-provision on
```

#### Pull and Push Secured Image
```bash
# Login to registry
az acr login --name bgpalerterregXXXX

# Import official image
az acr import \
  --name bgpalerterregXXXX \
  --source docker.io/nttgin/bgpalerter:latest \
  --image bgpalerter:latest

# Update container to use private registry
az container update \
  --resource-group bgpalerter-rg \
  --name bgpalerter \
  --image bgpalerterregXXXX.azurecr.io/bgpalerter:latest
```

### Step 4: Implement Private Endpoints

#### Storage Private Endpoint
```bash
az network private-endpoint create \
  --resource-group bgpalerter-rg \
  --name bgpalerter-storage-pe \
  --vnet-name bgpalerter-vnet \
  --subnet bgpalerter-subnet \
  --private-connection-resource-id "/subscriptions/$(az account show --query id -o tsv)/resourceGroups/bgpalerter-rg/providers/Microsoft.Storage/storageAccounts/bgpalerterstorageXXXX" \
  --group-id file \
  --connection-name bgpalerter-storage-connection
```

#### Key Vault Private Endpoint
```bash
az network private-endpoint create \
  --resource-group bgpalerter-rg \
  --name bgpalerter-kv-pe \
  --vnet-name bgpalerter-vnet \
  --subnet bgpalerter-subnet \
  --private-connection-resource-id "/subscriptions/$(az account show --query id -o tsv)/resourceGroups/bgpalerter-rg/providers/Microsoft.KeyVault/vaults/bgpalerter-kv-XXXX" \
  --group-id vault \
  --connection-name bgpalerter-kv-connection
```

### Step 5: Secure BGPalerter Configuration

Create a hardened `config.yml` in your file share:

```yaml
# BGPalerter Secure Configuration
environment:
  name: production

# Restrict REST API to localhost only
rest:
  host: 127.0.0.1
  port: 8011

# Enhanced logging for security monitoring
logging:
  directory: logs
  logRotatePattern: YYYY-MM-DD
  maxRetainedFiles: 30
  maxFileSizeMB: 10
  compressOnRotation: true
  useUTC: true

# Connectors - secure connection to RIPE RIS
connectors:
  - file: connectorRIS
    name: ris-live
    params:
      url: wss://ris-live.ripe.net/v1/ws
      subscription:
        moreSpecific: true
        lessSpecific: false
        peer: "*"

# Enhanced monitoring with higher thresholds
monitors:
  - file: monitorHijack
    channel: hijack
    name: enhanced-hijack-detection
    params:
      thresholdMinPeers: 5
      fadeOffSeconds: 180

  - file: monitorRPKI
    channel: rpki
    name: strict-rpki-monitor
    params:
      thresholdMinPeers: 3
      checkUncovered: true
      checkDisappearing: true

  - file: monitorVisibility
    channel: visibility
    name: visibility-monitor
    params:
      thresholdMinPeers: 5
      fadeOffSeconds: 300

  - file: monitorPath
    channel: path
    name: path-security-monitor
    params:
      thresholdMinPeers: 3

# Secure reporting with Key Vault integration
reports:
  - file: reportEmail
    channels: [hijack, visibility, rpki, path]
    params:
      senderEmail: bgpalerter@yourdomain.com
      smtp:
        host: smtp.gmail.com
        port: 587
        secure: true
        requireTLS: true
        auth:
          user: your-email@gmail.com
          pass: "@Microsoft.KeyVault(VaultName=bgpalerter-kv-XXXX;SecretName=smtp-password)"
      notifiedEmails:
        default: [admin@yourdomain.com]

  - file: reportSlack
    channels: [hijack, rpki]
    params:
      colors: true
      webhook: "@Microsoft.KeyVault(VaultName=bgpalerter-kv-XXXX;SecretName=slack-webhook-url)"

# Process monitoring for health checks
processMonitors:
  - file: uptimeApi
    params:
      useStatusCodes: true
      host: 127.0.0.1
      port: 8011
```

## Phase 3: Monitoring and Maintenance

### Step 1: Enable Azure Monitor

#### Create Log Analytics Workspace
```bash
az monitor log-analytics workspace create \
  --resource-group bgpalerter-rg \
  --workspace-name bgpalerter-logs \
  --location eastus
```

#### Enable Container Insights
```bash
az container update \
  --resource-group bgpalerter-rg \
  --name bgpalerter \
  --log-analytics-workspace bgpalerter-logs
```

### Step 2: Configure Security Alerts

#### Container Restart Alert
```bash
az monitor metrics alert create \
  --name "BGPalerter Container Restart" \
  --resource-group bgpalerter-rg \
  --scopes "/subscriptions/$(az account show --query id -o tsv)/resourceGroups/bgpalerter-rg/providers/Microsoft.ContainerInstance/containerGroups/bgpalerter" \
  --condition "count RestartCount static gt 3 PT5M" \
  --description "BGPalerter container restarted more than 3 times in 5 minutes" \
  --severity 2
```

#### High CPU Usage Alert
```bash
az monitor metrics alert create \
  --name "BGPalerter High CPU" \
  --resource-group bgpalerter-rg \
  --scopes "/subscriptions/$(az account show --query id -o tsv)/resourceGroups/bgpalerter-rg/providers/Microsoft.ContainerInstance/containerGroups/bgpalerter" \
  --condition "avg CpuUsage static gt 80 PT10M" \
  --description "BGPalerter CPU usage above 80% for 10 minutes" \
  --severity 3
```

#### Configuration Change Alert
```bash
az monitor activity-log alert create \
  --name "BGPalerter Config Change" \
  --resource-group bgpalerter-rg \
  --scopes "/subscriptions/$(az account show --query id -o tsv)/resourceGroups/bgpalerter-rg" \
  --condition category=Administrative \
  --condition operationName=Microsoft.ContainerInstance/containerGroups/write \
  --description "BGPalerter container configuration changed"
```

### Step 3: Backup Strategy

#### Create Backup Storage
```bash
az storage account create \
  --name bgpalerterbackupXXXX \
  --resource-group bgpalerter-rg \
  --location eastus \
  --sku Standard_GRS \
  --encryption-services blob \
  --https-only true

# Enable soft delete
az storage blob service-properties delete-policy update \
  --account-name bgpalerterbackupXXXX \
  --enable true \
  --days-retained 30
```

#### Automated Backup Script
Create a backup script to regularly backup configuration files:

```bash
#!/bin/bash
# backup-bgpalerter.sh

STORAGE_ACCOUNT="bgpalerterstorageXXXX"
BACKUP_ACCOUNT="bgpalerterbackupXXXX"
DATE=$(date +%Y%m%d-%H%M%S)

# Download current configuration
az storage file download-batch \
  --destination ./backup-$DATE \
  --source bgpalerter-data \
  --account-name $STORAGE_ACCOUNT

# Upload to backup storage
az storage blob upload-batch \
  --destination backups/bgpalerter-$DATE \
  --source ./backup-$DATE \
  --account-name $BACKUP_ACCOUNT

# Cleanup local files
rm -rf ./backup-$DATE

echo "Backup completed: bgpalerter-$DATE"
```

## Cost Analysis

### Monthly Cost Breakdown (East US region)

| Component | Configuration | Monthly Cost (USD) |
|-----------|---------------|-------------------|
| **Container Instance** | 4GB RAM, 1 vCPU, always-on | $35-45 |
| **Virtual Network** | Standard VNet with subnet | $0 |
| **Network Security Group** | Standard NSG with rules | $0 |
| **Storage Account** | 5GB Standard LRS | $1-2 |
| **Key Vault** | Standard tier, <10K operations | $1-3 |
| **Container Registry** | Premium tier, <100GB | $5-10 |
| **Log Analytics** | 5GB data ingestion | $2-8 |
| **Private Endpoints** | 2 endpoints | $14-22 |
| **Backup Storage** | 5GB GRS with soft delete | $2-4 |
| **Data Transfer** | BGP monitoring traffic | $1-5 |
| **Total** | | **$61-99** |

### Cost Optimization Tips

1. **Use Azure Free Tier**: First 12 months include many free services
2. **Regional Selection**: Choose lower-cost regions (East US, South Central US)
3. **Resource Sizing**: Monitor actual usage and adjust container resources
4. **Log Retention**: Configure appropriate log retention periods
5. **Backup Frequency**: Adjust backup frequency based on change rate

## Troubleshooting

### Common Issues

#### Container Won't Start
```bash
# Check container logs
az container logs --resource-group bgpalerter-rg --name bgpalerter

# Check container events
az container show --resource-group bgpalerter-rg --name bgpalerter --query instanceView.events
```

#### Network Connectivity Issues
```bash
# Test NSG rules
az network nsg rule list --resource-group bgpalerter-rg --nsg-name bgpalerter-nsg --output table

# Check private endpoint status
az network private-endpoint list --resource-group bgpalerter-rg --output table
```

#### Storage Access Problems
```bash
# Verify storage account access
az storage account show --name bgpalerterstorageXXXX --resource-group bgpalerter-rg --query networkRuleSet

# Check file share permissions
az storage share show --name bgpalerter-data --account-name bgpalerterstorageXXXX
```

#### Key Vault Access Issues
```bash
# Check access policies
az keyvault show --name bgpalerter-kv-XXXX --resource-group bgpalerter-rg --query properties.accessPolicies

# Test secret retrieval
az keyvault secret show --vault-name bgpalerter-kv-XXXX --name smtp-password
```

### Emergency Procedures

#### Isolate Compromised Container
```bash
# Create emergency isolation rule
az network nsg rule create \
  --resource-group bgpalerter-rg \
  --nsg-name bgpalerter-nsg \
  --name EmergencyIsolation \
  --protocol "*" \
  --direction Inbound \
  --priority 50 \
  --source-address-prefix "*" \
  --destination-address-prefix "*" \
  --destination-port-range "*" \
  --access Deny

# Stop container
az container stop --resource-group bgpalerter-rg --name bgpalerter
```

#### Restore from Backup
```bash
# List available backups
az storage blob list --container-name backups --account-name bgpalerterbackupXXXX --output table

# Download backup
az storage blob download-batch \
  --destination ./restore \
  --source backups/bgpalerter-YYYYMMDD-HHMMSS \
  --account-name bgpalerterbackupXXXX

# Upload to production storage
az storage file upload-batch \
  --destination bgpalerter-data \
  --source ./restore \
  --account-name bgpalerterstorageXXXX
```

### Performance Monitoring

#### Key Metrics to Monitor
- **Container CPU Usage**: Should stay below 70%
- **Memory Usage**: Should stay below 3.5GB (leaving headroom)
- **Network Throughput**: Monitor BGP message processing rate
- **Storage IOPS**: Monitor file share performance
- **Alert Response Time**: Time from BGP event to alert delivery

#### Monitoring Queries (Log Analytics)
```kusto
// Container performance
ContainerInstanceLog_CL
| where TimeGenerated > ago(1h)
| where ContainerGroup_s == "bgpalerter"
| summarize avg(CpuUsage_d), avg(MemoryUsage_d) by bin(TimeGenerated, 5m)

// BGPalerter specific logs
ContainerInstanceLog_CL
| where TimeGenerated > ago(1h)
| where ContainerGroup_s == "bgpalerter"
| where Message contains "alert" or Message contains "hijack"
| project TimeGenerated, Message
```

## Security Maintenance

### Monthly Security Checklist
- [ ] Review Azure Security Center recommendations
- [ ] Update container image to latest version
- [ ] Rotate Key Vault secrets (quarterly)
- [ ] Review NSG rules and access logs
- [ ] Analyze BGPalerter alert patterns for anomalies
- [ ] Test backup and restore procedures
- [ ] Review and update security policies
- [ ] Check for Azure service updates and security patches

### Security Incident Response
1. **Detection**: Monitor alerts from Azure Security Center and BGPalerter
2. **Assessment**: Analyze logs in Log Analytics workspace
3. **Containment**: Use NSG rules to isolate affected resources
4. **Investigation**: Collect forensic data from logs and metrics
5. **Recovery**: Restore from known-good configuration backup
6. **Lessons Learned**: Update security policies and procedures

This comprehensive guide provides enterprise-grade security for BGPalerter deployment on Azure while maintaining cost-effectiveness and operational simplicity. The security measures implemented protect against common attack vectors while ensuring reliable BGP monitoring capabilities.
