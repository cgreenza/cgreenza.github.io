---
layout: post
title:  "Backup Azure DevOps Git Repos to Blob Storage"
date:   2020-04-19 00:00:00 +0200
tags: []
---

Source code is the central asset of a software business, keeping it safe is critically important. [Regular backups](https://docs.microsoft.com/en-us/azure/devops/organizations/security/data-protection?view=azure-devops) are included as part of Azure DevOps; there are however some disaster scenarios that may warrant an additional level of protection:

1.	Microsoft keeps backups for 28 days which gives you limited time to detect a problem and request a restore. In some cases, you may only find out much later e.g. accidental or malicious deletion of a repository, branch or history.
2.	Your Azure DevOps instance may be hosted in a different country and may not be accessible from your home country if there is a global war.

To address above one can regularly (e.g. monthly) clone the Git repos and store them to an Azure blob storage account using the following:

1.	Azure Pipelines to run a PowerShell script on a schedule to perform the backup. The Azure Pipeline must have the project level pipeline setting `Limit job authorization scope to current project` off so that the pipeline’s `System.AccessToken` has access to all repositories in the DevOps account.
2.	Azure DevOps Services [REST API](https://docs.microsoft.com/en-us/rest/api/azure/devops/?view=azure-devops-rest-5.0) to get the list of repositories.
3.	`git clone --bare` and `git lfs fetch --all` to download the repo content
4.	PowerShell’s `Compress-Archive` to zip up the repo into a single file;
5.	`AzCopy` to upload the backup to blob storage. Here we use a [SAS token](https://docs.microsoft.com/en-us/azure/storage/common/storage-sas-overview) with only [create and write](https://docs.microsoft.com/en-us/rest/api/storageservices/create-service-sas#permissions-for-a-container) access on the storage container for authentication. The SAS URL is stored as a secret on the pipeline. Alternative: Azure File Copy task.
6.	Setup a [Blob storage lifecycle management policy ](https://docs.microsoft.com/en-us/azure/storage/blobs/storage-lifecycle-management-concepts) to manage costs by automatically archiving and/or deleting old backups.

YAML pipeline definition for above:
```yaml
schedules:
- cron: "0 0 2 * *"
  displayName: Monthly backups on 2nd at 00:00
  branches:
    include:
    - master
  always: true

pool:
  vmImage: 'vs2017-win2016' # windows-latest fails due to git checkout writing to the warning/error stream

steps:
- powershell: |
    # download & unzip azcopy. Download link from: (curl https://aka.ms/downloadazcopy-v10-windows -MaximumRedirection 0 -ErrorAction silentlycontinue).RawContent
    Start-BitsTransfer -Source "https://azcopyvnext.azureedge.net/release20200410/azcopy_windows_amd64_10.4.0.zip" -Destination $env:Build_SourcesDirectory\AzCopy.zip
    Expand-Archive $env:Build_SourcesDirectory\AzCopy.zip $env:Build_SourcesDirectory\azcopy\ -Force
    Get-ChildItem "$env:Build_SourcesDirectory\azcopy\*\*" | Move-Item -Destination "$env:Build_SourcesDirectory\azcopy\" -Force

    $repos = Invoke-RestMethod -Uri "$($env:System_TeamFoundationCollectionUri)_apis/git/repositories?api-version=5.0" -Headers @{
      Authorization = "Bearer $env:SYSTEM_ACCESSTOKEN"
    }
    
    if ($repos.value -eq $null)
    {
      Write-Error 'Error getting repo list' 
      Write-Error $repos
      throw
    }

    $date = Get-Date -Format "yyyy-MM-dd"
    new-item $env:Build_SourcesDirectory\clone -itemtype directory

    foreach ($repo in $repos.value) {
      Write-Host "**** REPO ****"
      Write-Host $repo.remoteUrl
      
      $projectName = $repo.project.name -replace '[^a-zA-Z0-9]', '-'
      $repoName = $repo.name -replace '[^a-zA-Z0-9]', '-'
      
      git clone -c http.extraheader="AUTHORIZATION: bearer $env:SYSTEM_ACCESSTOKEN" --bare --verbose --progress $repo.remoteUrl $env:Build_SourcesDirectory\clone
      if ($LASTEXITCODE) {
        throw "git clone error $LASTEXITCODE"
      }

      cd $env:Build_SourcesDirectory\clone
      git lfs fetch --all
      if ($LASTEXITCODE) {
        throw "git lfs error $LASTEXITCODE"
      }

      $zipFile = "$($date)_$($projectName)_$($repoName).zip"
      Compress-Archive -Path $env:Build_SourcesDirectory\clone\* -DestinationPath $env:Build_ArtifactStagingDirectory\$zipFile

      .$env:Build_SourcesDirectory\azcopy\azcopy.exe cp $env:Build_ArtifactStagingDirectory\$zipFile $env:SAS_URL_MAPPED --check-length=false
      if ($LASTEXITCODE) {
        throw "azcopy error $LASTEXITCODE"
      }

      Remove-Item -Path $env:Build_SourcesDirectory\clone\* -Recurse -Force
      Remove-Item -Path $env:Build_ArtifactStagingDirectory\* -Recurse -Force
    }
  displayName: 'Clone, zip and store repos'
  failOnStderr: false
  env:
    SYSTEM_ACCESSTOKEN: $(System.AccessToken)
    SAS_URL_MAPPED: $(SasUrl)

```

