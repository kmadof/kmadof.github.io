---
layout: post
title: DB deployments with FAKE tool
date: 2019-12-14
excerpt_separator:  <!--more-->
tags: DB FAKE-tool
---

This story has begun quite long ago. I got a chance to work on projects without automatic deployment process. It was strange a bit, because we had deployments process for both front-end and back-end projects, but not for databases. For databases, we were generating SQL scripts from Visual Studio, and then we executed them in our Test environment. This was perfect place to save our time and [FAKE](https://fake.build/) did the right job here.

We use in my company Visual Studio SQL Server DB projects to handle SQL scripts. And exactly this kind of project we are going to deploy using FAKE. I prepared suolution with two DB projects based on [AdventureWorks db](https://docs.microsoft.com/en-us/sql/samples/adventureworks-install-configure?view=sql-server-ver15). The code you can find in my repo [FsharpAdventCalendar2019](https://github.com/kmadof/FsharpAdventCalendar2019). In each of these projects I added two publish profile in Profiles folders. One for Dev and one for Test env. In my example they are almost the same. There is only one difference. Dev file has _Block incremental deployment if data los occur_ option enabled and Test file disabled. This is only to prove that we can parameterize selected profile.

Dev publish profile for FsharpAdventureWorks db
{% gist 171de5d4c3f361e23322aa382d67bfb6 Dev.publish.xml %}

Test publish profile for FsharpAdventureWorksDW db
{% gist 3f48e15414d5d837e1b251af96d4db06 Test.publish.xml %}

Our goal is to deploy two databases at once!

### Build target ###

Let's create build.fsx file at solution level. Once we have it we can define our first target and build our solution.

{% highlight fsharp %}
// Properties
let buildDir = "./build/"

// Helper methods
let getProjectName (fullPath: string) =
    fullPath.Split('\\').Last().Replace(".sqlproj", "")

let build (projectDirectoryPair:string * string) =
    let project, projectName = projectDirectoryPair
    MSBuild.runRelease  (fun p ->
        { p with Properties = [ ("DeployOnBuild", "false") ] } ) (buildDir + projectName + "/") "Build" (Seq.singleton project)
       |> Trace.logItems "DbBuild-Output: "

Target.create "BuildDb" (fun _ -> 

    !! "**/*.sqlproj" 
    |> Seq.map (fun x -> (x, getProjectName x))
    |> Seq.iter build
)

// start build
Target.runOrDefault "BuildDb"
{% endhighlight %}


BuildDb target gets all project files (files with sqlproj extension), extracts project name from file name, runs msbuild (via [msbuild module](https://fake.build/apidocs/v5/fake-dotnet-msbuild.html)) for each and saves DACPAC files in build directory. Just few lines of code and we have our solutiond build. Let's run fake build command and check result:

![Build DB target result](/images/fake_build_db_target.png)

Let's check what happen if we make not compilable mistake in table declaration.

![Build DB target failed result](/images/fake_build_db_target-failed.png)


### Publish target ###

Now is time to publish our databases. For readability I skipped here BuildDb target details.

{% highlight fsharp %}
let publish (projectDirectoryPair:string * string) =
    let project, projectName = projectDirectoryPair
    MSBuild.runRelease  (fun p ->
        { p with Properties = [ ("DeployOnBuild", "true"); ("SqlPublishProfilePath","./Profiles/DEV.publish.xml") ] } ) (buildDir + projectName + "/") "Publish" (Seq.singleton project)
       |> Trace.logItems "DbBuild-Output: "

Target.create "DeployDb" (fun _ -> 

    !! "**/*.sqlproj"
    |> Seq.map (fun x -> (x, getProjectName x))
    |> Seq.iter publish
)

"BuildDb"
    ==> "DeployDb"

// start build
Target.runOrDefault "DeployDb"
{% endhighlight %}

This step is actually pretty similar to Build database step. We just passed slighlty different parameters to msbuild tool:
- we have Publish target instead of Build
- we passed publish profiles

Let's run fake build command this time and check the result:

![Deploy DB target result](/images/fake_deploy_db_target.png)

In that way we just deployed our databases. How cool is that? Just few lines of code and we have all things done.

<div class="message">
  There is another way of deploying databases in FAKE. Since we already created DACPAC files in BuildDB target, we can use SqlPackage tool (which is delivered with VisualStudio). We can use this tool via [Fake.DacPac](https://fake.build/sql-dacpac.html) module. However, I found an issue there with defaults argument. We can pass plenty of prameters to SqlPackage via command line or we can put them in publish profile. In case where we set the same parameter in command line and publish profile, the first one wins. This is the same with Fake.DacPac, but there is one difference. If we do not pass for instance _block on possible data loss_ parameter, Fake.DacPac set it's own default. In that case even if we have this parameter set in publish profile and we don't set it directly in Fake.DacPac it will be overriden by Fake.DacPac default. I already created an <a href="https://github.com/fsharp/FAKE/issues/2321">issue</a> on GitHub and hopefully I should fix it soon
</div>

### Targets parameterization ###

Scripts above are good, but they are rigid. They deploy always all databases and use only Dev publish profile. We can easily change it and get in that way a tool which can be used to deploy database in our CI/CD pipeline. To add these parameters we need to sligtly modify our code.

{% highlight fsharp %}
let publish env (projectDirectoryPair:string * string) =
    let project, projectName = projectDirectoryPair
    MSBuild.runRelease  (fun p ->
        { p with Properties = [ ("DeployOnBuild", "true"); ("SqlPublishProfilePath","./Profiles/" + env + ".publish.xml") ] } ) (buildDir + projectName + "/") "Publish" (Seq.singleton project)
       |> Trace.logItems "DbBuild-Output: "

// Targets

Target.create "BuildDb" (fun _ -> 

    let db = Fake.Core.Environment.environVarOrDefault "db" "*"

    !! (sprintf "**/%s.sqlproj" db)
    |> Seq.map (fun x -> (x, getProjectName x))
    |> Seq.iter build
)

Target.create "DeployDb" (fun _ -> 

    let env = Fake.Core.Environment.environVarOrDefault "env" "Dev"
    let db = Fake.Core.Environment.environVarOrDefault "db" "*"

    !! (sprintf "**/%s.sqlproj" db)
    |> Seq.map (fun x -> (x, getProjectName x))
    |> Seq.iter (publish env)
)
{% endhighlight %}

Now if we want to deploy using Test publish profile we need to run:

> fake build -e env=Test

and if we want to deploy only FsharpAdventureWorks database:

> fake build -e db=FsharpAdventureWorks


and in case when we want to deploy FsharpAdventureWorks database using Test profile:

> fake build -e db=FsharpAdventureWorks -e env=Test

Changes in code, are very small. We only read parameters given in command line and pass them to functions to properly execute targets. This shows, how powerfull FAKE can be. The code is really expressive and easy to reason about. ANother benefit is having the same approach on DEV and other environments. Having such consistency we can faster catch any potential deployment issue.


### Clean target ###

As our last modification fo this script I want to add clean step to clear build directory before main targets are executed.

{% highlight fsharp %}
Target.create "Clean" (fun _ ->
    Shell.cleanDir buildDir
)

"Clean"
    ==> "BuildDb"
    ==> "DeployDb"

// start build
Target.runOrDefault "DeployDb"
{% endhighlight %}

### Summary ###

In my opinion FAKE is really good tool and with plenty options which we can adjust to our need. And even if there is sth what doesn't work as it sould be it is open source and community there is more than firendly. I hope that soon and I will do my contribution for the community.

This post is part of [F# Advent Calendar 2019](https://sergeytihon.com/2019/11/05/f-advent-calendar-in-english-2019/). Thanks to Sergey Tihon for running that event.