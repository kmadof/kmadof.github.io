---
layout: post
title: Caching (not only) NuGet packages on Azure DevOps
date: 2020-03-11
excerpt_separator:  <!--more-->
tags: DevOps AzureDevOps
---

The goal of [Cache@2](https://docs.microsoft.com/en-us/azure/devops/pipelines/tasks/utility/cache?view=azure-devops) task is improving build performance by caching files between pipeline runs. It supports multiple types of packages like

- Bundler gems
- npm packages
- Yarn packages
- NuGet packages
- Maven artifacts
- Gradle artifacts
- ccache artifacts

Further, we will focus on caching [NuGet packages](https://docs.microsoft.com/pl-pl/azure/devops/pipelines/release/caching?view=azure-devops#netnuget) however in a similar manner we can configure this task for other types.

<!--more-->

### Locking dependencies

Before we configure cache task we need to lock dependencies to create `packages.lock.json` file as we need that file to set a proper key for the cache. To do that we need to set MSBuild property `RestorePackagesWithLockFile` for a project.

{% highlight xml %}

<PropertyGroup>
    <!--- ... -->
    <RestorePackagesWithLockFile>true</RestorePackagesWithLockFile>
    <!--- ... -->
</PropertyGroup>

{% endhighlight %}

On the next `dotnet restore` a lock file `packages.lock.json` will be generated. We need to store that file in source control to be able to use cache task. However this is not such easy as it looks like. Because, if you use this file probably  you will get on CI server error NU1403:

    error NU1403: Package content hash validation failed for System.Linq.4.3.0. The package is different than the last restore. [/home/vsts/work/1/s/PipelineCaching.sln]


This is because we can get different on different machines/systems. So hash on my machine is different than on Linux host agent or Windows host agent. On [github](https://github.com/NuGet/Home/issues/7921#issuecomment-478152479) you can find the proposed solution. But this boils down to putting these commands:

{% highlight powershell %}

dotnet nuget locals all --clear
git clean -xfd
git rm **/packages.lock.json -f
dotnet restore Froto.sln

{% endhighlight %}

in your pipeline. In general, it is really strange that mechanism which should provide a consistent and reliable way of getting the same packages regardless you compile your source code requires to generate on each machine/system a new lock file. I took a different approach. I created the lock file on the host agent and published it as an artifact. Then I downloaded it and committed into source control. I know that it requires to reproduce that step over and over again when I add or update package. But for me, this is a better approach than removing and recreating lock file by CI server. If you want to do the same, you can achieve this using these steps:

{% highlight yaml %}
variables:
  buildDirectory: '$(Build.SourcesDirectory)'

steps:
- task: CopyFiles@2
  displayName: Copy packages.lock.json file
  enabled: true
  inputs:
    sourceFolder: $(buildDirectory)/PipelineCaching
    contents: packages.lock.json
    targetFolder:  $(Build.ArtifactStagingDirectory)

- task: PublishBuildArtifacts@1
  displayName: Publish packages.lock.json file
  enabled: true
  inputs:
    pathToPublish: $(Build.ArtifactStagingDirectory)
    artifactName: LockedJson

{% endhighlight %}

Once you create a lock file, you can disable these steps.

### NuGet packages location

To configure cache task we need also packages location. Following [this documentation](https://docs.microsoft.com/pl-pl/azure/devops/pipelines/release/caching?view=azure-devops#netnuget) you can assume that NuGet downloads packages to `$(Pipeline.Workspace)/.nuget/packages`. But this is wrong or some configuration part is missing like overriding [default location for global-packages](https://docs.microsoft.com/en-us/nuget/reference/nuget-config-file). If you list `$(Pipeline.Workspace)` you will find that there is no NuGet folder.

     Directory of D:\a\1

    03/10/2020  02:45 PM    <DIR>          .
    03/10/2020  02:45 PM    <DIR>          ..
    03/10/2020  02:45 PM    <DIR>          a
    03/10/2020  02:45 PM    <DIR>          b
    03/10/2020  02:45 PM    <DIR>          s
    03/10/2020  02:45 PM    <DIR>          TestResults

To find nuget packages folder, we can use this command `dotnet nuget locals global-packages -l` which allow us to set proper path programmatically.


{% highlight yaml %}

steps:
- task: PowerShell@2
  inputs:
    targetType: 'inline'
    script: |
      $cache = dotnet nuget locals global-packages -l
      $cacheLocation = $cache.Replace('info : global-packages: ', '')
      
      Write-Host $cacheLocation
      
      Write-Host "##vso[task.setvariable variable=NUGET_PACKAGES;]$cacheLocation"

    pwsh: true


{% endhighlight %}

What is worth noted here, that it works for both Linux and Windows host agents.

### Configuration for NuGet packages

Putting all together:

{% highlight yaml %}

trigger:
- master

pool:
  vmImage: 'ubuntu-latest'

variables:
  buildConfiguration: 'Release'
  buildDirectory: '$(Build.SourcesDirectory)'

steps:

- script: dotnet nuget locals global-packages -l
  displayName: 'dotnet nuget locals global-packages -l'

- task: PowerShell@2
  inputs:
    targetType: 'inline'
    script: |
      $cache = dotnet nuget locals global-packages -l
      $cacheLocation = $cache.Replace('info : global-packages: ', '')
      
      Write-Host $cacheLocation
      
      Write-Host "##vso[task.setvariable variable=NUGET_PACKAGES;]$cacheLocation"

    pwsh: true

- task: Cache@2
  inputs:
    key: 'nuget | "$(Agent.OS)" | PipelineCaching/packages.lock.json'
    restoreKeys: |
       nuget | "$(Agent.OS)"
       nuget
    path: $(NUGET_PACKAGES)
    cacheHitVar: CACHE_RESTORED
  displayName: Cache NuGet packages

- script: dotnet restore
  displayName: 'dotnet restore'
  condition: ne(variables.CACHE_RESTORED, 'true')
  workingDirectory: $(buildDirectory)

- script: dir
  displayName: 'Displaying NuGet folder'
  enabled: true
  condition: always()
  workingDirectory: $(NUGET_PACKAGES)

- task: CopyFiles@2
  displayName: Copy packages.lock.json file
  enabled: false
  inputs:
    sourceFolder: $(buildDirectory)/PipelineCaching
    contents: packages.lock.json
    targetFolder:  $(Build.ArtifactStagingDirectory)

- task: PublishBuildArtifacts@1
  displayName: Publish packages.lock.json file
  enabled: false
  inputs:
    pathToPublish: $(Build.ArtifactStagingDirectory)
    artifactName: LockedJson

- script: dotnet build --configuration $(buildConfiguration)
  displayName: 'dotnet build $(buildConfiguration)'
  workingDirectory: $(buildDirectory)
 

{% endhighlight %}

Looking into cache task logs, you may find that your key was resolved to `nuget|"Linux"|h6yiwZzAgkoiZL06WJLdgZToQpsZmeUIUL5u0xFpYHM=`, where last part is a hash of your lock file. It means that each time when the lock file will change, a new cache key will be created. In the first run, you may also find information that `There is a cache miss`. This is because setting cache takes a place in post-job task

    Information, Creating a pipeline cache artifact with the following fingerprint: `nuget|"Linux"|h6yiwZzAgkoiZL06WJLdgZToQpsZmeUIUL5u0xFpYHM=`
    Information, Cache item created.

In the next run, cache task gets a hit and download packages to the given folder.

    There is a cache hit: `nuget|"Linux"|h6yiwZzAgkoiZL06WJLdgZToQpsZmeUIUL5u0xFpYHM=`

Cache task set also variable CACHE_RESTORED to true which causes evaluating condition in next step as false. So we do not restore packages.


### Summary

Cache task is very useful as it may reduce significantly network calls to get packages. I recommend to use it especially if you have many dependencies. I can only complain about documentation, as for me there were too many things not working as it is written there. Or maybe it is just me and I didn’t understand all the pieces. In that case, my apologies. But, also this part can be improved. The code for this you can find on [my github](https://github.com/kmadof/azure-devops-caching).

