---
layout: post
title:  "Squeezing startup performance out of ASP.NET Core MVC apps"
date:   2016-08-01
categories: aspnetcore performance mvc
---

I love ASP.NET Core and its version of MVC. So much has been streamlined 
and development is just a breeze. However, I've noticed as I've ported more and
more apps to use it that the startup performance in production has been lackluster
in comparison to development --
moreso for some apps than others. This little post details my short journey to getting
better startup times after deployments; I hope you find it useful or enjoyable.

## A Low-Hanging Fruit for Some Applications

I was able to identify two camps of applications: those that were taking roughly
22 seconds to first render and those that were taking roughly 12 seconds to first render.
(Does this sound familiar at all?) It did not take long to figured out that the apps
taking 22 seconds to render were using Entity Framework 6. The EF6 assembly is around
5 megabytes in size, and that needs to be JIT-ted each time the application starts -- unacceptable!

As it turns out, EF6 had been added to the NGen cache on my development machine, likely
by the Visual Studio installer at some point. The solution here was to run the following
command from an administrator prompt on our servers, within one of our published apps' folders:

```
"C:\Windows\Microsoft.NET\Framework64\v4.0.30319\ngen.exe" install EntityFramework.SqlServer.dll
```

Note that this only needed to be run once; all of the apps will share the cached native image.
Once I had finished running this command, I recycled the application pools and saw an improvement
of around 4.5 to 5 seconds. 

## An Obvious but More Challenging Performance Point

One thing I already knew wasn't helping my startup times was the runtime compilation of Razor views.
ASP.NET Core RC1 (or was it beta8?) had support for precompilation, but it was pulled thereafter to
be worked on for a later release. I wanted to see just how much faster my apps could start up with
precompilation so this weekend I wrote some code that would create and load an assembly containing
the precompiled views and this morning I converted it into a library that is usable as a `dotnet-*` tool:

> <https://github.com/tuespetre/dotnet-precompile-views>

The increase of speed here was tremendous: the apps that were taking ~12s to initial render
were now only taking ~4.75s. 

## One Last Squeeze

By using MVC, the `Microsoft.CodeAnalysis` and `Microsoft.CodeAnalysis.CSharp` libraries get
roped in and loaded at runtime. Like EF6, these are larger assemblies and incur some JIT cost
(they are together about 6MB.) Also like EF6, I installed them into the NGen cache by running
the following command from an administrator prompt on our servers, within one of our published apps' folders:

```
"C:\Windows\Microsoft.NET\Framework64\v4.0.30319\ngen.exe" install /ExeConfig:MyAppName.exe Microsoft.CodeAnalysis.CSharp.dll
```

The `/ExeConfig` option is important: NGen needs it to grab some binding redirects, or it will fail
to generate and install the native image. This resulted in an additional gain of ~1.5s, for an initial
render time of ~3.3s for the simpler applications and ~8.5s for the applications using EF6 and other things.

## Final Thoughts

At some point the `dotnet` CLI tooling will be able to generate native images so we won't have
to mess around with the cache, and at some point MVC will have official support for view precompilation.
Some of our apps use NEST which is also a fairly large assembly at 2MB, and I suspect that, similar to
EF6, there may be some runtime overhead due to use of reflection or similar. I still need to investigate
what could be done to slice some time off of those things, but for now I feel a ~3.3s post-deploy startup 
time for simple MVC apps is acceptable.