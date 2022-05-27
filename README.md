# Virtual-Wan Custom Routing with BGP over IPSEC
<br>
In this lab, the topology will include one vWAN with one vhub and four attached VNETs with custom routing. We will also have an IPSEC tunnel with BGP to a CSR in the branch with one VM. Username for all VMs is azureuser and password is MyP@SSword123!. This assumes you are using serial console to connect to all VMs including the CSR which are not in the lab instructions. Change the paramaters and prefixes to your liking.
<br>
<br>
Topology:

![vWAN](https://user-images.githubusercontent.com/55964102/170731858-33bb09fc-cc2d-47c7-8c7a-baa08dad480c.jpg)
<br>

Commands:
<br>
#Parameters
<br>
loc=WestUS2
<br>
rg=lab-csrvwan
<br>
vwanname=myvwan
<br>
vhubname=myvhub
<br>
username=azureuser
<br>
password="MyP@SSword123!"
<br>
vmsize=Standard_D2_v2
<br>
<br>
#Create the RG
<br>
az group create -n $rg -l $loc --output none
