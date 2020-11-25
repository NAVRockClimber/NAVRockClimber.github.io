---
layout: post
title:  "Deploy artifacts to BC containers like on-prem"
date:   2020-11-22
desc: "Deploy artifacts to bc docker containers as if you working on-prem"
keywords: "Business Central,Docker,Artifacts,Environments"
categories: [DevOps]
tags: [Business Central,Devops Environments, Devops Artifacts]
icon: fa-key
---

Well, it has been a while. This blogging thing did not went as I planned to. It became sacrificed to the day to day business as an Product Architect with the aftermath of switching to AL like breaking up a monolithic application and the transition into this new home office thing which started at the dinning table and I had to set everything up from the scratch.

But let's get to the subject of this blog post: How can we deploy any Artifact - an app file in the BC world - into a docker container as if we are working on-prem without the leveraging the bccontainerhelpers. Azure Devops pipelines offer this awesome feature of [enviroments](https://docs.microsoft.com/en-us/azure/devops/pipelines/process/environments?view=azure-devops). Environments are basically the endpoint or resource for a special kind of jobs in devops called [deployment jobs](https://docs.microsoft.com/en-us/azure/devops/pipelines/process/deployment-jobs?view=azure-devops) 

## Environments

If you take a look into Environments you see that only Kubernetes clusters or virtual machines can be used as resource.  

## Setting Up Containers

## Deployment Task & Job

## Keep on rolling