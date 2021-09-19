---
layout: post
title:  "Downloading artifacts from DevOps artifact feeds (Part 2)"
date:   2021-09-17
desc: "Downloading universal artifacts from DevOps artifact feeds via the API"
keywords: "Azure Artifacts,Artifacts,DevOps API"
categories: [HTML]
tags: [Azure Artifacts,DevOps API, Devops Artifacts]
icon: fa-cube
---

In my [previous post]({% post_url 2021-09-10-downloading-devops-artifact %}) I described how we can utilize the internal artifacttool from Microsoft that is used in the Azure CLI and in the pipeline task to download a universal Artifact from a feed hosted in Azure DevOps. It is a good solution if you are sitting on a blank vm or docker container but if you want to download an artifact from an application you are currently writing you would need to open a new process and invoke the tool. I am not a fan of opening new processes and invoking external applications (at least under windows) due to it is in my experience always a point of failure. Meaning if it breaks it breaks there.

So I fired up Fiddler and send a request for downloading an artifact with the artifacttool.

## Fiddling around

For those of you not familiar with Fiddler it is a tool for debugging web development, sending request and logging send requests from you system.

After downloading an artifact using the artifacttool Fiddler logged request looking like this:

<img src="{{ site.img_path }}/artifact_api/Fiddler_Web_Debugger.png" width="40%">

At the end we see a quite big request or better response. Which is a good candidate for being our artifact (I downloaded a Business Central artifact). If we look at the payload of the response we see that we received some binary data with a file header starting "NAVX" which is the typical header for a business central app.

**Request:**

<img src="{{ site.img_path }}/artifact_api/Fiddler_BlobRequest.png" width="50%">

**Response:**

<img src="{{ site.img_path }}/artifact_api/Fiddler_Payload.png" width="50%">

Looking closer at these request we see the URL in the request and the headers in the response reveal a that we are looking at a download from an ordinary blob storage.
So I guessed the request with the cryptic server names like "vsblobprod..." or "pkgsprodsu3weu" are used to retrieve the Blob-Storage URL.

And, I was right looking at the request before I found a JSON response containing the BLOB URI.

<img src="{{ site.img_path }}/artifact_api/Fiddler_GetBlobRespsonse.png" width="50%">

## Analyzing the requests

If you analyze the requests further you find a feq parameters that are collected on the way to get the blob uri. A parameter "virtualDirectory", "ManifestId", "ManifestUri" and a "BlobId". After mostly tracing back where which parameter is retrieved I turned to the URLs. Investigating them further it looks like some aliases for directing you to the next local data center. Probably used by some kind of load balancer and/or geo redundancy service.
Cross checking them with the IPs from the servers in this [list from Microsoft's Documentation](https://docs.microsoft.com/en-us/azure/devops/pipelines/agents/v2-windows?view=azure-devops#for-organizations-using-the-devazurecom-domain) we can substitute them with prettier names.

So I started putting everything together in VS Code [Rest client file](https://github.com/NAVRockClimber/devops_cli_info/blob/master/artifact.http).

### 1. Virtual Directory

One of the first parameter you will need is a virtualDirectory which seems just to be a guid. At least in my investigation I found nothing we can substitute this id with.

**Request Structure:**

```
@Organization=<Your DevOps Organisation>
@PAT=<Your PAT>

GET https://dev.azure.com/{{Organization}}/_apis/connectionData?connectOptions=1
Authorization: Basic PAT:{{PAT}}
```

You retrieve a complex JSON where you need to find the property virtualDirectory in locationServiceData.accessMappings and copy the value.

### 2. My Manifest

In the next step we need the retrieve the ID of the manifest belonging to our package.

**Request Structure:**

```
@feed=<Your Devops Feed Name>
@package=<Your Package Name>
@version=<Desired Version>

GET https://pkgs.dev.azure.com/{{Organization}}/_packaging/{{feed}}/upack/packages/{{package}}/versions/{{version}}?intend=download
Authorization: Basic PAT:{{PAT}}
```

**Response:**

```JSON
{
    "version": "<Version>",
    "superRootId": "<RootID>", 
    "manifestId": "<manifestID>"
}
```

From this quite simple response we just need the manifest id.
With this we can retrieve the manifest which we need to get the blob id.

### 3. Give me the manifest

This slightly more difficult request will return the URL of our manifest.

**Request Structure:**

```
@VirtualDirectory=<virtualDirectory from 1.>
@ManifestId=<manifestID from 2.>

POST https://{{Organization}}.vsblob.visualstudio.com/{{VirtualDirectory}}/_apis/dedup/urls?allowEdge=true"
Content-Type: application/json; api-version=1.0
Accept: application/json; api-version=1.0
Authorization: Basic PAT:{{PAT}}

["{{ManifestId}}"]
```

**Response:**

```JSON
{
    "<manifestID>": "<ManifestUri>"
}
```

With this link we can download the manifest with a simple http get. It is probably noteworthy that this link is only valid for 24 hours.

```
@ManifestUri=

GET {{ManifestUri}}
```

**Response:**

```JSON
{
    "manifestFormat": "1.1.0",
    "items": [
        {
            "path": "/app/myTestApp_2.2.54246.0.app",
            "blob": {
                "id": "<Some Guid>",
                "size": 88622
            }
        }
    ],
    "manifestReferences": []
}
```

### 4. Give me that blob

We are nearly there. With the blob id from the manifest before we can query the blob URL like we did with the manifest id.

**Request Structure:**

@BlobId=

POST https://{{Organization}}.vsblob.visualstudio.com/{{VirtualDirectory}}/_apis/dedup/urls?allowEdge=true"
Content-Type: application/json; api-version=1.0
Accept: application/json; api-version=1.0
Authorization: Basic PAT:{{PAT}}

["{{BlobId}}"]

**Resonse:**

```JSON
{
    "<BlobId>": "<BlobUri>"
}
```

With the URL in the response we can now finally simply download the artifact we published at some time in Azure DevOps.

## Conclusion

Downloading an artifact via this undocumented or internal API is somewhat cumbersome but in some cases you prefer an api over a tool. If you choose the Azure cli, the artifacttool or the API is your decision. In the most cases I will probably use the azure cli and just now and than depending on the circumstances the artifacttool or the API. 