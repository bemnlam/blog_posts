---
title: "Notes on Bamboo"
date: 2020-09-12T12:08:18+08:00
draft: true
---

# Principles of a CI/CD workflow

In my opinion:

1. Make things predictable.
2. You should think of the deployment (CD) first, then work backward to the integration (CI) part.
3. Use *gitflow* in your source code.

## Fundamentals in Bamboo

The menu in Bamboo (version 6.5.0) contains 3 parts: The *Projects*, the *Build* and *Deploy*.

![image-20200913141147305](../../static/img/image-20200913141147305.png)

![image-20200913141246490](../../static/img/image-20200913141246490.png)



### Deployment (CD)

Here is a typical **deployment project summary** looks like:

<img src="../../static/img/image-20200913132645203.png" alt="a typical deployment project" style="zoom:50%;" />

You can check this out by clicking **Deploy** > **All Deployment Projects**. After that, choose one of the item on the list.

#### Glossary

*Artifact*: The content built in the CI part.

*Deployment Environment*: The place the *artifacts* should go to.

~~For example, you can deploy same *artifact* to 1  or multiple web servers, depends on how you define the tasks in each environment.~~

*Release*: A unique (and usually auto-increment) identifier of this deployment.

*Deployment Project*: A collection of build plans.

### Build (CI)

Here is a typical build dashboard:

<img src="../../static/img/image-20200913130048805.png" alt="a typical build dashboard" style="zoom:50%;" />

This is the default front page of Bamboo. You can also check this out by clicking **Build** > **All Build Plans**.

#### Glossary

*Build*: The build result of the default plan.

*Build Plan*: Usually it refers to the definitions of all build-related configuratoins. However, sometimes it *also* refers to the **default build barnch** or the **default branch configurations**.

I will try to avoid using *build plan* in this article as it is a bit confusing. I will use :

- *default branch configuratoins* to indicate the default settings of a *build plan*;

- *default build branch* to indicate the branch associated to the *default branch configuratoins*.
- When I use the word *build plan*, it refers to the item listed under **Plan** column in the build dashboard.

*Build Project*: Similar to *Deployment Project*, it's a collection of *build plans*.



Next, I will talk about how to **design** a CD flow, then a CI flow.

# Designing a CI/CD workflow

> Assumption: you are using [*gitflow*](https://datasift.github.io/gitflow/IntroducingGitFlow.html) to manage your git repository.

In short:

**Deployment**

- 1 *deoloyment project*, many *deployment plans*.
- 1 destination, 1 *deployment plan*.

**Integration**

- 1 Bitbucket git repository, (usually) 1 Bamboo *build plan*
  - build the `develop` branch using **default plan configuration**
  - git branch <--> *Branch config* of a build plan
- 1 Bitbucket project,  1Bamboo *build project*

## Deployment (CD)

All the deployments in my team can be categorized in to 2 types: either a **library** (i.e. nuget package) or a **website** (i.e. IIS website).

### Nuget Package

My team is using [MyGet](https://myget.org/) to host private packages. So, there is only 1 destination for my *artifact* goes.

![image-20200913153116958](../../static/img/image-20200913153116958.png)

This is how I organize the *deployment project*:

![image-20200913153838931](../../static/img/image-20200913153838931.png![image-20200913154357682](../../static/img/image-20200913154357682.png)

```
Deployment Project
├── Production environment: for builds from master
└── Pre-release environment: all other branches
```



### Web Application

We have 2 web servers for production and other environments has a corresponding web server. All running IIS. On the other hand, some of the website are hosting under a sub-directory (as a sub-site).

Therefore, an *artifact* may goes to multiple destinations.

In my design, 1 *deployment environment* responsibles for 1 destination (i.e. 1 IIS web server).

![image-20200913190536806](../../static/img/image-20200913190536806.png)

So you may ask: why not 1 deployment environment for 1 environment? Here's why:

#### 1-to-1 mapping of *deployment environment* and *IIS website*


##### The bad

One of the downside is: It takes longer time to deploy all websites. Consider the case of deploying 2 *deployment projects* at the same time, and each project has multiple *deployment environments*. The sequence of deployment activities can be:

1. deploy site A to environment A 
2. deploy site B to environment A
3. deploy site A to environment B (triggered by activity 1.)
4. deploy site B to environment B (triggered by activity 2.)

Which means I have to wait for the deployment of another website.

##### The good

The biggest advantage of doing so is that modefying one *deployment environment* does not affect any settings of the other *deployment environments*. Then, I can do the following things:

- A/B testing
- Rollback the version of a specific web server
- Check out the audit log / deployment history for a particular web server

Moreover, you can see the status of all environments in the **All Deployment Projects** page (**Deploy** > **All Deployment Projects**).

- If deployment of a particular web server is failed, I am able to find out which server is it easily
- Plus, I can just re-deploy to that particuler web server

You can create a new *deployment environment* by cloning an existing one (from the same *deployment project* or from another one). Thus, this is not a tedious task.

##### My decision

This is how I organize the *deployment project*:

![image-20200913154337848](../../static/img/image-20200913154337848.png)

```
Deployment Project
├── Production (A) environment: for builds from master / release
├── Production (B) environment: for builds from master / release
├── Staging environment: all other branches
└── UAT environment: all other branches
```

### Build (CI)

Similarly, my team has to build nuget packages, .NET Framework and .NET Core web applications.

#### Nuget Packages

#### .NET Framework Web Applications

#### .NET Core websites

