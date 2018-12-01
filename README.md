# ContainerTraining
This is a quick guide to walk you through getting up to speed using containers.

In this guide we will walk you through the following

 - Building an asp.net Core WebAPI
 - Putting your WebAPI in a Docker Container
 - Running the container locally
 - publishing the container to Docker Hub
 - publishing the container to ACR (Azure Container Registry)
 - Deploy the container using a web app in azure
 - Setting up Continuous Deployment for your container in a Web App
 - Running the container in Azure using ACI (Azure Container Service)
 - Running the container as part of a cluster using AKS ( Azure Kubernetes Service)
 
We will be pulling together a mixture of hands on labs from the docs and HOLs in this document that will walk you through these steps with commentary. 



## Prerequisites

The first thing you will need to do is to get your machine prepped and ready. 

You will need the following things installed on your system to walk through these exercises.

Azure CLI [Azure CLI Directions](https://docs.microsoft.com/en-us/cli/azure/install-azure-cli?view=azure-cli-latest)

Docker for [Windows](https://docs.docker.com/v17.09/docker-for-windows/install/) or [Mac](https://docs.docker.com/v17.09/docker-for-mac/install/)

[A Dockerhub Account](https://hub.docker.com/) - This is usually created when you install Docker for Mac or Windows and you sign in to the app. 

Kubectl [Full Directions](https://kubernetes.io/docs/tasks/tools/install-kubectl/). --  [Easiest way](https://docs.microsoft.com/en-us/cli/azure/acs/kubernetes?view=azure-cli-latest) Must have Azure CLI installed 

[Postman](https://www.getpostman.com/) for testing and making calls to our API

##### If you are running on Mac or want to use VS Code

[VSCode for Windows, Mac, or Linux](https://code.visualstudio.com/download) or other text editor (Vim, Sublime, etc...) 

C# for Visual Studio Code [https://marketplace.visualstudio.com/items?itemName=ms-vscode.csharp]

[.Net Core 2.1 SDK ](https://www.microsoft.com/net/download/all)  for running our asp.net core projects



## Creating an asp.net core WebAPI
The first thing we need to to set up an asp.net core WebAPI to have something to call when we testing out containers.  For our purposes, this needs to be asp.net core since we will be using kubernetes and running linux containers.  If you need to run windows containers and classic ASP.net then you could use Azure Service Fabric (not covered in this training)  [Service Fabric Quick Start]( https://docs.microsoft.com/en-us/azure/service-fabric/service-fabric-quickstart-containers)


##### Using the following tutorial to create a simple todo WebAPI app using VS Code  (or just use the one in the repository ToDoV1)



[Create a Web API with ASP.NET Core and Visual Studio Code](https://docs.microsoft.com/en-us/aspnet/core/tutorials/web-api-vsc?view=aspnetcore-2) <span style="color:red"> SKIP THE CALLING FROM JQUERY SECTION</span>

Now that we have created an asp.net core WebApi we want to prepare it for docker.  

## Running your asp.net core WebAPI in Docker

The first thing we want to do is to make some modifications to our project to prepare it for containerization. In our ***program.cs*** file, we need to add the ***.UserUrls()*** to our ***CreateWebHostBuilder*** method. By default, the app will listen to the localhost, ignoring any incoming requests from outside the container.  By adding "http://0.0.0.0:5000" or "http://*:5000" it will allow it to listen outside the container. There are many ways you can do this. This by far is the easiest. (We will also *EXPOSE* it in the Dockerfile too to allow it to be accessed in the cloud)

<pre><code>
        public static IWebHostBuilder CreateWebHostBuilder(string[] args) =>
            WebHost.CreateDefaultBuilder(args)
                <b>.UseUrls("http://0.0.0.0:5000")</b>
                .UseStartup<Startup>();
</code></pre>
--

Next, we want to to create a file called **Dockerfile**  (no extension) and place it in the root of your project.  [Dockerfile best practices](https://docs.docker.com/develop/develop-images/dockerfile_best-practices/)


There are many images to use as a base images for docker when using ASP.net Core. We are going to be using two of them. Docker 17.05 and higher allows multi-stage builds. 
 
One for building - **microsoft/dotnet:2.1.300-sdk**  
And one for deployment **microsoft/dotnet:2.1.0-aspnetcore-runtime**

The reason for this is we want the smallest possible size for deployment.  The SDK image is **1.73GB** vs **255MB** for the runtime.

Here are all the possible images. 


- [microsoft/dotnet:2.1.0-runtime-deps](https://andrewlock.net/exploring-the-net-core-2-1-docker-files-dotnet-runtime-vs-aspnetcore-runtime-vs-sdk/#1microsoftdotnet210runtimedeps) - use for deploying self-contained deployment apps

- [microsoft/dotnet:2.1.0-runtime](https://andrewlock.net/exploring-the-net-core-2-1-docker-files-dotnet-runtime-vs-aspnetcore-runtime-vs-sdk/#2microsoftdotnet210runtime) - use for deploying .NET Core console apps

- [microsoft/dotnet:2.1.0-aspnetcore-runtime](https://andrewlock.net/exploring-the-net-core-2-1-docker-files-dotnet-runtime-vs-aspnetcore-runtime-vs-sdk/#3microsoftaspnetcore210aspnetcoreruntime) - use for deploying ASP.NET Core apps

- [microsoft/dotnet:2.1.300-sdk](https://andrewlock.net/exploring-the-net-core-2-1-docker-files-dotnet-runtime-vs-aspnetcore-runtime-vs-sdk/#4microsoftdotnet21300sdk) - use for building .NET Core (or ASP.NET Core apps)

You can find out more information about these different builds [here](https://andrewlock.net/exploring-the-net-core-2-1-docker-files-dotnet-runtime-vs-aspnetcore-runtime-vs-sdk/) with a great post by Andrew Lock.  Or the official documentation [Here](https://docs.microsoft.com/en-us/dotnet/standard/microservices-architecture/net-core-net-framework-containers/official-net-docker-images) 

The first thing we need to add to the Dockerfile we create is the base image. As stated previously, we will be using the build image.  We name it build-env to reference it later in the file. 

The file itself will only have the code sections that you need to type, inbetween we explain what each section is for. Start by typing the following at the top of the file you just created (dockerfile)

<pre><code>
<b>FROM microsoft/dotnet:2.1.300-sdk AS build-env</b>

</code></pre>

Next we set the working directory and EXPOSE port 5000. (text in **Bold**)  This is where we will be building the app

<pre><code>
FROM microsoft/dotnet:2.1.300-sdk AS build-env
<b>WORKDIR /app
EXPOSE 5000</b>

</code></pre>

After that we copy the project file over and use the dotnet cli to call restore
<pre><code>
FROM microsoft/dotnet:2.1.300-sdk AS build-env
WORKDIR /app
EXPOSE 5000

<b># copy csproj and restore as distinct layers
COPY *.csproj ./
RUN dotnet restore</b>
</code></pre>

Then copy everything else and use the dotnet cli on the image to publish a release version of the app.
<pre><code>
FROM microsoft/dotnet:2.1.300-sdk AS build-env
WORKDIR /app
EXPOSE 5000

# copy csproj and restore as distinct layers
COPY *.csproj ./
RUN dotnet restore
<b>
# copy everything else and build
COPY . ./
RUN dotnet publish -c Release -o out</b>
</code></pre>

Now since everything is published we can use the runtime version to create the image. We set the working directory, copy the files from the out directory we created in the other image, and set the ENTRYPOINT (use dotnet cli to run ToDoV1.dll) 

<pre><code>
FROM microsoft/dotnet:2.1.300-sdk AS build-env
WORKDIR /app
EXPOSE 5000

# copy csproj and restore as distinct layers
COPY *.csproj ./
RUN dotnet restore

# copy everything else and build
COPY . ./
RUN dotnet publish -c Release -o out
<b>
# build runtime image
FROM microsoft/dotnet:2.1.0-aspnetcore-runtime
WORKDIR /app
COPY --from=build-env /app/out .
ENTRYPOINT ["dotnet", "TodoApi.dll"]
</b>
</code></pre>
> **IMPORTANT** You want to make sure that the dll referenced in the last line of your dockerfile (TodoApi.dll in this case) matches what will be produced by your application.  Meaning if you named your application something other than TodoApi then you will need to change this line.  If you dont, you will get a strange error like the following <b>Did you mean to run dotnet SDK commands? Please install dotnet SDK from</b> and you wont know what is causing it. 





We also want to create a <b>.dockerignore</b>  (don't forget the '.' in front of the name) this will make sure we keep the image as fast and as small as possible by ignoring files we don't care about. Place it in the root of your project directory (same as Dockerfile). We are excluding the bin and obj folders, as well as stuff associated with VSCode and git,  but you can add anything you don't need packaged in the container. 
<pre><code><b>.dockerignore
.env
.git
.gitignore
.vs
.vscode
*/bin
*/obj</b>
</code></pre>
--

Now we want to create our images using docker at the command line.  Open up your terminal (Command Window, PowerShell, Terminal), CD (Change Directory) into the folder that holds your project (and Dockerfile),  and type the following commands

Don't forget the '.' at the end. This will build our image according to our Dockerfile and give it a name of todov1.  It will be tagged as todov1:latest.  If we want our own tag we just add it in the command todov1:testversion or todov1:version1.0

```
-> docker build -t todov1 . 
```

It should produce something similar to this. You can see it creating the directories, restoring packages, creating the dll, copying files, destroying the build image, etc...
<pre><code>
Daniels-MacBook-Pro-2:ToDoV2AddingDocker danielegan$ docker build -t todov1 .
Sending build context to Docker daemon  1.302MB
Step 1/10 : FROM microsoft/dotnet:2.1.300-sdk AS build-env
 ---> 90a5a2ee9755
Step 2/10 : WORKDIR /app
 ---> Using cache
 ---> b2d282cb0d82
Step 3/10 : COPY *.csproj ./
 ---> 09cf1314b867
Step 4/10 : RUN dotnet restore
 ---> Running in 930012dd0c50
  Restoring packages for /app/ToDoV1.csproj...
  Generating MSBuild file /app/obj/ToDoV1.csproj.nuget.g.props.
  Generating MSBuild file /app/obj/ToDoV1.csproj.nuget.g.targets.
  Restore completed in 1.66 sec for /app/ToDoV1.csproj.
Removing intermediate container 930012dd0c50
 ---> b67933ac170b
Step 5/10 : COPY . ./
 ---> 64255972a9a7
Step 6/10 : RUN dotnet publish -c Release -o out
 ---> Running in 8098ed84f05d
Microsoft (R) Build Engine version 15.7.179.6572 for .NET Core
Copyright (C) Microsoft Corporation. All rights reserved.

  Restoring packages for /app/ToDoV1.csproj...
  Generating MSBuild file /app/obj/ToDoV1.csproj.nuget.g.props.
  Generating MSBuild file /app/obj/ToDoV1.csproj.nuget.g.targets.
  Restore completed in 656.92 ms for /app/ToDoV1.csproj.
  ToDoV1 -> /app/bin/Release/netcoreapp2.1/ToDoV1.dll
    ToDoV1 -> /app/out/
Removing intermediate container 8098ed84f05d
 ---> b92dc13c5e46
Step 7/10 : FROM microsoft/dotnet:2.1.0-aspnetcore-runtime
 ---> 76e0c475bb4c
Step 8/10 : WORKDIR /app
 ---> Using cache
 ---> 9acdd401e590
Step 9/10 : COPY --from=build-env /app/out .
 ---> 30f9a9c5e76a
Step 10/10 : ENTRYPOINT ["dotnet", "todoapp.dll"]
 ---> Running in e70e31f68997
Removing intermediate container e70e31f68997
 ---> 29928295685e
Successfully built 29928295685e
Successfully tagged todov1:latest
</code></pre>

--
Next we can run the container. 

```
 -> docker run -it --rm -p 5000:5000 -e "ASPNETCORE_URLS=http://+:5000" --name To_Do_App todov1
```

- -it is running it interactively at the command prompt (as opposed to detached)(i interactive t terminal)
- --rm automatically removes the container at exit (to clean up your local environment)
- -p is setting up the port to connect your local port of 5000 to the port 5000 on the container
- -e is setting an environment variable (we dont really need this since we set it in the program.cs file
- --name is of the container
- and lastly, the name you gave it in the previous step. 

To prove to yourself that the container is running the app on your machine, open a browser and navigate to http://localhost:5000/api/todo.

If you want to look around the container you can have it give you a bash prompt when you run it.  Normally you can just add /bin/bash  to the end of the command but if you have an entrypoint defined (we do) you have to run the command like this.

```
--> docker run -it --rm -p 5000:5000  --name ToDo_App --entrypoint /bin/bash todov1 
```


That will give you a bash prompt right where your files are in the container. The working directory we defined, the "app" folder.

To exit, you will need to type <b>exit</b> at the bash prompt (as opposed to Ctrl + C)

## Uploading to DockerHub
Once you have your docker container running locally, you will want to put it in a repository so that it can be used. You have many options for this and usually it depends on a couple of things. First, is this an image that will be open source and be used as either a base image or a starter image? Or is this a personal/company image that needs to stay private.






If your container is meant to be consumed (ala base image) DockerHub is probably the best place to host your container.  It allows easy access to it and works seamlessly in the Dockerfile.  If your container is a private container then you need to know where they are running. Your containers should be registered in the same cloud you use for hosting them. 

- Azure Container Registry (ACR)
- AWS EC2 Container Registry (ECR)
- Google Container Registry
- IBM Cloud Container Registry
- Hosting your own in the cloud or locally with Docker Registry

In this section, we are going to assume you want the world to see, and use your container, so we are going to host it in DockerHub. 

--
The first thing we need to do is to go to DockerHub https://hub.docker.com and create a repository to store our image. 


![](https://raw.githubusercontent.com/DanielEgan/ContainerTraining/master/images/CreateRepository.png)

Fill out the form with the desired information. 

![](https://raw.githubusercontent.com/DanielEgan/ContainerTraining/master/images/CreateRepositoryForm.png)

Now we can tag and push our image from the command line 

You should already be signed in and your username and password should be in keychain(mac) or credential manager(windows) from when you signed into Docker for Windows/Mac.  For more information on securely signing into Docker see [THIS](https://docs.docker.com/engine/reference/commandline/login/) page.


Open up your command line and type the following command (replacing it with your Docker Account Name)

You are using docker to tag the image we created (todov1) to associate it with your docker account.  

```
-> docker tag todov1 <_YourDockerAccountName_>/todov1
```

Next we want to push it to our DockerHub account. Run the following command at the command line. 

```
-> docker push <_YourDockerAccountName_>/todov1
```


When it is done you should see something like the following.

![](https://raw.githubusercontent.com/DanielEgan/ContainerTraining/master/images/CompletedPush.png)

You can check https://hub.Docker.com to verify it is there.  You now have your Docker Container in DockerHub for the world to use.


If we want to pull the image from  Docker Hub to your local machine, we need to type 

```
-> docker pull accountname/imagename:tag.
```

If you don’t specify the tag, you are going to pull the image tagged :latest


## Uploading your container to ACR (Azure Container Registry)

Before we can upload our container to the Azure Container Registry (ACR) we need to actually create the registry.  There are three ways to do this:

1. Use the Azure Portal to create the resource group and the registry ([Instructions here](https://docs.microsoft.com/en-us/azure/container-registry/container-registry-get-started-portal))
2. Use Powershell ([Instructions Here](https://docs.microsoft.com/en-us/azure/container-registry/container-registry-get-started-powershell))
3. User the Azure CLI 

We are going to do it using the Azure CLI.  One of the prerequisites before starting this tutorial was to have the Azure ClI installed. In addition to this, we need to also make sure that the version of the Azure CLI that we have is version 2.0.27 or higher.  Type the following command into your terminal.

```
-> az --version
```

This will give you not only the version of the az cli but also all the command line interfaces that the az cli utilizes. 

![](https://raw.githubusercontent.com/DanielEgan/ContainerTraining/master/images/azversion.png)


 Make sure the it is above 2.0.27.  If not, go to the link in the prereqs and upgrade/install it.
 
#### Log in to az
 
If you have not used the Azure CLI before you will need to log in using the command 

```
-> az login
```

This will print out the following line with a code to authenticate the azure cli with your Azure subscription.  If you have already done this, you can skip this step. 

**_To sign in, use a web browser to open the page https://microsoft.com/devicelogin and enter the code SOMECODEHERE to authenticate_**

If you want to test if you are signed in you can run the command to list the resource groups in your subscription.

```
-> az group list -o table
```

This will give you a list (in table format) of all your resource groups.

![](https://raw.githubusercontent.com/DanielEgan/ContainerTraining/master/images/listtables.png)

#### Create resource group and registry

Now that we have confirmed we are logged in to azure from the command line we can very quickly create our resource group and our registry.

To create the resource group type the following command

```
-> az group create --name todov1rg --location eastus

```

>Remember this resource group, we will be using it often during this workshop.** 

Now we can create our registry.  Just type the following command into the terminal. 

```
-> az acr create --resource-group todov1rg --name <_YourRegistryName_> --sku Basic
```


The registry name needs to be unique, so use <your email alias>todov1registry. 

We are using the Basic sku for the registry which works well for testing.  There is also **Standard** and **Premium** skus.  [You can read about them here](https://docs.microsoft.com/en-us/azure/container-registry/container-registry-skus).

When it returns you should see the follow output. 

![](https://raw.githubusercontent.com/DanielEgan/ContainerTraining/master/images/registrycreate.png)

#### Upload our image to ACR

The first thing we need to do is to log into the ACR that we just created. You can do that with the following command using the name we created in the last step. 

```
-> az acr login --name <_YourRegistryName_>
```

You should receive a **Login Succeeded** when it is complete. 

To push an image to ACR you need to have it locally.  You could pull it from github and tag it but we already have ours local.  We just need to tag it close to the same way when we uploaded it to Dockerhub.

We need to issue the following command using docker. 

First, just to remind us of the name of the local image, not the one we tagged for Dockerhub, run the following command to see a list of all of our local images.

```
-> docker image ls
```


You want to tag the todov1.  Not the ones that are already tagged for Dockerhub. (Your name will differ)

![](https://raw.githubusercontent.com/DanielEgan/ContainerTraining/master/images/taggedimages.png)


To tag it for ACR run the following fully qualified command. (which includes the .azurecr.io)

```
-> docker tag todov1 <_YourRegistryName_>.azurecr.io/todov1
```


remember if you don't add a tag (with a colon like this todov1:beta) to the end of the image name it will be tagged as todov1:latest

Finally, you can push the images using docker push.

```
-> docker push <_YourRegistryName_>.azurecr.io/todov1
```


When it completes, you should see output similar to the following. 

![](https://raw.githubusercontent.com/DanielEgan/ContainerTraining/master/images/pushsuccess.png)

If you want to see it in the repository, you can run the following command.

```
->az acr repository list --name <_YourRegistryName_> --output table
```

Or view it on the Azure portal http://portal.azure.com 

![](https://raw.githubusercontent.com/DanielEgan/ContainerTraining/master/images/portalacr.png)

Next we will run it in an Azure Web App.

## Deploying your image as and Azure Web App

We are not going to be spending time on what an Azure Web App is other than acknowledging that it is a place in azure that allows you to host websites, apis, and now containers, with the ability to easily scale them, create minor CD/CI, among other things. We are going to use one as one way to host our simple todo api container.  

For this section we are going to move from the command line and use the visual tools at http://portal.azure.com.  Once you sign on to your account, click on the **Create a resource** link and search for web app (or just navigate to it).  Click on where you see **Web App**  

![](https://raw.githubusercontent.com/DanielEgan/ContainerTraining/master/images/webapp1.png)

On the next page, simply click on the **Create** button.

![](https://raw.githubusercontent.com/DanielEgan/ContainerTraining/master/images/webapp2.png)

When the WebApp blade comes up you will need to do the following :

- Give it a name (must be unique across azure
- Select your subscription
- Create or use a Resource Group (we created **todov1rg** in other exercises)
- Select an **App Service plan/Location** -- Select a region that is in or near your hosted container
- Configure Container
 - Select **Single Container**
 - Select **Azure Container Registry** -- Although as you can see, you can select others
 - Fill in all the dropdowns (nothing in Startup File)
 - Click **OK**


![](https://raw.githubusercontent.com/DanielEgan/ContainerTraining/master/images/webapp3.png)

When your resource comes up you should be able to navigate to the url provided. Make sure you add the **/api/todo** to the end of the url

![](https://raw.githubusercontent.com/DanielEgan/ContainerTraining/master/images/webapp4.png)

You should see the same json returned that we saw locally.

![](https://raw.githubusercontent.com/DanielEgan/ContainerTraining/master/images/webapp5.png)

If you do not see the json, you may need to configure Azure to use port 5000. Go to the Application Settings blade. Click Add new setting. Add an App Setting of PORT and set its value to 5000. Then click Save at the top of the page.

![](https://raw.githubusercontent.com/DanielEgan/ContainerTraining/master/images/webapp6.png)

Next, we will set up CD for your container in a web app

## Setting up Continuous Deployment for your container in a Web App

There are multiple ways to enable CI/CD piplelines for your containers.  The most robust woudl be to use Azure Dev Ops and pipelines to implement gates and checks along the way. For this lab though, we are going to the ACR webhook.  Also, we are only setting up CI/CD for the staging slot.  In this simple scenario, we want to be able to swap staging and production manually once we are confident that the new container works properly. 

So to set up the webhook you will need to do the following:

Sign into the Azure portal if you have not done so already.

Navigate to the webapp we created in the previous lab.

Click on Deployments Slots (Or Deployment Slots Preview) and click on Add Slot
![](https://raw.githubusercontent.com/DanielEgan/ContainerTraining/master/images/cicd1.png)

Create a slot called Staging.  Under Configuration Source, select your production slot (it should just say the name of the webapp) then click OK. 

![](https://raw.githubusercontent.com/DanielEgan/ContainerTraining/master/images/cicd2.png)

This will clone the existing webapp and add the current container to this slot. Wait for this to complete.
When the stage appears, click on the stage to open up the overview. 

![](https://raw.githubusercontent.com/DanielEgan/ContainerTraining/master/images/cicd3.png)

As you can see, the URL for this slot is the same as your production webapp with -staging appended to it.  If you would like to verify that it is working correctly, you can click on the URL to open it (dont forget to append it with /api/todo)

![](https://raw.githubusercontent.com/DanielEgan/ContainerTraining/master/images/cicd4.png)

Once you are done testing, click on container settings for the staging slot you are on (not the main webapp production slot)
Under Continuous Deployment click the ON toggle and select save.  This will create a webhook.

![](https://raw.githubusercontent.com/DanielEgan/ContainerTraining/master/images/cicd5.png)


Next, navigate to the registry you created (todov1registry) and select Webhooks.  
Click the elipses (...) at the end of your webhook. 

![](https://raw.githubusercontent.com/DanielEgan/ContainerTraining/master/images/cicd6.png)

 From here you can make configuration changes.  For example you can set the scope, which tells the webhook which tagged image to look for.  If no tag is specified then :latest will be selected.

![](https://raw.githubusercontent.com/DanielEgan/ContainerTraining/master/images/cicd7.png)


If you would like to set up this same thing with Docker Hub you can follow the directions at the bottom of [This Page](https://docs.microsoft.com/en-us/azure/app-service/containers/app-service-linux-ci-cd)

If you would like to see how to use Azure DevOps to deploy your container you can look [Here](https://docs.microsoft.com/en-us/azure/devops/pipelines/apps/cd/deploy-docker-webapp?view=vsts)


Of course, now we want to test to make sure it works right?  The first thing we need to do is make some sort of change.  You can make any change you want to the web app but lets make this easy.  Go ito the ToDoController.cs file and change the word Item1 to Item1Chagned (or whatever you woudl like).  This is what we have been looking at when we test our service.

```
        public TodoController(TodoContext context)
        {
            _context = context;

            if (_context.TodoItems.Count() == 0)
            {
                _context.TodoItems.Add(new TodoItem { Name = "Item1Changed" });
                _context.SaveChanges();
            }
        }
```

Once we have done this we can use Docker to build the new image and push it to ACR. 

Build image(dont forget the '.' at the end)
(make sure you are in the directory that your dockerfile is in)

```
-> docker build -t todov1 .
```


Tag image for ACR

```
-> docker tag todov1 todov1registry.azurecr.io/todov1
```

Push to ACR (You may need to log in again to ACR - az acr login --name <YourRegistryName>)

```
-> docker push todov1registry.azurecr.io/todov1
```


  When done, your command line will look similar to this (look for errors)

```
➜  ToDoV2AddingDocker git:(master) ✗ docker push todov1registry.azurecr.io/todov1
The push refers to repository [todov1registry.azurecr.io/todov1]
ed663896211d: Layer already exists
8ddd1b1e11fa: Layer already exists
f3f4bb43b885: Layer already exists
280a1e8bf3b7: Layer already exists
fd98629870ee: Layer already exists
d626a8ad97a1: Layer already exists
expose: digest: sha256:4be2518323181c8954ac4a22b47ced6e21c1e10da6986332859e152aff5f2e02 size: 1579
f131451deafe: Layer already exists
8ddd1b1e11fa: Layer already exists
f3f4bb43b885: Layer already exists
280a1e8bf3b7: Layer already exists
fd98629870ee: Layer already exists
d626a8ad97a1: Layer already exists
latest: digest: sha256:4ba89a74e3f74fed7e18b88bac053fb070638d4362b8ee96003c57cb89ed8c12 size: 1579
```

If you go to the Webhooks section of your registry, you can check to make sure it succeeded. (200).

![](https://raw.githubusercontent.com/DanielEgan/ContainerTraining/master/images/cicd8.png)

And finally, make a call to the staging URL to confirm that the change you made shows up.

![](https://raw.githubusercontent.com/DanielEgan/ContainerTraining/master/images/cicd9.png)

Finally, you can go to the Deployment Slots on yoyur web app and swap images OR go to the Deployment Slots (Preview) and send a percentage of traffic to the new container. 

![](https://raw.githubusercontent.com/DanielEgan/ContainerTraining/master/images/cicd10.png)


Next, we will deploy to Azure Container Instances

## Deploying your image to ACI (Azure Container Instance)

Now normally you will be doing your deploying headless.  Meaning using some sort of CI/CD Pipeline to deploy your container from ACR to ACI or others. In that case you will want to authenticate using a service principal.  You can find a quickstart [here](https://docs.microsoft.com/en-us/azure/container-registry/container-registry-auth-aci), on how to do it or more information on using service principals [here](https://docs.microsoft.com/en-us/azure/container-registry/container-registry-auth-service-principal). 

For our image we will enable the admin user on the registry so we can do it manually from the command line interface. To enable it, you run the following command if we want. (keep in mind we are still logged in to acr). 

```
-> az acr update --name <_YourRegistryName_> --admin-enabled true
```


The rest of the section, we are going to again do it visually in the portal.

First we need to go to the portal and click on **Create A Resource** as we did before and search for **Container Instances**. Once the blade opens, click on **Create**.

![](https://raw.githubusercontent.com/DanielEgan/ContainerTraining/master/images/aci1.png)

From this page, you can see that we can create the instance from either a public image (Like our Dockerhub image) or from a private image (like our ACR image).  We will do a private images.  

When you click on the **Private** toggle, you will boxes for :
- Container Image
- Image Registry login server
- Image Registry username
- Image Registry password.


![](https://raw.githubusercontent.com/DanielEgan/ContainerTraining/master/images/aci2.png)

You can get all of these from the registry that you created earlier.  To find it, on the left, 
- click on Resource Groups 
- Click on the resource group you created ( I called it todov1rg )
- Click on the registry you created ( I called it todov1registry )
- On the left of this blade click on **Access Keys** this will give you the information you need for your private registry. 

![](https://raw.githubusercontent.com/DanielEgan/ContainerTraining/master/images/aci6.png)

In the Configuration section, you want to set the following:

- DNS name label -- The name that will be used, combined with the region you are in, to access your container instance. 
- Port - We are going to leave 80 there even though it is not being used so we can add additional ports
- Click on **Open additional ports** and add port 5000

We will leave everything else as is but you can see we can add environment variables and command overrides if needed.  Click **OK** when done

![](https://raw.githubusercontent.com/DanielEgan/ContainerTraining/master/images/aci3.png)

This will bring up a summary for you to review. You can see that you can download a template to do this in an automated fashion at a later date.   Click on **OK** when done. 

![](https://raw.githubusercontent.com/DanielEgan/ContainerTraining/master/images/aci4.png)

When it is deployed, you can view the resource. To access your container, you can use the FQDN address. Also notice that you have and IP address which means if you would like you could put a load balancer in front of this (something that Azure Web Does for you )

![](https://raw.githubusercontent.com/DanielEgan/ContainerTraining/master/images/aci5.png)

Make sure that you add the port and api route **:5000/api/todo** 


![](https://raw.githubusercontent.com/DanielEgan/ContainerTraining/master/images/aci7.png)

Next, we will deploy our container in a Kubernetes Cluster.


## Deploying your image in a Kubernetes Cluster

### Setting up the cluster
Now we will deoploy our container to the fourth of our options, a Kubernetes cluster. To do this we will be using AKS, the Microsoft Managed Kubernetes service.  We will use a combination of the azure-cli and the portal to complete our tasks.  

To begin, we first need to make sure that AKS is associated with your Azure subscrition.  To do that, run the following command.

```
-> az provider register -n Microsoft.ContainerService
```


When that finishes, we can create the AKS cluster inside the resource group that we have been using for this workshop **todov1rg** to do this, type the following command.

```
-> az aks create --resource-group todov1rg --name todov1service --node-count 2 --generate-ssh-keys
```


This will beging to create our cluster with two VMs and generate out ssh-keys for us. 

This will take a while to spin up so be prepared to wait 5 or 10 minutes.  When it completes you will see somthing similar to this.

![](https://raw.githubusercontent.com/DanielEgan/ContainerTraining/master/images/k8s1.png)

You now have your first kubernetes cluster in Azure. 

### Taking a look around
Before we deploy our application to the cluster, lets take a spin around what was created. 

If you go to the portal and look at the resource group you created your cluster in **todov1rg** 

![](https://raw.githubusercontent.com/DanielEgan/ContainerTraining/master/images/k8s2.png)

you will notice that there is only one item that relates to the cluster in there, the service itself. This is the pointer to the master node and is managed for you by azure (patching, upgrading, etc...). You don't have access to the master node.

![](https://raw.githubusercontent.com/DanielEgan/ContainerTraining/master/images/k8s3.png)

So where is all the other stuff associated with the service?  If you go back to your resource groups, you will notice that there is a new resource group called MC_todov1rg_todov1service_eastus (MC = Managed Cluster/ original resource group / name of service / region of service)

![](https://raw.githubusercontent.com/DanielEgan/ContainerTraining/master/images/k8s4.png)

If you click on that resource group you will see all of the other things that were created for your cluster. (The VM's the disks, Virtual Network, etc...)

![](https://raw.githubusercontent.com/DanielEgan/ContainerTraining/master/images/k8s5.png)

### Working with Kubctl 

Kubectl, pronouced "Kube cuttle" or "Kube control" depending on who you ask, is your command line interface to your kubernetes cluster. We installed this as part of the pre-reqs at the begining of the lab. If you did not, head back up there before continuing.  We will be using this tool to deploy and manage our applications and services to the nodes we created.  In addition, we are also going to utilizing YAML files to do our deployment.  We will not be going deep into YAML files here, they are a text based file that is used to set configurations (pods, deployments, etc...) in our cluster.  If you would like to learn more about them just search [Yaml files for beginners](https://www.google.com/search?q=Yaml+Files+for+beginers&oq=Yaml+Files+for+beginers).

So to communicate with our Azure Kubernetes Cluster we need to connect **Kubectl** to our cluster. To do that we need to execute the following command. 

```
  -> az aks get-credentials --resource-group todov1rg --name todov1service
```


You should see a message similar to the one below stating that it Merged as current context

![](https://raw.githubusercontent.com/DanielEgan/ContainerTraining/master/images/k8s6.png)

Once you have this done, you can run kubectl commands againts the cluster. Try a few of them out, starting with :

Get all the commands you can run
```
-> kubectl --help
```

Get all the nodes in the cluster
```
-> kubectl get nodes
```

Get Cluster component statuses
```
-> kubectl get cs
```

Get all pods(you dont have any yet)
```
-> kubectl get pods
```


Get all deployments
```
-> kubectl get deployoments
```

Feel free to try out other commands.

### Getting Secret to access private ACR

We could of course delploy our image from DockerHub since there is no authentication needed but in most cases you will not be doing that.  So for us to deploy our ACR image to the Kubernetes Cluster we will need to give Kubernetes access to the ACR.  They typical way an organization would do this would be to use [AAD and Service Principal] (https://docs.microsoft.com/en-us/azure/container-registry/container-registry-authentication) but for this exercise we will be generating a secret for AKS to use to access ACR. 

In the last exercise where we loaded our container to ACI (Azure container instance) we enabled an admin user on our ACR.  We will need that for this exercise.  If you did not keep it around you can find it by going to your ACR (Azure Container Registry) and click on Access Keys.

![](https://raw.githubusercontent.com/DanielEgan/ContainerTraining/master/images/k8s6.png)

Either keep that page open or save them for our next command.  The command we need to run is the following:

```
kubectl create secret docker-registry SECRET_NAME --docker-server=REGISTRY_NAME.azurecr.io --docker-username=USERNAME --docker-password=PASSWORD --docker-email=ANY_VALID_EMAIL
```

SECRET_NAME = todov1secret -- (This is the name you will use in the Yaml file.)

REGISTRY_NAME = todov1registry -- (This is what mine is named, yours may be different)

USERNAME = Copy from last step

PASSWORD = Copy form last step

ANY_VALID_EMAIL = ummm any valid email :)


When you have run this command you will then see the following on the command line.

```
secret "todov1secret" created
```


Now we can load our our container into our cluster using the Yaml file.  I have created the file for you. You can find it in this repository called aks.yaml.  Before we run it, I am going to break it down into pieces and explain what it does.

>**_IMPORTANT_** 
..I DO NOT want you to create your own Yaml file, they are notoriously fickle and easy to mess up by not having the correct amount of spaces. Please use the Yaml file (aks.yaml) that is in this repository when running it.  Below is an explanation as to what you will find in the Yaml file. 

First we are telling the cluster that we are creating a deployment called todo

```
apiVersion: apps/v1beta1
kind: Deployment
metadata:
  name: todo
```
Next, we tell it that we want three replicas of the image. The app is named todo (we need that later for the load balancer)

```
spec:
  replicas: 3 
  template:
    metadata:
      labels:
        app: todo
```
We point the the container in ACR (I am using one that I had with and EXPOSE tag. If you leave off the tag it will select :latest)

```
    spec:
      containers:
      - name: todo
        image: todov1registry.azurecr.io/todov1
```
Next, we set the port that we want access to for the app. As you remember we also use expose in the docker file to expose port 5000. We set a few resource limits.

```
        ports:
        - containerPort: 80
        resources:
          requests:
            cpu: 250m
          limits:
            cpu: 500m

```
We add the image pull secret that we created in the last step

```
      imagePullSecrets:
      - name: todov1secret
```
Finally, we create a load balancer to spread the load between the 3 replicas that we create.  We map port 80 to port 5000 to access the api.

```
---
apiVersion: v1
kind: Service
metadata:
  name: todo
spec:
  type: LoadBalancer
  ports:
  - port: 80
    targetPort: 5000
  selector:
    app: todo

``` 

Finally, we run the yaml file with the following command. (Make sure you are in the same directory as the Yaml file when you run it)  -f denotes filename

```
-> kubectl create -f aks.yaml
```

This will start to deploy your containters, pods, replicas, etc..   Try running the commands we played with before. 

Get all the nodes in the cluster
```
-> kubectl get nodes
```

Get Cluster component statuses
```
 -> kubectl get cs
```

Get all pods(you dont have any yet)
```
-> kubectl get pods
```

Get all deployments
```
-> kubectl get deployoments
```

Get all servides
```
-> kubectl get service
```


That last one is the one we need to test our cluster. Once created, the load balancer will have an external port that we can hit which will then send the call to the container on port 5000.

It will look something like this:

```
NAME         TYPE           CLUSTER-IP     EXTERNAL-IP     PORT(S)        AGE
kubernetes   ClusterIP      10.0.0.1       <none>          443/TCP        4h
todo         LoadBalancer   10.0.125.123   168.62.168.84   80:31318/TCP   4m
```

Once you have that External-IP you can test your cluster to see your app working. http://188.62.168.84/api/todo  (Yours will be different)


Finially, if you prefer to visually see the condition of your cluster you can do that through the portal (kind of).  In the Azure portal navigate to the service you created.  Remember, this can be found in the todov1rg resourse group called todov1service.  On the service's overview tab you will see three options :

- Monitor Container
- View Logs
- View Kubernetes dashboard

Select View Kubernetes Dashboard

![](https://raw.githubusercontent.com/DanielEgan/ContainerTraining/master/images/k8s8.png)

This will open up a blade that shows you how to view the dashboard. You have already done numbers one through three.  Select the copy icon from number four and paste it into your command line to run it.

![](https://raw.githubusercontent.com/DanielEgan/ContainerTraining/master/images/k8s9.png)

It will produce something like this on the command line.

```
ContainerTraining git:(master) az aks browse --resource-group todov1rg --name todov1service
Merged "todov1service" as current context in /var/folders/y1/8kxz0m9n3v9d8hw8p2nlsmhr0000gn/T/tmpuxlphmzg
Proxy running on http://127.0.0.1:8001/
Press CTRL+C to close the tunnel...
Forwarding from 127.0.0.1:8001 -> 9090
Forwarding from [::1]:8001 -> 9090
Handling connection for 8001
Handling connection for 8001
Handling connection for 8001
Handling connection for 8001
```

It will then launch a browser for you to view your cluster.  You should see somethig like this when it opens.  Starting with Kubernetes v1.7 RBAC (Role Binding Access Control) is enabled by default.  This prevents the dashboard from accessing your cluster.

![](https://raw.githubusercontent.com/DanielEgan/ContainerTraining/master/images/k8s10.png)

To learn about how to fix this you can visit this page. [Accessing RBAC enabled Kubernetes Dashboard](https://unofficialism.info/posts/accessing-rbac-enabled-kubernetes-dashboard/)

As in other parts of this tutorial, there is the way you should do it in production, and the way we are going to do it here.  In practice, you should utilize your service account to give access to the dashboard with tokens which has many steps.

We are going to grant admin privliges to the Dashboards Service account which will work but might be a security risk. [see here](https://github.com/kubernetes/dashboard/wiki/Access-control#admin-privileges)

Since this is just a demo for us, we will grant admin privliges with the follow command: 

Run this in your command line.

```
->  kubectl create clusterrolebinding kubernetes-dashboard -n kube-system --clusterrole=cluster-admin --serviceaccount=kube-system:kubernetes-dashboard
```



Once this is done, you can try to bring up the dashboard again by running the following (which you had copied earlier from the portal)

```
-> az aks browse --resource-group todov1rg --name todov1service
```

This will give you access to your dashboard as seen below.  You can get all the same information you get from Kubectl but now in an interface.

![](https://raw.githubusercontent.com/DanielEgan/ContainerTraining/master/images/k8s11.png)

From here feel free to play around with your cluster search the Kubectl commands (Kubectl --help) to remove pods, add pods, check on health, modify your Yaml.  Its just a demo so its ok if you break it. 

Enjoy!!












