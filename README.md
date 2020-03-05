# Trick Unity into using packages from NuGet

What passes for configuration management in Unity3D is a shambling, rotting, dumpster zombie. They write their own version of Mono, they write their own build system, they've even written their own package management system (originally a hacked-up version of NPM, so they not only broke an existing package manager, they broke an already broken package manager). In the process, they create sharp demarcation lines between themselves, all the code written by users of their framework, and the huge ecosystem of third-party libraries in the rest of the .NET community.

Thankfully, with somewhat recent advances in both Unity's support for standard .NET (the deprecation and subsequent removal of the ancient ".NET 3.5-like" Unity backend) and Microsoft's development of standardized, portable .NET libraries (.NET Standard 2.0 and SDK Style Projects), we can finally work around all this malarky and get back to building quality software.

Here's how it works. Incidentally, this process also means we can now use other .NET-targeting programming languages to build code for Unity.

## Overview

To get everything to work just right, we have to commit to using a relatively rigid folder structure for our project. In principle, I'm not a fan of "magic folders", different areas of your project getting special treatment just because their parent folder is named a certain way. I didn't like it in Ruby-On-Rails, and I don't like it here. We are adults, we can write configuration files and fully specify our dependencies. But in practice, we only need to use this structure as the "trick" to get Unity to use independently sourced and managed dependencies. In day-to-day development, we're still setting up our projects as we want for development.

 1. Out-of-band of our Unity project (or, at least the Assets folder), we create our own Visual Studio Solution for doing all the development we want that is not touching anything involving MonoBehaviour.
    1. While .NET Standard 2.0 DLLs are technically consumable in Unity, the way in which their transient dependencies are specified *are not*. .NET Standard 2.0 DLLs deploy with a DLL for the code that they encapsulate, plus a `dependencies.json` file listing its transient dependencies. This is in contrast to .NET Framework, where transient dependencies get copied to the output directory of a DLL project. With that in mind, we'll be creating One-Assembly-To-Rule-Them-All to take .NET Standard 2.0 code and assemblies and .NET Framework v4.7.2 code and assemblies and package them all up in a way that Unity will not barf over.
    2. To ease in managing all of these different target framework types, we'll want all Project Files within this solution to be built in the newer [SDK Style Project](https://docs.microsoft.com/en-us/dotnet/core/tools/csproj) format.
    3. An XCOPY command takes the collected DLLs from the Ring-Of-Power project and dumps them into your Unity project's Assets/Plugins folder.
 2. In-band of our Unity project, you may want to consider setting the Backend Type in the Project Settings to ".NET 4.x". Unfortunately, I've encountered legitimate defects within the Mono implementations of the .NET Standard 2.0 version of some dependencies.

You may also want to read up on [Target frameworks in SDK-style projects](https://docs.microsoft.com/en-us/dotnet/standard/frameworks). This explains how you reference individual frameworks within MSBuild.

### Create the Out-of-band VS Solution

As stated before, create a new Visual Studio solution, anywhere that is not a child directory of the Unity project's Assets folder. In this example repo, I have the following folder structure:

 1. `Assets/` - this is the jerkasaurus rex of Unity
    1. `Plugins/` - this is where we will take a dump on Unity's head
 2. `Packages/` - there is only ever one file in here, `manifest.json`, so why does it need its own folder? It would be one thing if the package cache were stored here. That might make some sense. But that is not the case.
 3. `ProjectSettings/` - blah blah blah
 4. `VisualStudio/` - the root of the out-of-band solution
    1. `ChuckNorris/` - an example of writing a .NET Standard 2.0 DLL, that itself consumes .NET Standard 2.0 dependencies, acquired out of NuGet.
    2. `ChuckNorris.Console/` - a .NET Core 3.1 app that exercises the `ChuckNorris` DLL
    3. `ChuckNorris.Unity/` - a .NET Framework v4.7.2 DLL that collects together the dependencies of `ChuckNorris`, getting around the issue of .NET Standard 2.0 libraries deploying with a `dependencies.json` file.

### SDK-Style project files

Gone are the days of gigantic csproj files. The simplest csproj file now looks like this:
```xml
<Project Sdk="Microsoft.NET.Sdk">

  <PropertyGroup>
    <OutputType>Exe</OutputType>
    <TargetFramework>netcoreapp3.1</TargetFramework>
  </PropertyGroup>

</Project>
```

Notice the lack of XML doctype declaration, the lack of importing MSBuild subprojects, even the lack of naming the assembly and root namespace. While those may certainly be added for more control, most projects will use the defualts of the project type specified by "Microsof.NET.Sdk" and the assembly and namespace names taken from the project directory name. For more information on SDKs, read [How to: Use MSBuild project SDKs](https://docs.microsoft.com/en-us/visualstudio/msbuild/how-to-use-project-sdk) and [.NET Core project SDKs](https://docs.microsoft.com/en-us/dotnet/core/project-sdk/overview). Once you've gone through that, also check out [nirin/msbuild-sdks on Github](https://github.com/nirin/msbuild-sdks) for some more options on target frameworks. Finally, you may want to brush up on [.NET Standard](https://docs.microsoft.com/en-us/dotnet/standard/net-standard) and which .NET implementations implement which .NET standards. Of note, .NET Standard 2.0 is the last standard that .NET Framework will support, with support going back to .NET Framework v4.6.2.

Also gone is the `packages.json` file. You now import NuGet dependencies with a much simpler PackageReference element. 

#### An example library (ChuckNorris)
In this example repo, the `ChuckNorris` project imports two such dependencies:

```xml
<Project Sdk="Microsoft.NET.Sdk">

  <PropertyGroup>
    <TargetFramework>netstandard2.0</TargetFramework>
  </PropertyGroup>

  <!-- snipping extraneous details... -->

  <ItemGroup>
    <PackageReference Include="RestSharp" Version="106.10.1" />
    <PackageReference Include="RestSharp.Serializers.NewtonsoftJson" Version="106.10.1" />
  </ItemGroup>

</Project>
```

I selected `netstandard2.0` (.NET Standard 2.0) as the TargetFramework for this project specifically because A) .NET Framework v4.7.2 is .NET Standard 2.0 compliant, and B) the deployment of such dependencies and their transient dependencies is not as simple as it used to be.

This project RestSharp to make a query against a REST service to retrieve "Chuck Norris Facts/Jokes". It has one source file, `ChuckNorris.cs`
```csharp
using RestSharp;
using RestSharp.Serializers.NewtonsoftJson;

namespace ChuckNorris
{
    public struct Joke
    {
        public string icon_url;
        public string id;
        public string url;
        public string value;
    }

    public class ChuckNorris
    {
        private readonly RestClient client = new RestClient("https://api.chucknorris.io/jokes");

        public ChuckNorris()
        {
            client.UseNewtonsoftJson();
            client.ThrowOnAnyError = true;
        }

        public Joke GetRandom()
        {
            var request = new RestRequest("random", DataFormat.Json);
            var response = client.Get<Joke>(request);
            if(response.ResponseStatus == ResponseStatus.Completed)
            {
                return response.Data;
            }
            else
            {
                throw new System.Net.WebException($"ERR[{response.StatusCode}] = {response.ErrorMessage}");
            }
        }
    }
}
```

#### Exercising the example (ChuckNorris.Console)
The `ChuckNorris.Console` project is set to `netcoreapp3.1` (.NET Core 3.1) to show that this code is useable without Unity, in other runtimes that support .NET Standard 2.0
```xml
<Project Sdk="Microsoft.NET.Sdk">

  <PropertyGroup>
    <OutputType>Exe</OutputType>
    <TargetFramework>netcoreapp3.1</TargetFramework>
  </PropertyGroup>

  <ItemGroup>
    <ProjectReference Include="..\ChuckNorris\ChuckNorris.csproj" />
  </ItemGroup>

</Project>
```

It also has only one source file, `Program.cs`

```csharp
using static System.Console;

namespace ChuckNorris
{
    class Program
    {
        static void Main(string[] args)
        {
            var chuck = new ChuckNorris();
            var joke = chuck.GetRandom();
            WriteLine(joke.value);
        }
    }
}

```

#### Exporting to Unity (ChuckNorris.Unity)
And finally, the darkness that binds us all, `ChuckNorris.Unity`
```xml
<Project Sdk="Microsoft.NET.Sdk">

  <PropertyGroup>
    <TargetFramework>net472</TargetFramework>
  </PropertyGroup>

  <!-- snipping extraneous details... -->

  <ItemGroup>
    <ProjectReference Include="..\ChuckNorris\ChuckNorris.csproj" />
  </ItemGroup>

  <Target Name="PostBuild" AfterTargets="PostBuildEvent" Condition="'$(Configuration)|$(TargetFramework)'=='Release|net472'">
    <Exec ContinueOnError="true" Command="XCOPY &quot;$(TargetDir)*&quot; &quot;..\..\..\Assets\Plugins&quot; /exclude:excludeFromUnity.txt /C /I /F /Y" />
  </Target>

</Project>
```

It has several things of note:

 1. **The TargetFramework is `net472` (.NET Framework v4.7.2)**. Unity *officially* only supports .NET Framework v4.7.1, but I've never found it to be an issue. Change to `net471` if you're feeling conservative.
 2. **The PostBuild Event**. I have it set to only run for Release builds, but you can enable it for Debug builds too by just deleting the `Condition` attribute on the `Target` element, if you don't mind releasing or cleaning up your PDB files all the time.
    1. Executes an XCOPY command, where the parameters are:
       1. `$(TargetDir)*` - i.e. every file in the output directory.
       2. `..\..\Assets\Plugins` - a relative path to where we want to dump the files.
       3. `/exclude:excludeFromUnity.cs` - references a file that lists files that should not be copied. More on this in a minute.
       4. `/C` - Continues copying even if errors occur.
       5. `/I` - If destination does not exist and copying more than one file, assumes that destination must be a directory.
       6. `/F` - Displays full source and destination file names while copying.
       7. `/Y` - Suppresses prompting to confirm you want to overwrite an existing destination file.
 3. **There are no source files**. We don't need 'em, but see "Optional Item 1" below for an idea of what could be put in here.
 4. **ecludeFromUnity.txt contains ChuckNorris.Unity.dll**. Even though there are no source files in the project, a DLL will still be generated. We don't need it, so no need to include it in the export. Also, occasionally, there are certain DLLs that don't play nicely with Unity, that were only added by default by MSBuild. You can use this file to prevent them from being copied. It's also useful in "Optional Item 1" below.
 

### Optional Item 1: Put some meat on the Binding DLLs bones.

You could use the binding DLL as an opportunity to write Unity-specific code that plays nicely with big-boy-pants build and test systems.

If:

 1. You are 
    a. Not writing MonoBehaviours, or
    b. Are writing only new MonoBehaviours (i.e. not attempting to pull existing, in-use MonoBehaviours out of your Assets folder), and
 3. Don't need to include UnityEditor-specific code in your MonoBehaviours (i.e. no `#if UNITY_EDITOR/#endif`), and
 4. Don't need to perform any other Platform-dependant, compiler-flagged #if/#endif blocks (i.e. no `#if UNITY_STANDALONE || UNITY_ANDROID` type things), 
 
Then your project might be a candidate for writing a [Managed Plugin for Unity](https://docs.unity3d.com/Manual/UsingDLL.html).

Here are some steps that I undertake to make it a little easier to maintain over time:

 1. I define an environment variable `UNITY_ROOT`, which points to the current version of Unity that I'm using. In my current case, I have it set to `C:\Program Files\Unity\Hub\2019.3.3f1`
 2. I manually add an assembly reference to UnityEngine.dll to my project file:
 3. I add UnityEngine.dll to `excludeFromUnity.txt`

Here is that assembly reference:
```xml
  <ItemGroup Condition="Exists('$(UNITY_ROOT)')">
    <Reference Include="$(UNITY_ROOT)\Editor\Data\Managed\UnityEngine.dll" />
  </ItemGroup>
```
 

### Optional Item 2: Symlink your DLLs folder

For added control over projects, instead of copying directly to your Unity project's (or projects'!) Assets/Plugins directory, copy them to a directory out-of-band of the Unity project root and create a Directory Junction from your output directory to a new directory within Assets/Plugins. This is especially useful if you are building one project for multiple platforms. You might create a folder structure such as:

 1. `Shared/`
    1. `Assets/`
    2. `Editor Plugins/`
    3. `Editor Scripts/`
    4. `Gizmos/`
    5. `Player Plugins/`
    6. `Player Scripts/`
    7. `Project Settings/`
    7. `Resources/`
    8. `Streaming Assets/`
 2. `VisualStudio/`
 3. `[Your Project Name] - Standalone`
    1. `Assets/`
        1. `[Your Project Name]/`
           1. `Assets/` - a junction to `Shared/Assets/`
           2. `Scripts/` - a junction to `Shared/Player Scripts/`
        2. `Editor/`
           1. `[Your Project Name]/` - a junction to `Shared/Editor Scripts/`
        3. `Gizmos/`
           1. `[Your Project Name]/` - a junction to `Shared/Gizmos/`
        3. `Plugins/`
           1. `Editor/`
              1. `[Your Project Name]/` -  a junction to `Shared/Editor Plugins/`
           2. `[Your Project Name]/` - a junction to `Shared/Player Plugins/`
        3. `Resources/`
           1. `[Your Project Name]/` - a junction to `Shared/Resources/`
        3. `StreamingAssets/`
           1. `[Your Project Name]/` - a junction to `Shared/Streaming Assets/`
    2. `ProjectSettings/` - a junction to `Shared/Project Settings/`
 4. `[Your Project] - Android/`
    1. etc.
 5. `[Your Project] - iOS/`
    1. etc.
 6. etc.

In my project, I have a large chunk of library code named "Juniper", which has its own BAT file that creates a similar structure for itself (*NOTE: the use of the `@` prefix on commands is for selectively excluding commands from echoing to the terminal, rather than `ECHO OFF`ing everything and getting no output*). :

```batch
@rem USAGE: `call init-project.bat "<platform>" "<dest>"`
	@setlocal enableextensions
	@set juniper=%~dp0
	@set platform=%~1
	@set dest=%~2

	@echo %juniper%
	@echo Platform: %platform%
	@echo Destination: %dest%

	@echo "Initializing Juniper Assets at: %dest%"

	mkdir "%dest%"
	mkdir "%dest%\Assets"
	mkdir "%dest%\Assets\Editor"
	mkdir "%dest%\Assets\Juniper"
	mkdir "%dest%\Assets\Plugins"
	mkdir "%dest%\Assets\Plugins\Editor"
	mkdir "%dest%\Assets\StreamingAssets"

	@call :link /J "%dest%\AssetStore" "%juniper%..\lib\Unity Asset Store Packages"
	@call :link /J "%dest%\Assets\ShaderControl" "%juniper%Unity Asset Store Packages\ShaderControl"
	@call :link /j "%dest%\Assets\TextMesh Pro" "%juniper%Unity Asset Store Packages\TextMesh Pro"
	@call :link /J "%dest%\Assets\SpeechSDK" "%juniper%Unity Asset Store Packages\SpeechSDK"
	@if not "%platform:Oculus=%" == "%platform%" call :link /J "%dest%\Assets\Oculus" "%juniper%Unity Asset Store Packages\Oculus"

	@call :link /H "%dest%\Assets\csc.rsp" "%juniper%csc.rsp"
	@call :link /H "%dest%\Assets\link.xml" "%juniper%link.xml"
	@call :link /J "%dest%\Assets\Editor\Juniper" "%juniper%Unity Editor Scripts"
	@call :link /J "%dest%\Assets\Plugins\Juniper" "%juniper%Unity Plugins"
	@call :link /J "%dest%\Assets\Plugins\Editor\Juniper" "%juniper%Unity Editor Plugins"
	@call :link /J "%dest%\Assets\Juniper\Assets" "%juniper%Unity Assets"
	@call :link /J "%dest%\Assets\Juniper\Scripts" "%juniper%Unity Scripts"
@exit /b

@rem ==== SUBROUTINES ====

@rem USAGE: `call :link (/H|/J) "<dest>" "<src>"`
:link
	@setlocal enableextensions
	@set switch=%1
	@set dest=%~2
	@set src=%~3

	mklink %switch% "%dest%" "%src%"
	@if exist "%src%.meta" (mklink /h "%dest%.meta" "%src%.meta")
@exit /b
```