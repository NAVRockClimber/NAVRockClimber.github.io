---
layout: post
title:  "Downloading artifacts from DevOps artifact feeds (Part 1)"
date:   2021-09-10
desc: "Downloading universal artifacts from DevOps artifact feeds"
keywords: "Azure Artifacts,Artifacts,DevOps API,Artifacttool"
categories: [HTML]
tags: [Azure Artifacts,DevOps API, Devops Artifacts,Artifacttool]
icon: fa-cube
---

Have you ever been in the situation that you needed to download an artifact from an Azure DevOps artifact feed and recognized there is not really a way next to using the azure cli? But you can't install or simply don't like to install the cli? I was and this annoyed me. So I took a closer look on how the cli downloads universal artifacts and what the pipeline task does. This is part 1 of 2 blog posts showing the first solution on how else we can download an universal artifact from Azure DevOps.

## Analyzing the cli and upload task

If you take a closer look into the output of cli or the download step you will see some lines stating that something called artifacttool is downloaded.

```
SYSTEMVSSCONNECTION exists true
Downloading: https://08wvsblobprodsu6weus73.vsblob.vsassets.io/artifacttool/artifacttool-win10-x64-Release_0.2.198.zip?<redacted token>
Caching tool: ArtifactTool 0.2.198 x64
SYSTEMVSSCONNECTION exists true
```

Seeing this made curious and I started digging a little deeper. I googled around for this tool hoping Microsoft published anything on this or maybe even its sources on GitHub. Well, apparently they did not publish the sources or anything useful on the artifacttool so I took the next best I got the azure cli and I started to look for the sources of the azure cli devops extension.

## Analyzing the cli source code

If you ever tried to do something in Azure DevOps using the azure cli you know that you have to install an extension for the cli first. Luckily Microsoft published the sources and we can take a look what Microsoft is doing to retrieve an artifact or to be precise to download and invoke the artifacttool.
Taking a look into the sources on [GitHub](https://github.com/Azure/azure-devops-cli-extension) I quickly discovered a method with the promising name [download_universal](https://github.com/Azure/azure-devops-cli-extension/blob/97dc4fe0655da7b1dc0e975bd65e74f7d72948b7/azure-devops/azext_devops/dev/common/artifacttool.py#L32).
Already in the first line an argument array is constructed containing a few familiar parameters I already knew from using the cli to download a universal artifact. Digging through the code I found Microsoft implemented some kind of auto updater where they query an API for the most current version of the artifacttool and then invoke the artifacttool passing an environment variable into the tool for authenticating.

## Getting the version information

As I already stated the cli has a build in auto update feature for this tool. The cli queries an rest api and gets back the version number and url of the latest artifacttool release.

The request needs a authentication header and has the following parameters:

- osName
- arch
- distroName
- distroVersion
- version

Powershell Example:

``` Powershell
$encoded = [System.Convert]::ToBase64String([System.Text.Encoding]::ASCII.GetBytes("azure:<PAT>"))
$basicAuthValue = "Basic $encoded"
$Headers = @{Authorization = $basicAuthValue}

$result = Invoke-WebRequest -Headers $headers -Method Get -Uri "https://vsblob.dev.azure.com/<Your Organization>/_apis/clienttools/ArtifactTool/release?osName=Windows&arch=x86_64"
```

Response:

``` json
{
    "name":"ArtifactTool",
    "rid":"win10-x64",
    "version":"0.2.195",
    "uri":"https://08wvsblobprodsu6weus73.vsblob.vsassets.io/artifacttool/artifacttool-win10-x64-Release_0.2.195.zip?..."
}
```

## Invoking the Artifacttool

With these information we can now download and invoke the artifacttool. The artifacttool can be used similar to azure cli with slightly different syntax. For authentication we need to set an environment variable with a PAT as value and pass the name of the environment variable to the artifacttool.

Example

``` PowerShell
$env:AZURE_DEVOPS_EXT_PAT = '<PAT>'

.\artifacttool universal download --service <URL to your devops organization> --patvar AZURE_DEVOPS_EXT_PAT --feed <feed name> --package-name <package name> --package-version '*' --path D:\Temp\Artifact\
```

## Conclusion

Well, we can now start downloading (and publishing it is the same tool) universal artifacts without having to install something first. Nevertheless, I still wasn't satisfied with the results of my findings yet. I simply don't like the idea of having to download a tool first for downloading an artifact. Therefore I went further down the rabbit hole and will describe in a second blog post how we can retrieve and artifact using API calls.
