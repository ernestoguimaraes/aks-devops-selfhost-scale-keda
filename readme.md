# Autoscaling Azure DevOps Self Hosted Agents on AKS with KEDA

This repo contains the necessary files and instructions do deploy an Azure DevOps Self-Hosted agent on AKS and enables Auto-Scaling with KEDA.

### Requirements

1. Running instance of Azure Kubernetes Services
2. Azure Container Register
3. Knowledge on managing containers and Kubernetes deployment
4. Have Docker on your dev machine



### Content

[deployment-selfagent.yml](deployment-selfagent.yml) - This file deploys the Self-Hosted Agent from Container Register on AKS 

[Dockerfile](Dockerfile) - Builds the Self-Hosted agent Image

[Keda_AutoScale_Config_Jobs.yml](Keda_AutoScale_Config_Jobs.yml) - This Options setups a Job on Keda that scales the Self-hosted Pod.

[Keda_AutoScale_Config.yml](Keda_AutoScale_Config.yml) - This option setup a Scaled Object on Keda. This is the recommended one. When applied, one Pipeline Job will trigger on Pod and destrou the Pod after the pipeline execution.

[Sample_parallel_pipeline.yml](Sample_parallel_pipeline.yml) - A sample pipeline that runs parallel jobs to demonstrate the scalability.


&nbsp;
## 1. Deploy Keda on AKS

The first step is to deploy Keda. I recommend to visit GitHub release pages from Keda and check for the last version. In a quick way, you can deploy it using the command below:


**GitHub Releases from Keda - Get the last version**
https://github.com/kedacore/keda/releases

```
kubectl apply -f https://github.com/kedacore/keda/releases/download/v2.2.0/keda-2.2.0.yaml
```
&nbsp;
## 2. Register a new Self-Hosted Agent Pool

The following article explains the Self-Hosted agent and how to deploy it. 

https://docs.microsoft.com/en-us/azure/devops/pipelines/agents/v2-windows?view=azure-devops#:~:text=Prepare%20permissions%201%20Information%20security%20for%20self-hosted%20agents,use%20has%20permission%20to%20register%20the%20agent.%20

One important step: it's necessary to create a PAT (Personal Access Token) on DevOps. Create and note the secret. This will be used on the config files.

**How to create PAT  on Azure DevOps** - https://docs.microsoft.com/en-us/azure/devops/organizations/accounts/use-personal-access-tokens-to-authenticate?view=azure-devops&tabs=Windows

&nbsp;

## Get the AgentPoolID

When you create a Agent Pool, there is a PoolID associated with it and it´s necessary for the future steps. 

Access the URL in your browser, logged with your AZ DevOps organization account and retrieve the ID from Agent Pool


**Url to get the Agend Pool ID** - 
https://dev.azure.com/<your org>/_apis/distributedtask/pools?poolname=<your agent name>

&nbsp;

## Build and Deploy Self-hosted image to Container Register

The [Dockerfile](Dockerfile)  on this repo creates the image that will host the Azure DevOps Agent. 

Follow the steps to build and publish in your ACR.

```
# Build Docker Image.
docker build  -t dockeragent:v1 .

# Login to ACR with your credentials.
az acr login --name <acrName>

# Tag the Azure DevOps Pipeline image 
 docker tag dockeragent:v1 <acrName>.azurecr.io/dockeragent:v1

# List the docker images
$ docker images
#
# Push the image to ACR
$ docker push <acrName>.azurecr.io/dockeragent:v1
```

&nbsp;

## Setup ScaledObject on Keda

Once Keda is deployed on AKS, you need to setup a Scaleb Object.Keda recommends to use this approach because in a perspective on Auto Scaling, there is a wish to scale on POD per Pipeline Job and once its finish, the POD is destroyed.

The other option can be used when you want to setup to run more pipelines jobs in a POD. Once its finish, by inactivity, the POD is destroyed.

If you see the file start.sh, you noticed the --once flag. This is the parameter necessary to tell the image to run once. It´s a essential step to enable the capabilty of Pod destruction on pipeline conclusion.

``` 
./run-docker.sh  "$@" & wait $! --once
```

Be aware that you need to update on file with your data. Look for '< >' and replace with the correct info.

```
 kubectl apply -f Keda_AutoScale_Config.yml
```

One important step is to improve the number of parallel jobs in a Self Hosted Environment. By default, it's on Job per time and no cost associated.

The following article explains how to enable them and gives some insights and how to define the number of jobs that you need.


https://docs.microsoft.com/en-us/azure/devops/pipelines/licensing/concurrent-jobs?view=azure-devops&tabs=ms-hosted




&nbsp;&nbsp;

## References
https://docs.microsoft.com/en-us/azure/devops/pipelines/agents/docker?view=azure-devops

https://keda.sh/docs/2.6/concepts/scaling-jobs/

