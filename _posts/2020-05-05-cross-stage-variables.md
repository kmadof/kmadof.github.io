---
layout: post
title: Cross stage variables (almost avialable) in Azure DevOps
date: 2020-05-05
excerpt_separator:  <!--more-->
tags: DevOps AzureDevOps
---

When you create a build pipeline you must sometime decide at runtime whether run some code or not. One of the options for this is output variables. It enables you to set a variable in one job and use this variable in the next job. In YAML we will write this in that way:

```yaml
stages:
- stage: A
  jobs:
  - job: JA
    steps:
    - script: |
        echo "This is job Foo."
        echo "##vso[task.setvariable variable=doThing;isOutput=true]Yes" #The variable doThing is set to true
      name: DetermineResult
    - script: echo $(DetermineResult.doThing)
      name: echovar
  - job: JA_2
    dependsOn: JA
    condition: eq(dependencies.JA.outputs['DetermineResult.doThing'], 'Yes')
    steps:
    - script: |
        echo "This is job Bar."
```

<!--more-->

In the task `DetermineResult` we set the variable `doThing` and mark it is as output `##vso[task.setvariable variable=doThing;isOutput=true]Yes` using [logging command syntax](https://docs.microsoft.com/en-us/azure/devops/pipelines/scripts/logging-commands?view=azure-devops&tabs=bash). In the next job `JA_2` we used this variable as a part of condition `eq(dependencies.JA.outputs['DetermineResult.doThing'], 'Yes')` which checks if the variable is set to `Yes`. So far this construct was limited a single stage. But it changes ...

### Cross stage variables

Azure DevOps team announced that we can access output variables from the previous stage. You can read about this in [the release note](https://docs.microsoft.com/en-us/azure/devops/release-notes/2020/sprint-168-update#azure-pipelines-1). The syntax is quite similar, and instead of `dependencies.jobName.outputs['stepName.variableName']` we should use this `stageDependencies.stageName.jobName.outputs['stepName.variableName']`. So let's try it:

```yaml
stages:
- stage: A
  jobs:
  - job: JA
    steps:
    - script: |
        echo "This is job Foo."
        echo "##vso[task.setvariable variable=doThing;isOutput=true]Yes" #The variable doThing is set to true
      name: DetermineResult
    - script: echo $(DetermineResult.doThing)
      name: echovar
  - job: JA_2
    dependsOn: JA
    condition: eq(dependencies.JA.outputs['DetermineResult.doThing'], 'Yes')
    steps:
    - script: |
        echo "This is job Bar."

#stage B runs if DetermineResult task set doThing variable n stage A
- stage: B
  dependsOn: A
  condition: eq(stageDependencies.A.JA.outputs['DetermineResult.doThing'], 'Yes') #map doThing and check if true
  jobs:
  - job: JB
    steps:
    - bash: echo "Hello world stage B job JB"
```

If you try to run this build you may still get an error:

> An error occurred while loading the YAML build pipeline. Unrecognized value: 'stageDependencies'. Located at position 4 within expression: eq(stageDependencies.A.JA.outputs['DetermineResult.doThing'], 'Yes'). For more help, refer to https://go.microsoft.com/fwlink/?linkid=842996

Because this is not available for everyone yet.

> These features will roll out over the next two to three weeks.

But it will be soon. I hope that this will simplify your builds and your lifes :)

As always you will find this code on my [github](https://github.com/kmadof/devops-manual/tree/master/cross-stage-variables).