---
layout: post
title: DB deployments with Fake.Sql.SqlPackage
date: 2019-12-23
excerpt_separator:  <!--more-->
tags: DB FAKE-tool
og_image: /assets/images/posts/2019/db-deployments-with-fake-sql-sqlpackage/og-image.png
---

In my previous [post]({% post_url 2019-12-14-DB-deployments-with-FAKE-tool %}), I presented how we can deploy multiple visual studio database projects using [Fake build](https://fake.build/) tool. Here, I will present another option which is possible with version [5.19.0](https://github.com/fsharp/FAKE/releases/tag/5.19.0).

This version has a new module called [Fake.Sql.SqlPackage](https://fake.build/sql-sqlpackage.html), which is a redesign of previous [Fake.Sql.DacPac](https://fake.build/sql-dacpac.html) module. The reason, why the previous module needed redesign, it was a missing option (from my point of view crucial option) - publish profiles. I raised that point on GitHub [here](https://github.com/fsharp/FAKE/issues/2321) and solved it with this [pull request](https://github.com/fsharp/FAKE/pull/2366). If you are interested in SqlPackage itself you should check [the documentation](https://docs.microsoft.com/en-us/sql/tools/sqlpackage?view=sql-server-ver15).

To sum up this a bit long introduction - all that means that I had a pleasure to be Fake build contributor.


![Fake build feature release list](/images/fake_release_5.19.0.png)

### New publish target ###

All code can be found [here](https://github.com/kmadof/FsharpAdventCalendar2019) and is a continuation of the previous [post]({% post_url 2019-12-14-DB-deployments-with-FAKE-tool %}). The logic of DeployDb target doesn't change and thus I will skip it here. I will focus only on publish function which is used by this target.

Previously we used MSBuild tool to publish database projects.

{% highlight fsharp %}

let publish env (projectDirectoryPair:string * string) =
    let project, projectName = projectDirectoryPair
    MSBuild.runRelease  (fun p ->
        { p with Properties = [ ("DeployOnBuild", "true"); ("SqlPublishProfilePath","./Profiles/" + env + ".publish.xml") ] } ) (buildDir + projectName + "/") "Publish" (Seq.singleton project)
       |> Trace.logItems "DbDeploy-Output: "

{% endhighlight %}


In this approach, we build and publish the database. It means that we build project twice (once in BuildDb target and once in DeployDb target). With Fake.Sql.SqlPackage we can avoid that and use dacpac files created in BuildDb target as it is presented below.

{% highlight fsharp %}

let publishWithSqlPackageModule env (projectDirectoryPair:string * string) =
    let project, directory = projectDirectoryPair
    let dacPacPath = sprintf "./build/%s/%s.dacpac" directory directory
    let profile = sprintf "./%s/Profiles/%s.publish.xml" directory env

    Fake.Sql.SqlPackage.deployDb (fun args -> { args  with Source = dacPacPath; Profile = profile })

{% endhighlight %}

The function itself is simpler (in both version we just invoke Fake modules, but here we need to provide fewer parameters), but what is important even more. It is faster too. Publish with MSBuild takes 78 seconds.

![Deployments statistics with MS Build](/images/fake-deploy-target-ms-build.png)

While deployment with SqlPackage takes 66 seconds.

![Deployments statistics with SqlPackage](/images/fake-deploy-target-sql-package.png)

It may not seem like much, but SqlPackage version is 15.5% faster just on this example project. I made a similar switch in my company projects, which has a more complex database structure than this here. Numbers are much better - 172 seconds with MSBuild version versus 90 seconds with SqlPackage version. What gives 47.7% speedup! And it saves not only my time but everyone's time as we use it on our CI server.

### Summary ###

The conclusion will be really short. It is not the end of the world if open source software doesn't have needed feature. It can be your opportunity to pay back to the community.