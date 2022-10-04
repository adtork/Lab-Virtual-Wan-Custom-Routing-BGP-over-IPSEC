# Lab: Virtual-Wan Custom Routing with BGP over IPSEC


## Introduction


In this lab, the topology will include one vWAN with one vhub and four attached VNETs with custom routing. We will also have an IPSEC tunnel with BGP to a CSR in the branch with one VM. Username for all VMs is azureuser and password is MyP@SSword123!. This assumes you are using serial console to connect to all VMs including the CSR which are not in the lab instructions. Change the paramaters and prefixes to your liking.

## Topology

![vWAN](https://user-images.githubusercontent.com/55964102/170731858-33bb09fc-cc2d-47c7-8c7a-baa08dad480c.jpg)
<br>

## Commands
```bash
#Paramaters
loc=WestUS2
rg=lab-csrvwan
vwanname=myvwan
vhubname=myvhub
username=azureuser
password="MyP@SSword123!"
vmsize=Standard_D2_v2

#Create the RG
az group create -n $rg -l $loc --output none

#Create the vWAN and vHub
az network vwan create -g $rg -n $vwanname --branch-to-branch-traffic true --location $loc --type Standard --output none
az network vhub create -g $rg --name $vhubname --address-prefix 172.16.1.0/24 --vwan $vwanname --location $loc --sku Standard --no-wait

#Create CSR Branch Vnet and Inside/Outside Subnets, Spoke VM Subnet
az network vnet create --address-prefixes 10.1.0.0/16 -n csrbranch -g $rg -l $loc --subnet-name inside --subnet-prefixes 10.1.0.0/24 --output none
az network vnet subnet create --address-prefixes 10.1.1.0/24 --name outside --resource-group $rg --vnet-name csrbranch --output none
az network vnet subnet create --address-prefixes 10.1.2.0/24 --name branch --resource-group $rg --vnet-name csrbranch --output none

#Create the Spoke Vnets
az network vnet create --address-prefixes 172.16.2.0/24 -n spoke1 -g $rg -l $loc --subnet-name default --subnet-prefixes 172.16.2.0/27 --output none
az network vnet create --address-prefixes 172.16.3.0/24 -n spoke2 -g $rg -l $loc --subnet-name default --subnet-prefixes 172.16.3.0/27 --output none
az network vnet create --address-prefixes 172.16.4.0/24 -n spoke3 -g $rg -l $loc --subnet-name default --subnet-prefixes 172.16.4.0/27 --output none
az network vnet create --address-prefixes 172.16.5.0/24 -n spoke4 -g $rg -l $loc --subnet-name default --subnet-prefixes 172.16.5.0/27 --output none

#Create the CSR in Branch
az vm image terms accept --urn cisco:cisco-csr-1000v:17_3_4a-byol:latest
az network public-ip create --name CSRPubIP --resource-group $rg --idle-timeout 30 --allocation-method Static --location $loc --output none
az network nic create --name CSROutsideInt -g $rg --subnet outside --vnet csrbranch --public-ip-address CSRPubIP --ip-forwarding true --location $loc --output none
az network nic create --name CSRInsideInt -g $rg --subnet inside --vnet csrbranch --ip-forwarding true --location $loc --output none
az vm create --resource-group $rg --location $loc --name CSR --size $vmsize --nics CSROutsideInt CSRInsideInt --image cisco:cisco-csr-1000v:17_3_4a-byol:latest --admin-username azureuser --admin-password MyP@SSword123!

#Get CSR PIP, will use late for branch setup
az network public-ip show -g $rg -n CSRPubIP --query "{address: ipAddress}"

#Create the VMs in Spoke Subnets and VM in Branch Subnet
az vm create -n spoke1VM  -g $rg --image ubuntults --public-ip-sku Standard --size $vmsize -l $loc --subnet default --vnet-name spoke1 --admin-username $username --admin-password $password --no-wait
az vm create -n spoke2VM  -g $rg --image ubuntults --public-ip-sku Standard --size $vmsize -l $loc --subnet default --vnet-name spoke2 --admin-username $username --admin-password $password --no-wait
az vm create -n spoke3VM  -g $rg --image ubuntults --public-ip-sku Standard --size $vmsize -l $loc --subnet default --vnet-name spoke3 --admin-username $username --admin-password $password --no-wait
az vm create -n spoke4VM  -g $rg --image ubuntults --public-ip-sku Standard --size $vmsize -l $loc --subnet default --vnet-name spoke4 --admin-username $username --admin-password $password --no-wait
az vm create -n branchVM  -g $rg --image ubuntults --public-ip-sku Standard --size $vmsize -l $loc --subnet branch --vnet-name csrbranch --admin-username $username --admin-password $password --no-wait 

#Create the spoke connections to vHub
az network vhub connection create -n spoke1connect --remote-vnet spoke1 -g $rg --vhub-name $vhubname --no-wait
az network vhub connection create -n spoke2connect --remote-vnet spoke2 -g $rg --vhub-name $vhubname --no-wait
az network vhub connection create -n spoke3connect --remote-vnet spoke3 -g $rg --vhub-name $vhubname --no-wait
az network vhub connection create -n spoke4connect --remote-vnet spoke4 -g $rg --vhub-name $vhubname --no-wait

#Create the vHub Gateway
az network vpn-gateway create -n $vhubname-vng -g $rg --location $loc --vhub $vhubname --no-wait
#Wait for around 20ish minutes before procedding on the next steps to make sure GW is provisoned 

#Get vHub PIP and BGP info for CSR Config
az network vpn-gateway show -n $vhubname-vng -g $rg --query 'bgpSettings.bgpPeeringAddresses[0].defaultBgpIpAddresses[0]' 
az network vpn-gateway show -n $vhubname-vng -g $rg --query 'bgpSettings.bgpPeeringAddresses[0].tunnelIpAddresses[0]'

#Create the connection to CSR, replace with your AZ GW VIP and BGP IPs
crypto ikev2 proposal Azure-Ikev2-Proposal
 encryption aes-cbc-256
 integrity sha1 sha256
 group 2
!
crypto ikev2 policy Azure-Ikev2-Policy
 match address local 10.1.1.4 
 proposal Azure-Ikev2-Proposal
!
crypto ikev2 keyring to-onprem-keyring
 peer Azure-VNGpubip
  address Azure-VNGpubip
  pre-shared-key abc123
!
crypto ikev2 profile Azure-Ikev2-Profile
 match address local 10.1.1.4 
 match identity remote address 20.252.3.8 255.255.255.255
 authentication remote pre-share
 authentication local pre-share
 keyring local to-onprem-keyring
 lifetime 28800
 dpd 10 5 on-demand
!
crypto ipsec transform-set to-Azure-TransformSet esp-gcm 256
 mode tunnel
!
crypto ipsec profile to-Azure-IPsecProfile
 set transform-set to-Azure-TransformSet
 set ikev2-profile Azure-Ikev2-Profile
!
interface Loopback11
 ip address 192.168.1.1 255.255.255.255
!
interface Tunnel11
 ip address 192.168.2.1 255.255.255.255
 ip tcp adjust-mss 1350
 tunnel source 10.1.1.4
 tunnel mode ipsec ipv4
 tunnel destination Azure-VNGpubip
 tunnel protection ipsec profile to-Azure-IPsecProfile
!
router bgp 65003
 bgp router-id 192.168.1.1
 bgp log-neighbor-changes
 neighbor 172.16.1.12 remote-as 65515
 neighbor 172.16.1.12 ebgp-multihop 255
 neighbor 172.16.1.12 update-source Loopback11
 !
 address-family ipv4
  network 10.1.2.0 mask 255.255.255.0
  neighbor 172.16.1.12 activate
 exit-address-family
!
!Static route to On-Prem-VNG BGP ip pointing to Tunnel11, so that it would be reachable
ip route 172.16.1.12 255.255.255.255 Tunnel11
!Static route for default subnet in branch pointing to CSR default gateway of internal subnet, this is added in order to be able to advertise this route using BGP
ip route 10.1.2.0 255.255.255.0 10.1.1.1

exit
exit
wr

#Create CSR VPN Branch
az network vpn-site create --ip-address 20.112.94.114 -n csrbranch -g $rg --asn 65003 --bgp-peering-address 192.168.1.1 -l $loc --virtual-wan $vwanname --device-model 'ASR1000v' --device-vendor 'Cisco' --link-speed '150' --with-link true --output none

#Create connection to CSR VPN Branch
az network vpn-gateway connection create --gateway-name $vhubname-vng -n csrbranch-conn -g $rg --enable-bgp true --remote-vpn-site csrbranch --internet-security --shared-key 'abc123' --output none

#Test to make sure tunnel is up including BGP and learned routes on CSR
sh int tunnel11
sh ip bgp summ
sh ip route

#Create the route table for branch VM to ping all Spoke VMs and assoicate it
az network route-table create -g $rg -n rt_branch --output none
az network route-table route create -g $rg --route-table-name rt_branch -n branchtospoke --next-hop-type VirtualAppliance --address-prefix 172.16.0.0/16 --next-hop-ip-address 10.1.1.4 --output none
az network vnet subnet update --vnet-name csrbranch --name branch --resource-group $rg --route-table rt_branch --output none

#Create the custom vhub routing 
az network vhub route-table create -n rt_yellow -g $rg --vhub-name $vhubname --labels rt_yellow --no-wait
az network vhub route-table create -n rt_blue -g $rg --vhub-name $vhubname --labels rt_blue --no-wait

#Associate the VNETs and config propogation
default_hub=$(az network vhub route-table show --name defaultroutetable --vhub-name $vhubname -g $rg --query id -o tsv)
rt_yellow=$(az network vhub route-table show --name rt_yellow --vhub-name $vhubname -g $rg --query id -o tsv)
rt_blue=$(az network vhub route-table show --name rt_blue --vhub-name $vhubname -g $rg --query id -o tsv)

#Create the isolation for yellow VNETS
az network vhub connection update --name spoke1connect --resource-group $rg --associated-route-table $rt_yellow --vhub-name $vhubname --propagated-route-tables $rt_yellow $default_hub --labels rt_yellow default --no-wait
az network vhub connection update --name spoke3connect --resource-group $rg --associated-route-table $rt_yellow --vhub-name $vhubname --propagated-route-tables $rt_yellow $default_hub --labels rt_yellow default --no-wait

#Create the isolation for blue VNETS
az network vhub connection update --name spoke2connect --resource-group $rg --associated-route-table $rt_blue --vhub-name $vhubname --propagated-route-tables $rt_blue $default_hub --labels rt_blue default --no-wait
az network vhub connection update --name spoke4connect --resource-group $rg --associated-route-table $rt_blue --vhub-name $vhubname --propagated-route-tables $rt_blue $default_hub --labels rt_blue default --no-wait

#Make sure CSR branch reaches all spokes including custom route tables
az network vpn-gateway connection update --gateway-name $vhubname-vng -n csrbranch-conn -g $rg --propagated $rt_yellow $rt_blue_blue $default_hub --label default rt_yellow rt_blue --output none
```

## Take Aways
We can see after creating the rt for blue and yellow, only those VMs in those route tables can ping each other. So blue vms can ping blue and yellow vms can ping yellow, but not across. Also, the spoke vms cannot ping the branch vm, since they are only associated to custom route table. For the branch vm, we created a custom route table and pointed to the CSR outside interface to ping all spoke vms across the ipsec tunnel. 
