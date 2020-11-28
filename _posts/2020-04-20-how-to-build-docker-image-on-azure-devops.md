---
layout: post
title: How to build docker image on Azure DevOps?
date: 2020-04-20
excerpt_separator:  <!--more-->
tags: DevOps AzureDevOps continuous-integration .NETCore Docker ACR
featured_image_thumbnail:
featured_image: /assets/images/posts/2020/frank-mckenna-tjX_sniNzgQ-unsplash.jpg
featured: true
hidden: true
og_image: /assets/images/posts/2020/how-to-build-docker-image-on-azure-devops/og-image.png
---

Nowadays with Kubernetes being so popular, building a Docker image is a must thing for CI/CD pipeline. For this kind of pipelines, an artifact is not a simple zip file wich compiled application, but a Docker image pushed to container registry. There is plenty of benefits of this approach but there is also price for this. We need to handle this in our pipelines. Hopefully, this price is not high. And we will explore today how we can build a Docker image for our dotnet core web app on Azure DevOps.

<!--more-->

### DOCKERFILE

Our app is straightforward. It's nothing more than scaffolding provided by Visual Studio on creating dotnet core web app. So let's skip this part and move to DOCKERFILE. We use [multi-stage build](https://docs.docker.com/develop/develop-images/multistage-build/) where in the first stage we use `mcr.microsoft.com/dotnet/core/sdk:3.1` image as our build environment to switch later to `mcr.microsoft.com/dotnet/core/aspnet:3.1` as our runtime, just to provide a minimal image.

```dockerfile
FROM mcr.microsoft.com/dotnet/core/sdk:3.1 AS build-env
WORKDIR /app

# Copy csproj and restore as distinct layers
COPY *.csproj ./
RUN dotnet restore

# Copy everything else and build
COPY . ./
RUN dotnet publish -c Release -o out --no-restore

# Build runtime image
FROM mcr.microsoft.com/dotnet/core/aspnet:3.1
WORKDIR /app
COPY --from=build-env /app/out .
ENTRYPOINT ["dotnet", "SampleAppForDocker.dll"]
```

### Building Docker image on host agent

We will push our images to Azure Container Registry, but before that we need to create one.

Create first a resource group using az cli:

`
az group create --name TheCodeManual --location westeurope
`

Then create Azure container registry:

`
az acr create --resource-group TheCodeManual --name devopsmanual --sku Basic
`

I selected Basic tier. Primary differences between Basic and Standard are included storage and number of web hooks. For our puproses Basic is enough.

Now we need to define service connection on Azure DevOps. You will need to navigate to `Service connections*` under your `Project settings`. Then click `New service connection` select `Docker Registry` and fill form like below:

![Adding new ACR service connection](/images/2020-04-20-acr-service-connection.png)

Now we are ready to define our pipeline:

{% highlight yml %}
steps:
- task: Docker@2
  displayName: Login to ACR
  inputs:
    command: login
    containerRegistry: devopsmanual-acr

- task: Docker@2
  displayName: Build and Push
  inputs:
    repository: $(imageName)
    command: buildAndPush
    Dockerfile: build-docker-image/SampleAppForDocker/DOCKERFILE
    tags: |
      build-on-agent

- task: Docker@2
  displayName: Logout of ACR
  inputs:
    command: logout
    containerRegistry: devopsmanual-acr

{% endhighlight %}

We have here three very simple steps:

1. Login to Azure Container Registry
2. Build and push Docker image
3. Logout from Azure Container Registry

This build took 55 seconds. 

### Building Docker image on Azure Container Registry

Another approach is to use ACR tasks. But what is ACR task? [The codumentation](https://docs.microsoft.com/en-us/azure/container-registry/container-registry-tasks-overview#what-is-acr-tasks) explains this very well:

<div class="note-box">
  <p>
    ACR Tasks is a suite of features within Azure Container Registry. It provides cloud-based container image building for platforms including Linux, Windows, and ARM, and can automate OS and framework patching for your Docker containers. ACR Tasks not only extends your "inner-loop" development cycle to the cloud with on-demand container image builds, but also enables automated builds triggered by source code updates, updates to a container's base image, or timers. For example, with base image update triggers, you can automate your OS and application framework patching workflow, maintaining secure environments while adhering to the principles of immutable containers.
  </p>
</div>

So basicly it will allow us to offload some part of CI steps to ACR. In our case it will be build and push docker image. You may wonder how much does cost. At the momen of writing this text it is $0.0001/second per CPU. 

Our pipeline definition for this approach has single step which calls `az acr build`.

{% highlight yml %}
steps:
- task: AzureCLI@2
  displayName: Azure CLI
  inputs:
    azureSubscription: rg-the-code-manual
    scriptType: pscore
    workingDirectory: $(Build.SourcesDirectory)/build-docker-image/SampleAppForDocker/
    scriptLocation: inlineScript
    inlineScript: |
      az acr build --image sampleappfordocker:build-on-acr --registry devopsmanual --file DOCKERFILE .
{% endhighlight %}

This build took 94 seconds to create and publish the image. It is almost twice longer, however for such simple projects we should not pay too much attention to these results. The purpose here was to show how we can build image and for benchmarking we should do more than this.

### Building Docker image on host agent with Build Kit

Looking at options how we can build Docker images on Azure DevOps I found a [Build Kit](https://github.com/moby/buildkit) project. You may ask what it is. Let me cyte the offical site:

<div class="note-box">
  <p>
  BuildKit is a toolkit for converting source code to build artifacts in an efficient, expressive and repeatable manner.
  Key features:
    <ul>
      <li>Automatic garbage collection</li>
      <li>Extendable frontend formats</li>
      <li>Concurrent dependency resolution</li>
      <li>Efficient instruction caching</li>
      <li>Nested build job invocations</li>
      <li>Distributable workers</li>
      <li>Multiple output formats</li>
      <li>Pluggable architecture</li>
      <li>Execution without root privileges</li>
    </ul> 
  </p>
</div>

I recommend you read [this article](https://brianchristner.io/what-is-docker-buildkit/) which will give you general overview about Build Kit. But, to sum up, this is a software whch speed up bulding Docker images. We can enable [Build Kit on Azure DevOps](https://docs.microsoft.com/en-us/azure/devops/pipelines/ecosystems/containers/build-image?view=azure-devops#buildkit) by setting DOCKER_BUILDKIT in the pipeline.

{% highlight yml %}

variables:
  imageName: 'SampleAppForDocker'
  DOCKER_BUILDKIT: 1

pool:
  vmImage: 'ubuntu-latest'

steps:
- task: Docker@2
  displayName: Login to ACR
  inputs:
    command: login
    containerRegistry: devopsmanual-acr

- task: Docker@2
  displayName: Build and Push
  inputs:
    repository: $(imageName)
    command: buildAndPush
    Dockerfile: build-docker-image/SampleAppForDocker/DOCKERFILE
    tags: |
      build-with-build-kit

- task: Docker@2
  displayName: Logout of ACR
  inputs:
    command: logout
    containerRegistry: devopsmanual-acr

{% endhighlight %}

You may wonder how such simple change impacts on build performance ;) It took only 45 seconds.

### Summary
In this post, I presented two ways of building docker images. I will not point which is better. It all depends on your preferences. Besides, ACR tasks have more capabilities than I presented here, but this is for the next post.

#### Links

- [Docker multi-stage builds](
https://docs.docker.com/develop/develop-images/multistage-build/)
- [Automate container image builds and maintenance with ACR Tasks](https://docs.microsoft.com/en-us/azure/container-registry/container-registry-tasks-overview#what-is-acr-tasks)
- [What is Docker BuildKit and What can I use it for?](https://brianchristner.io/what-is-docker-buildkit/)