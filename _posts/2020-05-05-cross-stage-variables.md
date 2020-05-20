---
layout: post
title: Cross stage variables in Azure DevOps and where you can use it
date: 2020-05-05
excerpt_separator:  <!--more-->
tags: DevOps AzureDevOps
---

<div class="dark-message">
  21 May 2020 - I specified where stageDependencies can be used and how.
</div>

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

You will get it because this is not available in condition at stage level. You can use `stageDependencies` in condition but at the job level. But not only there. Please check code below:

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

#job JB from stage B runs if DetermineResult task set doThing variable n stage A
- stage: B
  dependsOn: A
  jobs:
  - job: JB
    condition: eq(stageDependencies.A.JA.outputs['DetermineResult.doThing'], 'Yes') #map doThing and check if true
    variables:
      varFromStageA: $[ stageDependencies.A.JA.outputs['DetermineResult.doThing'] ]
    steps:
    - bash: echo "Hello world stage B first job"
    - script: echo $(varFromStageA)
```

We used variable from the previous stage in the next stage but at a job level in a condition expression. But not only there. We also mapped that variable to variable in this job and now we can use in further steps.

In my opinion, this is an oversight that we can't use `stageDependencies` and I hope that it will change in the future. And what do you think about this? Please share your opinions in comments.

And as always you will find this code on my [github](https://github.com/kmadof/devops-manual/tree/master/cross-stage-variables).