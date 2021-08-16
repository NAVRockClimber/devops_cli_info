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

## Get the latest version info

The cli has a build in autoupdate feature for this tool. The cli queries an rest api and gets back the url of Artifacttool release.