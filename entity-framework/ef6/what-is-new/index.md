---
title: "What's new - EF6"
author: divega
ms.date: "09/12/2019"
ms.assetid: 41d1f86b-ce66-4bf2-8963-48514406fb4c
uid: ef6/what-is-new/index
---
# What's new in EF6

We highly recommend that you use the latest released version of Entity Framework to ensure you get the latest features and the highest stability.
However, we realize that you may need to use a previous version, or that you may want to experiment with new improvements in the latest pre-release.
To install specific versions of EF, see [Get Entity Framework](~/ef6/fundamentals/install.md).

## EF 6.3.0

The EF 6.3.0 runtime was released to NuGet in September 2019. The main goal of this release was to facilitate migrating existing applications that use EF 6 to .NET Core 3.0. The community has also contributed several bug fixes and enhancements. See the issues closed in each 6.3.0 [milestone](https://github.com/aspnet/EntityFramework6/milestones?state=closed) for details. Here are some of the more notable ones:

- Support for .NET Core 3.0
  - The EntityFramework package now targets .NET Standard 2.1 in addition to .NET Framework 4.x
  - The migrations commands have been rewritten to execute out of process and work with SDK-style projects
- Support for SQL Server HierarchyId
- Improved compatibility with Roslyn and NuGet PackageReference
- Added `ef6.exe` utility for enabling, adding, scripting, and applying migrations from assemblies. This replaces `migrate.exe`

### EF designer support

There's currently no support for using the EF designer directly on .NET Core or .NET Standard projects. 

You can work around this limitation by adding the EDMX file and the generated classes for the entities and the DbContext as linked files to a .NET Core 3.0 or .NET Standard 2.1 project in the same solution.

The linked files will look like this in the project file:

``` csproj 
&lt;ItemGroup&gt;
  &lt;EntityDeploy Include="..\EdmxDesignHost\Entities.edmx" Link="Model\Entities.edmx" /&gt;
  &lt;Compile Include="..\EdmxDesignHost\Entities.Context.cs" Link="Model\Entities.Context.cs" /&gt;
  &lt;Compile Include="..\EdmxDesignHost\Thing.cs" Link="Model\Thing.cs" /&gt;
  &lt;Compile Include="..\EdmxDesignHost\Person.cs" Link="Model\Person.cs" /&gt;
&lt;/ItemGroup&gt;
```

Note that the EDMX file is linked with the EntityDeploy build action. This is a special MSBuild task (now included in the EF 6.3 package) that takes care of adding the EF model into the target assembly as embedded resources (or copying it as files in the output folder, depending on the Metadata Artifact Processing setting in the EDMX). For more details on how to get this set up, see our [EDMX .NET Core sample](https://aka.ms/EdmxDotNetCoreSample).

## Past Releases

The [Past Releases](past-releases.md) page contains an archive of all previous versions of EF and the major features that were introduced on each release.
