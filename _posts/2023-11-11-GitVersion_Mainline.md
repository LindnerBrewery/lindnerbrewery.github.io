---
layout: post
title: Versioning PowerShell modules with GitVersion in Mainline Mode
tags: powershell git gitversion
---

# Versioning PowerShell modules with GitVersion in Mainline Mode

In the world of software development, version management is a critical aspect of maintaining the integrity of a project and tracking its progress. For PowerShell module developers, the process of versioning can sometimes be a manual and error-prone task. In this blog post, we'll explore an alternative approach to versioning using GitVersion in mainline mode, which can help streamline the process and ensure consistent versioning across your projects.

### The Current Versioning Process

Before we look at GitVersion, let's take a look at more traditional approaches that many PowerShell developers use to version their modules. So far, you might be familiar with a few methods:

#### 1. Manual Versioning

A common but far from ideal method is to add version numbers manually to the `psd1` (module manifest) files, or by using `Update-ModuleManifest -ModuleVersion 0.2.0 -Path .\module.psd1`.  This approach is quick, easy and gives you full control over your versioning. The problem with this approach is that anyone working on the module will need to update the version or prerelease tag before committing to your repo.

#### 2. Incremental Patch Versioning depending on the latest version in your PsRepository

Another approach is to increment the patch version by retrieving the current version from your PsRepository, incrementing the patch version by 1, and updating the version number in the module's manifest. 
Here's a simple example:
```powershell
[Version]$currentVersion = (Find-Module -Name MyModule).version
$newVersion = [Version]::new($currentVersion.Major,$currentVersion.Minor, $currentVersion.Build, $currentVersion.Revision + 1)
Update-ModuleManifest -Path .\MyModule.psd1 -ModuleVersion $newVersion
```
While this method is fully automated, it doesn't allow any manual intervention and doesn't take advantage of the capabilities of version control systems such as Git.

#### 3. Hybrid Versioning

For many years, this was my main approach to versioning my modules. While manually updating major and minor, I would use the git revision to determine my patch number. This allows me to manually decide if it is a breaking change (major) or an added feature (minor).
However, this will result in a patch number that always counts up, and it can end up with very high patch numbers if you make a lot of changes to your module.
My build script would look something like this
```powershell
oldVersion = [version](Import-PowerShellDataFile -Path .\MyModule.psd1).moduleVersion
patch = git rev-list --count HEAD
$newVersion = [version]::new($oldVersion.Major, $oldVersion.Minor, $patch)
Update-ModuleManifest -ModuleVersion $newVersion -Path .\MyModuleg.psd1
```
The advantage of this approach is that it's simple, easy and you can be sure that each commit creates a unique version, while still being able to manually define major and minor.  

### Semantic versioning

Before we take a look at GitVersion and how it can help with versioning our modules, we need to talk briefly about semantic versioning. Semantic versioning, often abbreviated as SemVer, is a versioning scheme for software that helps describe the meaning of the underlying code changes and compatibility compared to other versions. The version number is usually expressed as three numbers separated by dots (e.g. 1.2.3), representing the major, minor and patch versions. In addition to these three components, SemVer allows for pre-release and build metadata.

Here's a breakdown of the three main components:
- MAJOR version increments when there are incompatible API changes. In PowerShell, this would be when a parameter is removed or the output of a cmdlet changes. 
- The MINOR version increments when functionality is added in a backward-compatible way. This would be the case if you add a new cmdlet to your module or if you add an additional parameter to a function without changing its original behaviour.
- The PATCH version increases when backward-compatible bug fixes are made. Patches don't change the way the feature works or add functionality.

For example, if the current version of a piece of software is 2.3.1, and a new backwards-compatible feature is added, the new version would be 2.4.0.

In addition to the standard MAJOR.MINOR.PATCH format, Semantic Versioning also supports pre-release versions. Pre-release versions have an additional label appended to the version number, such as 1.0.0-alpha or 1.0.0-beta.1. These versions are intended to allow users to test new features before the final release, and are not considered stable. For example, I always call my pre-releases *preview00X*.

For more information, see the official Semantic Versioning website: https://semver.org/


### Introducing GitVersion in Mainline Mode

While the hybrid approach mentioned above is an improvement over manual versioning, and is the way I version most of my modules, it doesn't support semantic versioning, and it doesn't support pre-releases. 
GitVersion is an open source tool that automates the versioning of your projects based on Git history. Out of the box, GitVersion uses a mode called "ContinuousDeployment". This mode does not suit the way I want to version my modules. Mainline mode is the mode that best suits the way I work. The main rule of mainline development is that **main is always in a state that could be deployed to production**. More information about mainline mode can be found [here] (https://GitVersion.net/docs/reference/modes/mainline).
When developing PowerShell modules, I create a new git branch for development, I never develop in main. Depending on the name of the branch, I want to increment the MINOR or PATCH number. I also want to build my module after each push as a pre-release.  

Let's get started with GitVersion!
First we need to install GitVersion. You can either download it from GitVersion.net or use our favourite package manager Chocolatey. If you use Chocolatey, make sure you install the GitVersion.portable package.
After installing GitVersion, we'll go into your module folder which is versioned using git.
Without a GitVersion.yml configuration file in your root folder, you will be in continuous delivery mode. To change this behaviour will create a new file called GitVersion.yml, put it in your root folder and add the following configuration to it:
```yaml
mode: Mainline
major-version-bump-message: '\s?(breaking|major|breaking\schange)'
minor-version-bump-message: '(adds?|features?|minor)\b'
patch-version-bump-message: '\s?(fix|patch)'
no-bump-message: '\+semver:\s?(none|skip)'
branches:
  main:
    tag: 
    increment: Patch
    regex: ^master$|^main$
  feature:
    tag: preview
    increment: Minor
    regex: ^features?[/-]
  hotfix:
    tag: preview
    increment: Patch
    regex: ^hotfix(es)?[/-]|^patch?[/-]
  support:
    tag: preview
    increment: Patch
```

What does the config tell us? The first line tells GitVersion, that we want to use mainline mode. The next lines tell how to bump the version, depending on the commit message. Everything under `branches:` tells GitVersion how to increment the version depending on the branch name. 
Let's see what happens. We've added the GitVersion.yml to our repository and committed the change. Now we can call `GitVersion`. The output should look something like this:
```json
{
  "Major": 0,
  "Minor": 1,
  "Patch": 2,
  "PreReleaseTag": "2",
  "PreReleaseTagWithDash": "",
  "PreReleaseLabel": "",
  "PreReleaseLabelWithDash": "",
  "PreReleaseNumber": null,
  "WeightedPreReleaseNumber": 60000,
  "BuildMetaData": null,
  "BuildMetaDataPadded": "",
  "FullBuildMetaData": "Branch.master.Sha.e4dca363071fea323c0342d4b1d61320cb5fd6c2",
  "MajorMinorPatch": "0.1.2",
  "SemVer": "0.1.2",
  "LegacySemVer": "0.1.2",
  "LegacySemVerPadded": "0.1.2",
  "AssemblySemVer": "0.1.2.0",
  "AssemblySemFileVer": "0.1.2.0",
  "FullSemVer": "0.1.2",
  "InformationalVersion": "0.1.2+Branch.master.Sha.e4dca363071fea323c0342d4b1d61320cb5fd6c2",
  "BranchName": "master",
  "EscapedBranchName": "master",
  "Sha": "e4dca363071fea323c0342d4b1d61320cb5fd6c2",
  "ShortSha": "e4dca36",
  "NuGetVersionV2": "0.1.2",
  "NuGetVersion": "0.1.2",
  "NuGetPreReleaseTagV2": "",
  "NuGetPreReleaseTag": "",
  "VersionSourceSha": "f892ad6191074e06758d1a7ce39e8d5ce8c25df2",
  "CommitsSinceVersionSource": 2,
  "CommitsSinceVersionSourcePadded": "0002",
  "UncommittedChanges": 0,
  "CommitDate": "2023-11-10"
}
```
With `GitVersion /showvariable NuGetVersionV2` you will only get the content saved in NuGetVersionV2 ("0.1.2"). Let's make a change to our repo, say by adding a new file, and commit that change. Our new version is now 0.1.3, because our config file tells GitVersion to increment the patch on every change to the main branch. 
Now let's say we want to add a new feature to our module. The first thing we'll do is create a new branch `git switch -c feature/myfunction`. In our config yaml, we've defined through a regex pattern that any branch starting with feature will be minorincremented and given the pre-release tag "preview".
```yml
 feature:
    tag: preview
    increment: Minor
    regex: ^features?[/-]
```
Running GitVersion will now return "0.2.0-preview0000". Each commit to this branch will cause the pre-release tag to be incremented by 1: "0.2.0-preview00001", "0.2.0-preview0002", and so on. When we are finished and happy with our changes, we can create a pull request and merge the branch back into main. Running GitVersion on main will now return "0.2.0", because we haven't defined a pre-release tag on the main/master branch.
In the next scenario, we want to fix a bug in our module. In this case we will create a new branch `git switch -c patch/somename`. Patch will tell git to increment the patch number by 1, resulting in "0.2.1-preview0000". While patching our module, we realise that the only way to fix the problem is to add a breaking change to our module, so we now want to increment the major value by one. We can do this by adding "breaking", "major" or "breaking change" to a commit message. Now we have 1.0.0-preview0001.
You can use git tags to define a starting point for GitVersion. For example, if you're starting a new module and don't want version 0.1.0 to be your starting point, you can create a git tag from which GitVersion will calculate the future version. If we want to start with version 1.0.0, we create a tag by typing `git tag 1.0.0`. The next version will be 1.0.1, and so on.  Don't forget to push your tags to your remote repo.

Now that you have your versions sorted out, you just need to update your module manifest when you build your module. 
```powershell
$version = GitVersion /showvariable MajorMinorPatch
Update-ModuleManifest -Path .\MyModule.psd1 -ModuleVersion $Version
$prerelease = GitVersion /showvariable NuGetPreReleaseTagV2
Update-ModuleManifest -Path .\MyModule.psd1 -Prerelease $prerelease
```

### Conlusion
There are many ways to version your PowerShell modules, but using GitVersion is the most elegant way for me. However, I only use GitVersion for larger modules. For small modules that I know I won't change often, or that only have a few functions, I tend to use hybrid versioning, using the git revision to define my patch number. GitVersion can be a bit intimidating at first, but once you've found the best way to use it for your needs, it becomes a valuable tool.     