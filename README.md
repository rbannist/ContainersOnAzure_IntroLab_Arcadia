# Intro to Containers on Azure
This lab aims to show a few ways you can quickly deploy container workloads to Azure. 
<br><br>
# What is it?

This intro lab serves to guide you on a few ways you can deploy a container on Azure, namely:

*	Deploy a container on App Service PaaS platform
*	Deploy a container on an Azure Container Instance (managed Kubernetes instance)
*	Deploy a managed Kubernetes cluster on Azure using Azure Kubernetes Service (AKS) and deploy our container onto it
* Write to Azure Cosmos DB. [Cosmos DB](https://azure.microsoft.com/en-us/services/cosmos-db/) is Microsoft's globally distributed, multi-model database 
* Use [Application Insights](https://azure.microsoft.com/en-us/services/application-insights/) to track custom events in the container
* Breaking things... intentionally...
* Deploy Helm to your Kubernetes cluster
<br><br>
# Technology used

* Our container contains a swagger enabled API developed in Go which writes a simple order via json to your specified Cosmos DB and tracks custom events via Application Insights.
<br><br>
# Preparing for this lab

For this Lab you will require:

* Install the Azure CLI 2.0, get it here - https://docs.microsoft.com/en-us/cli/azure/install-azure-cli
* Install Docker, get it here - https://docs.docker.com/engine/installation/
* Install Kubectl, get it here - https://kubernetes.io/docs/tasks/tools/install-kubectl/
* Install Postman, get it here - https://www.getpostman.com - this is optional but useful

When using the Azure CLI, after logging in, if you have more than one subscripton you may need to set the default subscription you wish to perform actions against. To do this use the following command:

```
az account set --subscription "<your requried subscription guid>"
```
<br><br>

Images key:

| **Line Colour** | **What is it showing?** |
| :-------- | :------ |
| Purple | The path of your access to various machines |
| Pink (dashed) | Azure Portal location / Resource created |
| Black | Console commands and locations |
| Dark Blue | Docker control plane |
| Green | Docker actions/events |
| Red | Standard 'Data Plane' comms |
| Yellow | Network (logical) |

Docker-specific key:

| **Character** | **Item** |
| :-------- | :------ |
| "i:" | Image |
| "c:" | Container (from image) |
| "p:" | Kubernetes Pod |

<br><br>

## 1. Provisioning a Cosmos DB instance

Let's start by creating a Cosmos DB instance in the portal or using CLI, this is a quick process.

### Portal:
Navigate to the Azure portal and create a new Azure Cosmos DB instance, enter the following parameters:

* ID: <yourdbinstance>
* API: Select MongoDB as the API as our container API will use this driver
* ResourceGroup: <yourresourcegroup>
* Location: westeurope

See below:
![alt text](https://github.com/shanepeckham/ContainersOnAzure_IntroLab/blob/master/images/CosmosDB.png)


Once the DB is provisioned, we need to get the Database Username and Password, these may be found in the Settings --> Connection Strings section of your DB. We will need these to run our container, so copy them for convenient access. See below:

![alt text](https://github.com/shanepeckham/ContainersOnAzure_IntroLab/blob/master/images/DBKeys.png)


### CLI:

```
az group create -n <yourresourcegroup> -l westeurope
```

```
az cosmosdb create -n <yourcosmosdbid> -g <yourresourcegroup> --kind MongoDB
```

```
az cosmosdb list-connection-strings -n <yourcosmosdbid> -g <yourresourcegroup>
```
-->  Store the output in a text file for later use.

<br><br>

If you wish to see the keys in isolation:

```
az cosmosdb list-keys -n <yourcosmosdbid> -g <yourresourcegroup>
```
<br><br>

![Cosmos DB Resource](images/ContainersOnAzure_IntroLab_1.png?raw=true)

<br><br>

## 2. Provisioning an Application Insights instance

In the Azure portal, select create new Application Insights instance, enter the following parameters:

* Name: <yourappinsightsinstance>
* Application Type: General
* ResourceGroup: <yourresourcegroup>
* Location: westeurope

See below:
![alt text](https://github.com/shanepeckham/ContainersOnAzure_IntroLab/blob/master/images/ApplicationInsights.png)

Once Application Insights is provisioned, we need to get the Instrumentation key, this may be found in the Overview section. We will need this to run our container, so copy it for convenient access. See below:

![alt text](https://github.com/shanepeckham/ContainersOnAzure_IntroLab/blob/master/images/AppKey.png)

<br><br>

![Application Insights Resource](images/ContainersOnAzure_IntroLab_2.png?raw=true)

<br><br>

## 3. Provisioning an Azure Container Registry instance

If you would like an example of how to setup an [Azure Container Registry](https://azure.microsoft.com/en-us/services/container-registry/) instance via ARM, have a look [here](https://github.com/shanepeckham/CADScenario_Recommendations)

### Portal:

Navigate to the Azure Portal and select create new Azure Container Registry, enter the following parameters:

* Registry Name: <yourcontainerregistryinstance>
* ResourceGroup: <yourresourcegroup>
* Location: westeurope
* Admin User: Enable
* SKU: Basic or Classic
* Storage Account: Select the default value provided (if Classic)

See below:
![alt text](https://github.com/shanepeckham/ContainersOnAzure_IntroLab/blob/master/images/ContainerRegistry.png)


### CLI:

```
az acr create -n <youracrname> -g <yourresourcegroup> --sku Basic --admin-enabled true -l westeurope
```

```
az acr show -n <youracrname> --query loginServer
```

-->  Save the ouput in the text file for later use.

<br><br>

![Azure Container Registry Resource](images/ContainersOnAzure_IntroLab_3.png?raw=true)

<br><br>

## 4. Pull the container to your environment and set the environment keys

Open up your docker machine in Guacamole and type the following:

```
sudo docker pull shanepeckham/go_order_sb
```
<br>

![Pull Original Container Image](images/ContainersOnAzure_IntroLab_4.png?raw=true)

<br>

We will now test the image locally to ensure that it is working and connecting to our CosmosDB and Application Insights instances. The keys you copied for the DB and App Insights keys are set as environment variables within the container, so we will need to ensure we populate these.

The environment keys that need to be set are as follows:
* DATABASE: <your cosmodb username from step 1>
* PASSWORD: <your cosmodb password from step 1>
* INSIGHTSKEY: <you app insights key from step 2>
* SOURCE: This is a free text field which we will use specify where we are running the container from. The values 'Localhost', 'AppService', 'ACI' and 'K8' are applicable to our tests.

So, to run the container on your local machine, enter the following command, substituting your environment variable values:

```
sudo docker run --name go_order_sb -p 8080:8080 --network=host -e DATABASE="<your cosmodb username from step 1>" -e PASSWORD="<your cosmodb password from step 1>" -e INSIGHTSKEY="<you app insights key from step 2>" -e SOURCE="localhost"  -i -t shanepeckham/go_order_sb
```
Note, the application runs on port 8080 which we will bind to the host as well (-p switch).  However, we're running the container directly on the 'host network' as well in this example (i.e. on 172.31.0.0/24) in order to access it from the neighbouring windows machine(s).

<br>

![Test Container Run Locally](images/ContainersOnAzure_IntroLab_5.png?raw=true)

<br>

If you wish to explore docker host's networks enter the following commands

```
sudo docker network ls

sudo docker network inspect [<bridgenetworkid> | <hostnetworkid>]
```

If all goes well, you should see the application running on (dockervmipaddress):8080.  'ifconfig -a' on the Docker VM will give you the IP address that you need.  See below for an example:
![alt text](https://github.com/shanepeckham/ContainersOnAzure_IntroLab/blob/master/images/localrun.png)

Now you can navigate to (dockervmipaddress):8080/swagger and test the api. Select the 'POST' /order/ section, select the button "Try it out" and enter some values in the json provided and select "Execute", see below:
![alt text](https://github.com/shanepeckham/ContainersOnAzure_IntroLab/blob/master/images/swagger.png)

If the request succeeded, you will get a CosmosDB Id returned for the order you have just placed, see below:
![alt text](https://github.com/shanepeckham/ContainersOnAzure_IntroLab/blob/master/images/swaggerresponse.png)

<br><br>

![Testing Container](images/ContainersOnAzure_IntroLab_6.png?raw=true)

<br><br>

You can also test using CURL on the Mgmt/'Mgmt VM'.  Firsty, install CURL:

```
choco install curl -y
```

Then send a POST call:

```
curl -X POST "http://<dockervmipaddress>:8080/v1/order/" -H  "accept: application/json" -H  "content-type: application/json" -d "{  \"EmailAddress\": \"<emailaddress<>\",  \"ID\": \"string\",  \"PreferredLanguage\": \"ENU\",  \"Product\": \"Latte\",  \"Source\": \"Localhost\",  \"Total\": 1}"
```
<br><br>

Enter 'Ctrl + C' to shutdown the running container before moving on.

We can now go and query CosmosDB to check our entry there, in the Azure portal, navigate back to your Cosmos DB instance and go to the section Data Explorer. We can now query for the order(s) that we placed. A collection called 'orders' will have been created within your database, you can then apply a filter for the id we created, namely:

```
{"id":"<orderid>"}
```

See below:

![alt text](https://github.com/shanepeckham/ContainersOnAzure_IntroLab/blob/master/images/CosmosQuery.png)

<br><br>

## 5. Retag the image and upload it your private Azure Container Registry

Navigate to the Azure Container Registry instance you provisioned within the Azure portal. Click on the *Quick Start* blade, this will provide you with the relevant commands to upload a container image to your registry, see below:

![alt text](https://github.com/shanepeckham/CADScenario_Recommendations/blob/master/images/quicksstartacs.png)

Now we will push the image up to the Azure Container Registry, enter the following (from the quickstart screen):

``` 
sudo docker login <yourcontainerregistryinstance>.azurecr.io -u <your acr admin username> -p <your acr admin password>
```

<br><br>

![Login to Azure Container Registry](images/ContainersOnAzure_IntroLab_7.png?raw=true)

<br><br>

To get the username and password, navigate to the *Access Keys* blade, see below:

![alt text](https://github.com/shanepeckham/CADScenario_Recommendations/blob/master/images/acskeys.png)

You will receive a 'Login Succeeded' message. Now type the following:
```
sudo docker tag shanepeckham/go_order_sb <yourcontainerregistryinstance>.azurecr.io/go_order_sb
sudo docker push <yourcontainerregistryinstance>.azurecr.io/go_order_sb
```

<br><br>

![Tag Image](images/ContainersOnAzure_IntroLab_8.png?raw=true)

<br><br>

![Push Image to Azure Container Registry](images/ContainersOnAzure_IntroLab_9.png?raw=true)

<br><br>

Once this has completed, you will be able to see your container uploaded to the Container Registry within the portal, see below:

![alt text](https://github.com/shanepeckham/ContainersOnAzure_IntroLab/blob/master/images/registryrepo.png)

<br><br>

## 6. Deploy the container to App Services

We will now deploy the container to Azure App Services via the Azure CLI. If you would like an example of how to setup an [App Service Application](https://docs.microsoft.com/en-us/azure/app-service-web/app-service-linux-intro) instance via ARM and associate the container with your Azure Container Registry, have a look [here](https://github.com/shanepeckham/CADScenario_Recommendations)

[Login to your Azure subscription via the Azure CLI](https://docs.microsoft.com/en-us/cli/azure/authenticate-azure-cli) and enter the following first command to create your App service plan:

```
az appservice plan create -g <yourresourcegroup> -n <yourappserviceplan> --is-linux
```

<br><br>

![Create App Service Plan for Web App for Containers Instance](images/ContainersOnAzure_IntroLab_10.png?raw=true)

<br><br>

Upon receiving the 'provisioningState': 'Succeeded' json response, enter the following to create your app which will run our API:

```
az webapp create -n <youruniquewebappname> -p <yourappserviceplan> -g <yourresourcegroup> --deployment-container-image-name <yourcontainerregistryinstance>.azurecr.io/go_order_sb
```

Upon receiving the successful completion json response, we will now associate our container from our private Azure Registry to the App Service App, type the following):

```
az webapp config container set -n <youruniquewebappname> -g <yourresourcegroup> --docker-custom-image-name <yourcontainerregistryinstance>.azurecr.io/go_order_sb:latest --docker-registry-server-url https://<yourcontainerregistryinstance>.azurecr.io --docker-registry-server-user <your acr admin username> --docker-registry-server-password <your acr admin password>
```

<br><br>

![Create Web App using image stored in ACR](images/ContainersOnAzure_IntroLab_11.png?raw=true)

<br><br>

### Associate the environment variables with API App

Now we need to go and set the environment variables for our container to ensure that we can connect to our Cosmos DB and Application Insights. Navigate to the *Application Settings* pane within the Azure portal for your Web App and add the following entries in the 'App Settings' section, namely:

The environment keys that need to be set are as follows:
* DATABASE: <your cosmodb username from step 1>
* PASSWORD: <your cosmodb password from step 1>
* INSIGHTSKEY: <you app insights key from step 2>
* SOURCE: This is a free text field which we will use specify where we are running the container from.
* WEBSITES_PORT: 8080 

See below:
![alt text](https://github.com/shanepeckham/ContainersOnAzure_IntroLab/blob/master/images/AppSettingsWeb.png)

Or, using Azure CLI:

```
az webapp config appsettings set -n <youruniquewebappname> -g <yourresourcegroup> --settings "DATABASE=<your cosmodb username from step 1>" "PASSWORD=<your cosmodb password from step 1>" "INSIGHTSKEY=<you app insights key from step 2>" "SOURCE=AppServices" "WEBSITES_PORT=8080"
```

<br><br>

![Set Web App environmental settings](images/ContainersOnAzure_IntroLab_12.png?raw=true)

<br><br>

Now we can test our app service container, the URL should be:
```
https://<youruniquewebappname>.azurewebsites.net/swagger
```
but you can also navigate to the Overview section to get the URL for your API, see below:

![alt text](https://github.com/shanepeckham/CADLab_Loyalty/blob/master/Images/App_URI.png)

Ensure you add ```/swagger``` on to the end of the URL to access the Swagger API test harness.

<br><br>

![Test Web App for Containers instance](images/ContainersOnAzure_IntroLab_13.png?raw=true)

<br><br>

### Stream the logs from the App Service container

To see the log stream of your container running in the web app, navigate to:
```
https://<youruniquewebappname>.scm.azurewebsites.net/api/logstream
```

<br><br>

## 7. Deploy the container to Azure Container Instance

Now we will deploy our container to [Azure Container Instances](https://azure.microsoft.com/en-us/services/container-instances/). 

In the command terminal, login using the AZ CLI and we will start off by creating a new resource group for our Container instance. At the time of writing this functionality is still in preview and is thus not available in all regions (it is currently available in westeurope, eastus, westus).

Enter the following:

```
az group create --name <yourACIresourcegroup> --location westeurope
```

### Associate the environment variables with Azure Container Instance

We will now deploy our container instance via an ARM template, which is [here](https://github.com/rbannist/ContainersOnAzure_IntroLab/blob/master/azuredeploy.json) but before we do, we need to edit this document to ensure we set our environment variables.


In the document, the following section needs to be amended, adding your environment keys like you did before:

```

"properties": {
                "containers": [
                    {
                        "name": "[variables('container1name')]",
                        "properties": {
                            "image": "[variables('container1image')]",
                            "environmentVariables": [
                                {
                                    "name": "DATABASE",
                                    "value": "<your cosmodb username from step 1>"
                                },
                                {
                                    "name": "PASSWORD",
                                    "value": "<your cosmodb password from step 1>"
                                },
                                {
                                    "name": "INSIGHTSKEY",
                                    "value": "<you app insights key from step 2>"
                                },
                                {
                                    "name": "SOURCE",
                                    "value": "ACI"
                                }
                            ],

```


Once this document is saved, we can create the deployment via the az CLI. Enter the following:

```
az group deployment create --name <yourACIname> --resource-group <yourACIresourcegroup> --template-file /<path to your file>/azuredeploy.json
```

It is also possible to create the container instance via the Azure CLI directly.

```
az container create -n go-order-sb -g <yourACIresourcegroup> -e DATABASE=<your cosmodb username from step 1> PASSWORD=<your cosmodb password from step 1> INSIGHTSKEY=<your app insights key from step 2> SOURCE="ACI"--image <yourcontainerregistryinstance>.azurecr.io/go_order_sb:latest --registry-password <your acr admin password>
```

<br><br>

![Run on Azure Container Instanced](images/ContainersOnAzure_IntroLab_14.png?raw=true)

<br><br>

You can check the status of the deployment by issuing the container list command:

```
az container show -n go-order-sb -g <yourACIresourcegroup> -o table
```

Once the container has moved to "Succeeded" state you will see your external IP address under the "IP:ports" column, copy this value and navigate to http://yourACIExternalIP:8080/swagger and test your API like before.

<br><br>

## 8. Deploy the container to an Azure Kubernetes Service (AKS) cluster

In this exercise we will deploy an Azure Kubernetes Service (AKS) cluster.  AKS provides a managed Kubernetes environment on Azure.  'Master' nodes (managed) are 'free' and managed by Microsoft and 'Worker' nodes are exposed. We will run a container using our image on a newly provisioned Kubernetes cluster.

We will start by once again creating a resource group for our cluster.  In a command window enter the following:

```
az group create --name <yourresourcegroupk8> --location westeurope
```

AKS provides a highly-simplified Kubernetes cluster deployment proces.  One command is needed to get started:

```
az aks create -n <youraksname> -g <yourresouregroupk8>
```

<br><br>

![Create AKS Cluster](images/ContainersOnAzure_IntroLab_15.png?raw=true)

<br><br>

The kubernetes client should already be installed on your 'Mgmt VM' but, should you ever need to install it, this command will do that for you:

```
az aks install-cli
```

Once the deployment has completed, you will be able to connect to your new cluster using the following command:

```
az aks get-credentials -g <yourresouregroupk8> -n <youraksname>
```

We can then verify the connection by taking a look at the Worker nodes in the cluster using the following command:

```
kubectl get nodes
```
<br>
### Verifying using the Kubernetes dashboard UI

You can also now access the Kubernetes dashboard UI.  Enter the following at your command prompt (the link should automatically opened after hitting enter but, if not, please browse to http://127.0.0.1:8001/api/v1/namespaces/kube-system/services/kubernetes-dashboard/proxy/ using a browser on your 'Mgmt VM'):

```
az aks browse -g <yourresourcegroupk8> -n <youraksname>
```
<br>
##### Note.
It is always a good idea to apply an auto shutdown policy to your VMs to avoid unnecessary costs for a test cluster, you can do this in the portal by navigating to the VMs provisioned within your resource group <yourresourcegroupk8> and navigating to the Auto Shutdown section for each one, see below:

![alt text](https://github.com/shanepeckham/ContainersOnAzure_IntroLab/blob/master/images/autoshutdown.png)
<br><br>
### Register our Azure Container Registry within Kubernetes

We now want to register our private Azure Container Registry on our Kubernetes cluster to ensure that we can pull images from it. Enter the following in your command window:

```
kubectl create secret docker-registry <yourcontainerregistryinstance>.azurecr.io --docker-server=<yourcontainerregistryinstance>.azurecr.io --docker-username=<yourcontainerregistryinstanceusername> --docker-password=<yourcontainerregistryinstancepassword> --docker-email=<youremailaddress>
```
<br><br>

![Login AKS cluster to ACR](images/ContainersOnAzure_IntroLab_16.png?raw=true)

<br><br>

In the Kubernetes dashboard you should now see this created within the 'Secrets' section.  Click on 'Secrets' under the 'Config and Storage' section to see your new entry:

![alt text](https://github.com/shanepeckham/ContainersOnAzure_IntroLab/blob/master/images/K8secrets.png)
<br><br>
### Associate the environment variables with the container that we want to deploy to Kubernetes

We will now deploy our container via a yaml file, which is [here](https://github.com/shanepeckham/ContainersOnAzure_IntroLab/blob/master/go_order_sb.yaml). However, before we do, we need to edit this file to ensure we set our environment variables and ensure that you have set your private Azure Container Registry correctly.  Fill out the relevant lines and then save the file to your user profile directory as 'go_order_sb.yaml'.

```

spec:
      containers:
      - name: goordersb
        image: <containerregistry>.azurecr.io/go_order_sb
        env:
        - name: DATABASE
          value: "<your cosmodb username from step 1>"
        - name: PASSWORD
          value: "<your cosmodb password from step 1>"
        - name: INSIGHTSKEY
          value: ""<you app insights key from step 2>"
        - name: SOURCE
          value: "K8"
        ports:
        - containerPort: 8080
      imagePullSecrets:
        - name: <yourcontainerregistry>

```


Once the yaml file has been updated, we can now deploy our container. Within the command line enter the following:

```
kubectl create -f ./<your path>/go_order_sb.yaml
```

Use the following command to wait for the service to be fully provisioned ('EXTERNAL-IP' from 'pending' to <i>an ip address</i>):
```
kubectl get service goordersb --watch
```
'Ctrl+c' to break.

<br><br>

![Deploy to AKS](images/ContainersOnAzure_IntroLab_17.png?raw=true)

<br><br>

You should get a success message that a deployment and service has been created. Navigate back to the Kubernetes dashboard and you should see the following:
<br><br>
#### Your deployments running 

![alt text](https://github.com/shanepeckham/ContainersOnAzure_IntroLab/blob/master/images/k8deployments.png)
<br><br>
#### Your three pods

![alt text](https://github.com/shanepeckham/ContainersOnAzure_IntroLab/blob/master/images/k8pods.png)
<br><br>
#### Your service and external endpoint

![alt text](https://github.com/shanepeckham/ContainersOnAzure_IntroLab/blob/master/images/k8service.png)
<br><br>
You can now navigate to http://k8serviceendpoint:8080/swagger and test your API

<br><br>

![Test AKS Pods](images/ContainersOnAzure_IntroLab_18.png?raw=true)

<br><br>