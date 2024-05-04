---
layout: post
title: "Creating and publishing a simple NuGet package"
categories: blog
---

I publish my first [NuGet package](https://www.nuget.org/packages/Blazer/) the other day, and I figured I'd do a short write-up of the steps involved. It's actually _really_ easy, there are no difficult technical challenges to overcome here, nor are there challenges of any kind. So, this post is mainly meant as just a quick reference for myself, in case I decide to publish more stuff in the future.

## Prerequisites

Number one: get a NuGet account, if you don't already have one. You'll also need the NuGet command line tool, which you can get [here](https://dist.nuget.org/index.html). Put the .exe in your PATH somewhere, or just stick it in the project folder that you want to publish (next to the .csproj file).

## Creating the package

Open a command prompt in the project folder that you want to publish, and call

    > nuget spec

This will create the .nuspec file, and fill it with some defaults. The .nuspec file is an Xml file containing package metadata. It can contain hard-coded values, as well as tokens that will be replaced during package creation. For an overview of the replacement tokens, see the [NuGet docs](https://docs.nuget.org/ndocs/schema/nuspec#replacement-tokens). My .nuspec file ended up looking like this:

```xml
<?xml version="1.0"?>
<package>
    <metadata>
        <id>$id$</id>
        <version>$version$</version>
        <title>$title$</title>
        <authors>$author$</authors>
        <owners>b-w</owners>
        <licenseUrl>https://github.com/b-w/Blazer/blob/master/LICENSE.txt</licenseUrl>
        <projectUrl>https://github.com/b-w/Blazer</projectUrl>
        <requireLicenseAcceptance>true</requireLicenseAcceptance>
        <description>$description$</description>
        <copyright>$copyright$</copyright>
        <tags>SQL ADO.NET data-access ORM micro-ORM object-mapper</tags>
    </metadata>
</package>
```

The owner tag contains my NuGet account name. I also entered project- and license URLs, and some relevant tags. The remaining values come from my project's assembly information. I changed the assembly properties to use fixed versioning, instead of having the build- and revision numbers autogenerate.

After creating the .nuspec file, I built the project in release mode from Visual Studio. I then called

```
> nuget pack Blazer.csproj -properties Configuration=Release
```

This generates the .nupkg file. This file is then uploaded to the NuGet gallery, and we're done. That's all it takes to publish a NuGet package.
