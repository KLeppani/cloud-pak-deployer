# Create an Azure Service Principal

## Login to Azure
Login to the Microsoft Azure using your subscription-level credentials.
``` { .bash .copy }
az login
```

If you have a subscription with multiple tenants, use:
``` { .bash .copy }
az login --tenant <TENANT_ID>
```

Example:
``` { .bash .copy }
az login --tenant 869930ac-17ee-4dda-bbad-7354c3e7629c8
To sign in, use a web browser to open the page https://microsoft.com/devicelogin and enter the code AXWFQQ5FJ to authenticate.
[
  {
    "cloudName": "AzureCloud",
    "homeTenantId": "869930ac-17ee-4dda-bbad-7354c3e7629c8",
    "id": "72281667-6d54-46cb-8423-792d7bcb1234",
    "isDefault": true,
    "managedByTenants": [],
    "name": "Azure Account",
    "state": "Enabled",
    "tenantId": "869930ac-17ee-4dda-bbad-7354c3e7629c8",
    "user": {
      "name": "your_user@domain.com",
      "type": "user"
    }
  }
]
```

## Set subscription (optional)

If you have multiple Azure subscriptions, specify the relevant subscription ID: `az account set --subscription <SUBSCRIPTION_ID>`

You can list the subscriptions via command:
``` { .bash .copy }
az account subscription list
```

```output
[
  {
    "authorizationSource": "RoleBased",
    "displayName": "IBM xxx",
    "id": "/subscriptions/dcexxx",
    "state": "Enabled",
    "subscriptionId": "dcexxx",
    "subscriptionPolicies": {
      "locationPlacementId": "Public_2014-09-01",
      "quotaId": "EnterpriseAgreement_2014-09-01",
      "spendingLimit": "Off"
    }
  }
]
```

## Create service principal

Create the service principal that will do the installation and assign the `Contributor role`

### Set environment variables for Azure

``` { .bash .copy }
export AZURE_SUBSCRIPTION_ID=72281667-6d54-46cb-8423-792d7bcb1234
export AZURE_LOCATION=westeurope
export AZURE_RESOURCE_GROUP=pluto-01-rg
export AZURE_SP=pluto-01-sp
```

- `AZURE_SUBSCRIPTION_ID`: The id of your Azure subscription. Once logged in, you can retrieve this using the `az account show` command.
- `AZURE_LOCATION`: The Azure location of the resource group, for example `useast` or `westeurope`.
- `AZURE_SP`: Azure service principal that is used to create the resources on Azure.

### Create a service principal with subscription permissions
In the situation where you have full access to the Azure subscription, you can create a service principal with subscription-level roles.
``` { .bash .copy }
az ad sp create-for-rbac \
  --role Contributor \
  --name ${AZURE_SP} \
  --scopes /subscriptions/${AZURE_SUBSCRIPTION_ID} | tee /tmp/${AZURE_SP}-credentials.json
```

Example output:
```output
{
  "appId": "a4c39ae9-f9d1-4038-b4a4-ab011e769111",
  "displayName": "pluto-01-sp",
  "password": "xyz-xyz",
  "tenant": "869930ac-17ee-4dda-bbad-7354c3e7629c8"
}
```

Set the permissions of the service principal to allow creation of the OpenShift cluster.
``` { .bash .copy }
az role assignment create \
  --role "User Access Administrator" \
  --scope /subscriptions/${AZURE_SUBSCRIPTION_ID} \
  --assignee-object-id $(az ad sp list --display-name=${AZURE_SP} --query='[].id' -o tsv)
```

### Create a service principal with resource group permissions
If the service principal must not have permissions at the subscription-level, you can grant permissions for the resource group(s) the service principal needs to create resources in.

First you have to make sure that the resource group exists:
``` { .bash .copy }
az group create \
  --name ${AZURE_RESOURCE_GROUP} \
  --location ${AZURE_LOCATION}
```

``` { .bash .copy }
az ad sp create-for-rbac \
  --role Contributor \
  --name ${AZURE_SP} \
  --scopes /subscriptions/${AZURE_SUBSCRIPTION_ID}/resourceGroups/${AZURE_RESOURCE_GROUP} | tee /tmp/${AZURE_SP}-credentials.json
```

Example output:
```output
{
  "appId": "a4c39ae9-f9d1-4038-b4a4-ab011e769111",
  "displayName": "pluto-01-sp",
  "password": "xyz-xyz",
  "tenant": "869930ac-17ee-4dda-bbad-7354c3e7629c8"
}
```

Set the permissions of the service principal to allow creation of the OpenShift cluster.
``` { .bash .copy }
az role assignment create \
  --role "User Access Administrator" \
  --scope /subscriptions/${AZURE_SUBSCRIPTION_ID}/resourceGroups/${AZURE_RESOURCE_GROUP} \
  --assignee-object-id $(az ad sp list --display-name=${AZURE_SP} --query='[].id' -o tsv)
```

#### Set permissions for additional resource groups
There are scenarios where the permissions must be set for additional resource groups, for example when the Azure virtual network is in a different resource group than the OpenShift cluster. This is common in private network installations.

```
export AZURE_NETWORK_RESOURCE_GROUP=pluto-01-network-rg
```

Set the permissions of the service principal to allow updates to the network resource group and allow creation of the OpenShift cluster.
``` { .bash .copy }
az role assignment create \
  --role "Contributor" \
  --scope /subscriptions/${AZURE_SUBSCRIPTION_ID}/resourceGroups/${AZURE_NETWORK_RESOURCE_GROUP} \
  --assignee-object-id $(az ad sp list --display-name=${AZURE_SP} --query='[].id' -o tsv)

az role assignment create \
  --role "User Access Administrator" \
  --scope /subscriptions/${AZURE_SUBSCRIPTION_ID}/resourceGroups/${AZURE_NETWORK_RESOURCE_GROUP} \
  --assignee-object-id $(az ad sp list --display-name=${AZURE_SP} --query='[].id' -o tsv)
```
