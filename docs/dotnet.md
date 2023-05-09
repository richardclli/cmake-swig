| Linux | macOS | Windows |
|-------|-------|---------|
| [![Status][dotnet_linux_svg]][dotnet_linux_link] | [![Status][dotnet_macos_svg]][dotnet_macos_link] | [![Status][dotnet_windows_svg]][dotnet_windows_link] |

[dotnet_linux_svg]: https://github.com/Mizux/cmake-swig/actions/workflows/amd64_linux_dotnet.yml/badge.svg
[dotnet_linux_link]: https://github.com/Mizux/cmake-swig/actions/workflows/amd64_linux_dotnet.yml
[dotnet_macos_svg]: https://github.com/Mizux/cmake-swig/actions/workflows/amd64_macos_dotnet.yml/badge.svg
[dotnet_macos_link]: https://github.com/Mizux/cmake-swig/actions/workflows/amd64_macos_dotnet.yml
[dotnet_windows_svg]: https://github.com/Mizux/cmake-swig/actions/workflows/amd64_windows_dotnet.yml/badge.svg
[dotnet_windows_link]: https://github.com/Mizux/cmake-swig/actions/workflows/amd64_windows_dotnet.yml


# .Net Wrapper Status
* [x] GNU/Linux wrapper
* [x] MacOS wrapper
* [ ] Windows wrapper

# Introduction
Try to build a .NetStandard2.0 native (for win-x64, linux-x64 and osx-x64) nuget multi package using [`dotnet/cli`](https://github.com/dotnet/cli) and the *new* .csproj format.  

# Build the Binary Packages
To build the .Net nuget packages, simply run:
```sh
cmake -S. -Bbuild -DBUILD_DOTNET=ON
cmake --build build --target dotnet_package -v
```
note: Since `dotnet_package` is in target `all`, you can also ommit the
`--target` option.

# Technical Notes
First you should take a look at my [dotnet-native](https://github.com/Mizux/dotnet-native) project to understand the layout.  
Here I will only focus on the CMake/SWIG tips and tricks.

## Build directory layout
Since .Net use a `.csproj` project file to orchestrate everything we need to
generate them and have the following layout:

```shell
<CMAKE_BINARY_DIR>/dotnet
build/dotnet
├── CMakeSwig
│   ├── Bar
│   │   ├── Bar.cs
│   │   ├── csBar.cs
│   │   ├── csBarPINVOKE.cs
│   │   └── SWIGTYPE_p_int64_t.cs
│   ├── Foo
│   │   ├── csFoo.cs
│   │   ├── csFooPINVOKE.cs
│   │   ├── Foo.cs
│   │   └── SWIGTYPE_p_int64_t.cs
│   └── FooBar
│       ├── csFooBar.cs
│       ├── csFooBarPINVOKE.cs
│       ├── FooBar.cs
│       └── SWIGTYPE_p_int64_t.cs
├── Directory.Build.props
├── logo.png
├── Mizux.CMakeSwig
│   └── Mizux.CMakeSwig.csproj
└── Mizux.CMakeSwig.runtime.linux-x64
    └── Mizux.CMakeSwig.runtime.linux-x64.csproj
```
src: `tree build/dotnet --prune -I "obj|bin"`

## Table of Content
* [Requirement](#requirement)
* [Directory Layout](#directory-layout)
* [Build Process](#build-process)
  * [Local Mizux.CMakeSwig Package](#local-mizuxfoo-package)
    * [Building a runtime Mizux.CMakeSwig Package](#building-local-runtime-mizuxfoo-package)
    * [Building a Local Mizux.CMakeSwig Package](#building-local-mizuxfoo-package)
    * [Testing the Local Mizux.CMakeSwig Package](#testing-local-mizuxfoo-package)
  * [Complete Mizux.CMakeSwig Package](#complete-mizuxfoo-package)
    * [Building all runtime Mizux.CMakeSwig Package](#building-all-runtime-mizuxfoo-package)
    * [Building a Complete Mizux.CMakeSwig Package](#building-complete-mizuxfoo-package)
    * [Testing the Complete Mizux.CMakeSwig Package](#testing-complete-mizuxfoo-package)
* [Appendices](#appendices)
  * [Ressources](#ressources)
  * [Issues](#issues)
* [Misc](#misc)

# Requirement
You'll need the ".Net Core SDK 3.1" to get the dotnet cli.
i.e. We won't/can't rely on VS 2019 since we want a portable cross-platform [`dotnet/cli`](https://github.com/dotnet/cli) pipeline. 

# Directory Layout
* [`src/dotnet/Mizux.CMakeSwig.runtime.linux-x64`](src/dotnet/Mizux.CMakeSwig.runtime.linux-x64)
Contains the hypothetical C++ linux-x64 shared library.
* [`src/dotnet/Mizux.CMakeSwig.runtime.osx-x64`](src/dotnet/Mizux.CMakeSwig.runtime.osx-x64)
Contains the hypothetical C++ osx-x64 shared library.
* [`src/dotnet/Mizux.CMakeSwig.runtime.win-x64`](src/dotnet/Mizux.CMakeSwig.runtime.win-x64)
Contains the hypothetical C++ win-x64 shared library.
* [`src/dotnet/Mizux.CMakeSwig`](src/dotnet/Mizux.CMakeSwig)
Is the .NetStandard2.0 library which should depends on all previous available packages.
* [`src/dotnet/Mizux.CMakeSwigApp`](src/dotnet/Mizux.CMakeSwigApp)
Is a .NetCoreApp2.1 application with a **`PackageReference`** to `Mizux.CMakeSwig` project to test.

note: While Microsoft use `runtime-<rid>.Company.Project` for native libraries naming,
it is very difficult to get ownership on it, so you should prefer to use`Company.Project.runtime-<rid>` instead since you can have ownership on `Company.*` prefix more easily.

# Build Process
To Create a native dependent package we will split it in two parts:
* A bunch of `Mizux.CMakeSwig.runtime.{rid}.nupkg` packages for each 
[Runtime Identifier (RId)](https://docs.microsoft.com/en-us/dotnet/core/rid-catalog) targeted.
* A meta-package `Mizux.CMakeSwig.nupkg` depending on each runtime packages.

note: [`Microsoft.NetCore.App` packages](https://www.nuget.org/packages?q=Microsoft.NETCore.App)
follow this layout.

We have two use case scenario:
1. Locally, be able to build a Mizux.CMakeSwig package which **only** target the local `OS Platform`,
i.e. building for only one 
[Runtime Identifier (RID)](https://docs.microsoft.com/en-us/dotnet/core/rid-catalog).  
note: This is usefull when the C++ build is a complex process for Windows, Linux and MacOS.  
i.e. You don't support cross-compilation for the native library.

2. Be able to create a complete cross-platform (ed. platform as multiple rid) Mizux.CMakeSwig package.  
i.e. First you generate each native Nuget package (`runtime.{rid}.Mizux.CMakeSwig.nupkg`) 
on each native architecture, then copy paste these artifacts on one native machine
to generate the meta-package `Mizux.CMakeSwig`.

## Local Mizux.CMakeSwig Package
Let's start with scenario 1: Create a *Local* `Mizux.CMakeSwig.nupkg` package targeting **one** [Runtime Identifier (RID)](https://docs.microsoft.com/en-us/dotnet/core/rid-catalog).  
We would like to build a `Mizux.CMakeSwig.nupkg` package which only depends on one `Mizux.CMakeSwig.runtime.{rid}.nupkg` in order to dev/test locally.  

The pipeline for `linux-x64` should be as follow:  
note: The pipeline will be similar for `osx-x64` and `win-x64` architecture, don't hesitate to look at the CI log.
![Local Pipeline](doc/local_pipeline.svg)
![Legend](doc/legend.svg)

### Building local Mizux.CMakeSwig.runtime Package
disclaimer: for simplicity, in this git repository, we suppose the `g++` and `swig` process has been already performed.  
Thus we have the C++ shared library `Native.so` and the swig generated C# wrapper `Native.cs` already available.  
note: For a C++ CMake cross-platform project sample, take a look at [Mizux/cmake-cpp](https://github.com/Mizux/cmake-cpp).   
note: For a C++/Swig CMake cross-platform project sample, take a look at [Mizux/cmake-swig](https://github.com/Mizux/cmake-swig). 

So first let's create the local `Mizux.CMakeSwig.runtime.{rid}.nupkg` nuget package.

Here some dev-note concerning this `Mizux.CMakeSwig.runtime.{rid}.csproj`.
* `AssemblyName` must be `Mizux.CMakeSwig.dll` i.e. all {rid} projects **must** generate an assembly with the **same** name (i.e. no {rid} in the name)
  ```xml
  <RuntimeIdentifier>{rid}</RuntimeIdentifier>
  <AssemblyName>Mizux.CMakeSwig</AssemblyName>
  <PackageId>Mizux.CMakeSwig.runtime.{rid}</PackageId>
  ```
* Once you specify a `RuntimeIdentifier` then `dotnet build` or `dotnet build -r {rid}` 
will behave identically (save you from typing it).
  - note: not the case if you use `RuntimeIdentifiers` (notice the 's')
* It is [recommended](https://docs.microsoft.com/en-us/nuget/create-packages/native-packages)
to add the tag `native` to the 
[nuget package tags](https://docs.microsoft.com/en-us/dotnet/core/tools/csproj#packagetags)
  ```xml
  <PackageTags>native</PackageTags>
  ```
* Specify the output target folder for having the assembly output in `runtimes/{rid}/lib/{framework}` in the nupkg
  ```xml
  <BuildOutputTargetFolder>runtimes/$(RuntimeIdentifier)/lib</BuildOutputTargetFolder>
  ```
  note: Every files with an extension different from `.dll` will be filter out by nuget.
* Add the native shared library to the nuget package in the repository `runtimes/{rib}/native`. e.g. for linux-x64:
  ```xml
  <Content Include="*.so">
    <PackagePath>runtimes/linux-x64/native/%(Filename)%(Extension)</PackagePath>
    <Pack>true</Pack>
    <CopyToOutputDirectory>PreserveNewest</CopyToOutputDirectory>
  </Content>
  ```
* Generate the runtime package to a defined directory (i.e. so later in meta CMakeSwig package we will be able to locate it)
  ```xml
  <PackageOutputPath>{...}/packages</PackageOutputPath>
  ```
* Generate the Reference Assembly (but don't include it to this runtime nupkg !, see below for explanation) using:
  ```xml
  <ProduceReferenceAssembly>true</ProduceReferenceAssembly>
  ```

Then you can generate the package using:
```bash
dotnet pack src/runtime.{rid}.CMakeSwig
```
note: this will automatically trigger the `dotnet build`.

If everything good the package (located where your `PackageOutputPath` was defined) should have this layout:
```
{...}/packages/Mizux.CMakeSwig.runtime.{rid}.nupkg:
\- Mizux.CMakeSwig.runtime.{rid}.nuspec
\- runtimes
   \- {rid}
      \- lib
         \- {framework}
            \- Mizux.CMakeSwig.dll
      \- native
         \- *.so / *.dylib / *.dll
... 
```
note: `{rid}` could be `linux-x64` and `{framework}` could be `netstandard2.0`

tips: since nuget package are zip archive you can use `unzip -l <package>.nupkg` to study their layout.

### Building local Mizux.CMakeSwig Package
So now, let's create the local `Mizux.CMakeSwig.nupkg` nuget package which will depend on our previous runtime package.

Here some dev-note concerning this `CMakeSwig.csproj`.
* This package is a meta-package so we don't want to ship an empty assembly file:
  ```xml
  <IncludeBuildOutput>false</IncludeBuildOutput>
  ```
* Add the previous package directory:
  ```xml
  <RestoreSources>{...}/packages;$(RestoreSources)</RestoreSources>
  ```
* Add dependency (i.e. `PackageReference`) on each runtime package(s) availabe:
  ```xml
  <ItemGroup Condition="Exists('{...}/packages/Mizux.CMakeSwig.runtime.linux-x64.1.0.0.nupkg')">
    <PackageReference Include="Mizux.CMakeSwig.runtime.linux-x64" Version="1.0.0" />
  </ItemGroup>
  ```
  Thanks to the `RestoreSource` we can work locally we our just builded package
  without the need to upload it on [nuget.org](https://www.nuget.org/).
* To expose the .Net Surface API the `CMakeSwig.csproj` must contains a least one 
[Reference Assembly](https://docs.microsoft.com/en-us/nuget/reference/nuspec#explicit-assembly-references) of the previously rumtime package.
  ```xml
  <Content Include="../Mizux.CMakeSwig.runtime.{rid}/bin/$(Configuration)/$(TargetFramework)/{rid}/ref/*.dll">
    <PackagePath>ref/$(TargetFramework)/%(Filename)%(Extension)</PackagePath>
    <Pack>true</Pack>
    <CopyToOutputDirectory>PreserveNewest</CopyToOutputDirectory>
  </Content>
  ```

Then you can generate the package using:
```bash
dotnet pack src/.CMakeSwig
```

If everything good the package (located where your `PackageOutputPath` was defined) should have this layout:
```
{...}/packages/Mizux.CMakeSwig.nupkg:
\- Mizux.CMakeSwig.nuspec
\- ref
   \- {framework}
      \- Mizux.CMakeSwig.dll
... 
```
note: `{framework}` could be `netstandard2.0`

### Testing local Mizux.CMakeSwig Package 
We can test everything is working by using the `CMakeSwigApp` project.

First you can build it using:
```
dotnet build src/CMakeSwigApp
```
note: Since CMakeSwigApp `PackageReference` CMakeSwig and add `{...}/packages` to the `RestoreSource`.
During the build of CMakeSwigApp you can see that `Mizux.CMakeSwig` and
`Mizux.CMakeSwig.runtime.{rid}` are automatically installed in the nuget cache.

Then you can run it using:
```
dotnet build src/CMakeSwigApp
```

You should see something like this
```bash
[1] Enter CMakeSwigApp
[2] Enter CMakeSwig
[3] Enter CMakeSwig.{rid}
[3] Exit CMakeSwig.{rid}
[2] Exit CMakeSwig
[1] Exit CMakeSwigApp
```

## Complete Mizux.CMakeSwig Package
Let's start with scenario 2: Create a *Complete* `Mizux.CMakeSwig.nupkg` package targeting multiple [Runtime Identifier (RID)](https://docs.microsoft.com/en-us/dotnet/core/rid-catalog).  
We would like to build a `Mizux.CMakeSwig.nupkg` package which depends on several `Mizux.CMakeSwig.runtime.{rid}.nupkg`.  

The pipeline should be as follow:  
note: This pipeline should be run on any architecture,
provided you have generated the three architecture dependent `Mizux.CMakeSwig.runtime.{rid}.nupkg` nuget packages.
![Full Pipeline](doc/full_pipeline.svg)
![Legend](doc/legend.svg)

### Building All runtime Mizux.CMakeSwig Package 
Like in the previous scenario, on each targeted OS Platform you can build the coresponding
`runtime.{rid}.Mizux.CMakeSwig.nupkg` package.

Simply run on each platform
```bash
dotnet build src/runtime.{rid}.CMakeSwig
dotnet pack src/runtime.{rid}.CMakeSwig
```
note: replace `{rid}` by the Runtime Identifier associated to the current OS platform.

Then on one machine used, you copy all other packages in the `{...}/packages` so
when building `CMakeSwig.csproj` we can have access to all package...

### Building Complete Mizux.CMakeSwig Package 
This is the same step than in the previous scenario, since we "see" all runtime
packages in `{...}/packages`, the project will depends on each of them.

Once copied all runtime package locally, simply run:
```bash
dotnet build src/CMakeSwig
dotnet pack src/CMakeSwig
```

### Testing Complete Mizux.CMakeSwig Package 
We can test everything is working by using the `CMakeSwigApp` project.

First you can build it using:
```
dotnet build src/CMakeSwigApp
```
note: Since CMakeSwigApp `PackageReference` CMakeSwig and add `{...}/packages` to the `RestoreSource`.
During the build of CMakeSwigApp you can see that `Mizux.CMakeSwig` and
`runtime.{rid}.Mizux.CMakeSwig` are automatically installed in the nuget cache.

Then you can run it using:
```
dotnet run --project src/CMakeSwigApp
```

You should see something like this
```bash
[1] Enter CMakeSwigApp
[2] Enter CMakeSwig
[3] Enter CMakeSwig.{rid}
[3] Exit CMakeSwig.{rid}
[2] Exit CMakeSwig
[1] Exit CMakeSwigApp
```

# Ressources
Few links on the subject...

## Documention
First take a look at my [dotnet-native](https://github.com/Mizux/dotnet-native) project.

## Related Issues
Some issue related to this process
* [Nuget needs to support dependencies specific to target runtime #1660](https://github.com/NuGet/Home/issues/1660)
* [Improve documentation on creating native packages #238](https://github.com/NuGet/docs.microsoft.com-nuget/issues/238)

## Runtime IDentifier (RID)
* [.NET Core RID Catalog](https://docs.microsoft.com/en-us/dotnet/core/rid-catalog)
* [Creating native packages](https://docs.microsoft.com/en-us/nuget/create-packages/native-packages)
* [Blog on Nuget Rid Graph](https://natemcmaster.com/blog/2016/05/19/nuget3-rid-graph/)

## Reference on .csproj format
* [Common MSBuild project properties](https://docs.microsoft.com/en-us/visualstudio/msbuild/common-msbuild-project-properties?view=vs-2017)
* [MSBuild well-known item metadata](https://docs.microsoft.com/en-us/visualstudio/msbuild/msbuild-well-known-item-metadata?view=vs-2017)
* [Additions to the csproj format for .NET Core](https://docs.microsoft.com/en-us/dotnet/core/tools/csproj)
