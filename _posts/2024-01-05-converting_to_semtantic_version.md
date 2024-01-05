---
layout: post
title: Converting conventional version to semantic version with PowerShell
tags: powershell semver 
---
Lately I've been needing to validate and convert versions to semantic versions. For instance, a version defined as "23.01" needed to be converted to "23.1.0". This led me to create a PowerShell function, `ConvertTo-SemVersion`, made to transform a conventional version numbers into the semantic versioning format.

#### What is Semantic Versioning?

Semantic Versioning (SemVer) is a versioning scheme for software that aims to convey meaning about the underlying changes. A standard SemVer string is in the format of `MAJOR.MINOR.PATCH`, where:

- **MAJOR** version increments mean incompatible API changes,
- **MINOR** version increments add functionality in a backwards compatible manner, and
- **PATCH** version increments are for backwards compatible bug fixes.

Semantic versioning also supports prereleases and release metadata to provide additional context and information about the build and release stage of the software. Prerelease versions can be indicated with a hyphen and additional labels after the PATCH version, such as "1.0.0-alpha" or "2.1.0-beta.1", to signify that the version is unstable and not ready for production. Similarly, build metadata can be added using a plus sign and additional identifiers, like "1.0.0+20130313" or "2.1.0-beta.1+202411", providing extra details like build or commit references. Both of these extensions are optional.

More Information about Semantic Versioning can be found on [semver.org](https://semver.org/)
#### The ConvertTo-SemVersion Function

The `ConvertTo-SemVersion` function is a simple tool that accepts a version string and converts it into a semantic versioning string. It handles versions with different degrees of complexity, from simple major-only versions like "1" to more complex forms including pre-release or build metadata.

Here are a few examples of how versions are converted:

- "1" becomes "1.0.0"
- "23.01" becomes "23.1.0"
- "1.1.1.0" becomes "1.1.1"
- "1-Alpha" becomes "1.0.0-Alpha"

The function uses a regular expression to dissect the version input and then reformats it according to semantic versioning rules. It ensures that the output has at least a major, minor, and patch number. It also handles revision numbers (the fourth number in some versioning schemes) by dropping it if it's "0". Admittedly, I still don't fully understand the regex details, but I'm proud to have gotten it to work and to see it effectively managing version conversions.

Here are a few examples:
```powershell
ConvertTo-SemVersion -Version 23.01
23.1.0​

ConvertTo-SemVersion -Version 2.1.0.0
2.1.0

ConvertTo-SemVersion -Version 1-Alpha
1.0.0-Alpha
```

And heres the complete function:
```powershell
function ConvertTo-SemVersion {
    <#
.SYNOPSIS
    Converts a Version to to semantic versioning v2
.DESCRIPTION
    Converts a Version to to semantic versioning v2. The function accepts a version as a string and will try to convert it.
    It will return a string representing at least major.minor.patch. If revision is 0 it will be dropped, unless is has a prerelease or buildmetadata.
    1           -> 1.0.0
    1.1         -> 1.1.0
    1.1.1.0     -> 1.1.1
    1.1.1.1     -> 1.1.1.1
    1.01        -> 1.1.0
    1-Alpha     -> 1.0.0-Alpha
    1.1.0.0-RC  -> 1.1.0-RC
    1.1.0.0-RC+2019 -> 1.1.0-RC+2019

.EXAMPLE
    PS C:\> ConvertTo-SemVersion -Version 1.0
    1.0.0
.INPUTS
    [String]
.OUTPUTS
    [String]
#>
    [CmdletBinding()]
    param (
        [Parameter(Mandatory = $true,
            ValueFromPipeline = $true,
            Position = 0,
            HelpMessage = "Specifies the version to convert.")]
        [String]$Version
    )

    begin {}

    process {
        $normVersion = $null
        $regexPattern = '^(?<major>\d*)?(?:\.(?<minor>\d*))?(?:\.(?<patch>\d*))?(?:\.(?<build>\d*))?(?:-(?<prerelease>[0-9a-zA-Z-]+(?:\.[0-9a-zA-Z-]+)*))?(?:\+(?<buildmetadata>[0-9a-zA-Z-]+(?:\.[0-9a-zA-Z-]+)*))?$'
        $result = [Regex]::Match($Version, $regexPattern)
        
        # throw error if version not valid
        if (! $result.Success) {
            Throw "$Version is not a valid semantic version"
        }

        # convert the major minor and patch
        Write-Verbose "Converting version"
        $normVersion = [version]::new($result.groups[1].Value, $result.groups[2].Value, $result.groups[3].Value)
        $rev = $result.groups[4].Value
        if ($rev -gt 0) {
            Write-Verbose "Revision ($rev) is not 0 and will be preserved"
            $normVersion = [System.Version]::Parse("$($normVersion).$($result.groups[4].Value)")
        } elseif ($rev -eq 0) {
            Write-Verbose "Revision is 0 and will be removed from the converted version"
        }
        $versionString = $normVersion.ToString()

        # if version is a prerelease or has build metadata then return converted version with metadata

        if ($result.Groups[5].Success -or $result.Groups[6].Success) {
            Write-Verbose "Version has prerelease or buildmetadata and will be returned as is"
            if ($result.groups[5].Value) {
                Write-Verbose "Version has prerelease tag"
                $versionString += "-" + $result.groups[5].Value
            }
            if ($result.groups[6].Value) {
                Write-Verbose "Version has buildmetadata tag"
                $versionString += "+" + $result.groups[6].Value
            }

        }

        return $versionString
    }

    end {}
}  
```

#### Conclusion

Adopting standard and comparable versions through Semantic Versioning has significantly helped me with my work, particularly with managing NuGet repositories. Being able to quickly compare versions with the versions in my repositories.
Being able to quickly and reliably compare versions helps not just my workflow, but also enhances collaboration with my team. 
You can find the code and Pester tests on my [gist](https://gist.github.com/LindnerBrewery/5a4a7acdde660640cc86c6bf8c14ef0b)
