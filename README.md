# phpipam
phpipam

Deploying phpIpam to Azure with azure CLI, using Azure Container instances, mySql and keeping it private

* [AZ CLI Commands](https://learn.microsoft.com/en-us/cli/azure/?view=azure-cli-latest)

* [phpipam-www](https://hub.docker.com/r/phpipam/phpipam-www)

* [ACI Example](https://learn.microsoft.com/en-us/azure/container-instances/container-instances-multi-container-yaml)

* [YAML Reference](https://learn.microsoft.com/en-us/azure/container-instances/container-instances-reference-yaml) 

* [MySQL Example](https://learn.microsoft.com/en-us/azure/mysql/single-server/quickstart-create-mysql-server-database-using-azure-cli)

* [Flexible Server aka mysql](https://learn.microsoft.com/en-us/azure/mysql/flexible-server/quickstart-create-server-cli)

* [Flexible Server TLS](https://learn.microsoft.com/en-us/azure/mysql/flexible-server/how-to-connect-tls-ssl)

This is all done via the Azure CLI Shell

Clone or create the yaml file in this repo and modify as per instructions 

Remember to clean up everything at the end. 

Variables

    project=phpacidemo
    MyResourceGroup=rg-$project
    vnetName=vnet-1

Create a RG 

    az group create -l uksouth -n $MyResourceGroup

Create a Vnet and its associated subnets 

    az network vnet create \
    --name $vnetName \
    --resource-group $MyResourceGroup \
    --address-prefix 10.0.0.0/16 \
    --subnet-name vm-subnet \
    --subnet-prefixes 10.0.0.0/24

    az network vnet subnet create \
    --vnet-name $vnetName \
    --resource-group $MyResourceGroup \
    --name aci-subnet \
    --address-prefixes 10.0.1.0/24

    az network vnet subnet create \
    --vnet-name $vnetName \
    --resource-group $MyResourceGroup \
    --name sql-subnet \
    --address-prefixes 10.0.2.0/24

    az network vnet subnet update \
    --resource-group $MyResourceGroup \
    --name aci-subnet \
    --vnet-name $vnetName \
    --delegations Microsoft.ContainerInstance/containerGroups

Create a public IP for the VM 

    az network public-ip create \
        --resource-group $MyResourceGroup \
        --name myPublicIP \
        --version IPv4 \
        --sku Standard \
        --zone 1 2 3

Create VM for accessing ACI via a browser a "Bastion" or "management" server (you will be prompted for a password)

    az vm create \
        --name $project-bst \
        --resource-group $MyResourceGroup \
        --public-ip-address myPublicIP \
        --size Standard_B2ms \
        --image MicrosoftWindowsServer:WindowsServer:2019-Datacenter:latest \
        --vnet-name $vnetName \
        --subnet vm-subnet \
        --admin-username testadmin \
        --output json 

Create a Mysql Database - This will prompt you to confirm the creation of a pribate dns press 'Y'

    az mysql flexible-server create \
    --name $project-mysql \
    --resource-group $MyResourceGroup \
    --admin-user mylogin \
    --admin-password <password> \
    --sku-name Standard_B1ms \
    --vnet $vnetName \
    --subnet sql-subnet \
    --private-dns-zone $project-mysql.private.mysql.database.azure.com

Disables SSL - see footer for issues with phpipam

    az mysql flexible-server parameter set \
    --resource-group $MyResourceGroup \
    --server-name $project-mysql \
    --name require_secure_transport \
    --value OFF

Output the Subnet ID to update the yaml file

    az network vnet subnet list \
     --resource-group $MyResourceGroup \
     --vnet-name $vnetName \
     --query "[?name=='aci-subnet'].id"

Create the Container (remember to upload the file)

    az container create \
    --resource-group $MyResourceGroup \
    --file deploy-aci.yaml

Get the A record (remember to update the yml file)

![image](https://github.com/knowlesy/phpipam/assets/20459678/f41c799d-fe29-4dd4-955f-d0679b0224c7)

![image](https://github.com/knowlesy/phpipam/assets/20459678/a24c98fc-48a6-4b20-8f4c-513fdf696bdd)


Get the IP of the container 

    az container list --query "[?contains(name, 'phpipam')].[ipAddress.ip]" --output tsv

RDP to the server and browse to the IP 




*NOTE* 

SSL to mySql not working  - ensure SSL is disabled in mySql DB

MySQL Cert https://learn.microsoft.com/en-us/azure/mysql/single-server/how-to-configure-ssl

Possible requirement to update config.php file see https://github.com/phpipam/phpipam/issues/3550




