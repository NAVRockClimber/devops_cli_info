# Azure DevOps Cli Info

Automating artifact uploads and downloads in non pipeline environment is quite tricky e.g. from c# or typescript due to Microsoft does not provide API calls for this purpose. Here my findings from investigating the sources of the azure devops extension for the azure cli.

## Authentication without login

Usually you need to run az login for authenticating. In automation scenarios this is anoying if you just want to pass in a PAT for authentication.
In the [const.py](https://github.com/Azure/azure-devops-cli-extension/blob/master/azure-devops/azext_devops/dev/common/const.py) some environment variables are defined. Amongst others here is a environment variable declared for using a PAT to authenticate called *AZURE_DEVOPS_EXT_PAT*.

Bash Example:
``` BASH
export AZURE_DEVOPS_EXT_PAT=<Your PAT>
az artifacts universal download --organization <URL to your devops organisation> --feed <feed name> --name <package name> --version '*' --path /tmp/
```

PowerShell Example:
``` PowerShell
$env:AZURE_DEVOPS_EXT_PAT = '<Your PAT>'
az artifacts universal download --organization <URL to your devops organisation> --feed <feed name> --name <package name> --version '*' --path D:\Temp\
```

## The Artifacttool

In large areas the azure devops extension just queries the azure devops rest api. But for downloading and uploading artifact it uses a tool called the Artifacttool. Well that not too surprising as I already saw pipelines downloading this tool if you use one of the upload or download steps.

### Get the latest version info

The cli has a build in autoupdate feature for this tool. The cli queries an rest api and gets back the version number and url of the last artifacttool release.

The request has the following parameters:
- osName
- arch
- distroName
- distroVersion
- version

Powershell Example:
``` Powershell
$result = Invoke-WebRequest -Headers $headers -Method Get -Uri "https://vsblob.dev.azure.com/<Your Organisation>/_apis/clienttools/ArtifactTool/release?osName=Windows&arch=x86_64"
```

Reponse:
``` json
{
    "name":"ArtifactTool",
    "rid":"win10-x64",
    "version":"0.2.195",
    "uri":"https://08wvsblobprodsu6weus73.vsblob.vsassets.io/artifacttool/artifacttool-win10-x64-Release_0.2.195.zip?..."
}
```

Powershell Example2:
``` Powershell
$result = Invoke-WebRequest -Headers $headers -Method Get -Uri "https://vsblob.dev.azure.com/<Your Organisation>/_apis/clienttools/ArtifactTool/release?osName=Linux&arch=x86_64&distroName=Debian&distroVersion=10
```

Response:
``` json
{
    "name":"ArtifactTool",
    "rid":"linux-x64",
    "version":"0.2.195",
    "uri":"https://08wvsblobprodsu6weus73.vsblob.vsassets.io/artifacttool/artifacttool-linux-x64-Release_0.2.195.zip?..."}
```

### Using the Artifacttool

The artifact can be used similar to azure cli with slightly different syntax. Here we can pass in a PAT as well with the difference we need to tell the tool the name of the environment variable containing the PAT.

Example
``` PowerShell
.\artifacttool universal download --service <URL to your devops organisation> --patvar AZURE_DEVOPS_EXT_PAT --feed <feed name> --package-name <package name> --package-version '*' --path D:\Temp\Artifact\
```

# Artifacttool

## Get Virtual Directory

```
GET https://dev.azure.com/{Organization}/_apis/connectionData?connectOptions=1&lastChangeId=183750697&lastChangeId64=183750697
```

Response:
```JSON
    ...
    "locationServiceData": {
        "serviceOwner": "00000000-0000-0000-0000-000000000000", // removed
        "accessMappings": [
            {
                "displayName": "Host Guid Access Mapping",
                "moniker": "HostGuidAccessMapping",
                "accessPoint": "https://tfsprodweu3.visualstudio.com/",
                "serviceOwner": "00000000-0000-0000-0000-000000000000",
                "virtualDirectory": "<GUID / Virtual Directory>"
            },
            {}
        ]}
    ...
```

## Get Manifest ID

```
GET https://pkgsprodsu3weu.pkgs.visualstudio.com/{{VirtualDirectory}}/_packaging/{{feed}}/upack/packages/{{package}}/versions/{{version}}?intent=Download
```
might be able to be substituated by: https://pkgs.dev.azure.com/{{Organization}}/_packaging/{{feed}}/upack/packages/{{package}}/versions
https://github.com/Azure/azure-devops-cli-extension/issues/567

Response:
```JSON
{
    "version": "<Version>",
    "superRootId": "<RootID>", 
    "manifestId": "<manifestID>"
}
```

## Get Manifest URl

```
POST https://vsblobprodsu6weu.vsblob.visualstudio.com/{{VirtualDirectory}}/_apis/dedup/urls?allowEdge=true 

["<manifestID>"]
```
can be supplemented by: "https://{{Organization}}.vsblob.visualstudio.com/{{VirtualDirectory}}/_apis/dedup/urls?allowEdge=true&api-version=1.0" -Body '["<manifestID>"]'

Docu: https://docs.microsoft.com/de-de/azure/devops/pipelines/agents/v2-windows?view=azure-devops#for-organizations-using-the-devazurecom-domain

Response:
```JSON
{
    "<manifestID>": "<Storage Link Valid 1 Day>"
}
```

## Get Package URl

```
POST https://vsblobprodsu6weu.vsblob.visualstudio.com/{{VirtualDirectory}}/_apis/dedup/urls?allowEdge=true 

["<BlobId>"]
```
can be supplemented by: "https://{{Organization}}.vsblob.visualstudio.com/{{VirtualDirectory}}/_apis/dedup/urls?allowEdge=true&api-version=1.0" -Body '["<BlobId>"]'

Response:
```JSON
{
    "<BlobId>": "<Storage Link Valid 1 Day>"
}
```