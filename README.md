# Github Actions and Kubernetes on Azure

This tutorial explains how to set up a basic "infrastructure as code"-pipeline in Github to create a Kubernetes cluster in Azure using Azure Kubernetes Service, AKS.

## Azure subscription

To complete the tutorial, you need an Azure subscription. If you do not have one already, you can create a free 12-month subscription here: <https://azure.microsoft.com/free>

## Github account 

Also, you need a Github account. If you do not have one, create one for free here: <https://github.com/join>

At the end of the sign-up, you may be asked to create your first repository, but you don't have to that for now.

## Azure Portal

To make sure you are correctly setup with a working subscription, make sure you can log in to the Azure portal. Go to <https://portal.azure.com>

## Azure Cloud Shell

Azure Cloud Shell is a web based shell which has a lot of good tools pre-installed (like kubectl, az cli, helm, etc).

Start cloud shell by typing the address ````shell.azure.com```` into a web browser. If you have not used cloud shell before, you will be asked to create a storage location for cloud shell. Accept that and make sure that you run bash as your shell (not powershell).


## Create a Service Principal for Github
You need to somehow allow Github to act on "your behalf" when deploying things in Azure. The way this is done, is by creating a **Service Principal**. 

The service principal is simply an identity. You can configure this identity to be allowed to do various things in your azure subscription. 

The easiest way to create a service principal is by using **azure cli**, a.k.a ````az````. 

Lets use Azure CLoud shell to do this. In your cloud shell, do the following:

````bash
az ad sp create-for-rbac --sdk-auth --name githubSP
````
This will create a Service Principal that has the role **Contributor** in your Azure subscription. To read more about roles, have a look here: <https://docs.microsoft.com/en-us/azure/role-based-access-control/built-in-roles>

Output from this command is a json object, and should look something like the below. You will need this output in a later step, so keep it available somehow.
````json
{
  "clientId": "8e65cbad-bc58-4cc7-88f4-4eada9450c7d",
  "clientSecret": "869118ba-a0e1-4bd7-a746-fbeda735bdc6",
  "subscriptionId": "6f65205f-d352-482f-970b-a1d2a478fb64",
  "tenantId": "72f543bf-86f1-41af-91ab-2d7cd011db47",
  "activeDirectoryEndpointUrl": "https://login.microsoftonline.com",
  "resourceManagerEndpointUrl": "https://management.azure.com/",
  "activeDirectoryGraphResourceId": "https://graph.windows.net/",
  "sqlManagementEndpointUrl": "https://management.core.windows.net:8443/",
  "galleryEndpointUrl": "https://gallery.azure.com/",
  "managementEndpointUrl": "https://management.core.windows.net/"
}
````

For instance, copy this and paste it into a text editor. Do not share it with anyone, since this can be used to log into your Azure account. In other words: if you share it, prepare to have someone using your azure account to mine bitcoins :-)

## Create Service Principal for AKS

AKS will also need an identity in Azure, since it will create resources like Loadbalancers etc.

In the following example, the --skip-assignment parameter prevents any additional default assignments being assigned

````bash
az ad sp create-for-rbac --skip-assignment --name KubernetesSP
````

The output will look similar to this
````json
{
  "appId": "559513bd-0c19-4a1c-87cd-851a26afd5fc",
  "displayName": "KubernetesSP",
  "name": "http://KubernetesSP",
  "password": "e763725a-5eee-40e8-a466-dc88d91e2415",
  "tenant": "72f988bf-86f1-41af-91ab-2d7cd100db48"
}
````
Hang on to this, as it will be used in a later step.

## Create SSH key pair for AKS

If, by chance, you need to access the nodes in the AKS cluster, you connect using an SSH key pair. Use the ssh-keygen command to generate SSH public and private key files.

The following command creates an SSH key pair using RSA encryption and a bit length of 2048. The keys will be named ````aks-key```` and ````aks-key.pub```` and will be located in the directory from where you executed the command.

````bash
ssh-keygen -t rsa -b 2048 -f aks-key
````

In a later step, you will use this key, so make a mental note of this...

## Create Azure Keyvault

Azure Keyvault is a service that can be used to securely store secrets. 

To create a Keyvault you can type the following in cloud shell. The commands first create a Resource Group and then the Keyvault, and places it inside the Resource Group. The keyvault must have a unique name.  

````bash
az group create -l westeurope -n keyvault-rg
az keyvault create --location westeurope --name <unique keyvault name> --resource-group keyvault-rg
````

## Add AKS Service principal to keyvault

Since the Service Principal is a secret

Add the AKS Service Principal secret to your keyvault:

````bash
az keyvault secret set -n aks-sp-secret --vault-name <unique keyvault name> --value <service principal secret>
````

Where <service principal secret> is the password value from your service principal, e.g.

````bash
"e763725a-5eee-40e8-a466-dc88d91e2415"
````

Then add the Service Principal name as well, which (slightly confusingly) is called app-ip in this case.

````bash
az keyvault secret set -n aks-sp-name --vault-name <unique keyvault name> --value <service principal app-id>
````

Where <service principal app-id> is the app-id value from your service principal, e.g.

````bash
"559513bd-0c19-4a1c-87cd-851a26afd5fc"
````

## Add SSH key to keyvault

You should add the ssh key to keyvault, so that the pipeline can retrieve the key. The key pair you generated previously is located in the same directory from where you generated the keys (as mentioned previously).

To add the the key (assuming its named aks-key) to keyvault:

````bash
az keyvault secret set -n aks-ssh-key --vault-name <unique keyvault name> --file aks-key
````


## Fork Github repository

The easiest way to get the code for this tutorial, is by **forking** the Github repository where I put my code. Forking means that you copy the contents of my repository into a repository in your own github account.

To fork the repository, first navigate to <https://github.com/pelithne/aks-pipeline> then select the **fork** button in the top right corner.

<p align="left">
  <img width=100%" src="./media/fork.PNG">
</p>

You should now get a forked repository in your account, which looks similar to this:

<p align="left">
  <img width=100%" src="./media/forked-repo.PNG">
</p>

When you browse around the files, you will notice that the repository has some template files in the ````arm-templates```` folder, and a pipeline definition in ````.github/workflows````. 

The template files constitute an **ARM Template** (Azure Resource Management Template) and contain the definitions of the resources you will deploy to azure (in this case, a Kubernetes cluster). More about this later.

The file ````main.yaml```` in the workflows directory is the pipeline definition, which Github calls workflow. I will be using these terms interchangeably.

## Insert the Service Principal for Azure in Github

As mentioned before, in order for Github actions to be able to interact with Azure, it needs an identity. For this you will use the service principal you created in a previous step.

In order for Github to be "made aware" of this identity, and the associated credentials, you will add it into the **secrets vault** in Github. This secret will be referenced by the pipeline.

To import the secret to Github, first select **settings** to the right in your github toolbar.

<p align="left">
  <img width=100%" src="./media/settings.png">
</p>
 
Then selclick on **secrets** in the left hand navigation bar

<p align="left">
  <img width=30%" src="./media/secrets.PNG">
</p>

After this, select **Add new secret** and paste in the entire json object you (hopefully saved before).

Reminder: It should look similar to this:

````json
{
  "clientId": "8e65cbad-bc58-4cc7-88f4-4eada9450c7d",
  "clientSecret": "869118ba-a0e1-4bd7-a746-fbeda735bdc6",
  "subscriptionId": "6f65205f-d352-482f-970b-a1d2a478fb64",
  "tenantId": "72f543bf-86f1-41af-91ab-2d7cd011db47",
  "activeDirectoryEndpointUrl": "https://login.microsoftonline.com",
  "resourceManagerEndpointUrl": "https://management.azure.com/",
  "activeDirectoryGraphResourceId": "https://graph.windows.net/",
  "sqlManagementEndpointUrl": "https://management.core.windows.net:8443/",
  "galleryEndpointUrl": "https://gallery.azure.com/",
  "managementEndpointUrl": "https://management.core.windows.net/"
}
````

The name of the secret should be **AZURE_CREDENTIALS**. (it could be anything, but the pipeline definition expects this name, so if you name it differently there will be some extra hacking).

Don't forget to click **Add Secret**

## Activate Actions

When you forked the repository, you also copied the pipeline definition. However, github will deactivate the pipeline on forked repos, so you need to activate it again.

Do that by clicking on **actions** in the Github toolbar

<p align="left">
  <img width=100%" src="./media/actions.png">
</p>

You will be asked to to "go ahead and run" your workflows. Do that.

<p align="left">
  <img width=100%" src="./media/activate-actions.PNG">
</p>



In order for the pipeline to work properly, you need to update the parameter file in the ````arm-templates```` directory. This part of the tutorial is a bit clunky and will hopefully see some improvement ahead...

Browse to the file ````params.json````  in the ````arm-templates```` directory, then start editing the file by clicking the pen in the right corner:

<p align="left">
  <img width=30%" src="./media/pen.png">
</p>

You need to update the file to include references to your keyvault, which is defined in 3 different places in the file. Look for this:

````json
"keyVault": {
  "id": "/subscriptions/<Subscription ID>/resourceGroups/KeyVaultRG/providers/Microsoft.KeyVault/vaults/<Your unique keyvault name>"
},
````

And change the line to contain your subscription ID and your keyvault.

Then repeat for all the 3 occurrences of the keyvault parameter.

If you followed the naming suggestions for your AKS Service Principle, and AKS Secret you should not have to change anything else for the pipeline to work as it should.

Then commit the change by clicking the **start commit** button

<p align="left">
  <img width=15%" src="./media/start-commit.PNG">
</p>

Give the commit a *good* description and click on **Commit changes** to complete the commit.

This will "save" your change to the repository, and at the same time trigger off the pipeline that will build your AKS Kubernetes cluster.

## Actions

If all went well, a new workflow should have been triggered by your commit. You can follow the progress of that workflow under that actions tab on the Github toolbar.

The name of the workflow should be "AzureARMTest" and there should be a "job" named "build-infra". If you click on "build-infra" you can monitor the progress of the job (or possibly identify errors that prevents the pipeline from finishing...).

Creating an AKS cluster takes a while, perhaps 7-8 minutes. After this you can go into the Azure portal and confirm that your cluster is up and running.

## More details about the pipeline/workflow

If you are curious how the pipeline definition and ARM template works together, I will try to explain that in a little bit more detail.

First, lets start with the pipeline (a.k.a **workflow**). The definition looks something like this:

````yaml
on: [push]

env:
  AZURE_RESOURCE_GROUP: action-rg
  ARM_TEMPLATE_FILE: template.json
  ARM_PARAMETERS_FILE: params.json

name: AzureARMTest

jobs:
  build-infra:
    runs-on: ubuntu-latest
    steps: 
    # Azure login   
    - uses: azure/login@v1
      with:
        creds: ${{ secrets.AZURE_CREDENTIALS }}
    # Create Resource Group
    - run: az group create -l westeurope -n ${{ env.AZURE_RESOURCE_GROUP }}
    # Check out code to the agent
    - uses: actions/checkout@v1
    # create AKS cluster using ARM templates
    - run: az group deployment create -n test -g ${{ env.AZURE_RESOURCE_GROUP }} --template-file ${{ env.ARM_TEMPLATE_FILE }} --parameters params.json
      working-directory: ./arm-templates
````

The first line ````on: [push]```` tells us that the workflow will be executed when something is pushed to the repository. This is what triggered the worksflow to run when you previously committed a change to a file in the repository.

The next few lines are environment variables, which are used for substitution of values in the worksflow;

````yaml
env:
  AZURE_RESOURCE_GROUP: action-rg
  ARM_TEMPLATE_FILE: template.json
  ARM_PARAMETERS_FILE: params.json
````

After this, the ````jobs:```` tag indicates the start of the workflow, which is called ````build-infra````. 

````runs-on: ubuntu-latest```` means that this job will be run on an agent (a virtual machine) that has the latest Ubuntu OS. I a production grade pipeline, you would obviously want to be more specific than that, and use a specific Ubuntu version.

After ````steps```` comes the different, well, steps of the workflow. First we log into Azure using the credentials stored in Github:

````yaml
# Azure login   
    - uses: azure/login@v1
      with:
        creds: ${{ secrets.AZURE_CREDENTIALS }}
````

Then a resource group is created, using the **Azure CLI** with the ````az group create```` command. The resource group is the "container" for the resources in Azure. All resources that exist in Azure must "live" inside a **Resource Group**. Note that the name is picked up from the environment variable ````AZURE_RESOURCE_GROUP````, and that the location for the RG is in **West Europe**.

````yaml
# Create Resource Group
    - run: az group create -l westeurope -n ${{ env.AZURE_RESOURCE_GROUP }}
````

The next section checks out the "code" (which is actually the ARM templates that define the infrastructure) from the repository to make sure it's available on the build agent where the workflow is running

````yaml
# Check out code to the agent
    - uses: actions/checkout@v1
````

Finally, the AKS cluster is created, using Azure CLI

````yaml
# create AKS cluster using ARM templates
    - run: az group deployment create -n test -g ${{ env.AZURE_RESOURCE_GROUP }} --template-file ${{ env.ARM_TEMPLATE_FILE }} --parameters params.json
      working-directory: ./arm-templates
````

This last command, uses two files as input, a template file and a parameter file. The template file contains the basic definitions of the ARM template, and the parameters file contains parameters which are inserted into the template.

The template file, which is located in ````./arm-templates/template.json```` looks like this (most of the content is reasonably self-explanatory but if I find the time I will try to break it down some more)

````yaml
{  
    "$schema":"https://schema.management.azure.com/schemas/2019-04-01/deploymentTemplate.json#",
    "contentVersion":"1.0.0.0",
    "parameters": {
        "resourceName": {
            "type": "string",
            "defaultValue":"AKS",
            "metadata": {
                "description": "The name of the Managed Cluster resource."
            }
        },
        "location": {
            "type": "string",
            "defaultValue": "[resourceGroup().location]",
            "metadata": {
                "description": "The location of the Managed Cluster resource."
            }
        },
        "dnsPrefix": {
            "type": "string",
            "metadata": {
                "description": "Optional DNS prefix to use with hosted Kubernetes API server FQDN."
            }
        },       
        "agentCount": {
            "type": "int",
            "defaultValue": 3,
            "metadata": {
                "description": "The number of nodes for the cluster."
            },
            "minValue": 1,
            "maxValue": 50
        },
        "agentVMSize": {
            "type": "string",
            "defaultValue": "Standard_B2s",
            "metadata": {
                "description": "The size of the Virtual Machine."
            }
        },
        "sshRSAPublicKey": {
            "type": "string",
            "metadata": {
                "description": "Configure all linux machines with the SSH RSA public key string. Your key should include three parts, for example 'ssh-rsa AAAAB...snip...UcyupgH azureuser@linuxvm'"
            }
        },
        "servicePrincipalClientId": {
            "metadata": {
                "description": "Client ID (used by cloudprovider)"
            },
            "type": "securestring"
        },
        "servicePrincipalClientSecret": {
            "metadata": {
                "description": "The Service Principal Client Secret."
            },
            "type": "securestring"
        },       
        "kubernetesVersion": {
            "type": "string",
            "defaultValue": "1.15.7",
            "allowedValues": [
                "1.13.11",
                "1.14.1",
                "1.14.7",
                "1.14.8",
                "1.15.5",
                "1.15.7"
            ],
            "metadata": {
                "description": "The version of Kubernetes."
            }
        }
    },
    "resources":[  
       {  
         "apiVersion": "2019-04-01",
          "type":"Microsoft.ContainerService/managedClusters",
          "location":"[parameters('location')]",
          "name":"[parameters('resourceName')]",
          "properties":{  
            "kubernetesVersion": "[parameters('kubernetesVersion')]",
            "dnsPrefix": "[parameters('dnsPrefix')]",
             "agentPoolProfiles":[  
                {  
                   "name":"agentpool1",
                   "count": "[parameters('agentCount')]",
                   "vmSize": "[parameters('agentVMSize')]",
                   "storageProfile":"ManagedDisks",
                   "osType":"Linux",
                   "maxPods":30,
                   "type":"VirtualMachineScaleSets",
                   "availabilityZones":[  
                      "1",
                      "2",
                      "3"
                   ]
                }
             ],
             "networkProfile": {
                 "networkPlugin": "kubenet",
                 "networkPolicy": "",
                 "podCidr": "10.244.0.0/16",
                 "serviceCidr":"10.0.0.0/16",
                 "dnsServiceIP": "10.0.0.10",
                 "dockerBridgeCidr": "172.17.0.1/16",
                 "loadBalancerSku": "standard"
             },            
             "linuxProfile":{  
                "adminUsername":"peter",
                "ssh":{  
                   "publicKeys":[  
                      {  
                        "keyData": "[parameters('sshRSAPublicKey')]"
                      }
                   ]
                }
             },
             "servicePrincipalProfile":{  
                "clientId": "[parameters('servicePrincipalClientId')]",
                "secret": "[parameters('servicePrincipalClientSecret')]"
             }
          }
       }
    ],
    "outputs":{  
       "controlPlaneFQDN":{  
          "type":"string",
          "value":"[reference(parameters('resourceName')).fqdn]"
       }
    }
 }
````

As you may have noticed, the template file contains references to the **parameters**. These are defined in the parameters file. The parameter file is located in ````./arm-templates/params.json```` and looks like this:

````yaml
{
  "$schema": "https://schema.management.azure.com/schemas/2015-01-01/deploymentParameters.json#",
  "contentVersion": "1.0.0.0",
  "parameters": {
        "resourceName": {
      "value": "k8s"
    },
    "dnsPrefix": {
      "value": "dns"
    },
        "sshRSAPublicKey": {
            "reference": {
                "keyVault": {
                  "id": "/subscriptions/6f66105f-d352-482f-970b-a1d2a478fb64/resourceGroups/KeyVaultRG/providers/Microsoft.KeyVault/vaults/pelithneKeyVault"
                },
                "secretName": "AKS-secret"
            }
        },
    "servicePrincipalClientId": {
      "value": "52b63a54-cdfb-44e7-8955-624c74ef11f3"
    },
    "agentVMSize": {
            "value": "Standard_B2s"
        },
        "agentCount": {
            "value": 3
        },
        "kubernetesVersion": {
            "value": "1.15.7"
        },
        "servicePrincipalClientSecret": {
            "reference": {
              "keyVault": {
                "id": "/subscriptions/6f66105f-d352-482f-970b-a1d2a478fb64/resourceGroups/KeyVaultRG/providers/Microsoft.KeyVault/vaults/pelithneKeyVault"
              },
              "secretName": "aks-sp"
            }
          }
  }
}
````