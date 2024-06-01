---
layout: post
title:  "Tracking Field Usage Across Azure DevOps Project"
date:   2024-06-01 23:00:05 +0100
---

`azure-devops` `work-items` `powershell-script`

### In this post

- [Overview](#overview)
- [Script](#script)
- [Dive into the script](#dive-into-the-script)
- [Running the Script](#running-the-script)
- [References](#references)

## Overview

In response to a practical need to identify and quantify the usage of a specific field across projects in Azure DevOps, I developed this PowerShell script.

This script interacts with the Azure DevOps API, to retrieve a list of projects and analyze each for the presence of a designated field. It not only identifies which projects utilize this field but also counts the work items within those projects that include it. 

Additionally, the script compiles these findings into a markdown file. This file serves as a summary, presenting a clear breakdown of the number of work items per project that make use of the field, thus providing valuable insights into field utilization across your Azure DevOps environment.

## Script

{% highlight ruby %}

# Prompt user for inputs
$organization = Read-Host -Prompt "Enter the Azure DevOps organization URL in format https://dev.azure.com/ORGNAME"
$personalAccessToken = Read-Host -Prompt "Enter your personal access token"
$field = "FIELD_REFERENCE_NAME"
$batchSize = 10  # Define the number of projects to process in each batch

# Base64-encode the personal access token for authentication
$base64AuthInfo = [Convert]::ToBase64String([Text.Encoding]::ASCII.GetBytes(":$personalAccessToken"))

# Function to get the list of all projects
function Get-AllProjects {
    $url = "$organization/_apis/projects?api-version=7.0"
    return (Invoke-RestMethod -Uri $url -Method Get -Headers @{ Authorization = "Basic $base64AuthInfo" }).value
}

# Function to get the list of fields in a project
function Get-ProjectFields {
    param (
        [string]$projectId
    )
    $url = "$organization/$projectId/_apis/wit/fields?api-version=7.1-preview.3"
    return (Invoke-RestMethod -Uri $url -Method Get -Headers @{ Authorization = "Basic $base64AuthInfo" }).value
}

# Function to query work items for a project
function Query-WorkItems {
    param (
        [string]$projectName,
        [string]$field
    )
    $query = @"
    SELECT [System.Id], [System.WorkItemType]
    FROM workitems
    WHERE [System.TeamProject] = '$projectName'
    AND [System.WorkItemType] <> ''
    AND [$field] <> ''
"@
    $url = "$organization/$projectName/_apis/wit/wiql?api-version=6.0"
    $body = @{ query = $query } | ConvertTo-Json
    return (Invoke-RestMethod -Uri $url -Method Post -Headers @{ Authorization = "Basic $base64AuthInfo" } -Body $body -ContentType "application/json").workItems
}

# Main script to retrieve and filter projects by the specified field
$allProjects = Get-AllProjects
$projectsUsingField = @()

# Implement batching mechanism
for ($i = 0; $i -lt $allProjects.Count; $i += $batchSize) {
    $currentBatch = $allProjects[$i..($i + $batchSize - 1)]
    foreach ($project in $currentBatch) {
        $projectId = $project.id
        $projectName = $project.name
        $projectFields = Get-ProjectFields -projectId $projectId

        if ($projectFields | Where-Object { $_.referenceName -eq $field }) {
            $projectsUsingField += $projectName
        }
    }
}

# Prepare markdown content
$markdownContent = @"
|No|Project Name| Field Usage|
|--|------------|------------|
"@

# Output the list of projects using the specified field and query work items
if ($projectsUsingField.Count -eq 0) {
    Write-Output "No projects are using the field '$field'."
} else {
    $index = 1
    foreach ($projectName in $projectsUsingField) {
        $workItems = Query-WorkItems -projectName $projectName -field $field
        $workItemCount = if ($workItems) { $workItems.Count } else { 0 }
        $markdownContent += "`n|$index|$projectName|$workItemCount|"
        $index++
    }

    # Define the markdown file path relative to the script's directory
    $markdownFilePath = Join-Path -Path $PSScriptRoot -ChildPath "Field-Usage.md"

    # Write the markdown content to the file
    $markdownContent | Out-File -FilePath $markdownFilePath -Encoding utf8

    Write-Output "Markdown file created at: $markdownFilePath"
}

{% endhighlight %}


## Dive into the script

Let's break down each part of the script for a clearer understanding:

**1. Prompting User for Inputs**

The script begins by requesting the user to input the Azure DevOps organization URL and their personal access token. These are essential for authenticating API requests.

```
$organization = Read-Host -Prompt "Enter the Azure DevOps organization URL in format https://dev.azure.com/ORGNAME"
$personalAccessToken = Read-Host -Prompt "Enter your personal access token"

```

**2. Authentication Preparation**

The personal access token is Base64-encoded to prepare it for use in HTTP headers for API authentication.

```
$base64AuthInfo = [Convert]::ToBase64String([Text.Encoding]::ASCII.GetBytes(":$personalAccessToken"))

```

**3. Defining Functions**

- Get-AllProjects

This function constructs a URL to access the list of all projects within the specified organization and sends a GET request. The returned projects are stored in the $allProjects variable.

```
function Get-AllProjects {
    $url = "$organization/_apis/projects?api-version=7.0"
    return (Invoke-RestMethod -Uri $url -Method Get -Headers @{ Authorization = "Basic $base64AuthInfo" }).value
}

```
- Get-ProjectFields

This function retrieves all fields associated with a specific project, identified by $projectId.

```
function Get-ProjectFields {
    param ([string]$projectId)
    $url = "$organization/$projectId/_apis/wit/fields?api-version=7.1-preview.3"
    return (Invoke-RestMethod -Uri $url -Method Get -Headers @{ Authorization = "Basic $base64AuthInfo" }).value
}

```

- Query-WorkItems
  
This function executes a Work Item Query Language (WIQL) query to fetch work items for a given project that use a specified field.

```
function Query-WorkItems {
    param ([string]$projectName, [string]$field)
    $query = @"
    SELECT [System.Id], [System.WorkItemType]
    FROM workitems
    WHERE [System.TeamProject] = '$projectName'
    AND [System.WorkItemType] <> ''
    AND [$field] <> ''
"@
    $url = "$organization/$projectName/_apis/wit/wiql?api-version=6.0"
    $body = @{ query = $query } | ConvertTo-Json
    return (Invoke-RestMethod -Uri $url -Method Post -Headers @{ Authorization = "Basic $base64AuthInfo" } -Body $body -ContentType "application/json").workItems
}

```

**4. Main Script Execution**

The script retrieves all projects and filters them based on the presence of the specified field, 
`$field`, using a batching mechanism to process a defined number of projects at a time.

```
$allProjects = Get-AllProjects
$projectsUsingField = @()
...

```

**5. Output Preparation**

A markdown table is dynamically created to document each project and the count of relevant work items. This table is saved to a markdown file.

```
$markdownContent = ...
$markdownFilePath = Join-Path -Path $PSScriptRoot -ChildPath "Field-Usage.md"
$markdownContent | Out-File -FilePath $markdownFilePath -Encoding utf8
...

```

## Running the Script

**1. Set Up Your Environment**

- PowerShell: Ensure you have PowerShell installed on your computer. PowerShell 5.1 or higher is recommended for better compatibility with the script features.

**2. Prepare the Script**

- Place the Script: Save your script in a known directory.
- Customize the Script: Make sure the $field variable in your script matches the field you want to analyze.

**3. Run the Script**

- Open PowerShell: Start PowerShell from your Start menu or taskbar.
  
- Navigate to Your Script: `cd path\to\your\script`
  
- Execute the Script: `.\YourScriptName.ps1`

- Input Requirements: The script will prompt you for the Azure DevOps organization URL and your personal access token. Make sure to enter these accurately to authenticate your API requests.

  - Organization URL: Enter the URL in the format https://dev.azure.com/ORGNAME.
  - Personal Access Token (PAT): Input your generated PAT from Azure DevOps. Ensure that the token has the necessary scopes for reading projects and work items.

**4. View Results**

- Review the Markdown File: After the script finishes executing, it outputs the path to the markdown file it created (Field-Usage.md). You can open this file in any markdown viewer or text editor to see the report.

This is an example of how the generated markdown file looks like:

![Generated Markdown File](/assets/images/field-usage/output-markdown-file.png)


## References

[Projects - List](https://learn.microsoft.com/en-us/rest/api/azure/devops/core/projects/list?view=azure-devops-rest-7.1&tabs=HTTP)

[Fields - List](https://learn.microsoft.com/en-us/rest/api/azure/devops/wit/fields/list?view=azure-devops-rest-7.1&tabs=HTTP)

[Wiql - Query By Wiql](https://learn.microsoft.com/en-us/rest/api/azure/devops/wit/wiql/query-by-wiql?view=azure-devops-rest-7.0&tabs=HTTP)

