---
layout: post
title: Build templates on Azure DevOps
date: 2020-04-02
excerpt_separator:  <!--more-->
tags: DevOps AzureDevOps continuous-integration .NETCore
---

Last time we created [a gated check-in build for .NET Core app]({% post_url 2020-03-26-gated-check-in-build-on-azure-devops-for-dotnet-core-app %}). It works very well, but we did there one thing which is in general a bad practice in our proficiency. We duplicated build steps for building and testing .NET Core app. We can do better than that, we can use [templates](https://docs.microsoft.com/en-us/azure/devops/pipelines/process/templates?view=azure-devops). Following the documentation:

> Templates let you define reusable content, logic, and parameters. Templates function in two ways. You can insert reusable content with a template or you can use a template to control what is allowed in a pipeline.

We will do both things.
<!--more-->

### Template definition

First what we need is a template definition. And we are halfway there as we already defined common steps. Now we need to extract them to a separate file and make a small change to parametrize it.

{% gist d943422f6ac198042abf14441e8deff0 build-and-test.yaml %}

There is a small little change comparing to the original steps definition. We extracted buildConfiguration as a parameter. Instead of

{% highlight yml %}

arguments: '--configuration $(buildConfiguration)'

{% endhighlight %}

we have now:


{% highlight yml %}

parameters:
- name: buildConfiguration # name of the parameter; required
  default: false

{% endhighlight %}

and 

{% highlight yml %}
{% raw %}

arguments: '--configuration ${{ parameters.buildConfiguration }}'

{% endraw %}
{% endhighlight %}

It will allow us to pass build configuration as a parameter when we reuse this template.

### Gated check-in (GC) build

Gated check-in build does exactly what is defined in the template and nothing more. For that purpose we use `extends` option.

{% highlight yml %}

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
    - gated-checkin-with-template/*
    exclude:
    - gated-checkin-with-template/azure-pipelines.yml

variables:
  buildConfiguration: 'Release'

extends:
  template: build-and-test.yaml
  parameters:
      buildConfiguration: $(buildConfiguration)

{% endhighlight %}

As you see this build definition is minimal. What we do here is:

- define triggers
- define variable
- select build template


Now is time for regular CI build.

### Standard CI build

Standard CI build apart of steps defined in the template may contain additional steps.

{% highlight yml %}

trigger:
  branches:
    include:
    - master
  paths:
    include:
    - gated-checkin-with-template/*
    exclude:
    - gated-checkin-with-template/azure-pipelines-gc.yml

pr: none

pool:
  vmImage: 'ubuntu-latest'

variables:
  buildConfiguration: 'Release'

steps:

- template: build-and-test.yaml
  parameters:
      buildConfiguration: $(buildConfiguration)

- script: echo Some steps to create artifacts!
  displayName: 'Run a one-line script'

{% endhighlight %}

This definition contains a little bit more:

- define triggers
- define pool
- define variable
- select build template
- define other steps

There is more than one difference between definitions. GC build doesn't have a pool definition. <strong>If you try to run build with pool definition and `extends` you will get an error.<strong>

<div class="message">
  /gated-checkin-with-template/azure-pipelines-gc.yml (Line: 29, Col: 1): Unexpected value 'extends'
</div>

I haven't found yet a reason why it behaves like that, but when I find it I will update this blog post.

### Summary

Templates are great option to simplify your builds. They help you reuse code and keep definition clean. And if you have many projects and many repositories, you will also find them useful. For this case, what you need is a central repository where you keep build templates and then reuse them in your repositories.

#### Links

- GitHub repo with [code for this article](https://github.com/kmadof/devops-manual/tree/master/gated-checkin-with-template)
- [My blog post about gated check-in build for .NET Core app]({% post_url 2020-03-26-gated-check-in-build-on-azure-devops-for-dotnet-core-app %})
- [Microsoft documentation for templates](https://docs.microsoft.com/en-us/azure/devops/pipelines/process/templates?view=azure-devops)