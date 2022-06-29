#!/bin/bash

# VARIABLES
rg="1hub-udr"
loc="eastus"

BLACK="\033[30m"
RED="\033[31m"
GREEN="\033[32m"
YELLOW="\033[33m"
BLUE="\033[34m"
PINK="\033[35m"
CYAN="\033[36m"
WHITE="\033[37m"
NORMAL="\033[0;39m"

usessh=true
vmname="frrVM"
vmspoke1="spoke1VM"
vmspoke2="spoke2VM"
vmspoke3="spoke3VM"
username="azureuser"
password="MyP@ssword123"
vmsize="Standard_D2S_v3"

# create a resource group
echo -e "$WHITE$(date +"%T")$GREEN Creating Resource Group$CYAN" $rg"$GREEN in $CYAN"$loc"$WHITE"
az group create -n $rg -l $loc -o none

# create a virtual network
echo -e "$WHITE$(date +"%T")$GREEN Creating Virtual Network hubVnet $WHITE"
az network vnet create --address-prefixes 10.1.0.0/16 -n hubVnet -g $rg --subnet-name internal --subnet-prefixes 10.1.4.0/24 -o none

# create subnets
echo -e "$WHITE$(date +"%T")$GREEN Creating subnets $WHITE"
echo ".... creating subnet1"
az network vnet subnet create -g $rg --vnet-name hubVnet -n subnet1 --address-prefixes 10.1.2.0/24 -o none
echo ".... creating external"
az network vnet subnet create -g $rg --vnet-name hubVnet -n external --address-prefixes 10.1.3.0/24 -o none
echo ".... creating GatewaySubnet"
az network vnet subnet create -g $rg --vnet-name hubVnet -n GatewaySubnet --address-prefixes 10.1.5.0/24 -o none
echo ".... creating AzureBastionSubnet"
az network vnet subnet create -g $rg --vnet-name hubVnet -n AzureBastionSubnet --address-prefixes 10.1.6.0/26 -o none

# Create ExR Gateway
echo -e "$WHITE$(date +"%T")$GREEN Creating ExpressRoute Gateway Standard $WHITE"
az network public-ip create -n exrGW-pip -g $rg --location $loc --sku Standard --only-show-errors -o none
az network vnet-gateway create -n exrGW --public-ip-addresses exrGW-pip -g $rg --vnet hubVnet --gateway-type ExpressRoute -l $loc --sku Standard -o none

# Create ExR Circuit
echo -e "$WHITE$(date +"%T")$GREEN Creating ExpressRoute Circuit $WHITE"
az network express-route create -g $rg -n exr-hubcircuit --bandwidth '50 Mbps' --peering-location "Washington DC" --provider "Megaport" -l $loc --sku-family MeteredData --sku-tier Standard -o none

# create spoke virtual networks
echo -e "$WHITE$(date +"%T")$GREEN Creating Virtual Network spoke1Vnet $WHITE"
az network vnet create --address-prefixes 10.10.0.0/16 -n spoke1Vnet -g $rg --subnet-name app --subnet-prefixes 10.10.0.0/24 -o none

# create spoke virtual networks
echo -e "$WHITE$(date +"%T")$GREEN Creating Virtual Network spoke2Vnet $WHITE"
az network vnet create --address-prefixes 10.11.0.0/16 -n spoke2Vnet -g $rg --subnet-name app --subnet-prefixes 10.11.0.0/24 -o none

# create spoke virtual networks
echo -e "$WHITE$(date +"%T")$GREEN Creating Virtual Network spoke3Vnet $WHITE"
az network vnet create --address-prefixes 10.12.0.0/16 -n spoke3Vnet -g $rg --subnet-name app --subnet-prefixes 10.12.0.0/24 -o none

# Create bastion to support access to other VM's that are not reachable publicly
echo -e "$WHITE$(date +"%T")$GREEN Creating Bastion $WHITE"
az network public-ip create --name bastion-pip --resource-group $rg -l $loc --sku Standard -o none --only-show-errors
az network bastion create -g $rg -n bastion --public-ip-address bastion-pip --vnet-name hubVnet -l $loc -o none --only-show-errors

# Turn on SSH tunneling
# az cli does not have a property to enable SSH tunneling, so must be done via rest API
echo -e "$WHITE$(date +"%T")$GREEN Turn on SSH Tunneling $WHITE"
subid=$(az account show --query 'id' -o tsv)
uri='https://management.azure.com/subscriptions/'$subid'/resourceGroups/'$rg'/providers/Microsoft.Network/bastionHosts/bastion?api-version=2021-08-01'
json='{
  "location": "'$loc'",
  "properties": {
    "enableTunneling": "true",
    "ipConfigurations": [
      {
        "name": "bastion_ip_config",
        "properties": {
          "subnet": {
            "id": "/subscriptions/'$subid'/resourceGroups/'$rg'/providers/Microsoft.Network/virtualNetworks/hubVnet/subnets/AzureBastionSubnet"
          },
          "publicIPAddress": {
            "id": "/subscriptions/'$subid'/resourceGroups/'$rg'/providers/Microsoft.Network/publicIPAddresses/bastion-pip"
          }
        }
      }
    ]
  }
}'

az rest --method PUT \
    --url $uri  \
    --body "$json"  \
    --output none

# peer virtual networks (spoke to hub)
echo -e "$WHITE$(date +"%T")$GREEN Peer hub to spokes $WHITE"
hubid=$(az network vnet show -g $rg -n hubVnet --query id -o tsv)
spoke1id=$(az network vnet show -g $rg -n spoke1Vnet --query id -o tsv)
spoke2id=$(az network vnet show -g $rg -n spoke2Vnet --query id -o tsv)
spoke3id=$(az network vnet show -g $rg -n spoke3Vnet --query id -o tsv)
# peer spoke1
echo ".... peering spoke1"
az network vnet peering create -n "hubTOspoke1" -g $rg --vnet-name hubVnet --remote-vnet $spoke1id --allow-vnet-access --allow-forwarded-traffic --allow-gateway-transit -o none
az network vnet peering create -n "spoke1TOhub" -g $rg --vnet-name spoke1Vnet --remote-vnet $hubid --allow-vnet-access --allow-forwarded-traffic --use-remote-gateways -o none
# peer spoke2
echo ".... peering spoke2"
az network vnet peering create -n "hubTOspoke2" -g $rg --vnet-name hubVnet --remote-vnet $spoke2id --allow-vnet-access --allow-forwarded-traffic --allow-gateway-transit -o none
az network vnet peering create -n "spoke2TOhub" -g $rg --vnet-name spoke2Vnet --remote-vnet $hubid --allow-vnet-access --allow-forwarded-traffic --use-remote-gateways -o none
# peer spoke3
echo ".... peering spoke3"
az network vnet peering create -n "hubTOspoke3" -g $rg --vnet-name hubVnet --remote-vnet $spoke3id --allow-vnet-access --allow-forwarded-traffic --allow-gateway-transit -o none
az network vnet peering create -n "spoke3TOhub" -g $rg --vnet-name spoke3Vnet --remote-vnet $hubid --allow-vnet-access --allow-forwarded-traffic --use-remote-gateways -o none

# create route tables
# nva external NIC to internet
echo -e "$WHITE$(date +"%T")$GREEN Creating route table for frrVM external interace to internet $WHITE"
az network route-table create -g $rg -n nvaroute -o none
az network route-table route create -g $rg --route-table-name nvaroute -n tointernet \
    --next-hop-type Internet --address-prefix 0.0.0.0/0 -o none
az network vnet subnet update -g $rg -n external --vnet-name hubVnet --route-table nvaroute -o none

# spoke to NVA
echo -e "$WHITE$(date +"%T")$GREEN Creating route table for spokes to NVA $WHITE"
az network route-table create -g $rg -n spokeroute -o none
az network route-table route create -g $rg --route-table-name spokeroute -n tonva \
    --next-hop-type VirtualAppliance  --address-prefix 0.0.0.0/0 --next-hop-ip-address 10.1.4.10 -o none
az network vnet subnet update -g $rg -n app --vnet-name spoke1Vnet --route-table spokeroute -o none
az network vnet subnet update -g $rg -n app --vnet-name spoke2Vnet --route-table spokeroute -o none
az network vnet subnet update -g $rg -n app --vnet-name spoke3Vnet --route-table spokeroute -o none

# ExR to NVA
echo -e "$WHITE$(date +"%T")$GREEN Creating route table for ExpressRoute Gateway $WHITE"
az network route-table create -g $rg -n gwroute -o none
az network route-table route create -g $rg --route-table-name gwroute -n spoke1tonva \
    --next-hop-type VirtualAppliance  --address-prefix 10.10.0.0/16 --next-hop-ip-address 10.1.4.10 -o none
az network route-table route create -g $rg --route-table-name gwroute -n spoke2tonva \
    --next-hop-type VirtualAppliance  --address-prefix 10.11.0.0/16 --next-hop-ip-address 10.1.4.10 -o none
az network route-table route create -g $rg --route-table-name gwroute -n spoke3tonva \
    --next-hop-type VirtualAppliance  --address-prefix 10.12.0.0/16 --next-hop-ip-address 10.1.4.10 -o none
az network vnet subnet update -g $rg -n GatewaySubnet --vnet-name hubVnet --route-table gwroute -o none

# create frrVM
mypip=$(curl -4 ifconfig.io -s)
echo -e "$WHITE$(date +"%T")$GREEN Create Public IP, NSG, and Allow SSH on port 22 for IP: $WHITE"$mypip
az network nsg create -g $rg -n $vmname"NSG" -o none
az network nsg rule create -n "Allow-SSH" --nsg-name $vmname"NSG" --priority 300 -g $rg --direction Inbound --protocol TCP --source-address-prefixes $mypip --destination-port-ranges 22 -o none
az network public-ip create -n $vmname"-pip" -g $rg --version IPv4 --sku Standard -o none --only-show-errors 

echo -e "$WHITE$(date +"%T")$GREEN Creating frr VM $WHITE"
az network nic create -g $rg --vnet-name hubVnet --subnet internal -n $vmname"IntNIC" --private-ip-address 10.1.4.10 --ip-forwarding true -o none
az network nic create -g $rg --vnet-name hubVnet --subnet external -n $vmname"ExtNIC" --public-ip-address $vmname"-pip" --private-ip-address 10.1.3.10 --network-security-group $vmname"NSG" --ip-forwarding true -o none
if [ $usessh = "true" ]; then
    az vm create -n $vmname \
        -g $rg \
        --image ubuntults \
        --size $vmsize \
        --nics $vmname"ExtNIC" $vmname"IntNIC" \
        --authentication-type ssh \
        --admin-username $username \
        --ssh-key-values @~/.ssh/id_rsa.pub \
        --custom-data cloud-init \
        -o none \
        --only-show-errors
else
    az vm create -n $vmname \
        -g $rg \
        --image ubuntults \
        --size $vmsize \
        --nics $vmname"ExtNIC" $vmname"IntNIC" \
        --admin-username $username \
        --admin-password $password \
        --ssh-key-values @~/.ssh/id_rsa.pub \
        --custom-data cloud-init \
        -o none \
        --only-show-errors
fi

# create Spoke1 VM
echo -e "$WHITE$(date +"%T")$GREEN Creating Spoke1 VM $WHITE"
az network nic create -g $rg --vnet-name spoke1Vnet --subnet app -n $vmspoke1"NIC" -o none
if [ $usessh = "true" ]; then
    az vm create -n $vmspoke1 \
        -g $rg \
        --image ubuntults \
        --size $vmsize \
        --nics $vmspoke1"NIC" \
        --authentication-type ssh \
        --admin-username $username \
        --ssh-key-values @~/.ssh/id_rsa.pub \
        -o none \
        --only-show-errors
else
    az vm create -n $vmspoke1 \
        -g $rg \
        --image ubuntults \
        --size $vmsize \
        --nics $vmspoke1"NIC" \
        --admin-username $username \
        --admin-password $password \
        --ssh-key-values @~/.ssh/id_rsa.pub \
        -o none \
        --only-show-errors
fi

# create Spoke2 VM
echo -e "$WHITE$(date +"%T")$GREEN Creating Spoke2 VM $WHITE"
az network nic create -g $rg --vnet-name spoke2Vnet --subnet app -n $vmspoke2"NIC" -o none
if [ $usessh = "true" ]; then
    az vm create -n $vmspoke2 \
        -g $rg \
        --image ubuntults \
        --size $vmsize \
        --nics $vmspoke2"NIC" \
        --authentication-type ssh \
        --admin-username $username \
        --ssh-key-values @~/.ssh/id_rsa.pub \
        -o none \
        --only-show-errors
else
    az vm create -n $vmspoke2 \
        -g $rg \
        --image ubuntults \
        --size $vmsize \
        --nics $vmspoke2"NIC" \
        --admin-username $username \
        --admin-password $password \
        --ssh-key-values @~/.ssh/id_rsa.pub \
        -o none \
        --only-show-errors
fi

# create Spoke3 VM
echo -e "$WHITE$(date +"%T")$GREEN Creating Spoke3 VM $WHITE"
az network nic create -g $rg --vnet-name spoke3Vnet --subnet app -n $vmspoke3"NIC" -o none
if [ $usessh = "true" ]; then
    az vm create -n $vmspoke3 \
        -g $rg \
        --image ubuntults \
        --size $vmsize \
        --nics $vmspoke3"NIC" \
        --authentication-type ssh \
        --admin-username $username \
        --ssh-key-values @~/.ssh/id_rsa.pub \
        -o none \
        --only-show-errors
else
    az vm create -n $vmspoke3 \
        -g $rg \
        --image ubuntults \
        --size $vmsize \
        --nics $vmspoke3"NIC" \
        --admin-username $username \
        --admin-password $password \
        --ssh-key-values @~/.ssh/id_rsa.pub \
        -o none \
        --only-show-errors
fi

echo "To check frr route table"
echo "ssh azureuser@"$(az vm show -g $rg -n $vmname --show-details --query "publicIps" -o tsv)
echo "vtysh"
echo "show ip bgp"
