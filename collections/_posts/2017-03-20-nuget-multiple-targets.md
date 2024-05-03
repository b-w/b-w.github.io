---
layout: post
title: "Targeting multiple .NET platforms in a single NuGet package with Visual Studio 2017"
categories: blog
---

As the proud owner of a [NuGet package](https://www.nuget.org/packages/Blazer) (72 downloads and counting!), I'm interested in targeting the new [.NET Standard](https://docs.microsoft.com/en-us/dotnet/articles/standard/library), in order to make my package available on a wider variety of platforms. In a [previous post](/Blog/Post/80), I showed how to create and publish a simple NuGet package. However, this only covered the basic scenario of targeting a single platform. I now want to target multiple platforms: I want to target .NET Standard, while also remaining compatible with version 4.5 of the full .NET Framework.

NuGet has [supported](https://docs.microsoft.com/en-us/nuget/create-packages/supporting-multiple-target-frameworks) placing multiple versions of the same library in one package for quite a while now. However, exactly how to create such a package (including setting up Visual Studio) is never fully described. There is [some documentation](https://docs.microsoft.com/en-us/nuget/create-packages/creating-a-package), but it's all scattered information that fails to paint the full picture.

In this post, I will demonstrate how to:

1.  Convert an existing codebase to .NET Standard.
2.  Target multiple .NET platforms from a single Visual Studio project.
3.  Create a NuGet package that targets multiple .NET platforms.

We'll be using Visual Studio 2017, and my [Blazer](https://github.com/b-w/Blazer) library as a working example.

## The new csproj file

We're starting off with a Visual Studio 2015 solution and an old csproj file. The first step involves upgrading to the new csproj format. To do this, we'll simply get rid of the old project file, and replace it with a new Blazer.csproj with the following contents:

```xml
<Project Sdk="Microsoft.NET.Sdk">

    <PropertyGroup>
    <TargetFramework>netstandard1.3</TargetFramework>
    </PropertyGroup>

</Project>
```

Yes, that's the entire file. The new format is pretty lean. Anyway, when we place this file in the old project folder and add it to the solution, all of the source files automatically show up in the Project Explorer.

![Visual Studio project explorer](/assets/img/blog/2017/03/vs2017-project.png)

That was easy! Unfortunately, the project doesn't compile. Which brings us to...

## Fixing the code

Our new project now targets .NET Standard 1.3, which isn't compatible with our old .NET 4.5 code. Although Standard 1.3 contains some APIs which are not in Framework 4.5, the real problem is that Framework 4.5 contains a lot of things not covered by Standard 1.3.

If this stuff confuses you, simply refer to this handy table to clear things up:

<table border="1" cellpadding="1" cellspacing="1">
    <thead>
        <tr>
            <th scope="row">Platform</th>
            <th colspan="8" scope="col">.NET Standard</th>
        </tr>
    </thead>
    <tbody>
        <tr>
            <th scope="row">&nbsp;</th>
            <th scope="col">1.0</th>
            <th scope="col">1.1</th>
            <th scope="col">1.2</th>
            <th scope="col">1.3</th>
            <th scope="col">1.4</th>
            <th scope="col">1.5</th>
            <th scope="col">1.6</th>
            <th scope="col">2.0</th>
        </tr>
        <tr>
            <th scope="row">.NET Core</th>
            <td>&rarr;</td>
            <td>&rarr;</td>
            <td>&rarr;</td>
            <td>&rarr;</td>
            <td>&rarr;</td>
            <td>&rarr;</td>
            <td>1.0</td>
            <td>vNext</td>
        </tr>
        <tr>
            <th scope="row">.NET Framework</th>
            <td>&rarr;</td>
            <td>4.5</td>
            <td>4.5.1</td>
            <td>4.6</td>
            <td>4.6.1</td>
            <td>4.6.2</td>
            <td>vNext</td>
            <td>4.6.1</td>
        </tr>
    </tbody>
</table>

So anyway, what does this mean for us? Simply put, we'll have to go through our code and fix all of the incompatibilities. It's tedious work but we have no choice. There's basically two kinds of errors we encounter: missing packages and missing APIs.

### Missing packages

Some types, like the things we use from System.Data, are not found in the [.NET Standard SDK](https://www.nuget.org/packages/NETStandard.Library/). As such, they must be included as separate NuGet packages. To figure out in which packages your missing types are hiding, you can use the extremely useful [Reverse Package Search](http://packagesearch.azurewebsites.net/) site. I ended up needing four additional packages.

![Visual Studio project dependencies](/assets/img/blog/2017/03/vs2017-packages.png)

### Missing APIs

Some APIs which are available in the full .NET Framework are not available in .NET Standard. For instance, Type.IsEnum and Type.IsValueType are commonly used across several Blazer classes, but don't exist in Standard 1.3\. In Standard 1.3, you're supposed to do Type.GetTypeInfo().IsEnum. And while this particular API does exist in Framework 4.5, it does not exist in Framework 4.0, which I also intend on targeting.

So, we'll solve this problem through feature toggles. In the csproj file, I add the following:

```xml
<Project Sdk="Microsoft.NET.Sdk">

    <!-- [more stuff...] -->

    <PropertyGroup Condition="'$(TargetFramework)'=='netstandard1.3'">
    <DefineConstants>FEATURE_TYPE_INFO</DefineConstants>
    </PropertyGroup>

</Project>
```

This defines a feature flag for us whenever we are targeting Standard 1.3\. Then, everywhere this particular feature is used, I change my code like in the following example:

```csharp
#if FEATURE_TYPE_INFO
    if (type.GetTypeInfo().IsEnum && !m_getMethodMap.ContainsKey(type))
#else
    if (type.IsEnum && !m_getMethodMap.ContainsKey(type))
#endif
    {
        type = Enum.GetUnderlyingType(type);
    }
```

We simply create multiple versions of our code, based on which features are available. It's not the prettiest solution, but it's the price we pay for targeting multiple platforms.

### AssemblyInfo

Last but not least, we run into a problem involving the old AssemblyInfo.cs file. In it, we define a bunch of attributes (AssemblyTitle, AssemblyDescription, etc) which the compiler is now complaining are duplicates. It turns out that with the new csproj format, the compiler generates these attributes for you based on information from the csproj file.

The fix is simple: we remove all of the attributes from the AssemblyInfo.cs file that the compiler complains about. To replace them, we add the following info to the csproj file:

```xml
<Project Sdk="Microsoft.NET.Sdk">

    <!-- [more stuff...] -->

    <PropertyGroup>
    <Version>0.1.2</Version>
    <FileVersion>0.1.2</FileVersion>
    <Authors>Bart Wolff</Authors>
    <Description>High-performance ADO.NET object mapper.</Description>
    <Copyright>Copyright (c) 2016 Bart Wolff</Copyright>
    </PropertyGroup>

</Project>
```

...and that's it! We now have a working library, that compiles, and targets .NET Standard 1.3!

## Targeting multiple platforms

Now that we have successfully converted our project to target .NET Standard 1.3, it's time re-introduce our targeting of .NET Framework 4.5, as well as Framework 4.0 and 4.6\. Doing so requires an edit of the csproj file. This:

```xml
<Project Sdk="Microsoft.NET.Sdk">

    <PropertyGroup>
    <TargetFramework>netstandard1.3</TargetFramework>
    </PropertyGroup>

    <!-- [more stuff...] -->

</Project>
```

...is replaced with this:

```xml
<Project Sdk="Microsoft.NET.Sdk">

    <PropertyGroup>
    <TargetFrameworks>netstandard1.3;net40;net45;net46</TargetFrameworks>
    </PropertyGroup>

    <!-- [more stuff...] -->

</Project>
```

Note that TargetFramework becomes TargetFramework**s**!

We also need to include our imports, so we also add the following:

```xml
<Project Sdk="Microsoft.NET.Sdk">

    <!-- [more stuff...] -->

    <ItemGroup Condition="'$(TargetFramework)'=='net40' OR '$(TargetFramework)'=='net45' OR '$(TargetFramework)'=='net46'">
    <Reference Include="System" />
    <Reference Include="System.Core" />
    <Reference Include="Microsoft.CSharp" />
    <Reference Include="System.Data" />
    </ItemGroup>

    <ItemGroup Condition="'$(TargetFramework)'=='net45' OR '$(TargetFramework)'=='net46'">
    <Reference Include="System.ComponentModel.DataAnnotations" />
    </ItemGroup>

</Project>
```

If we now look at a code file in Visual Studio, we notice the following options in the projects drop-down:

![Visual Studio project target platforms](/assets/img/blog/2017/03/vs2017-project-targets.png)

Here, we can switch between the various platforms we're targeting. Doing so affects the feature constants we've defined, as expected:

![Visual Studio feature toggle](/assets/img/blog/2017/03/vs2017-feature-1.png)

![Visual Studio feature toggle](/assets/img/blog/2017/03/vs2017-feature-2.png)

I needed to create some more feature toggles to ensure compatibility with Framework 4.0 (e.g. Async is not available), but other than that this is all it took to introduce these additional target platforms. Building the project produces four folders in our bin\Debug folder, one for each target:

![Build output](/assets/img/blog/2017/03/vs2017-build-targets.png)

## Creating a NuGet package

Now that we have our multi-platform project, it's time to turn this project into a multi-platform NuGet package. To do this, we first need to add some package metadata. In the olden days we used the nuspec file for this, but with the new build system all of this stuff is fully integrated into the csproj file. We add the following:

```xml
<Project Sdk="Microsoft.NET.Sdk">

    <!-- [more stuff...] -->

    <PropertyGroup>
    <PackageId>Blazer</PackageId>
    <PackageVersion>0.1.2</PackageVersion>
    <PackageRequireLicenseAcceptance>true</PackageRequireLicenseAcceptance>
    <PackageTags>SQL ADO.NET data-access ORM micro-ORM object-mapper</PackageTags>
    <PackageLicenseUrl>https://github.com/b-w/Blazer/blob/master/LICENSE.txt</PackageLicenseUrl>
    <PackageProjectUrl>https://github.com/b-w/Blazer</PackageProjectUrl>
    </PropertyGroup>

</Project>
```

It's a good thing Visual Studio 2017 allows you to edit the csproj file without unloading the project, because we need to be in here quite regularly.

Then, to create the NuGet package, we don't use nuget.exe. Instead, we open the Visual Studio Developer Command Prompt, browse to the project folder, and run:

```
> msbuild /t:pack /p:Configuration=Release
```

This builds your project and creates a NuGet package for all of its target platforms. We can verify this by opening the nupkg file (which is just a zip file) and browsing to the lib folder:

![NuGet package contents](/assets/img/blog/2017/03/vs2017-package.png)

We then simply upload this to nuget.org and we're [done](https://www.nuget.org/packages/Blazer)! And there we have it: a single Visual Studio 2017 project that targets multiple platforms, and is able to produce a NuGet package. Now let's see that download counter go through the roof now that I'm targeting .NET Standard...
