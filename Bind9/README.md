# Private Link DNS with Bind9
This repo contains example for configuring DNS entries with private link private IPs for PaaS service FQDN. Design is to use onpremises-DNS Server to host onprem zones and custom DNS Server in Azure to be used as custom DNS for Azure resources, forwarder to onprem and forwarder to Azure Private DNS for registration of VMs and Private Links.

## Architecture

```
onprem-dns  ------------------------------------------------------------------------------------- azure-dns (VM) ---------------------------------------- Azure Private DNS (168.63.129.16)
  |                                                                                                 |                                                           | 
  |- Forwarder to public DNS 208.67.222.222                                                         |- Forwarder to Azure DNS                                   |- Host tomas.azure zone
  |                                                                                                 |                                                           |
  |- host onprem zone tomas.onprem                                                                  |- Forwarder to tomas.azure via Azure Private DNS           |- Host privatelink.database.windows.net
  |                                                                                                 |
  |- Forwarder to tomas.azure via azure-dns (VM)                                                    |- Forwarder to tomas.onprem via onprem-dns
  |
  |- Forwarder to privatelink.database.windows.net via azure-dns (VM) 
```

## Prepare Azure networking

```bash
# Create resource group
az group create -n dns -l westeurope

# Create hub-and-spoke networking
az network vnet create -n hub-net \
    -g dns \
    --address-prefix 10.0.0.0/16 \
    --subnet-name subnet1 \
    --subnet-prefix 10.0.0.0/24

az network vnet create -n spoke-net \
    -g dns \
    --address-prefix 10.1.0.0/16 \
    --subnet-name subnet1 \
    --subnet-prefix 10.1.0.0/24

# Create network peering
az network vnet peering create -n hub-to-spoke \
    -g dns \
    --vnet-name hub-net \
    --remote-vnet $(az network vnet show -n spoke-net -g dns --query id -o tsv) \
    --allow-vnet-access \
    --allow-forwarded-traffic

az network vnet peering create -n hub-to-spoke \
    -g dns \
    --vnet-name spoke-net \
    --remote-vnet $(az network vnet show -n hub-net -g dns --query id -o tsv) \
    --allow-vnet-access \
    --allow-forwarded-traffic
```

## Prepare Azure PaaS
Prepare two Azure SQL Databases, only one will have private link

```bash
# Provision 2 SQL server objects
az sql server create -l westeurope -g dns -n tomas-sqlserver-1 -u tomas -p Azure12345678
az sql server create -l westeurope -g dns -n tomas-sqlserver-2 -u tomas -p Azure12345678

# Provision 2 SQL DB
az sql db create -g dns -s tomas-sqlserver-1 -n mydb --service-objective S0
az sql db create -g dns -s tomas-sqlserver-2 -n mydb --service-objective S0
```

## Create Private Link for SQL1

```bash
# Update subnet configuration
az network vnet subnet update \
 --name subnet1 \
 --resource-group dns \
 --vnet-name spoke-net \
 --disable-private-endpoint-network-policies true

# Create Private Endpoint
az network private-endpoint create \
    --name sql1-private-endpoint \
    --resource-group dns \
    --vnet-name spoke-net  \
    --subnet subnet1 \
    --private-connection-resource-id $(az sql server show -g dns -n tomas-sqlserver-1 --query id -o tsv) \
    --group-ids sqlServer \
    --connection-name myConnection
```

## Create and configure Azure Private DNS for privatelink.database.windows.net

```bash
# Create private zone for privatelink
az network private-dns zone create -g dns -n "privatelink.database.windows.net" 

# Link zone to both VNETs - disable VM registration as we will not use Azure DNS for VMs
az network private-dns link vnet create -g dns \
   --zone-name  "privatelink.database.windows.net" \
   -n hubDnsLink \
   --virtual-network hub-net \
   --registration-enabled false 
az network private-dns link vnet create -g dns \
   --zone-name  "privatelink.database.windows.net" \
   -n spokeDnsLink \
   --virtual-network spoke-net \
   --registration-enabled false 

# Get Private Link IP
export plIp=$(az network nic show \
    --ids $(az network private-endpoint show -n sql1-private-endpoint -g dns --query networkInterfaces[].id -o tsv) \
    --query ipConfigurations[].privateIpAddress \
    -o tsv)

# Create private DNS zone privatelink.database.windows.net
az network private-dns record-set a create -n tomas-sqlserver-1 --zone-name privatelink.database.windows.net -g dns  

# Create private DNS record for sq1
az network private-dns record-set a add-record \
    --record-set-name tomas-sqlserver-1 \
    --zone-name privatelink.database.windows.net \
    -g dns \
    -a $plIp
```

## Create and configure Azure Private DNS for tomas.azure

```bash
# Create private zone for tomas.azure
az network private-dns zone create -g dns -n "tomas.azure" 

# Link zone to both VNETs - enable VM registration
az network private-dns link vnet create -g dns \
   --zone-name  "tomas.azure" \
   -n hubDnsLink \
   --virtual-network hub-net \
   --registration-enabled true 
az network private-dns link vnet create -g dns \
   --zone-name  "tomas.azure" \
   -n spokeDnsLink \
   --virtual-network spoke-net \
   --registration-enabled true 
```

## Simulate onpremises DNS server (we will run it in Azure)

```bash
# Create VM
az vm create -n dns-onprem \
    -g dns \
    --image UbuntuLTS \
    --size Standard_B1ms \
    --admin-username tomas \
    --ssh-key-values ~/.ssh/id_rsa.pub \
    --nsg "" \
    --vnet-name hub-net \
    --subnet subnet1 \
    --public-ip-address dns-onprem-ip \
    --private-ip-address 10.0.0.5 \
    --storage-sku Standard_LRS

# Get VM public IP
export dnsOnprem=$(az network public-ip show -n dns-onprem-ip -g dns --query ipAddress -o tsv)

# Connect to VM
ssh tomas@$dnsOnprem

# Install Bind
sudo apt install -y bind9

# Copy files to /etc/bind and restart service
sudo systemctl restart bind9
```

## Deploy and configure custom DNS in Azure

```bash
# Create VM
az vm create -n dns-azure \
    -g dns \
    --image UbuntuLTS \
    --size Standard_B1ms \
    --admin-username tomas \
    --ssh-key-values ~/.ssh/id_rsa.pub \
    --nsg "" \
    --vnet-name hub-net \
    --subnet subnet1 \
    --public-ip-address dns-azure-ip \
    --private-ip-address 10.0.0.4 \
    --storage-sku Standard_LRS

# Get VM public IP
export dnsAzure=$(az network public-ip show -n dns-azure-ip -g dns --query ipAddress -o tsv)

# Connect to VM
ssh tomas@$dnsAzure

# Install Bind
sudo apt install -y bind9

# Copy files to /etc/bind and restart service
sudo systemctl restart bind9
```

## Test Azure VM
```bash
# Create public IP (for testing)
az network public-ip create -n azurevm-ip -g dns --sku Basic

# Create NIC using Azure DNS node 10.0.0.4
az network nic create -n azurevm-nic \
    -g dns \
    --vnet-name spoke-net \
    --subnet subnet1 \
    --dns-servers 10.0.0.4 \
    --public-ip-address azurevm-ip

# Create VM
az vm create -n azurevm \
    -g dns \
    --image UbuntuLTS \
    --size Standard_B1ms \
    --admin-username tomas \
    --ssh-key-values ~/.ssh/id_rsa.pub \
    --nics azurevm-nic \
    --storage-sku Standard_LRS

# Connect to VM
export azurevm=$(az network public-ip show -n azurevm-ip -g dns --query ipAddress -o tsv)
ssh tomas@$azurevm

# Test DNS to onprem resource
dig vm1.tomas.onprem

# Test DNS within Azure
dig azurevm.tomas.azure

# Test DNS to SQL with Private Link
dig tomas-sqlserver-1.database.windows.net

# Test DNS to SQL without Private Link
dig tomas-sqlserver-2.database.windows.net
```

## Test onprem VM (simulated in Azure)
```bash
# Create public IP (for testing)
az network public-ip create -n onpremvm-ip -g dns --sku Basic

# Create NIC using Onprem DNS node 10.0.0.5
az network nic create -n onpremvm-nic \
    -g dns \
    --vnet-name spoke-net \
    --subnet subnet1 \
    --dns-servers 10.0.0.5 \
    --public-ip-address onpremvm-ip

# Create VM
az vm create -n onpremvm \
    -g dns \
    --image UbuntuLTS \
    --size Standard_B1ms \
    --admin-username tomas \
    --ssh-key-values ~/.ssh/id_rsa.pub \
    --nics onpremvm-nic \
    --storage-sku Standard_LRS

# Connect to VM
export onpremvm=$(az network public-ip show -n onpremvm-ip -g dns --query ipAddress -o tsv)
ssh tomas@$onpremvm

# Test DNS to azure zone
dig azurevm.tomas.azure

# Test DNS to SQL with Private Link
dig tomas-sqlserver-1.database.windows.net

# Test DNS to SQL without Private Link
dig tomas-sqlserver-2.database.windows.net
```
