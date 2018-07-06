# ContainerTraining
This is a quick guide to walk you through getting up to speed using containers.

In this guide we will walk you through the following

 - Building an asp.net Core WebAPI
 - Creating a Docker container with your app
 - Running the container locally
 - publishing the container to Docker Hub
 - publishing the container to ACR (Azure Container Registry)
 - Running the container in Azure using ACS (Azure Container Service)
 - Running the container as part of a cluster using AKS ( Azure Kubernetes Service)
 
We will be pulling together hands on labs that will walk you through these steps with commentary. 



## Prerequisites

The first thing you will need to do is to get your machine prepped and ready. 

You will need the following things installed on your system to walk through these exercises.

Azure CLI [Azure CLI Directions](https://docs.microsoft.com/en-us/cli/azure/install-azure-cli?view=azure-cli-latest)

Docker for [Windows](https://docs.docker.com/v17.09/docker-for-windows/install/) or [Mac](https://docs.docker.com/v17.09/docker-for-mac/install/)

Kubectl [Full Directions](https://kubernetes.io/docs/tasks/tools/install-kubectl/). --  [Easiest way](https://docs.microsoft.com/en-us/cli/azure/acs/kubernetes?view=azure-cli-latest) Must have Azure CLI installed 

[Postman](https://www.getpostman.com/) for testing and making calls to our API

##### If you are running on Mac or want to use VS Code

[VSCode for Windows, Mac, or Linux](https://code.visualstudio.com/download) or other text editor (Vim, Sublime, etc...) 

C# for Visual Studio Code [https://marketplace.visualstudio.com/items?itemName=ms-vscode.csharp]

[.Net Core 2.1 SDK ](https://www.microsoft.com/net/download/all)  for running our asp.net core projects



## Getting Started
The first thing we need to to set up a asp.net core WebAPI to have something to call when we testing out containers.  For our purposes, this needs to be asp.net core since we will be using kubernetes and running linux containers.  If you need to run windows containers and classic ASP.net then you could use Azure Service Fabric (not covered in this training)  [Service Fabric Quick Start]( https://docs.microsoft.com/en-us/azure/service-fabric/service-fabric-quickstart-containers)


##### Using the following tutorial to create a simple todo WebAPI app using VS Code  (or just use the one in the repository)



[Create a Web API with ASP.NET Core and Visual Studio Code](https://docs.microsoft.com/en-us/aspnet/core/tutorials/web-api-vsc?view=aspnetcore-2) <span style="color:red"> SKIP THE CALLING FROM JQUERY SECTION</span>

Now that we have create an asp.net core WebApi we want to prepare it for docker.  

There are many images to use as base images for docker when using ASP.net Core. We are going to be using two of them.  One for building microsoft/dotnet:2.1.300-sdk  and one for deployment 


- [microsoft/dotnet:2.1.0-runtime-deps](https://andrewlock.net/exploring-the-net-core-2-1-docker-files-dotnet-runtime-vs-aspnetcore-runtime-vs-sdk/#1microsoftdotnet210runtimedeps) - use for deploying self-contained deployment apps

- [microsoft/dotnet:2.1.0-runtime](https://andrewlock.net/exploring-the-net-core-2-1-docker-files-dotnet-runtime-vs-aspnetcore-runtime-vs-sdk/#2microsoftdotnet210runtime) - use for deploying .NET Core console apps

- [microsoft/dotnet:2.1.0-aspnetcore-runtime](https://andrewlock.net/exploring-the-net-core-2-1-docker-files-dotnet-runtime-vs-aspnetcore-runtime-vs-sdk/#3microsoftaspnetcore210aspnetcoreruntime) - use for deploying ASP.NET Core apps

- [microsoft/dotnet:2.1.300-sdk](https://andrewlock.net/exploring-the-net-core-2-1-docker-files-dotnet-runtime-vs-aspnetcore-runtime-vs-sdk/#4microsoftdotnet21300sdk) - use for building .NET Core (or ASP.NET Core apps)

You can find out more information about these different builds [here](https://andrewlock.net/exploring-the-net-core-2-1-docker-files-dotnet-runtime-vs-aspnetcore-runtime-vs-sdk/) 












