# Hands-on with Container Apps

The purpose of this repo is to help you quickly get hands-on with Container Apps. It is meant to be consumed either through GitHub codespaces or through a local Dev Container. The idea being that everything you need from tooling to runtimes is already included in the Dev Container so it should be as simple as executing a run command.

* **Date:** 10th January 2022
* **Squad:** Cloud Native
* **Duration:** 30 minutes

## Scenario

As a retailer, you want your customers to place online orders, while providing them the best online experience. This includes an API to receive orders that is able to scale out and in based on demand. You want to asynchronously store and process the orders using a queing mechanism that also needs to be auto-scaled. With a microservices architecture, Container Apps offer a simple experience that allows your developers focus on the services, and not infrastructure.

In this sample you will see how to:

1. Deploy the solution and configuration through IaaC, no need to understand Kubernetes
2. Ability to troubleshoot using built-in logging capability with Azure Monitor (Log Analytics)
3. Out of the box Telemetry with Dapr + Azure Monitor (Log Analytics)
4. Ability to split http traffic when deploying a new version
5. Ability to configure scaling to meet usage needs

![Image of sample application architecture and how messages flow through queue into store](/images/th-arch.png)

### Pre-requisites

There are two options:

1. [Access to GitHub Codespaces](#getting-started-via-codespaces)
1. [VS Code + Docker Desktop on Local Machine](#getting-started-via-vs-code-and-local-dev-container)

### Getting Started

As this is currently a preview service, you will need to install an Azure CLI extension to work with Container Apps.

Run the following command.

```bash
az extension add --name containerapp
```

We will be using the `hey` load testing tool later on. If you are using Codespaces, the container includes Homebrew, so you can install `hey` like this:

```bash
brew install hey
```

If you are using an environment other than Codespaces, you can find installation instructions for `hey` here - [https://github.com/rakyll/hey](https://github.com/rakyll/hey)

### Setup Solution

Let's start by setting some variables that we will use for creating Azure resources in this demo, and a resource group for those resources to reside in.

```bash
# Generate a random name

name=ca$(cat /dev/urandom | tr -dc '[:lower:]' | fold -w ${1:-5} | head -n 1)

# Set variables for the rest of the demo

resourceGroup=${name}-rg
location=northeurope
containerAppEnv=${name}-env
logAnalytics=${name}-la
appInsights=${name}-ai
acr=${name}acr
```

Optional -  if using Codespaces or not logged into Azure CLI

```bash
# Login into Azure CLI
az login --use-device-code

# Check you are logged into the right Azure subscription. Inspect the name field
az account show

# In case not the right subscription
az account set -s <subscription-id>

# in order to be able to use Azure CLI with container apps, we need to add the container apps cli extension
# more information about the azure container apps extension commandos: https://docs.microsoft.com/en-us/cli/azure/containerapp?view=azure-cli-latest

az extension add --name containerapp --upgrade

```

```bash

# Create Resource Group
az group create --name $resourceGroup --location $location -o table
```

### Deploy container registry

We will start with deployment of a Azure Container Registry to save images. 

```bash
az deployment group create \
  -g $resourceGroup \
  --template-file v0_template.bicep \
  --parameters containerRegistryName=$acr
```

### Deploy version 1 of the application

We'll deploy the first version of the application to Azure. This typically takes around 3 to 5 minutes to complete.

```bash
az deployment group create \
  -g $resourceGroup \
  --template-file v1_template.bicep \
  --parameters @v1_parametersbicep.json \
  --parameters \
    ContainerApps_Environment_Name=$containerAppEnv \
    LogAnalytics_Workspace_Name=$logAnalytics \
    AppInsights_Name=$appInsights \
    Container_Registry_Name=$acr 
```

Now the application is deployed, let's determine the URL we'll need to use to access it and store that in a variable for convenience

```bash
storeURL=https://storeapp.$(az containerapp env show -g $resourceGroup -n $containerAppEnv --query 'properties.defaultDomain' -o tsv)/store
```

Let's see what happens if we call the URL of the store with curl.

> Alternatively, you can run `echo $storeURL` to get the URL for the application and then open that in a browser.

```bash
curl $storeURL
```

The response you should see is `[]` which means no data was returned. Something's not working, but what?

ContainerApps integrates with Application Insights and Log Analytics. In the Azure Portal, go to the Log Analytics workspace in the resource group we're using for this demo and run the following query to view the logs for the `queuereader` application.

```text
ContainerAppConsoleLogs_CL
| where ContainerAppName_s has "queuereader" and ContainerName_s has "queuereader"
| where Log_s has "null"
| project TimeGenerated, ContainerAppName_s, RevisionName_s, ContainerName_s, Log_s
| order by TimeGenerated desc
```

Alternatively, if you prefer to stay in the CLI, you can run the Log Analytics query from there.

```bash
workspaceId=$(az monitor log-analytics workspace show -n $logAnalytics -g $resourceGroup --query "customerId" -o tsv)
az monitor log-analytics query -w $workspaceId --analytics-query "ContainerAppConsoleLogs_CL | where ContainerAppName_s has 'queuereader' and ContainerName_s has 'queuereader' | where Log_s has 'null' | project TimeGenerated, ContainerAppName_s, RevisionName_s, ContainerName_s, Log_s | order by TimeGenerated desc"
```

You should see a number of log file entries which will likely all contain the same error. Drill down on one of them to reveal more. You should see something like the following:

![Image of an Azure Log Analytics log entry showing an error from the queuereader application indicating that a config value is missing](/images/LogAnalyticsDaprPortError.png)

> "Log_s": "      Value cannot be null. (Parameter ''DaprPort' config value is required. Please add an environment variable or app setting.')",

Looks like we're missing a configuration value relating to Dapr. So, we've gone ahead and made the necessary changes to our code and packaged that up in version 2 of our application's container. Let's deploy version 2.

### Deploy Version 2

We'll repeat the deployment command from earlier, but we've updated our template to use version 2 of the queuereader application.

```bash
az deployment group create \
  -g $resourceGroup \
  --template-file v2_template.bicep \
  --parameters @v2_parametersbicep.json \
  --parameters ContainerApps_Environment.Name=$containerAppEnv \
    LogAnalytics_Workspace.Name=$logAnalytics \
    AppInsights_Name=$appInsights 
```

This time, we'll store the URL for the HTTP API application in a variable

```bash
dataURL=https://httpapi.$(az containerapp env show -g $resourceGroup -n ${name}-env --query 'properties.defaultDomain' -o tsv)/Data
```

Now let's see what happens if we access that URL

> As before, you can type `echo $dataURL` to get the URL of the HTTP API and then open it in a browser if you prefer

``` bash
curl $dataURL
```

The result tells us that `demoqueue` has no messages:

> `Queue 'demoqueue' has 0 messages`

We can call the HTTP API endpoint to add a test message.

```bash
curl -X POST $dataURL?message=test
```

Ok, let's check our Store URL and see what happens this time

```bash
curl $storeURL
```

> `[{"id":"f30e1eb6-d9d1-458b-b8d3-327e5597ffc7","message":"57e88c1e-f4a4-4c66-8eb5-128bb235b08d"}]`

Ok, that's something but that's not the message we sent. Let's take a look at the application code

**DataController.cs**

```c#
...
  [HttpPost]
  public async Task PostAsync()
  {
      try
      {
          await queueClient.SendMessageAsync(Guid.NewGuid().ToString());

          Ok();
      }
...
```

It looks like the code is set to send a GUID, not the message itself. Must have been something the developer left in to test things out. Let's modify that code:

**DataController.cs** (version 2)

```c#
  [HttpPost]
  public async Task PostAsync(string message)
  {
      try
      {
          await queueClient.SendMessageAsync(DateTimeOffset.Now.ToString() + " -- " + message);

          Ok();
      }
```

We've fixed the code so that the message received is now actually being sent and we've packaged this up into a new container ready to be redeployed.

But maybe we should be cautious and make sure this new change is working as expected. Let's perform a controlled rollout of the new version and split the incoming network traffic so that only 20% of requests will be sent to the new version of the application.

To implement the traffic split, the following has been added to the deployment template

```json
  "ingress": {
      "external": true,
      "targetPort": 80,
      "traffic":[
          {
              "revisionName": "[concat('httpapi--', parameters('ContainerApps.HttpApi.CurrentRevisionName'))]",
              "weight": 80
          },
          {
              "latestRevision": true,
              "weight": 20
          }
      ]
```

Effectively, we're asking for 80% of traffic to be sent to the current version (revision) of the application and 20% to be sent to the new version that's about to be deployed.

### Deploy version 3

Once again, let's repeat the deployment command from earlier, now using version 2 of the HTTP API application and with traffic splitting configured

```bash
az deployment group create \
  -g $resourceGroup \
  --template-file v3_template.bicep \
  --parameters @v3_parametersbicep.json \
  --parameters ContainerApps_Environment_Name=$containerAppEnv \
    LogAnalytics_Workspace_Name=$logAnalytics \
    AppInsights_Name=$appInsights 
```

With the third iteration of our applications deployed, let's try and send another order.

```bash
curl -X POST $dataURL?message=secondtest
```

And let's check the Store application again to see if the messages have been received

```bash
curl $storeURL | jq
```

```json
[
  {
    "id": "b205d410-5150-4ac6-9e26-7079ebcae67b",
    "message": "a39ecc22-cece-4442-851e-25d7329a1f55"
  },
  {
    "id": "318e72bb-8c55-486c-99ec-18a5bd76bc1d",
    "message": "01/03/2022 13:28:33 +00:00 -- secondtest"
  }
]
```

That's looking better. We can still see the original message, but we can also now see our "test" message with the date and time appended to it.

We configured traffic splitting, so let's see that in action. First we will need to send multiple messages to the application. We can use the load testing tool `hey` to do that.

```bash
hey -m POST -n 25 -c 1 $dataURL?message=hello
```

```bash
curl $storeURL | jq
```

Let's check the application logs for the Queue Reader application

```text
ContainerAppConsoleLogs_CL
| where ContainerAppName_s has "queuereader" and ContainerName_s has "queuereader"
| where Log_s has "Message"
| project TimeGenerated, Log_s
| order by TimeGenerated desc
```

Looking through those logs, you should see a mix of messages, with some containing "hello" and others still containing a GUID. It won't be exact, but roughly one out of every five messages should contain "hello".

So, is our app ready for primetime now? Let's change things so that the new app is now receiving all of the traffic, plus we'll also setup some scaling rules. This will allow the container apps to scale up when things are busy, and scale to zero when things are quiet.

## Deploy version 4

One final time, we'll now deploy the new configuration with scaling configured. We will also add a simple dashboard for monitoring the messages flow.

```bash
az deployment group create \
  -g $resourceGroup \
  --template-file v4_template.bicep \
  --parameters @v4_parametersbicep.json \
  --parameters ContainerApps_Environment_Name=$containerAppEnv \
    LogAnalytics_Workspace_Name=$logAnalytics \
    AppInsights_Name=$appInsights 
```

First, let's confirm that all of the traffic is now going to the new application

```bash
hey -m POST -n 10 -c 1 $dataURL?message=testscale
```

Let's check the number of orders in the queue

```bash
curl $dataURL
```

As before, we can check the application log files in Log Analytics to see what messages are being received

```text
ContainerAppConsoleLogs_CL
| where ContainerAppName_s has "queuereader" and ContainerName_s has "queuereader"
| where Log_s has "Message"
| project TimeGenerated, Log_s
| order by TimeGenerated desc
```

Now let's see scaling in action. To do this, we will generate a large amount of messages which should cause the applications to scale up to cope with the demand.

> [Optional] While the scaling script is running, you can also have this operations dashboard open to visually see the messages flowing through queue into the store

> ```bash
> dashboardURL=https://dashboardapp.$(az containerapp env show -g $resourceGroup -n ${name}-env --query 'properties.defaultDomain' -o tsv)
> echo 'Open the URL in your browser of choice:' $dashboardURL
> ```

To demonstrate this, a script that uses the `tmux` command is provided in the `scripts` folder of this repository. Run the following commands:

```bash
cd scripts
./appwatch.sh $resourceGroup $dataURL
```

This will split your terminal into four separate views.

* On the left, you will see the output from the `hey` command. It's going to send 10,000 requests to the application, so there will be a short delay, around 20 to 30 seconds, whilst the requests are sent. Once the `hey` command finishes, it should report its results.
* On the right at the top, you will see a list of the container app versions (revisions) that we've deployed. One of these will be the latest version that we just deployed. As `hey` sends more and more messages, you should notice that one of these revisions of the app starts to increase its replica count
* Also on the right, in the middle, you should see the current count of messages in the queue. This will increase to 10,000 and then slowly decrease as the app works it way through the queue.

Once `hey` has finished generating messages, the number of instances of the HTTP API application should start to scale up and eventually max out at 10 replicas. After the number of messages in the queue reduces to zero, you should see the number of replicas scale down and return to 1.

> Tip! To exit from tmux when you're finished, type `CTRL-b`, then `:` and then the command `kill-session`

## Deploy version 5

Until now we have worked with docker images that others have created, now we are going to do a code change on our own code base and push the changes to Container Apps using GitHub Actions. [Learn more](https://docs.microsoft.com/en-us/azure/container-apps/github-actions-cli?tabs=bash)

We are now going to use a Azure CLI command to create a GitHub Action that builds the queuereader C# project and pushes the image to Azure Container Registry and deploys it to our Container App.

First, we need to create a service principal that can be used from the GitHub action to deploy the changes we are introducing.

```bash
az ad sp create-for-rbac \
  --name <SERVICE_PRINCIPAL_NAME> \
  --role "contributor" \
  --scopes /subscriptions/<SUBSCRIPTION_ID>/resourceGroups/<RESOURCE_GROUP_NAME> \
  --sdk-auth
```
The return value from this command is a JSON payload, which includes the service principal's `tenantId`, `clientId`, and `clientSecret`.

Set the variables in bash.

```bash
spClientid=[Replace with the clientId of the service principal]
spClientSecret=[Replace with the clientSecret of the service principal]
tenantid=[Replace with the tenantId of the service principal]
```
Now we need to get information about the Azure Container Registry that we created in the beginning.

Set the variables in bash.
```bash
acrUrl=$(az acr show -n $acr -g $resourceGroup --query 'loginServer' -o tsv)
acrUsername=$(az acr show -n $acr -g $resourceGroup --query 'name' -o tsv)
acrSecret=$(az acr credential show -n $acr -g $resourceGroup --query passwords[0].value -o tsv)
```

Now we need a GitHub Personal Access Token (PAT) so we can authenticate against GitHub from Azure CLI.

Go to github.com --> Settings --> Developer Settings --> Personal access tokens Click on ”Generate new token” button.
 
Password prompt might appear. Enter password.

In the “Note” textbox enter a name for the PAT, such as “ca-pat”.
Give the PAT the following scopes: 
-	repo (Full control of private repositories) 
-	workflows (Update GitHub Action workflows)

![pat](images/pat.png)


Click “Generate token”, copy the generated token and assign the variable. 
```bash
ghToken=[Replace with the PAT]
```
Now all the variables are set so we can run the Azure CLI command, make sure you are located at the root of the repo and run the following command.

```bash
az containerapp github-action add \
  --repo-url $repoUrl \
  --docker-file-path "./queuereaderapp/Dockerfile" \
  --branch main \
  --name queuereader \
  --resource-group $resourceGroup \
  --registry-url $acrUrl \
  --registry-username $acrUsername \
  --registry-password $acrSecret \
  --service-principal-client-id $spClientid \
  --service-principal-client-secret $spClientSecret \
  --service-principal-tenant-id $tenantid \
  --token $ghToken
```

The command will create a GitHub Action and run it, it takes a couple of minutes, please check the status at github.com --> Actions and see the progress of the GitHub Action after it has been created by the Azure CLI command.

Dive into the logs and locate the “latestRevisionName”, then go to the Azure portal and verify that the revision name is the same for the “queuereader” Container App.

![ghaction1](images/ghaction1.png)

![ghaction2](images/ghaction2.png)

![revision](images/revision.png)


Now it’s time to do a code change and validate that it has been deployed.

Open VS Code --> queuereaderapp folder --> Open Worker.cs and scroll down to line number 51, where we are writing to the log.  

```c#
logger.LogInformation($"Message ID: '{message.MessageId}', contents: '{message.Body?.ToString()}'");
```
Below this line insert the following code.

```c#
logger.LogInformation("This is a new log message!");
```
Then open the Terminal in VS Code and make sure you are in the “queuereaderapp” folder. Run this command.
```bash
dotnet build . 
```
Make sure that the build was succeeded.

Commit the change in VS Code.

![commit](images/commit.png)

After the commit, the previous created GitHub Action starts, follow the progress at github.com --> Actions.

After the deployment has succeeded, please verify that the revision number has changed using the Azure portal.

Now it’s time to validate that the changes we made has taken affect. Send a message to the API.

```bash
curl -X POST $dataURL?message=mynewlogmessage
```
Validate the change by looking in Log Analytics.

```text
ContainerAppConsoleLogs_CL
| where ContainerAppName_s has "queuereader" and ContainerName_s has "queuereader"
| where Log_s has "Message"
| project TimeGenerated, Log_s
| order by TimeGenerated desc
| limit 50
``` 

Here you should see one row with the text "This is a new log message!".

### Cleanup

Deleting the Azure resource group should remove everything associated with this demo.

```bash
az group delete -g $resourceGroup --no-wait -y
```

## Contributors

* Kevin Harris - kevin.harris@microsoft.com
* Mahmoud El Zayet - mahmoud.elzayet@microsoft.com
* Mark Whitby - mark.whitby@microsft.com
* Anu Bhattacharya - anulekha.bhattacharya@microsoft.com

* Jimmy Karlsson - jimmy.karlsson@microsoft.com
* Jonas Norlund - jonas.norlund@microsoft.com
* Peter Williams - pewill@microsoft.com
* Arash Rassoulpour - arash.rassoulpour@microsoft.com
* Anders Heden - anders.heden@microsoft.com
