# .NET Microservices on AKS — CI/CD with Azure DevOps, Docker Hub & Trivy

> Building a reliable delivery path for containerized .NET microservices — with real cost constraints, actual failures, and the engineering trade-offs that came out of them.

This project started as a learning exercise built on top of the excellent [aspnetrun/run-devops](https://github.com/aspnetrun/run-devops) repository.  
I kept the original application structure (Shopping API + Shopping Client + MongoDB), then rebuilt the entire delivery and deployment path around constraints I actually hit while running this on an **Azure for Students** subscription.

The goal was never to build the most complex pipeline possible.  
It was to understand what happens between a `git push` and a running pod — and to make every piece of that path work for real.


| Image | Status |
| ------------- | ------------- |
| Shopping Client | [![Build Status](https://dev.azure.com/hardikaroracs22/Azure-DevOps-Full_Microservice_Pipeline/_apis/build/status%2Fci-shopping-client?branchName=main&jobName=Build%2C%20Test%20%26%20Security%20Scan)](https://dev.azure.com/hardikaroracs22/Azure-DevOps-Full_Microservice_Pipeline/_build/latest?definitionId=4&branchName=main) |
| Shopping API | [![Build Status](https://dev.azure.com/hardikaroracs22/Azure-DevOps-Full_Microservice_Pipeline/_apis/build/status%2Fci-shopping-api?branchName=main)](https://dev.azure.com/hardikaroracs22/Azure-DevOps-Full_Microservice_Pipeline/_build/latest?definitionId=3&branchName=main) | | |


### Overall Picture
See the overall picture. You can see that we will have 3 microservices which we are going to develop and deploy together.

![Overall Picture of Repository](https://user-images.githubusercontent.com/1147445/105671396-b152f580-5ef3-11eb-8f3b-7f9f7c9c4d24.png)

### Shopping MVC Client Application
First of all, we are going to develop Shopping MVC Client Application For Consuming Api Resource which will be the Shopping.Client Asp.Net MVC Web Project. But we will start with developing this project as a standalone Web application which includes own data inside it. And we will add container support with DockerFile, push docker images to Docker hub and see the deployment options like “Azure Web App for Container” resources for 1 web application.
### Shopping API Application
After that we are going to develop Shopping.API Microservice with MongoDb and Compose All Docker Containers.
This API project will have Products data and performs CRUD operations with exposing api methods for consuming from Shopping Client project.
We will containerize API application with creating dockerfile and push images to Azure Container Registry.
### Mongo Db
Our API project will manage product records stored in a no-sql mongodb database as described in the picture.
we will pull mongodb docker image from docker hub and create connection with our API project.
At the end of the section, we will have 3 microservices whichs are Shopping.Client — Shopping.API — MongoDb microservices.
As you can see that, we have
* Created docker images,
Compose docker containers and tested them,
Deploy these docker container images on local Kubernetes clusters,
Push our image to ACR,
Shifting deployment to the cloud Azure Kubernetes Services (AKS),
Update microservices with zero-downtime deployments.
### Deploy to Azure Kubernetes Services (AKS) through CI/CD Azure Pipelines
And the last step, we are focusing on automation deployments with creating CI/CD pipelines on Azure Devops tool. We will develop separate microservices deployment pipeline yamls with using Azure Pipelines.
When we push code to Github, microservices pipeline triggers, build docker images and push the ACR, deploy to Azure Kubernetes services with zero-downtime deployments.

![cicd](https://user-images.githubusercontent.com/1147445/105671542-f37c3700-5ef3-11eb-9532-59a5855214d0.png)

You’ll see how to deploy your multi-container microservices applications with automating all deployment process seperately.
