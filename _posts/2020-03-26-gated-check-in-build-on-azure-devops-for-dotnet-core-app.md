---
layout: post
title: Gated check-in build on Azure DevOps for .NET Core app
date: 2020-03-26
excerpt_separator:  <!--more-->
tags: DevOps AzureDevOps continuous-integration .NETCore
---

Standard CI build checks integrity of committed code after merge. This approach can lead to a state when code is not deployable due to failing tests or even failing compilation. Gated check-in helps to protect the integrity by verifying state before the merge. In that way, you can protect your master branch and avoid build breaks. In that way, you can ensure that your master branch is always deployable (what is crucial in GitHub flow) and you will not interrupt your colleagues with your obvious mistakes. Gated check-in feature was originally introduced in TFS 2010. It can also be easily adopted in Azure DevOps YAML based builds.

<!--more-->

### Assumptions

We will define two builds for .NET Core application which source code is hosted on GitHub:

- standard CI build - which build, test and publish artifact
- gated check-in (GC) build - which build and test

The difference between these two indicates why we want to separate builds. We don't want to produce artifacts each time when a commit is pushed or PR is created however we want to execute steps which ensure us that code integrity won't be affected. 

Code below contains parts of the builds which are common for both definitions. It is:

 - restore NuGet packages
 - compile code
 - run unit tests
 - install report generator (to create code coverage)
 - create report
 - publish code coverage

{% gist 9629b19f8de0c505d1806563ad3dabc9 CommonSteps.yaml %}

I decided to use ubuntu host agent and because of that, I had to use [Coverlet](https://github.com/tonerdo/coverlet) for code coverage. It requires to install `coverlet.collector` package for a unit test project. If you use windows host agent you can use built-in coverage data collector. More information you can find [here](https://docs.microsoft.com/en-us/azure/devops/pipelines/ecosystems/dotnet-core?view=azure-devops#collect-code-coverage).

### Standard CI build

Azure DevOps should run Standard CI build only when a commit is done on the master branch. To achieve this we need to exclude non-master branches and all pull requests.

{% highlight xml %}

trigger:
  branches:
    include:
    - master
  paths:
    include:
    - gated-checkin/*
    exclude:
    - gated-checkin/azure-pipelines-gc.yml

pr: none

{% endhighlight %}

I limited scope to the gated-checking folder where I put code for this article.

### Gated check-in (GC) build

Azure DevOps should run GC build on each commit for non-master branch and when we create a pull request for the master branch.

{% highlight xml %}

trigger:
  branches:
    include:
    - '*'
    exclude:
    - master

pr:
  branches:
    include:
    - master
  paths:
    include:
    - gated-checkin/*
    exclude:
    - gated-checkin/azure-pipelines.yml

{% endhighlight %}

### Branch protection rule

Once we have build definitions we should add branch protection rule on GitHub. You will find it going to `Settings` -> `Branches` -> `Add rule` then check:

- Require status checks to pass before merging
- Require branches to be up to date before merging
- and select GC build - in my case it is `kmadof.devops-manual-gated-checkin-gc`

That configuration protects the master branch from breaks and ensures that our branch is up to date before we merge it with the master branch.

![Branch protection rule](/images/2020-03-26-branch-protection-rule.png)


### GC build in action on GitHub

With this configuration, GitHub displays build information on the branch page:

![Branch page with build information](/images/2020-03-26-branch-page.png)

and we will not be able to merge in case of a failing build (unless you are an administrator).

![Pull request build check](/images/2020-03-26-pull-request-check.png)

### Summary

Gated check-in is an approach which helps you keep the integrity of your code. It is very important when many developers work at the same time on the same repository or if you follow GitHub flow because you always want to have the master branch in a deployable state. Do you use GC build in your project? Let me know in comments. I wonder how you keep your source code integrity.

#### Links

- GitHub repo with [code for this article](https://github.com/kmadof/devops-manual/tree/master/gated-checkin)
- Microsoft dev blog post about [GC build in TFS and VSTS](https://devblogs.microsoft.com/buckh/gated-checkin-for-git-using-branch-policies-to-run-a-build-in-vsts-and-tfs/)
- Microsoft documentation about build pipeline for .NET Core - [build, test, and deploy .NET Core apps](https://docs.microsoft.com/en-us/azure/devops/pipelines/ecosystems/dotnet-core)